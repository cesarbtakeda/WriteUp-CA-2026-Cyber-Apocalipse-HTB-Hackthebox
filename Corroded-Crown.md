# Corroded Crown

## Resolvida por Nathan

```
Durante a análise do binário corroded_crown, foi identificada uma vulnerabilidade de corrupção de memória decorrente da confiança em metadados controlados pelo próprio programa para determinar o tamanho de uma leitura realizada com read().

A exploração permite sobrescrever estruturas adjacentes na memória e, posteriormente, redirecionar o fluxo de execução através da sobrescrita de um ponteiro de função, culminando na execução de código arbitrário.

Informações do binário
Item	Valor
Arquitetura	x86-64
Tipo	ELF
PIE	Sim
NX	Habilitado
Stack Canary	Habilitado
RELRO	Parcial
Reconhecimento

Inicialmente foi realizada engenharia reversa utilizando:

objdump
Ghidra
pwntools
gdb

A análise concentrou-se nas funções:

check_slot()
inscribe_relic()

conforme mostrado no disassembly fornecido.

Estrutura dos registros

Cada entrada possui tamanho de:

0x10 bytes

O programa utiliza três campos principais:

struct relic
{
    char *buffer;
    int length;
    bool used;
};

A função check_slot() percorre essas estruturas procurando um slot livre.

Análise da função check_slot()

A rotina percorre todos os 64 registros:

cmp DWORD PTR [rbp-0xc],0x3f

Verificando o campo responsável por indicar se o slot está ocupado.

Quando encontra um valor igual a zero, retorna seu índice.

Caso contrário retorna:

-1
Análise da função inscribe_relic()

O fluxo principal é:

Ler índice escolhido pelo usuário.
Validar se está entre 0 e 63.
Recuperar o tamanho associado ao registro.
Executar:
read(0,
     relic[index].buffer,
     relic[index].length);

A chamada observada no disassembly corresponde a:

mov rdx,eax
mov rsi,buffer
mov edi,0
call read

onde:

RDX = tamanho
RSI = destino
Vulnerabilidade

A principal vulnerabilidade decorre do fato de que o programa utiliza diretamente o campo:

relic[index].length

como tamanho da operação de leitura.

Esse campo pode assumir valores inconsistentes com o tamanho real do buffer associado.

Na prática:

buffer real
↓

+--------------------+
|                    |
|                    |
+--------------------+

read(..., length)

           ↑
     valor excessivo

Assim, a leitura ultrapassa o limite do buffer.

Impacto

O overflow permite sobrescrever memória adjacente.

Entre os dados localizados após os buffers encontram-se ponteiros utilizados pelo programa.

Uma vez corrompidos, esses ponteiros passam a referenciar endereços escolhidos pelo atacante.

Estratégia de exploração

A exploração ocorreu em duas etapas.

1. Preparação

Primeiramente foram preenchidos registros até posicionar um buffer adjacente à estrutura alvo.

Em seguida foi utilizado o overflow para modificar campos vizinhos.

2. Redirecionamento

Após controlar o ponteiro utilizado pelo programa, foi possível fazer com que uma chamada indireta executasse uma função de interesse.

O fluxo tornou-se:

Antes

Programa
   │
   ▼
ponteiro legítimo
   │
   ▼
função original

Após a corrupção:

Programa
   │
   ▼
ponteiro sobrescrito
   │
   ▼
endereço controlado
Mitigações encontradas

Durante a análise foram observadas algumas proteções:

NX

Impede execução direta de shellcode na pilha.

Consequência:

Foi necessário reutilizar código existente no processo.

Stack Canary

Protege retornos de função.

Como o ataque não depende da sobrescrita direta do endereço de retorno, essa mitigação não impediu a exploração.

PIE

Randomiza o endereço base do binário.

A exploração exigiu obter ou calcular corretamente os endereços utilizados durante a execução.

Resultado

Foi possível:

controlar estruturas internas;
sobrescrever ponteiros utilizados pelo programa;
redirecionar o fluxo de execução;
obter execução arbitrária de código.
Conclusão

A vulnerabilidade decorre da utilização de um tamanho de leitura inconsistente com o tamanho efetivo do buffer.

Embora o binário possua mecanismos modernos de mitigação (NX, PIE e Stack Canary), a corrupção de estruturas em memória permitiu contornar essas proteções sem necessidade de sobrescrever diretamente o endereço de retorno.

O desafio evidencia a importância de validar rigorosamente metadados utilizados em operações de leitura e escrita, especialmente quando esses valores determinam o tamanho de acessos à memória.
```
