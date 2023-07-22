# Execução e fluxo de código - SeaBIOS
Esta página fornece uma descrição de alto nível de algumas das principais fases do código pelas quais o SeaBIOS transita e informações gerais sobre o fluxo geral do código.

O código `SeaBIOS` passa por algumas fases de código distintas durante seu ciclo de vida de execução. Entender essas fases do código pode ajudar na leitura e aprimoramento do código.

## A fase de POST

A fase _Power On Self Test_ (POST) é a fase de inicialização da BIOS. Esta fase é percorrida quando o SeaBIOS inicia a execução pela primeira vez. O objetivo da fase é:

* -**inicializar o estado interno**;
* -**inicializar as interfaces externas**;
* -**detectar e configurar o hardware**; e, 
* -**iniciar a fase de inicialização**.

Em emuladores, esta fase começa quando a CPU inicia a execução no **modo 16 bits** em: 

* **0xFFFF0000:FFF0.**

Os emuladores mapeiam o binário SeaBIOS para este endereço e o SeaBIOS organiza para que 

* **romlayout.S:reset\_vector()**

esteja presente lá. Este código chama

* **romlayout.S:entry\_post()**

que então chama 

* **post.c:handle\_post()**

no **modo de 32 bits**.

No **coreboot**, a compilação faz com que 

* **romlayout.S:entry\_elf()**
  
seja chamado no **modo de 32 bits**. Isso chama 

* **post.c:handle\_post().**

No CSM, a construção organiza para que 

* **romlayout.S:entry\_csm()**

seja chamado (no **modo de 16 bits** ). Isso então chama

* **csm.c:handle\_csm()**

no **modo de 32 bits**. Ao contrário dos emuladores e do coreboot, a fase SeaBIOS CSM POST é orquestrada com UEFI e há várias chamadas entre SeaBIOS e UEFI via 

*  **handle\_csm()**

durante todo o processo POST.

### Sub-Fases da POST
A própria fase POST possui várias subfases.

