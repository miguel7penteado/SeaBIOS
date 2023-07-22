
## Bem-vindo ao projeto SeaBIOS! 
Este projeto implementa uma bios no formato histórica X86 que é construída usando-se com ferramentas padrão GNU.

## Compilação
Consulte as informações de compilação e desenvolvedor em:

  [http://seabios.org/Developer_Documentation](http://seabios.org/Developer_Documentation)

Para os impacientes, o SeaBIOS foi desenvolvido para QEMU e testado no QEMU com:

## Execução Rápida
```bash
# baixe o projeto 
# crie uma pasta local_do_projeto e salve e descompacte o fonte do seabios la dentro
cd local_do_projeto
make
# considerando que você tenha o qemu instalado
qemu -bios out/bios.bin
```
## Configuração
A `SeaBIOS` pode ser configurado com `kconfig` (Sistema de Construção do Kernel do Linux). 
Para alterar o padrão configuração pode-se executar `make menuconfig` antes de executar `make`.
Para outros tipos de compilações e para desenvolvedores mais detalhados documentação, consulte a documentação on-line listada acima.

## Como funciona ?
Para saber como a bios configura o hardware e deixa tudo pronto para o Sistema Operacional de uma olhada em [Fluxo de Carregamento](FluxoCarregamento.md).


