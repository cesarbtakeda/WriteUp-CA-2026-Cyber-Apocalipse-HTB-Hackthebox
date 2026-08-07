
Resposta: 
```python
# Read the first line
n = input().strip()

# Read the first lines values
N, M, Q = map(int, n.split())

# Read Each Swap
swaps = []
for _ in range(M):
    linha = input().strip()  # Each swap line
    a, b = map(int, linha.split())
    swaps.append((a, b))

# Read all queries
queries = []
for _ in range(Q):
    linha = input().strip()  # Read each query line
    p = int(linha)
    queries.append(p)

# Simulate all queries to get the final res
for p in queries:
    cur_pos = p  # position
    
    # all swaps in order applied
    for a, b in swaps:
        if cur_pos == a:
            cur_pos = b
        elif cur_pos == b:
            cur_pos = a
    
    print(cur_pos)  # Result
```


**Problema:** Temos N copos em um bar, numerados de 1 a N. Inicialmente, o copo **i** contém o item **i**. Realizamos M trocas (swaps) de posições em ordem. Depois, fazemos Q perguntas: "O item que começou na posição p, onde ele termina após todas as trocas?"

**Lógica do Algoritmo:**

1. **Rastreamento de Posição (Position Tracking)**
   - Em vez de simular todos os itens (o que seria O(N×M)), rastreamos apenas os itens consultados
   - Para cada consulta `p`, acompanhamos a **posição atual** do item que começou em `p`
   - Inicialmente, `cur_pos = p` (o item está na posição onde começou)

2. **Aplicação das Trocas em Ordem**
   - Percorremos todas as trocas na ordem em que foram realizadas
   - Para cada troca `(a, b)`:
     - Se o item está na posição `a`, ele é movido para `b` (`cur_pos = b`)
     - Se o item está na posição `b`, ele é movido para `a` (`cur_pos = a`)
     - Se não está em nenhuma das duas, permanece onde está

1. **Solução**
   - Quando fazemos `swap(a, b)`, apenas os itens nas posições `a` e `b` mudam de lugar
   - Portanto, para saber se nosso item se moveu, só precisamos verificar se ele está em `a` ou `b`
   - Isso é exatamente o que fazemos: verificamos se `cur_pos == a` ou `cur_pos == b`

4. **Complexidade:**
   - Tempo: O(Q × M) - para cada consulta, percorremos todas as trocas
   - Memória: O(M) - armazenamos apenas as trocas
   - Como Q ≤ 2000 e M ≤ 5000, isso é bem eficiente (até 10 milhões de operações)

#### Exemplo Visual / Visual Example
**Input:**
```
5 4 2
3 2
2 1
5 2
3 1
4
2
```

**Simulação para p=4:**
```
Start:  cur_pos = 4
Swap(3,2): cur_pos=4 (no change)
Swap(2,1): cur_pos=4 (no change)
Swap(5,2): cur_pos=4 (no change)
Swap(3,1): cur_pos=4 (no change)
Result: 4 ✓
```

**Simulação para p=2:**
```
Start:  cur_pos = 2
Swap(3,2): cur_pos=2 → cur_pos=3 (moved to 3)
Swap(2,1): cur_pos=3 (no change)
Swap(5,2): cur_pos=3 (no change)
Swap(3,1): cur_pos=3 → cur_pos=1 (moved to 1)
Result: 1 ✓
```

| Conceito                   | Explicação                                                 |
| -------------------------- | ---------------------------------------------------------- |
| **Rastreamento Invertido** | Acompanhamos apenas os itens que nos interessam, não todos |
| **Simulação Direta**       | Aplicamos as trocas na ordem exata em que ocorreram        |
| **Estado Atual**           | Mantemos apenas a posição atual do item, não todo o array  |
| **Eficiência**             | O(Q×M) é aceitável para os limites do problema             |

---
### Main Problem
Se simulássemos todos os N itens com todas as M trocas, seria O(N×M) = 2000×5000 = 10⁷, que também é aceitável! Mas nossa abordagem é ainda mais eficiente quando Q é pequeno, pois fazemos apenas O(Q×M).