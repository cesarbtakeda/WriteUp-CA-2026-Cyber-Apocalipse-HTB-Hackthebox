## Resolvido por Doz

```python
# Read the first line: N E S T
n = input().split()
N, E, S, T = int(n[0]), int(n[1]), int(n[2]), int(n[3])

# Find a path from a to b where capacitiy > 0
def find_path(graph, capacity, s, t, total_nodes):
    visited = [False] * total_nodes
    parent = [-1] * total_nodes
    stack = [s]
    visited[s] = True
    
    while stack:
        u = stack.pop()
        if u == t:
            path = []
            v = t
            while v != -1:
                path.append(v)
                v = parent[v]
            path.reverse()
            return path
        
        for v in graph[u]:
            if not visited[v] and capacity[u][v] > 0:
                visited[v] = True
                parent[v] = u
                stack.append(v)

    return None

# Figures the max qty flux that can pass thro a to b
def ford_fulkerson(total_nodes, graph, capacity, s, t):
    total_flow = 0
    while True:
        path = find_path(graph, capacity, s, t, total_nodes)
        if path is None:
            break
        
        bottleneck = float('inf')
        for i in range(len(path) - 1):
            u, v = path[i], path[i + 1]
            if capacity[u][v] < bottleneck:
                bottleneck = capacity[u][v]
        
        for i in range(len(path) - 1):
            u, v = path[i], path[i + 1]
            capacity[u][v] -= bottleneck
            capacity[v][u] += bottleneck
        
        total_flow += bottleneck

    return total_flow

# Main logic build to get the final result
def build_transformed_graph(n, e, s, t):
    total_nodes = 2 * n
    capacity = [[0] * total_nodes for _ in range(total_nodes)]
    graph = [[] for _ in range(total_nodes)]
    
    in_node = lambda v: 2 * v
    out_node = lambda v: 2 * v + 1
    
    # Splits In|out on all nodes
    for v in range(n):
        u, w = in_node(v), out_node(v)
        graph[u].append(w)
        graph[w].append(u)
        
        # If S and T can't be cut they get infinite capacity.
        # normal nodes get 1 cap.
        if v == s or v == t:
            capacity[u][w] = 999999
        else:
            capacity[u][w] = 1
            
    for _ in range(e):
        u, v = map(int, input().split())
        
        # conects in and out
        u_out = out_node(u)
        v_in = in_node(v)
        
        graph[u_out].append(v_in)
        graph[v_in].append(u_out)
        capacity[u_out][v_in] = 999999
        
    return total_nodes, graph, capacity

total_nodes, graph, capacity = build_transformed_graph(N, E, S, T)

# Ford-Fulkerson begins on out S to in T
print(ford_fulkerson(total_nodes, graph, capacity, 2 * S + 1, 2 * T))
```


**A Marcha Silenciosa (Corte Mínimo de Vértices)**
Miren Vale precisa interromper a cadeia do sinal de preparação sem tocar no coordenador ($S$) ou no distrito alvo ($T$) diretamente. O objetivo é descobrir o número mínimo de "mãos silenciosas" (nós intermediários) que devem ser expostas para cortar todos os caminhos possíveis por onde o sinal possa passar.

**Lógica do Algoritmo**
- **Desconexão de Vértices via Capacidade de Arestas:** Como o problema pede o número mínimo de mãos (nós, excluindo $S$ e $T$) a serem expostas, transformamos o problema de corte de vértices (vertex-cut) em um problema de fluxo máximo. **Todos os nós** $v$ (incluindo $S$ e $T$) são divididos em dois: um nó de entrada ($2v$) e um nó de saída ($2v+1$), conectados internamente por uma aresta. Para garantir que o coordenador ($S$) e o distrito alvo ($T$) nunca sejam cortados, a capacidade interna deles é definida como **infinita** ($999999$). Para as demais "mãos silenciosas" (nós intermediários), a capacidade interna é estritamente **1**.

- **Mapeamento de Links Direcionados:** As conexões originais por onde o sinal viaja são mapeadas da saída do nó $u$ para a entrada do nó $v$ com capacidade infinita ($999999$).
- **Execução do Ford-Fulkerson:** Usando Busca em Profundidade (DFS) para encontrar os caminhos aumentantes, o algoritmo calcula o fluxo máximo partindo do **nó de saída de S** ($2S+1$) até o **nó de entrada de T** ($2T$). Pelo teorema _Max-Flow Min-Cut_ (Fluxo Máximo, Corte Mínimo), esse valor será exatamente igual ao número mínimo de "mãos" que precisamos expor para desconectar a rede.

**Complexidade**
- **Tempo:** $\mathcal{O}(E \cdot V^2)$ onde $V = 2N$ (total de nós após a transformação) e $E$ é o número de conexões originais.
- **Espaço:** $\mathcal{O}(V^2)$ para armazenar a matriz de capacidades e as listas de adjacência.
    

**Decisões Técnicas**
- **Divisão de Nós (Node Splitting):** Permite converter a limitação imposta aos vértices (expor a mão silenciosa) em uma limitação de arestas, possibilitando o uso seguro de algoritmos clássicos de fluxo em redes.
- **DFS para Augmentação:** Implementação 100% nativa sem imports externos, focada em manter o código simples, leve e livre de dependências.
- **Capacidade Infinita para Links Originais:** Garante que o gargalo do fluxo (e consequentemente o corte) ocorra única e exclusivamente nas arestas internas dos nós intermediários, respeitando a premissa de que a rede só se rompe ao "queimar" as mãos silenciosas, e não as rotas em si.
    

**Fluxo de Execução**
```text
Input (N, E, S, T + linhas de conexões) -> Constrói o Grafo Transformado (Divisão de Nós) -> Executa Ford-Fulkerson (Max Flow / Min Cut) -> Output (Número mínimo de mãos expostas)
```

flag: HTB{0n3_c00rd1n4t3d_m0m3nt}
