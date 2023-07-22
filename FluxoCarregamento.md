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
reset_vector()
```

esteja presente lá. Este código chama

```asm
;;romlayout.S
entry_post()
```

que então chama 

```cpp
// post.c
handle_post()
```

no **modo de 32 bits**.
No **coreboot**, a compilação faz com que 

```asm
;;romlayout.S
entry_elf()
```
  
seja chamado no **modo de 32 bits**. Isso chama 

```cpp
// post.c
handle_post()
```

No CSM, a construção organiza para que 

```asm
;;romlayout.S
entry_csm()
```

seja chamado (no **modo de 16 bits** ). Isso então chama

```cpp
// csm.c
handle_csm()
```
no **modo de 32 bits**. Ao contrário dos emuladores e do coreboot, a fase SeaBIOS CSM POST é orquestrada com UEFI e há várias chamadas entre SeaBIOS e UEFI via 

```cpp
// csm.c
handle_csm()
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
entry_19()
```
 ou
 ```asm
;;romlayout.S
entry_18()
```
que então faz a transição para o **modo de 32 bits** e chama
```cpp
// boot.c
handle_19()**
```
ou
```cpp
// boot.c
handle_18()
```

A fase de inicialização também faz parte tecnicamente da fase de **execuçã principal** do SeaBIOS. Normalmente, ele é chamado imediatamente após a fase POST, mas também pode ser chamado por um sistema operacional ou várias vezes na tentativa de encontrar uma mídia de inicialização válida. Embora o código C da fase de inicialização seja executado no **modo de 32 bits**, ele não tem acesso de gravação à região de memória 
* **0x0f0000-0x100000**
e não pode chamar as várias chamadas **malloc_X()**.

Consulte [Modelo de memória](https://seabios.org/Memory_Model "Modelo de memória") para obter mais informações.


## A fase de Execução Principal

A fase de **execução principal** ocorre depois que a fase de inicialização inicia o sistema operacional. Uma vez nesta fase, o código SeaBIOS pode ser invocado pelo sistema operacional usando várias chamadas de **16 bits** e **32 bits**. O objetivo desta fase é oferecer suporte a essas interfaces de chamada herdadas e **fornecer compatibilidade** com os padrões do BIOS. 
Existem vários pontos de entrada para o BIOS - consulte as funções assembler 
```asm
;; romlayout.S
entry_XXX()
```
em **romlayout.S.**

Os chamadores usam a maioria desses pontos de entrada herdados configurando um determinado estado de registro da CPU, invocando o BIOS e, em seguida, inspecionando o estado de registro da CPU retornado. Para lidar com isso, o SeaBIOS fará backup do estado atual do registro em um "struct bregs" (consulte **romlayout.S**, **entryfuncs.S** e **bregs.h**) na entrada da chamada e, em seguida, passará essa estrutura para o código C. O código C pode então inspecionar o estado do registrador e modificá-lo. As funções de entrada de assembler restaurarão o estado de registro (possivelmente modificado) de "struct bregs" ao retornar a função.

## Continuar e Reiniciar

Conforme observado acima, em emuladores SeaBIOS lida com o vetor de execução de inicialização da máquina **0xFFFF0000:FFF0**. Este vetor também é chamado em falhas de máquina e em alguns eventos de "retorno" de máquina. Ele também pode ser chamado (como **0xF0000:FFF0**) pelo software como uma solicitação para reinicializar a máquina (em emuladores, coreboot e CSM).

O código SeaBIOS "resume and reboot" lida com essas chamadas e tenta determinar a ação desejada do chamador. O fluxo de código começa no modo de 16 bits em

```asm
;; arquivo romlayout.S
reset_vector()
```
que chama 
```asm
;; arquivo romlayout.S
entry_post() 
```

que chama 
```asm
;;romlayout.S
entry_resume()
```
que chama 
```cpp
//resume.c
handle_resume();
```
Dependendo da solicitação, o código **handle_resume()** pode fazer a transição para o **modo de 32 bits**.

Tecnicamente, esse código faz parte da fase de _execução principal_, portanto, embora partes dele sejam executadas no **modo de 32 bits**, ele ainda possui as mesmas limitações da fase de _execução principal_.

Internamente, o SeaBIOS implementa um **sistema multitarefa** cooperativo simples. O sistema funciona dando a cada "thread" sua própria **pilha**, e o sistema **alterna** entre essas **pilhas** sempre que um **thread** emite uma chamada 
```cpp
yield();
```
Este sistema de linhas de controle paralelas **threading** pode ser mais apropriadamente descrito como [**co-rotinas**](http://en.wikipedia.org/wiki/Coroutine). Esses "threads" não são executados em várias CPUs e não são interrompidos, portanto, **acessos atômicos à memória** e **bloqueio complexo** não são necessários.

O objetivo desses encadeamentos é reduzir o tempo geral de inicialização paralelizando os atrasos de hardware. (Por exemplo, permitindo que a espera de um disco rígido ATA gire e responda a comandos ocorra em paralelo com a espera de um teclado PS/2 responder a um comando de configuração).
O código que implementa threads está em **stacks.c**.

O código SeaBIOS C sempre é executado com as interrupções de hardware desativadas. Todos os pontos de entrada do código C (consulte romlayout.S) têm o cuidado de desabilitar explicitamente as interrupções de hardware (via "cli"). Como a execução com as interrupções desabilitadas aumenta a latência da interrupção, qualquer código C que possa fazer um loop por um período de tempo significativo (mais de 1 ms) deve chamar yield() periodicamente. A chamada yield() permitirá brevemente a ocorrência de interrupções de hardware, depois desabilitará as interrupções e retomará a execução do código C.

Existem duas razões principais pelas quais o SeaBIOS sempre executa o código C com as interrupções desativadas. A primeira razão é que o software externo pode substituir os manipuladores SeaBIOS padrão que são chamados em um evento de interrupção de hardware. De fato, é comum que aplicativos baseados em DOS façam isso. Esses manipuladores de interrupção herdados de terceiros podem ter expectativas não documentadas (como localização e tamanho da pilha) e podem tentar chamar de volta os vários serviços de software SeaBIOS. Maior compatibilidade e resultados mais reprodutíveis podem ser alcançados permitindo apenas interrupções de hardware em pontos específicos (via chamadas yield()). A segunda razão é que grande parte do SeaBIOS é executado no modo de 32 bits. Tentar lidar com interrupções no modo de 16 bits e no modo de 32 bits e alternar entre os modos para delegar essas interrupções é uma complexidade desnecessária. Embora desabilitar as interrupções possa aumentar a latência da interrupção, isso afeta apenas os sistemas legados nos quais é improvável que o pequeno aumento na latência da interrupção seja perceptível.

O SeaBIOS implementa manipuladores de modo real de 16 bits para interrupções de hardware e "interrupções" de solicitação de software. Em um BIOS tradicional, essas solicitações usariam o espaço de pilha do chamador. No entanto, a quantidade mínima de espaço que o chamador deve fornecer não foi padronizada e observou-se que programas DOS muito antigos alocam quantidades muito pequenas de espaço de pilha (100 bytes ou menos).

Por padrão, o SeaBIOS agora muda para sua própria pilha na maioria dos pontos de entrada do modo real de 16 bits. Este espaço de pilha extra é alocado em ["memória baixa"](https://seabios.org/Memory_Model "Memory Model"). Ele garante que o SeaBIOS use uma quantidade mínima de uma pilha de chamadores (normalmente não mais que 16 bytes) para essas chamadas herdadas. (Interfaces de BIOS definidas mais recentemente, como aquelas que suportam chamadas de modo protegido de 16 bits e 32 bits, padronizam um tamanho mínimo de pilha com espaço adequado, e o SeaBIOS geralmente não usará sua pilha extra nesses casos.)

O código para implementar esse "salto" de pilha está em **romlayout.S** e em **stacks.c**.

