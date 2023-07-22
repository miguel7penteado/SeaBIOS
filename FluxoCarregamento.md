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

```asm
0xFFFF0000:FFF0
```

Os emuladores mapeiam o binário SeaBIOS para este endereço e o SeaBIOS organiza para que 

```asm
;;romlayout.S
reset\_vector()
```

esteja presente lá. Este código chama

```asm
;;romlayout.S
entry\_post()
```

que então chama 

```cpp
# post.c
handle\_post()
```

no **modo de 32 bits**.
No **coreboot**, a compilação faz com que 

```asm
;;romlayout.S
entry\_elf()
```
  
seja chamado no **modo de 32 bits**. Isso chama 

```cpp
post.c
handle\_post()
```

No CSM, a construção organiza para que 

```asm
;;romlayout.S
entry\_csm()
```

seja chamado (no **modo de 16 bits** ). Isso então chama

```cpp
#csm.c
handle\_csm()
```
no **modo de 32 bits**. Ao contrário dos emuladores e do coreboot, a fase SeaBIOS CSM POST é orquestrada com UEFI e há várias chamadas entre SeaBIOS e UEFI via 

```cpp
#csm.c
handle\_csm()
```
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
```asm
;;romlayout.S
entry\_19()
```
 ou
 ```asm
;;romlayout.S
entry\_18()
```
que então faz a transição para o **modo de 32 bits** e chama
```cpp
# boot.c
handle\_19()**
```
ou
```cpp
# boot.c
handle\_18()
```

A fase de inicialização também faz parte tecnicamente da fase de **execuçã principal** do SeaBIOS. Normalmente, ele é chamado imediatamente após a fase POST, mas também pode ser chamado por um sistema operacional ou várias vezes na tentativa de encontrar uma mídia de inicialização válida. Embora o código C da fase de inicialização seja executado no **modo de 32 bits**, ele não tem acesso de gravação à região de memória 
* **0x0f0000-0x100000**
e não pode chamar as várias chamadas **malloc\_X()**.

Consulte [Modelo de memória](https://seabios.org/Memory_Model "Modelo de memória") para obter mais informações.


## A fase de Execução Principal

A fase de **execução principal** ocorre depois que a fase de inicialização inicia o sistema operacional. Uma vez nesta fase, o código SeaBIOS pode ser invocado pelo sistema operacional usando várias chamadas de **16 bits** e **32 bits**. O objetivo desta fase é oferecer suporte a essas interfaces de chamada herdadas e **fornecer compatibilidade** com os padrões do BIOS. 
Existem vários pontos de entrada para o BIOS - consulte as funções assembler 
* **entry\_XXX()**
em **romlayout.S.**

Os chamadores usam a maioria desses pontos de entrada herdados configurando um determinado estado de registro da CPU, invocando o BIOS e, em seguida, inspecionando o estado de registro da CPU retornado. Para lidar com isso, o SeaBIOS fará backup do estado atual do registro em um "struct bregs" (consulte **romlayout.S**, **entryfuncs.S** e **bregs.h**) na entrada da chamada e, em seguida, passará essa estrutura para o código C. O código C pode então inspecionar o estado do registrador e modificá-lo. As funções de entrada de assembler restaurarão o estado de registro (possivelmente modificado) de "struct bregs" ao retornar a função.

## Continuar e Reiniciar

Conforme observado acima, em emuladores SeaBIOS lida com o vetor de execução de inicialização da máquina **0xFFFF0000:FFF0**. Este vetor também é chamado em falhas de máquina e em alguns eventos de "retorno" de máquina. Ele também pode ser chamado (como **0xF0000:FFF0**) pelo software como uma solicitação para reinicializar a máquina (em emuladores, coreboot e CSM).

O código SeaBIOS "resume and reboot" lida com essas chamadas e tenta determinar a ação desejada do chamador. O fluxo de código começa no modo de 16 bits em

```asm
;; arquivo romlayout.S
reset\_vector()
```
que chama 
```asm
;; arquivo romlayout.S
entry\_post() 
```

que chama romlayout.S:entry\_resume() que chama resume.c:handle\_resume(). Dependendo da solicitação, o código handle\_resume() pode fazer a transição para o modo de 32 bits.

Technically this code is part of the "runtime" phase, so even though parts of it run in 32bit mode it still has the same limitations of the runtime phase.

Internally SeaBIOS implements a simple cooperative multi-tasking system. The system works by giving each "thread" its own stack, and the system round-robins between these stacks whenever a thread issues a yield() call. This "threading" system may be more appropriately described as [coroutines](http://en.wikipedia.org/wiki/Coroutine). These "threads" do not run on multiple CPUs and are not preempted, so atomic memory accesses and complex locking is not required.

The goal of these threads is to reduce overall boot time by parallelizing hardware delays. (For example, by allowing the wait for an ATA hard drive to spin-up and respond to commands to occur in parallel with the wait for a PS/2 keyboard to respond to a setup command.) These hardware setup threads are only available during the "setup" sub-phase of the [POST phase](#POST_phase).

The code that implements threads is in stacks.c.

The SeaBIOS C code always runs with hardware interrupts disabled. All of the C code entry points (see romlayout.S) are careful to explicitly disable hardware interrupts (via "cli"). Because running with interrupts disabled increases interrupt latency, any C code that could loop for a significant amount of time (more than about 1 ms) should periodically call yield(). The yield() call will briefly enable hardware interrupts to occur, then disable interrupts, and then resume execution of the C code.

There are two main reasons why SeaBIOS always runs C code with interrupts disabled. The first reason is that external software may override the default SeaBIOS handlers that are called on a hardware interrupt event. Indeed, it is common for DOS based applications to do this. These legacy third party interrupt handlers may have undocumented expectations (such as stack location and stack size) and may attempt to call back into the various SeaBIOS software services. Greater compatibility and more reproducible results can be achieved by only permitting hardware interrupts at specific points (via yield() calls). The second reason is that much of SeaBIOS runs in 32bit mode. Attempting to handle interrupts in both 16bit mode and 32bit mode and switching between modes to delegate those interrupts is an unneeded complexity. Although disabling interrupts can increase interrupt latency, this only impacts legacy systems where the small increase in interrupt latency is unlikely to be noticeable.

SeaBIOS implements 16bit real mode handlers for both hardware interrupts and software request "interrupts". In a traditional BIOS, these requests would use the caller's stack space. However, the minimum amount of space the caller must provide has not been standardized and very old DOS programs have been observed to allocate very small amounts of stack space (100 bytes or less).

By default, SeaBIOS now switches to its own stack on most 16bit real mode entry points. This extra stack space is allocated in ["low memory"](https://seabios.org/Memory_Model "Memory Model"). It ensures SeaBIOS uses a minimal amount of a callers stack (typically no more than 16 bytes) for these legacy calls. (More recently defined BIOS interfaces such as those that support 16bit protected and 32bit protected mode calls standardize a minimum stack size with adequate space, and SeaBIOS generally will not use its extra stack in these cases.)

The code to implement this stack "hopping" is in romlayout.S and in stacks.c.
