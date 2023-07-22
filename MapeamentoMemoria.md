# Modelos de Memória - SeaBIOS
O código SeaBIOS é necessário para suportar vários **modelos de memória de CPU x86**. Este requisito afeta o **layout do código** e o **armazenamento interno** do SeaBIOS.

A **linha de CPUs x86** evoluiu ao longo de muitos anos. O processador **8086 original** usava **ponteiros de 16 bits** e só podia **endereçar 1 Megabyte de memória**. O processador **80286** ainda usava **ponteiros de 16 bits**, mas podia **endereçar até 16 Megabytes de memória**. Os processadores **80386** já podiam processar **instruções de 32 bits** e **acessar até 4 Gigabytes de memória**. Os processadores x86 mais recentes podem processar **instruções de 64 bits** e acessar **16 Hexabytes de memória** RAM.

Durante a evolução das CPUs x86 do 8086 para o 80386, o **BIOS foi estendido** para lidar com chamadas nos vários modos implementados pela CPU.

Esta seção descreve os cinco diferentes modelos de execução de CPU x86 e acesso à memória suportados pelo SeaBIOS.

## Modo Real (16 bits)

Este modo é um modo de memória [**segmentada**](http://en.wikipedia.org/wiki/Memory_segmentation) invocado pelas funções. O padrão da CPU é executar **instruções de 16 bits**. As funções normalmente invocam a BIOS emitindo uma instrução 
```asm
int "número"
```
que causa uma [**interrupção**](http://en.wikipedia.org/wiki/Interrupt) **de software** que é **manipulada pela BIOS**. O código SeaBIOS também lida com **interrupções de hardware** neste modo. O SeaBIOS **só pode acessar o primeiro 1 Megabyte de memória** neste modo, mas pode acessar **qualquer parte** desse primeiro megabyte.

# O "Grande" Modo Real (16 bits)

Este modo é um modo de memória segmentada usado para [option roms](http://en.wikipedia.org/wiki/Option_ROM). O padrão da CPU é **executar instruções de 16 bits** e os **acessos segmentados à memória** ainda são usados. No entanto, os limites do segmento são aumentados para que todos os **primeiros 4 gigabytes de memória** sejam totalmente acessíveis. Os chamadores podem invocar todas as funções do [Modo Real (16 bits)](#Modo_Real_(16_bits)) enquanto estiverem neste modo e também podem invocar as funções **Post Memory Manager (PMM)** que estão disponíveis durante a execução da opção rom.

## O Modo Protegido em 16 bits

CPU execution in this mode is similar to [16bit real mode](#16bit_real_mode). The CPU defaults to executing 16bit instructions. However, each segment register indexes a "descriptor table", and it is difficult or impossible to know what the physical address of each segment is. Generally speaking, the BIOS can only access code and data in the f-segment. The PCIBIOS, APM BIOS, and PNP BIOS all have documented 16bit protected mode entry points.

Some old code may attempt to invoke the standard [16bit real mode](#16bit_real_mode) entry points while in 16bit protected mode. The PCI BIOS specification explicitly requires that the legacy "int 1a" real mode entry point support 16bit protected mode calls if they are for the PCI BIOS. Callers of other legacy entry points in protected mode have not been observed and SeaBIOS does not support them.

## O Modo Segmentado em 32 bits

In this mode the processor runs in 32bit mode, but the segment registers may have a limit and may have a non-zero offset. In effect, this mode has all of the limitations of [16bit protected mode](#16bit_protected_mode) - the main difference between the modes is that the processor defaults to executing 32bit instructions. In addition to these limitations, callers may also run the SeaBIOS code at varying virtual addresses and so the code must support code relocation. The PCI BIOS specification and APM BIOS specification define 32bit segmented mode interfaces.

## O Modo Protegido em 32 bits (modo plano)

Nesse modo, o padrão do processador é executar instruções de 32 bits e todos os registradores de segmento têm um deslocamento de zero e permitem o acesso a todos os primeiros 4 gigabytes de memória. Este é o único modo "sensato" para código de 32 bits - compiladores modernos e sistemas operacionais modernos geralmente suportam apenas este modo (ao executar código de 32 bits). Ironicamente, é o único modo que não é estritamente necessário para o suporte de um BIOS. SeaBIOS usa este modo internamente para suportar [as fases de execução POST e BOOT](https://seabios.org/Execution_and_code_flow "Execution and code flow").

Para produzir um código que possa ser executado quando o processador estiver no modo de 16 bits, o SeaBIOS usa a FLAG de ASSEMBLER [binutils ".code16gcc" ](http://en.wikipedia.org/wiki/GNU_Binutils) . Isso instrui o Assembler a emitir opcodes de prefixo extras para que o código de 32 bits produzido por [gcc](http://en.wikipedia.org/wiki/GNU_Compiler_Collection) seja executado mesmo quando o processador estiver no **modo de 16 bits**. 
>> Observe que o gcc sempre produz código de 32 bits - ele não conhece o sinalizador ".code 16 gcc" e não sabe que o código será executado no modo de 16 bits.

O SeaBIOS usa o mesmo código para todos os modos de 16 bits ([Modo Real em 16 bits](#16bit_real_mode), [Grande Modo Real em 16 bits](#16bit_bigreal_mode) e [Modo Protegido em 16 bits](#16bit_protected_mode)) e esse código é montado (Assembler) usando ".code16gcc". O SeaBIOS tem o cuidado de usar **registradores de segmento adequadamente** para que o mesmo código possa ser executado nos diferentes modos de 16 bits que ele precisa suportar.

Dois sinalizadores de tempo de compilação estão disponíveis para determinar o modelo de memória para o qual o código se destina: MODE16 e MODESEGMENT. Ao compilar para os modos de 16 bits, MODE16 é verdadeiro e MODESEGMENT é verdadeiro. No modo segmentado de 32 bits, MODE16 é falso e MODESEGMENT é verdadeiro. No modo plano de 32 bits, MODE16 e MODESEGMENT são falsos.

Existem várias áreas de memória que o SeaBIOS "runtime" [fase](https://seabios.org/Execution_and_code_flow "Execução e fluxo de código") utiliza:

* 0x000000-0x000400: Tabela do descritor de interrupção (IDT). Esta área define 256 vetores de interrupção conforme definido pela especificação da CPU Intel para manipuladores irq de 16 bits. Esta área é lida/gravável em tempo de execução e pode ser acessada a partir do modo real de 16 bits e chamadas de modo bigreal de 16 bits. O SeaBIOS usa apenas esta área para manter a compatibilidade com os sistemas legados.
  
* 0x000400-0x000500: Área de dados do BIOS (BDA). Esta área contém vários sinalizadores e atributos herdados. A área é lida/gravável em tempo de execução e pode ser acessada a partir do modo real de 16 bits e chamadas de modo bigreal de 16 bits. O SeaBIOS usa apenas esta área para manter a compatibilidade com os sistemas legados.
  
* 0x09FC00-0x0A0000 (típico): Extended BIOS Data Area (EBDA). Esta área contém alguns sinalizadores e atributos herdados. A área geralmente está localizada em 0x9FC00, mas pode ser movida por ROMs de opção, por sistemas operacionais legados e por SeaBIOS se CONFIG_MALLOC_UPPERMEMORY não estiver definido. Sua localização real é determinada por um ponteiro no BDA. A área é lida/gravável em tempo de execução e pode ser acessada a partir do modo real de 16 bits e chamadas do modo bigreal de 16 bits. O SeaBIOS usa apenas esta área para manter a compatibilidade com os sistemas legados.
  
* 0x0E0000-0x0F0000 (típico): memória "baixa". Esta área é usada para armazenamento personalizado de leitura/gravação interno do SeaBIOS. A área é lida/gravável em tempo de execução e pode ser acessada a partir do modo real de 16 bits e chamadas do modo bigreal de 16 bits. A área geralmente está localizada no final do segmento eletrônico, mas a compilação pode posicioná-la em qualquer lugar na região 0x0C0000-0x0F0000. No entanto, se CONFIG_MALLOC_UPPERMEMORY não estiver definido, essa região estará entre 0x090000-0x0A0000. O espaço é alocado nesta região marcando uma variável global com o sinalizador "VARLOW" ou chamando malloc_low() durante a inicialização. A área pode ser aumentada dinamicamente (via malloc_low), mas nunca ultrapassará 64K.
  
* 0x0F0000-0x100000: O segmento do BIOS. Essa área é usada para código de tempo de execução e variáveis estáticas. O espaço é alocado nessa região marcando uma variável global com VAR16, um dos sinalizadores VARFSEG, ou chamando malloc_fseg() durante a inicialização. A área é somente leitura em tempo de execução e pode ser acessada a partir do modo real de 16 bits, modo bigreal de 16 bits, modo protegido de 16 bits e chamadas de modo segmentado de 32 bits.
 
Todas as áreas acima também são lidas/gravadas durante a fase de inicialização do SeaBIOS e são acessíveis no modo plano de 32 bits.

As funções de entrada do montador para chamadas de modo segmentado (todos os modos, exceto [32bit flat mode](#32bit_flat_mode)) irão configurar o segmento de dados (%ds) para ser o mesmo que o segmento de pilha (%ss) antes de chamar qualquer código C. Isso permite que todas as variáveis C localizadas na pilha e os ponteiros C para os dados localizados na pilha funcionem normalmente.

No entanto, todo código em execução no modo segmentado deve agrupar os acessos à memória não empilhada em macros especiais. Essas macros garantem que o registro de segmento correto seja usado. A falha em usar a macro correta resultará em um acesso incorreto à memória que provavelmente causará erros difíceis de encontrar.

Existem três macros de acesso à memória de baixo nível:

* GET_VAR / SET_VAR : acessa uma variável usando o registrador de segmento especificado. Normalmente, isso não é usado diretamente pelo código C.
* GET_FARVAR / SET_FARVAR : Atribui o segmento extra (%es) ao ID do segmento fornecido e, em seguida, executa o acesso à memória fornecido por meio de %es.
* GET_FLATPTR / SET_FLATPTR : Essas macros pegam um ponteiro de 32 bits, constroem um par segmento/offset válido em modo real e então realizam o acesso dado. Essas macros não devem ser usadas no modo protegido de 16 bits ou no modo segmentado de 32 bits.

Como a maioria dos acessos à memória são para [memória comum usada em tempo de execução](#Common_memory_used_at_run-time), várias macros auxiliares também estão disponíveis.

* GET_IDT / SET_IDT : Acesse a tabela do descritor de interrupção (IDT).
* GET_BDA / SET_BDA : Acesse a área de dados do BIOS (BDA).
* GET_EBDA / SET_EBDA : Acesse a área de dados do BIOS estendida (EBDA).
* GET_LOW / SET_LOW : Acessar variáveis internas marcadas com VARLOW. (Também existem macros relacionadas GET_LOWFLAT / SET_LOWFLAT para acessar o armazenamento alocado com malloc_low.)
* GET_GLOBAL : Acessar variáveis internas marcadas com os sinalizadores VAR16 ou VARFSEG. (Há também a macro relacionada GET_GLOBALFLAT para acessar o armazenamento alocado com malloc_fseg.)

Durante a [fase] do POST (https://seabios.org/Execution_and_code_flow "Execução e fluxo de código"), o código pode acessar totalmente os primeiros 4 gigabytes de memória. No entanto, os acessos à memória são geralmente limitados à [memória comum usada em tempo de execução](#Common_memory_used_at_run-time) e áreas alocadas em tempo de execução por meio de uma das chamadas malloc:

* **malloc_high** : Zona permanente de alta memória. Esta área é usada para armazenamento personalizado de leitura/gravação interno do SeaBIOS. A área está localizada no topo dos primeiros 4 gigabytes de RAM. É comumente usado para armazenar tabelas padrão acessadas pelo sistema operacional em tempo de execução (ACPI, SMBIOS e MPTable) e para buffers DMA usados por drivers de hardware. A área é lida/gravável em tempo de execução e uma entrada no mapa de memória do e820 é usada para reservá-la. Ao rodar em um emulador que tenha apenas 1 megabyte de ram esta zona ficará vazia.

* **malloc_tmphigh** : Zona temporária de alta memória. Esta área é usada para armazenamento personalizado de leitura/gravação durante a fase de inicialização do SeaBIOS. A área geralmente começa após o primeiro 1 megabyte de RAM (0x100000) e termina antes da zona permanente de alta memória. Ao rodar em um emulador que tenha apenas 1 megabyte de ram esta zona estará vazia. A área não é reservada do sistema operacional, portanto não deve ser acessada após a fase de inicialização do SeaBIOS.

* **malloc_tmplow** : Zona temporária de pouca memória. Esta área é usada para armazenamento personalizado de leitura/gravação durante a fase de inicialização do SeaBIOS. A área reside entre 0x07000-0x90000. A área não é reservada do sistema operacional e por especificação é necessário zerá-la ao final da fase de inicialização.

As regiões "tmplow" e "tmphigh" só estão disponíveis durante a fase de inicialização. Qualquer acesso (leitura ou gravação) após a conclusão da fase de inicialização pode resultar em erros difíceis de encontrar.