*   A subfase **preinit**: código executado antes [code relocation](https://seabios.org/Linking_overview#Code_relocation "Linking overview").
*   A subfase **init**: código para inicializar variáveis internas e interfaces.
*   A subfase **configuração**: código para configurar hardware e drivers.
*   A subfase **prepboot**: código para finalizar as interfaces e preparar para a fase de inicialização.

Na conclusão da fase POST, o SeaBIOS invoca uma interrupção de software **int 0x19** no **modo de 16 bits** que inicia a fase de inicialização.

# A fase BOOT

O objetivo da **fase de inicialização** (**Fase BOOT**)  é carregar a primeira parte do carregador de inicialização do sistema operacional na memória e iniciar a execução desse carregador de inicialização. Esta fase começa quando uma interrupção de software (**int 0x19** ou **int 0x18**) é invocada. 
O fluxo de código **começa no modo de 16 bits** em 
* **romlayout.S:entry\_19()**
 ou
* **romlayout.S:entry\_18()**
que então faz a transição para o **modo de 32 bits** e chama
* **boot.c:handle\_19()**
ou
* **boot.c:handle\_18()**.

The boot phase is technically also part of the "runtime" phase of SeaBIOS. It is typically invoked immediately after the POST phase, but it can also be invoked by an operating system or be invoked multiple times in an attempt to find a valid boot media. Although the boot phase C code runs in 32bit mode it does not have write access to the 0x0f0000-0x100000 memory region and can not call the various malloc\_X() calls. See [Memory Model](https://seabios.org/Memory_Model "Memory Model") for more information.

Main runtime phase
------------------

The main runtime phase occurs after the boot phase starts the operating system. Once in this phase, the SeaBIOS code may be invoked by the operating system using various 16bit and 32bit calls. The goal of this phase is to support these legacy calling interfaces and to provide compatibility with BIOS standards. There are multiple entry points for the BIOS - see the entry\_XXX() assembler functions in romlayout.S.

Callers use most of these legacy entry points by setting up a particular CPU register state, invoking the BIOS, and then inspecting the returned CPU register state. To handle this, SeaBIOS will backup the current register state into a "struct bregs" (see romlayout.S, entryfuncs.S, and bregs.h) on call entry and then pass this struct to the C code. The C code can then inspect the register state and modify it. The assembler entry functions will then restore the (possibly modified) register state from the "struct bregs" on return to the caller.

Resume and reboot
-----------------

As noted above, on emulators SeaBIOS handles the 0xFFFF0000:FFF0 machine startup execution vector. This vector is also called on machine faults and on some machine "resume" events. It can also be called (as 0xF0000:FFF0) by software as a request to reboot the machine (on emulators, coreboot, and CSM).

The SeaBIOS "resume and reboot" code handles these calls and attempts to determine the desired action of the caller. Code flow starts in 16bit mode in romlayout.S:reset\_vector() which calls romlayout.S:entry\_post() which calls romlayout.S:entry\_resume() which calls resume.c:handle\_resume(). Depending on the request the handle\_resume() code may transition to 32bit mode.

Technically this code is part of the "runtime" phase, so even though parts of it run in 32bit mode it still has the same limitations of the runtime phase.

Internally SeaBIOS implements a simple cooperative multi-tasking system. The system works by giving each "thread" its own stack, and the system round-robins between these stacks whenever a thread issues a yield() call. This "threading" system may be more appropriately described as [coroutines](http://en.wikipedia.org/wiki/Coroutine). These "threads" do not run on multiple CPUs and are not preempted, so atomic memory accesses and complex locking is not required.

The goal of these threads is to reduce overall boot time by parallelizing hardware delays. (For example, by allowing the wait for an ATA hard drive to spin-up and respond to commands to occur in parallel with the wait for a PS/2 keyboard to respond to a setup command.) These hardware setup threads are only available during the "setup" sub-phase of the [POST phase](#POST_phase).

The code that implements threads is in stacks.c.

The SeaBIOS C code always runs with hardware interrupts disabled. All of the C code entry points (see romlayout.S) are careful to explicitly disable hardware interrupts (via "cli"). Because running with interrupts disabled increases interrupt latency, any C code that could loop for a significant amount of time (more than about 1 ms) should periodically call yield(). The yield() call will briefly enable hardware interrupts to occur, then disable interrupts, and then resume execution of the C code.

There are two main reasons why SeaBIOS always runs C code with interrupts disabled. The first reason is that external software may override the default SeaBIOS handlers that are called on a hardware interrupt event. Indeed, it is common for DOS based applications to do this. These legacy third party interrupt handlers may have undocumented expectations (such as stack location and stack size) and may attempt to call back into the various SeaBIOS software services. Greater compatibility and more reproducible results can be achieved by only permitting hardware interrupts at specific points (via yield() calls). The second reason is that much of SeaBIOS runs in 32bit mode. Attempting to handle interrupts in both 16bit mode and 32bit mode and switching between modes to delegate those interrupts is an unneeded complexity. Although disabling interrupts can increase interrupt latency, this only impacts legacy systems where the small increase in interrupt latency is unlikely to be noticeable.

SeaBIOS implements 16bit real mode handlers for both hardware interrupts and software request "interrupts". In a traditional BIOS, these requests would use the caller's stack space. However, the minimum amount of space the caller must provide has not been standardized and very old DOS programs have been observed to allocate very small amounts of stack space (100 bytes or less).

By default, SeaBIOS now switches to its own stack on most 16bit real mode entry points. This extra stack space is allocated in ["low memory"](https://seabios.org/Memory_Model "Memory Model"). It ensures SeaBIOS uses a minimal amount of a callers stack (typically no more than 16 bytes) for these legacy calls. (More recently defined BIOS interfaces such as those that support 16bit protected and 32bit protected mode calls standardize a minimum stack size with adequate space, and SeaBIOS generally will not use its extra stack in these cases.)

The code to implement this stack "hopping" is in romlayout.S and in stacks.c.
