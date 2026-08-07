# Ringtrue — Engenharia Reversa
## Resolvido por Joaov1t

## Contexto

O desafio entrega um executável chamado `ringtrue`.

O enunciado fala sobre um fragmento que foi ensinado a reconhecer uma única voz formada por oito tons.

Na prática, precisamos analisar o programa, entender como os oito valores são processados e encontrar a combinação correta sem realizar brute force.

---

## Identificando o arquivo

Primeiro verificamos o tipo do arquivo:

```bash
file ringtrue
```

A saída foi:

```text
ringtrue: ELF 64-bit LSB pie executable, x86-64, dynamically linked, not stripped
```

Isso mostra que o arquivo é:

- Um executável Linux de 64 bits;
- Compilado para a arquitetura x86-64;
- PIE;
- Não stripado.

O fato de não estar stripado facilita a análise, pois os nomes das funções e variáveis ainda estão presentes no binário.

Adicionamos permissão de execução:

```bash
chmod +x ringtrue
```

---

## Executando o programa

Executamos o binário para entender qual entrada ele espera:

```bash
./ringtrue
```

O programa apresenta:

```text
Present the First Mark
(eight tone samples, space-separated)
attune>
```

Isso indica que precisamos fornecer oito números separados por espaço.

Testamos uma entrada qualquer:

```bash
echo '1 2 3 4 5 6 7 8' | ./ringtrue
```

A entrada não foi aceita, então precisamos encontrar os oito valores corretos.

---

## Procurando strings

Usamos o comando `strings`:

```bash
strings -a -n 4 ringtrue
```

Entre as strings encontradas estavam:

```text
%d %d %d %d %d %d %d %d
npu: resonance_core.tflm - MLP 8-8-8-8, int8
weights symmetric int8
ECHO_S
L0_W
L0_B
L1_W
L1_B
L2_W
L2_B
VOW_CIPHER
VOW_LEN
dense
main
```

Essas strings revelam que o programa utiliza uma pequena rede neural.

A estrutura `MLP 8-8-8-8` indica:

```text
8 entradas
    ↓
8 neurônios
    ↓
8 neurônios
    ↓
8 saídas
```

Os nomes encontrados representam:

```text
L0_W e L0_B = pesos e biases da primeira camada
L1_W e L1_B = pesos e biases da segunda camada
L2_W e L2_B = pesos e biases da terceira camada
ECHO_S       = saída esperada
```

---

## Listando os símbolos

Como o executável não está stripado, podemos visualizar os símbolos usando:

```bash
nm -S -n ringtrue
```

Alguns dos símbolos importantes encontrados foram:

```text
dense
main
ECHO_S
L0_W
L0_B
L1_W
L1_B
L2_W
L2_B
VOW_CIPHER
VOW_LEN
```

Isso confirma que os pesos, biases e a saída esperada estão armazenados diretamente dentro do executável.

---

## Analisando a função `dense`

Desmontamos a função responsável pelo cálculo das camadas:

```bash
objdump -d -Mintel --disassemble=dense ringtrue
```

A função é equivalente a:

```c
void dense(
    int8_t weights[8][8],
    int32_t bias[8],
    int64_t input[8],
    int64_t output[8]
) {
    for (int j = 0; j < 8; j++) {
        int64_t total = bias[j];

        for (int i = 0; i < 8; i++) {
            total += weights[j][i] * input[i];
        }

        output[j] = total;
    }
}
```

Em matemática:

```text
saída = pesos × entrada + bias
```

Ou:

```text
Y = W × X + B
```

---

## Fluxo da rede

O programa chama a função `dense` três vezes.

Podemos localizar as chamadas usando:

```bash
objdump -d -Mintel ringtrue | grep -B5 -A8 'call.*dense'
```

O fluxo reconstruído é equivalente a:

```python
entrada = ler_oito_numeros()

camada0 = dense(L0_W, L0_B, entrada)
camada0 = ativacao(camada0)

camada1 = dense(L1_W, L1_B, camada0)
camada1 = ativacao(camada1)

saida = dense(L2_W, L2_B, camada1)

if saida == ECHO_S:
    abrir_cofre()
```

---

## Entendendo a ativação

Após as duas primeiras camadas, o programa aplica uma ativação.

A lógica encontrada é equivalente a:

```python
def activation(value):
    if value < 0:
        return value * 2

    return value
```

Ou seja:

```text
Valor positivo → permanece igual
Valor negativo → é multiplicado por 2
```

Exemplo:

```text
100  → 100
-100 → -200
```

