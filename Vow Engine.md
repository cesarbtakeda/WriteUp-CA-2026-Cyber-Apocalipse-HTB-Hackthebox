
```python
# Read the first line: N (witnesses), M (seal marks), Q (disputes / queries)
n = input().split()
N, M, Q = int(n[0]), int(n[1]), int(n[2])

# Initialize the graph
graph = [[] for _ in range(N)]
for _ in range(M):
    u, v, w = map(int, input().split())
    graph[u].append((v, w))
    graph[v].append((u, w))

# Structures for Depth-First Search (DFS)
visited = [False] * N
dist = [0] * N          # dist[i] stores the XOR sum of the path from the component's root to i
comp_id = [-1] * N      # comp_id[i] stores the ID of the connected component of node i
cycle_space = []        # cycle_space[c] stores the set of possible cycle XOR sums in component c

current_comp = 0

# Pre-processing: Builds the Spanning Tree and Cycle Basis
for i in range(N):
    if not visited[i]:
        stack = [i]
        visited[i] = True
        comp_id[i] = current_comp
        
        # The set starts with 0 (representing "no cycles used")
        reachable_cycles = {0}
        
        while stack:
            u = stack.pop()
            
            for v, w in graph[u]:
                if not visited[v]:
                    # Spanning tree edge
                    visited[v] = True
                    dist[v] = dist[u] ^ w
                    comp_id[v] = current_comp
                    stack.append(v)
                else:
                    # Back-edge - We found a cycle!
                    # The cycle XOR is the distance of u, distance of v, and the weight w of the edge connecting them
                    cycle_xor = dist[u] ^ dist[v] ^ w
                    
                    # If this cycle generates a new XOR value, we combine it with all previous ones
                    if cycle_xor not in reachable_cycles:
                        new_reachable = set(reachable_cycles)
                        for val in reachable_cycles:
                            new_reachable.add(val ^ cycle_xor)
                        reachable_cycles = new_reachable
        
        # Save all possible cycle XOR values of this component
        cycle_space.append(reachable_cycles)
        current_comp += 1

# Processing the Queries
for _ in range(Q):
    u, v, target = map(int, input().split())
    
    # If they are not in the same component, there is no path
    if comp_id[u] != comp_id[v]:
        print("NO")
    else:
        # Base path in the spanning tree
        base_path_xor = dist[u] ^ dist[v]
        
        # In order for (base_path_xor ^ cycles) == target,
        # we need to find cycles such that cycles == (base_path_xor ^ target)
        needed_cycle_xor = base_path_xor ^ target
        
        # Check if this combination of cycles is reachable
        if needed_cycle_xor in cycle_space[comp_id[u]]:
            print("YES")
        else:
            print("NO")
```

**O Motor do Juramento (Árvore Geradora + XOR Basis)**
Caldrin Vowmark precisa processar milhares de contestações quase instantaneamente. Como a rede de testemunhas de Garran possui caminhos redundantes (ciclos), a solução não exige testar todos os caminhos possíveis (o que causaria um erro de "Time Limit Exceeded"). Em vez disso, utilizamos propriedades algébricas da operação XOR ($GF(2)$) para comprimir toda a rede em um pequeno conjunto de possibilidades.

**Lógica do Algoritmo**
- **Arestas Base vs. Ciclos:** Qualquer caminho entre $u$ e $v$ é essencialmente um "caminho principal" (através da Árvore Geradora) somado (via XOR) a um conjunto de ciclos (rotas redundantes) que o caminho decide visitar e voltar.
- **O Caminho Principal (`dist`):** Durante a DFS, calculamos o XOR acumulado da raiz de um componente até cada nó. Pela propriedade do XOR ($X \oplus X = 0$), o caminho direto entre $u$ e $v$ na árvore é sempre `dist[u] ^ dist[v]`. A raiz comum se anula.
- **Capturando Ciclos (`cycle_space`):** Toda vez que a DFS tenta visitar um nó já visitado, encontramos um ciclo (Back-edge). O valor desse ciclo é `dist[u] ^ dist[v] ^ w`. Adicionamos esse valor ao nosso conjunto (`reachable_cycles`).
- **Fechamento de Possibilidades:** Se temos um ciclo valendo $A$ e outro valendo $B$, o usuário pode percorrer ambos, gerando $A \oplus B$. Por isso, sempre que achamos um ciclo novo, combinamos (fazemos XOR) dele com **todos** os valores já descobertos no conjunto, gerando a "Base Linear" completa.
- **Query:** Queremos saber se o caminho base modificado por alguns ciclos atinge o `target`.
    
    $$Base \oplus Ciclos = Target$$
    
    Aplicando XOR de $Base$ dos dois lados, isolamos o que precisamos buscar:
    
    $$Ciclos = Base \oplus Target$$
    
    Basta checar se esse valor existe no nosso conjunto de ciclos (pesquisa $\mathcal{O}(1)$).
    

**Complexidade**

- **Tempo (Pré-processamento):** $\mathcal{O}(N + M \cdot 64)$. A DFS visita os $N$ nós e $M$ arestas. Ao encontrar um ciclo, atualizar o `set` custa no máximo 64 operações (pois $W \le 63$, o que significa apenas 6 bits, logo, no máximo $2^6 = 64$ valores possíveis de XOR).
- **Tempo (Por Query):** $\mathcal{O}(1)$. É apenas uma verificação de igualdade em um array de IDs e uma busca no `set` Python (que tem lookup em tempo constante).
- **Espaço:** $\mathcal{O}(N + M)$ para guardar o grafo, variáveis de estado e os pequenos `sets` de ciclos (que ocupam espaço desprezível, max 64 inteiros por componente).

**Decisões Técnicas**
- **Descarte Seguro do Pai (No-op):** A DFS processa todas as arestas, incluindo a aresta de onde viemos (a árvore). Se a DFS estiver em $v$ e checar o pai $u$, ela calculará o "ciclo" $dist[v] \oplus dist[u] \oplus w$. Isso magicamente resulta em $0$. Como o conjunto já possui $0$, essa operação é inofensiva. Isso permitiu simplificar o código, removendo a necessidade de rastrear o "pai" de cada nó.
- **`set` do Python para Base Linear:** Em problemas com restrições maiores (ex: 32 bits), usaríamos eliminação Gaussiana para construir a Base XOR (tamanho max 32). Como os pesos do problema são minúsculos (6 bits, $W \le 63$), usar a estrutura nativa de `set` (`{}`) com atualização iterativa é muito mais rápido de codar, isento de bugs e tem a mesma eficiência prática.

flag: HTB{th3_b3ll_r1ngs_wr0ng_0n_purp0s3}