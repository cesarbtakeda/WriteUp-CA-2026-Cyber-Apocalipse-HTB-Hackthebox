### TollSchedule.md
## Resolvido por Dozz

```python
lines = []
while True:
    try:
        line = input()
        lines.append(line)
    except EOFError:
        break

# Parse the input
# First line: N and G
n, g = map(int, lines[0].split())

# Extract N arrival times - G clearance opening times 
arrivals = list(map(int, lines[1].split())) 
clearances = list(map(int, lines[2].split()))

# Sort both arrays
arrivals.sort()
clearances.sort()

# Greedy algorithm
total_wait = 0
clearance_idx = 0

for arrival in arrivals:
    # Find the first clearance that opens at or after arrival
    while clearances[clearance_idx] < arrival:
        clearance_idx += 1
    
    # Assign this clearance to the convoy
    total_wait += clearances[clearance_idx] - arrival
    clearance_idx += 1

print(total_wait)
```

**Estrategia usando algoritmo greedy:**
-  1. Ordenar chegadas das caravanas (crescente)
-  2. Ordenar aberturas das autorizações (crescente)
-  3. Para cada caravana (na ordem), encontrar a primeira autorização disponível que abre depois da chegada
-  4. Calcular e somar a espera

**Solução**
- Ordenar ambos garante que estamos fazendo o matching mais eficiente
- Atribuir a autorização mais cedo possível minimiza o tempo de espera de cada caravana
- É ótimo porque qualquer troca (swap) entre duas caravanas aumentaria ou manteria o tempo total

**Complexidade**
- Tempo: O(N log N + G log G) devido à ordenação
- Espaço: O(1) além dos arrays de entrada

**Exemplo Visual**
```text
Chegadas:  [2, 4, 6, 9]  → ordenadas
Aberturas: [3, 5, 8, 11] → ordenadas

2 → 3 (espera 1)
4 → 5 (espera 1)
6 → 8 (espera 2)
9 → 11 (espera 2)
Total = 6

```

flag: HTB{th3_p4ss_pr3f3rs_0n3_b4nn3r}