Como essa transformação é simples, também podemos desfazê-la:

```python
def inverse_activation(value):
    if value < 0:
        return value // 2

    return value
```

---

## Comparação final

A saída final da rede é comparada com o vetor `ECHO_S`.

O vetor esperado é:

```python
ECHO_S = [
     1542223,
      574187,
    -2694563,
    -3518303,
      383776,
      576877,
     2637871,
    -2518822,
]
```

O programa realiza uma comparação equivalente a:

```python
if output == ECHO_S:
    print("IT RINGS TRUE")
```

Portanto, precisamos encontrar quais oito valores de entrada produzem exatamente esse vetor.

---

## Invertendo a rede

Cada camada utiliza a seguinte fórmula:

```text
Y = W × X + B
```

Como conhecemos:

- A saída `Y`;
- Os pesos `W`;
- Os biases `B`;

podemos calcular a entrada:

```text
X = W⁻¹ × (Y - B)
```

Assim, resolvemos a rede de trás para frente:

```text
ECHO_S
    ↓
Inverter a terceira camada
    ↓
Desfazer a ativação
    ↓
Inverter a segunda camada
    ↓
Desfazer a ativação
    ↓
Inverter a primeira camada
    ↓
Recuperar os oito tons
```

Não foi necessário realizar brute force.

---

## Criando o solver

Instalamos o SymPy caso necessário:

```bash
python3 -m pip install sympy
```

Criamos o solver:

```bash
cat > solve.py <<'PY'
#!/usr/bin/env python3

from pathlib import Path
import struct
import sympy as sp

BINARY = Path("./ringtrue")
data = BINARY.read_bytes()


def read_i8(offset, count):
    return list(
        struct.unpack_from(f"{count}b", data, offset)
    )


def read_i32(offset, count):
    return list(
        struct.unpack_from("<" + ("i" * count), data, offset)
    )


def read_i64(offset, count):
    return list(
        struct.unpack_from("<" + ("q" * count), data, offset)
    )


def read_matrix_i8(offset):
    values = read_i8(offset, 64)
    return sp.Matrix(8, 8, values)


def invert_dense(weights, bias, output):
    return weights.inv() * (
        sp.Matrix(output) - sp.Matrix(bias)
    )


def invert_activation(values):
    result = []

    for value in values:
        value = sp.Rational(value)

        if value < 0:
            result.append(value / 2)
        else:
            result.append(value)

    return sp.Matrix(result)


ECHO_S = read_i64(0x5060, 8)

L2_B = read_i32(0x50A0, 8)
L1_B = read_i32(0x50C0, 8)
L0_B = read_i32(0x50E0, 8)

L2_W = read_matrix_i8(0x5100)
L1_W = read_matrix_i8(0x5140)
L0_W = read_matrix_i8(0x5180)


layer1_activated = invert_dense(
    L2_W,
    L2_B,
    ECHO_S,
)

layer1_normal = invert_activation(
    layer1_activated
)

layer0_activated = invert_dense(
    L1_W,
    L1_B,
    layer1_normal,
)

layer0_normal = invert_activation(
    layer0_activated
)

tones = invert_dense(
    L0_W,
    L0_B,
    layer0_normal,
)

tones = [int(value) for value in tones]

print("[+] Tons:", " ".join(map(str, tones)))
print("[+] ASCII:", "".join(chr(value) for value in tones))
PY
```

Executamos:

```bash
python3 solve.py
```

A saída foi:

```text
[+] Tons: 83 97 108 116 67 114 119 110
[+] ASCII: SaltCrwn
```

---

## Validando no executável

Enviamos os oito valores encontrados para o programa:

```bash
echo '83 97 108 116 67 114 119 110' | ./ringtrue
```

O programa respondeu:

```text
IT RINGS TRUE
The First Mark is yours.

vault-seal: OPEN
```

E apresentou a flag:

```text
HTB{h3y_s1gn3t_1_4m_y0ur_k1ng}
```

---

## Conclusão

O executável implementava uma pequena rede neural com três camadas densas.

Os pesos, biases e a saída esperada estavam armazenados diretamente dentro do binário.

Como as operações utilizadas eram matematicamente reversíveis, foi possível inverter cada camada e recuperar os oito valores corretos sem realizar brute force.

Os tons encontrados foram:

```text
83 97 108 116 67 114 119 110
```

Em ASCII:

```text
SaltCrwn
```

A flag final é:

```text
HTB{h3y_s1gn3t_1_4m_y0ur_k1ng}
```
