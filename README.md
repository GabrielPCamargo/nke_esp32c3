# NKE (fakeOS) versão ESP32C3

Esse repositório apresenta uma versão do NKE para o ESP32C3 (RISC-V) que baseia-se no framework para compilação baremetal [MDK](https://github.com/cpq/mdk) (sem nenhuma utilização do ESP-IDF) e na ferramenta de flash [Esputil](https://github.com/cpq/esputil) com pequenas alterações comentadas [aqui](#alterações).


## Requisitos

Este projeto utiliza o framework bare-metal MDK e herda seus requisitos de compilação.

**Os testes foram realizados apenas em Linux. O funcionamento em Windows não foi validado.

É necessário ter instalado:

- Docker
- Git

O compilador RISC-V é executado dentro de uma imagem Docker, portanto não é necessário instalar uma toolchain localmente.

## Obtenção do código

```bash
git clone https://github.com/GabrielPCamargo/nke_esp32c3
cd nke_esp32c3
```

## Configuração do Makefile

Antes de compilar o projeto, ajuste algumas variáveis no arquivo `src/Makefile`.

### Porta serial

Configure a porta serial correspondente à sua ESP32-C3:

```makefile
export PORT := /dev/ttyACM0
```

Alguns exemplos:

```makefile
# Linux
export PORT := /dev/ttyACM0

# Linux (outras placas)
export PORT := /dev/ttyUSB0

# Windows
export PORT := COM3
```

### Arquivos fonte

Adicione à variável `SOURCES` todos os arquivos `.c` que devem ser compilados (os arquivos devem estar na pasta /src):

Exemplo de arquivo fonte padrão:

```makefile
SOURCES = NKE.c
```

Por exemplo, para compilar multiplos arquivos:

```makefile
SOURCES = timer.c uart.c gpio.c main.c
```

### Aquivos de cabeçalho
Arquivos de cabeçalho (`*.h`) podem ser adicionados na pasta /include que já serão automaticamente incluídos na compilação.

No entanto, caso algum arquivo de cabeçalho (`*.h`) seja alterado, execute:

```bash
make clean
make
```

Isso é necessário porque o Makefile não possui regras de dependência para arquivos de cabeçalho.

## Execução

Após realizar as alterações necessárias no `Makefile`, entre na pasta /src (`cd src`) e utilize os comandos abaixo para compilar, gravar e monitorar o programa.

### Compilar

Compila o projeto e gera o binário para a ESP32-C3:

```bash
make build
```

### Gravar na placa

Grava o binário compilado na ESP32-C3 através da porta serial configurada:

```bash
make flash
```

### Monitor serial

Abre um monitor serial para visualizar mensagens enviadas pela aplicação:

```bash
make monitor
```

### Fluxo completo

Os comandos também podem ser executados em sequência:

```bash
make build flash monitor
```

Equivalente a:

```bash
make build
make flash
make monitor
```

Esse fluxo compila o projeto, grava o firmware na placa e, em seguida, abre o monitor serial para acompanhar a execução da aplicação.
## Alterações:

O arquivo esputil.c foi atualizado permitindo a entrada no modo download em placas com USB-ACM (sem conversor serial) alterando-se a função reset_to_bootloader:

```c
static void reset_to_bootloader(int fd) {
  sleep_ms(100);       // Wait
  set_dtr(fd, false);  // IO0 -> HIGH
  set_rts(fd, false);   // EN -> LOW
  sleep_ms(100);       // Wait
  set_dtr(fd, true);   // IO0 -> LOW
  set_rts(fd, false);  // EN -> HIGH
  sleep_ms(100);       // Wait
  set_dtr(fd, true);  // IO0 -> HIGH
  set_rts(fd, true);   // EN -> LOW
  sleep_ms(100);       // Wait
  set_dtr(fd, false);  // IO0 -> HIGH
  set_rts(fd, true);   // EN -> LOW
  sleep_ms(100);        // Wait
  set_dtr(fd, false);  // IO0 -> HIGH
  set_rts(fd, false);  // IO0 -> HIGH
  flushio(fd);         // Discard all data
}

```
A função de monitor também foi alterada para reinicar a ESP32C3 no começo da leitura da serial com a função hard_reset:

```c
 if (strcmp(*command, "info") == 0) {
    info(&ctx);
  } else if (strcmp(*command, "flash") == 0) {
    flash(&ctx, &command[1]);
  } else if (strcmp(*command, "readmem") == 0) {
    readmem(&ctx, &command[1]);
  } else if (strcmp(*command, "readflash") == 0) {
    readflash(&ctx, &command[1]);
  } else if (strcmp(*command, "monitor") == 0) {
    if (atoi(ctx.baud) != 115200) {
      change_baud(ctx.fd, atoi(ctx.baud), ctx.verbose);
    }
    hard_reset(ctx.fd);  // Linha alterada
    while (s_signo == 0) monitor(&ctx);
  } else {
    printf("Unknown command: %s\n", *command);
    usage(&ctx);
  }

```