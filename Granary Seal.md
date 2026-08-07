
```python

lines = []
while True:
    try:
        line = input()
        lines.append(line)
    except EOFError:
        break

n = lines
idx = 0

# Get C (clerks count)
c = int(n[idx])
idx += 1
clerks = set(n[idx:idx+c])
idx += c

# Get CS (countersigners count)
cs = int(n[idx])
idx += 1
countersigners = set(n[idx:idx+cs])
idx += cs

# Get R (couriers count)
r = int(n[idx])
idx += 1
couriers = set(n[idx:idx+r])
idx += r

# Get N (orders count)
n_orders = int(n[idx])
idx += 1

# Process orders
valid_orders = 0

for i in range(n_orders):
    # Get the order line
    order_line = n[idx + i]
    
    # Split into 3 names: clerk, countersigner, courier
    parts = order_line.split()
    
    if len(parts) != 3:
        continue
    
    clerk_name, countersigner_name, courier_name = parts
    
    # Check if all three are in their respective sets
    if (clerk_name in clerks and 
        countersigner_name in countersigners and 
        courier_name in couriers):
        valid_orders += 1

print(valid_orders)
```

**Complexidade**
Tempo: O(C + CS + R + N) onde N é o número de ordens
Espaço: O(C + CS + R) para armazenar os sets

**Decisões Técnicas**
set em vez de list para busca: Busca O(1) vs O(n)
Ponteiro idx em vez de fórmulas: Código mais legível e menos propenso a erro
Loop de leitura com input(): Evita importações e funciona em qualquer ambiente
Slicing de listas: lines[idx:idx+c] é mais rápido que loop manual

**Fluxo de Execução**

```text
Input → Lista de linhas → Ponteiro idx caminha → 
→ Converte números e coleta nomes em sets → 
→ Verifica cada ordem contra os sets → 
→ Conta válidas → Output
```

flag: HTB{th3_0ld_s3qu3nc3_n3v3r_h3s1t4t3s}