# SecondStamp — Security Audit Report
## Resolvido por Dozz

**Target**: SecondStamp (Sui Move CTF Challenge)  
**Blockchain**: Sui (Move)  
**Scope**: `setup.move`, `sharehouse` (v1/v2/v3), `old_counter`, `travel_counter`, `witness_oracle`, `claim_marks`  
**Date**: 2026-07-25

---

## 1. Condições de Vitória (`is_solved`)

**Arquivo**: `sources/setup.move` L102-L124

```move
public fun is_solved(challenge, buffer_a, buffer_b, fee_a, fee_b,
    old_counter_a, old_counter_b, travel_counter_a, travel_counter_b, ctx): bool {
    ctx.sender() == challenge.player
        && challenge.claimed
        && within_residual_a(buffer_a)       // <= 25_000_000_000_000_000
        && within_residual_b(buffer_b)       // <= 100_000_000
        && within_residual_a(fee_a)          // <= 25_000_000_000_000_000
        && within_residual_b(fee_b)          // <= 100_000_000
        && within_residual_a(old_counter_a)  // <= 25_000_000_000_000_000
        && within_residual_b(old_counter_b)  // <= 100_000_000
        && within_residual_a(travel_counter_a) // <= 25_000_000_000_000_000
        && within_residual_b(travel_counter_b) // <= 100_000_000
}
```

### Resumo dos Requisitos

| Condição | Threshold |
|---|---|
| `sender == player` | Autenticação via chave fornecida |
| `claimed == true` | Chamar `setup::claim()` primeiro |
| `buffer (PALE_WAX)` | ≤ 25e15 (≈2.5% do seed de 1e18) |
| `buffer (GOLD_FLECK)` | ≤ 100e6 (≈10% do seed de 1e9) |
| `fee (PALE_WAX)` | ≤ 25e15 |
| `fee (GOLD_FLECK)` | ≤ 100e6 |
| `old_counter (PALE_WAX)` | ≤ 25e15 |
| `old_counter (GOLD_FLECK)` | ≤ 100e6 |
| `travel_counter (PALE_WAX)` | ≤ 25e15 |
| `travel_counter (GOLD_FLECK)` | ≤ 100e6 |

**Objetivo**: Drenar quase todos os fundos (PALE_WAX + GOLD_FLECK) de **todos** os compartimentos do sistema: buffer, fees/rewards dos pools, old_counter pool e travel_counter pool — restando apenas resíduo tolerável.

---

## 2. Arquitetura do Sistema

```
DisputedMark (setup.move)
  ├── pale_wax_treasury   → Minta PALE_WAX (18 decimais)
  ├── gold_fleck_treasury → Minta GOLD_FLECK (6 decimais)
  └── claim_mark_treasury → Minta CLAIM_MARK (LP token, 9 decimais)

Sharehouse (v1→v2→v3, upgradeable)
  ├── base_buffer (PALE_WAX)
  ├── quote_buffer (GOLD_FLECK)
  ├── old_counter_position → old_counter::pool::Position
  ├── travel_counter_position (v3 DOF) → travel_counter::pool::Position
  ├── protocol_fee_base / protocol_fee_quote
  ├── claim_mark_treasury (LP minting/burning)
  └── oracle_id → witness_oracle::Oracle

GlobalConfig
  ├── ACL (player = KEEPER role)
  ├── protocol_fee_bps = 0
  └── withdraw_fee_bps = 0

Versioned (version = 1)
```

### Seed de Deploy

| Pool | PALE_WAX | GOLD_FLECK |
|---|---|---|
| old_counter | 1e18 | 2.5e9 |
| travel_counter | 14e18 | 55e9 |
| Sharehouse buffer | 20e15 | 1e6 |
| Player (claim) | 1e15 | 1e6 |

---

## 3. Vulnerabilidades Encontradas

---

### VULN-01: Hardcoded `sqrt_price_q64` no Cálculo de AUM (Oracle Manipulation — CRITICAL)

**Arquivo**: `v3/sources/sharehouse/accounting.move` L26-L32

```move
let (travel_counter_base, travel_counter_quote) = travel_counter::pool::amounts_from_liquidity(
    travel_counter::pool::liquidity(travel_counter_position),
    travel_counter::pool::sqrt_lower_q64(travel_counter_position),
    36_893_488_147_419_103_232,  // <-- HARDCODED sqrt_price_q64
    travel_counter::pool::sqrt_upper_q64(travel_counter_position),
);
```

**Impacto**: O cálculo do AUM do travel_counter usa um `sqrt_price_q64` fixo em vez de ler o preço real do pool. Isso cria uma **discrepância entre o AUM reportado e o valor real dos ativos** no travel_counter pool.

O valor `36_893_488_147_419_103_232` = `2 * Q64` (onde Q64 = 18_446_744_073_709_551_616), que é exatamente o `sqrt_price_q64` inicial do pool. Porém o preço do pool pode ser alterado via `set_sqrt_price` (se tiver admin cap) ou movimentações de liquidez.

**Consequência**: Ao alterar o preço real do travel_counter pool (via withdraw/deposit de liquidez), o AUM calculado fica desatualizado/incorreto, permitindo:
1. **Depositar** com AUM inflado → receber mais LP tokens do que deveria
2. **Sacar** com AUM deflado → drenar mais ativos proporcionalmente

---

### VULN-02: Flash Loan sem Verificação de Reentrada no AUM (CRITICAL)

**Arquivo**: `v3/sources/sharehouse/flash.move` L21-L38

O flash loan empresta PALE_WAX do buffer, mas o `repay_flash_loan_base` **apenas verifica se o repagamento ≥ principal + fee** e retorna os fundos ao buffer. Não há verificação de que o AUM esteja consistente durante o empréstimo.

**Vetor de ataque**: 
1. Flash loan → toma 20e15 PALE_WAX do buffer
2. Dentro da mesma tx, usa esses fundos para manipular pools
3. Refresh AUM com pool manipulado → AUM distorcido
4. Deposita com AUM favorável → recebe LP inflado
5. Repaga o flash loan
6. Saca LP inflado para drenar fundos

**Nota**: Flash loan requer `KEEPER` role. O player **já é KEEPER** (config.move L56).

---

### VULN-03: Player como KEEPER — Acesso Privilegiado (HIGH)

**Arquivo**: `v1/sources/config.move` L54-L66

```move
public fun new(player: address, ctx: &mut TxContext): GlobalConfig {
    let mut acl = acl::new();
    acl::set_roles(&mut acl, player, 1 << ROLE_KEEPER);  // player = KEEPER
    ...
}
```

O player recebe automaticamente a role `KEEPER`, que permite:
- Executar flash loans (`flash::flash_loan_base` verifica `check_keeper`)
- Potencialmente outras operações keeper

Isso **não é um bug per se** (é intencional no CTF), mas é um enabler para explorar o flash loan.

---

### VULN-04: `attach_travel_counter_position` sem Controle de Acesso (HIGH)

**Arquivo**: `v3/sources/sharehouse.move` L126-L129

```move
public fun attach_travel_counter_position(house: &mut Sharehouse, position: travel_counter::pool::Position) {
    assert!(!dynamic_object_field::exists_with_type<...>(...), ETravelCounterAlreadyAttached);
    dynamic_object_field::add(&mut house.id, TRAVEL_COUNTER_POSITION_KEY, position);
}
```

**Qualquer pessoa** pode chamar essa função (é `public fun`, sem verificação de ACL ou AdminCap). A única proteção é que só pode ser chamada uma vez (assert de já existir). No deploy, o bootstrapper já chama, então isso fecha a janela. Porém indica design de permissão fraco.

---

### VULN-05: AUM Refresh sem Rate Limiting (MEDIUM)

**Arquivo**: `v3/sources/sharehouse/accounting.move` L37-L50

```move
public fun refresh_aum(house, versioned, old_counter_pool, oracle, clock): u128 {
    versioned::assert_supported(versioned);
    let aum = calculate_aum(house, old_counter_pool, oracle, clock);
    sharehouse::set_last_aum(house, aum);
    ...
}
```

Qualquer pessoa pode chamar `refresh_aum` a qualquer momento, sem cooldown ou verificação de role. Isso permite manipular o AUM livremente entre operações de deposit/withdraw.

---

### VULN-06: Precisão Truncada no `base_value_in_quote` v1/v2 (MEDIUM)

**Arquivo**: `v1/sources/sharehouse/math.move` L14-L16

```move
public(package) fun base_value_in_quote(amount: u64, price_e6: u64): u128 {
    ((amount / BASE_TO_QUOTE_DECIMAL_FACTOR) as u128) * (price_e6 as u128) / QUOTE_PRICE_SCALE
}
```

A **divisão inteira** `amount / 1_000_000_000_000` **antes** da multiplicação perde até `~1e12` unidades de precisão por operação. V3 corrige isso parcialmente com `base_value_in_quote_q18_floor` que divide por `1e18` **depois** da multiplicação, mas o v3 `calculate_aum` continua usando o novo cálculo enquanto `deposit` no v1/v2 usa o truncado.

**Impacto**: Depósitos/saques em v1/v2 podem ter arredondamento favorável ao atacante.

---

### VULN-07: Withdraw sem Verificação de AUM Atualizado (MEDIUM)

**Arquivo**: `v3/sources/sharehouse/withdraw.move` L150-L160

O `new_receipt` verifica `sharehouse::last_aum(house) > 0` mas **não verifica se o AUM foi recalculado recentemente**. Um atacante pode:
1. Refresh AUM em estado normal
2. Manipular pool(s) para alterar o valor real
3. Sacar usando o AUM antigo (stale) como referência

---

### VULN-08: `old_counter::pool::collect_position_fee/reward` sem Autenticação (MEDIUM)

**Arquivo**: `old_counter/sources/pool.move` L282-L316

```move
public fun collect_position_fee(pool, position, ctx): (Coin, Coin) {
    assert!(position.pool_id == object::id(pool), EWrongPool);
    // Nenhuma verificação de ownership da position!
    let amount_a = pool.fee_a;
    ...
}
```

Não há verificação de quem é o dono da position. Qualquer pessoa com referência mútua à position (via o Sharehouse shared object) pode coletar fees/rewards. No contexto do CTF, o withdraw flow expõe isso — o player pode drenar fees/rewards do pool.

Mesmo padrão em `travel_counter::pool`.

---

### VULN-09: Rounding em `remove_liquidity_by_percent` (LOW)

**Arquivo**: `old_counter/sources/pool.move` L255-L280

```move
let amount_a = mul_div_floor(pool.reserve_a.value(), numerator, denominator);
let amount_b = mul_div_floor(pool.reserve_b.value(), numerator, denominator);
```

Remove liquidez proporcional às **reserves totais do pool**, não ao principal da position. Isso significa que se o pool acumulou fees/rewards nas reserves, o saque retira mais do que o proporcional da position do LP.

---

### VULN-10: `travel_counter::pool::remove_share` — Mesmo padrão de VULN-09 (LOW)

**Arquivo**: `travel_counter/sources/pool.move` L217-L237

```move
let amount_a = mul_div_u64(pool.reserve_a.value(), numerator, denominator);
let amount_b = mul_div_u64(pool.reserve_b.value(), numerator, denominator);
```

Remove das reserves totais, não só do proporcional de liquidez da position.

---

## 4. Fluxo de Ataque (Estratégia de Solve)

### Pré-Requisitos
1. Restart a instância → obter credenciais frescas
2. `setup::claim()` → recebe 1e15 PALE_WAX + 1e6 GOLD_FLECK, seta `claimed = true`

### Fase 1: Refresh AUM Inicial
- Chamar `accounting::refresh_aum()` com o oracle e old_counter pool para inicializar o `last_aum > 0`

### Fase 2: Deposit Inicial
- Depositar tokens do player na Sharehouse via `accounting::deposit()` → receber LP tokens (CLAIM_MARK)

### Fase 3: Flash Loan + Manipulação de AUM
1. `flash::flash_loan_base()` → tomar PALE_WAX do buffer (player é KEEPER)
2. Usar os fundos para manipular a posição/pool (ex: adicionar liquidez no old_counter para inflar AUM)
3. `refresh_aum()` → AUM inflado é registrado
4. Depositar com AUM inflado → receber LP tokens a preço favorável
5. Restaurar pool, `refresh_aum()` → AUM normal
6. Repagar flash loan

### Fase 4: Withdraw Massivo
- Usar LP tokens acumulados para `withdraw()` ou `new_withdraw_cert()` → drenar:
  - old_counter pool (via `process_old_counter`)
  - travel_counter pool (via `withdraw_travel_counter`)
  - Fees (via `collect_position_fees`)
  - Rewards (via `collect_position_rewards`)
  - Buffer (via `process_buffer`)

### Fase 5: Verificação
- Todos os compartimentos devem ter saldos residuais ≤ limites de `is_solved`
- Chamar check endpoint: `POST /api/instance/check`

---

## 5. Superfície de Ataque — Funções Públicas Disponíveis ao Player

| Módulo | Função | Acesso |
|---|---|---|
| `setup` | `claim()` | player only |
| `accounting` | `refresh_aum()` | public (qualquer) |
| `accounting` | `deposit()` | public (qualquer, se deposits enabled) |
| `withdraw` | `begin_withdraw()` / `withdraw()` | public |
| `withdraw` | `new_withdraw_cert()` | public |
| `withdraw` | `withdraw_travel_counter()` | public (profile 3) |
| `flash` | `flash_loan_base()` | KEEPER (= player) |
| `flash` | `repay_flash_loan_base()` | public (com receipt) |
| `sharehouse` | `attach_travel_counter_position()` | public (1x) |
| `old_counter` | `remove_liquidity_by_percent()` | public |
| `old_counter` | `collect_position_fee()` | public |
| `old_counter` | `collect_position_reward()` | public |
| `travel_counter` | `decrease_liquidity()` | public |
| `travel_counter` | `remove_share()` | public |
| `travel_counter` | `collect_position_fee()` | public |
| `travel_counter` | `collect_position_reward()` | public |

---

## 6. Constantes Relevantes

| Constante | Valor | Contexto |
|---|---|---|
| `RESIDUAL_LIMIT_A` | 25_000_000_000_000_000 (25e15) | Max PALE_WAX residual |
| `RESIDUAL_LIMIT_B` | 100_000_000 (100e6) | Max GOLD_FLECK residual |
| `FLASH_LOAN_FACILITY_BASE` | 20_000_000_000_000_000 (20e15) | Flash loan fixo |
| `FLASH_LOAN_FEE_BPS` | 9 (0.09%) | Fee do flash loan |
| `REFERENCE_PRICE_E6` | 2_500_000_000 (2500e6) | Preço referência |
| Oracle price_e6 inicial | 2_500_000_000 | 1 WAX = 2500 GOLD (6 dec) |
| `SHAREHOUSE_BUFFER_WAX` | 20_000_000_000_000_000 (20e15) | Buffer PALE_WAX |
| `SHAREHOUSE_BUFFER_GOLD` | 1_000_000 (1e6) | Buffer GOLD_FLECK |

---

## 7. Resumo de Severity

| ID | Vulnerabilidade | Severidade | Exploitável |
|---|---|---|---|
| VULN-01 | Hardcoded sqrt_price no AUM | 🔴 CRITICAL | ✅ Sim |
| VULN-02 | Flash Loan → AUM Manipulation | 🔴 CRITICAL | ✅ Sim |
| VULN-03 | Player = KEEPER | 🟠 HIGH | ✅ Enabler |
| VULN-04 | attach_travel sem ACL | 🟠 HIGH | ⚠️ Janela fechada |
| VULN-05 | refresh_aum sem rate limit | 🟡 MEDIUM | ✅ Sim |
| VULN-06 | Truncamento base_value_in_quote | 🟡 MEDIUM | ⚠️ Parcial |
| VULN-07 | Withdraw com AUM stale | 🟡 MEDIUM | ✅ Sim |
| VULN-08 | Fee/reward collect sem auth | 🟡 MEDIUM | ✅ Sim |
| VULN-09 | Rounding remove_liquidity | 🟢 LOW | ⚠️ Minor |
| VULN-10 | Rounding remove_share | 🟢 LOW | ⚠️ Minor |


---
notas:
1. Por que estava dando Timeout?
O comando sui client quando executado pela primeira vez (ou se não encontrar um arquivo de configuração no caminho padrão ~/.sui/sui_config/client.yaml) pergunta de forma interativa no terminal: No sui config found in /home/buzzeto/.sui/sui_config/client.yaml, create one [Y/n]?

Como o script Python utiliza o subprocess.run(capture_output=True) (que roda de forma não-interativa e sem receber dados do teclado), o executável do Sui CLI ficava esperando indefinidamente por uma resposta no canal de entrada padrão (stdin), travando a execução e estourando o timeout de 30 segundos definido no script.

2. Como testar via CLI antes de executar o script?
Já configurei o seu ambiente global do Sui CLI e importei a chave privada ativa atual.

Para testar a chamada via CLI, execute o seguinte comando no terminal (ele faz um dry-run da função de claim):

```bash
sui client ptb \
  --move-call 0x76fcfef1d7fbda1bc8a81a2200e9ac1db1add62be3d0fb7d71c1f02bba6988cc::setup::claim @0x439e00ce1320698191099b902ca04f6fc0c72584e56901693e6700d3e0f8000a \
  --assign coins \
  --transfer-objects "[coins.0, coins.1]" @0xa6a9ce3fba4f202aa510e5d870e76b54cda3ac2b044c374c52a91f65179d8f42 \
  --dry-run
```
Como testar na CLI com a nova instância ativa:
Assim que você iniciar uma nova instância do lab, você poderá testar o comando equivalente diretamente na CLI usando a nova configuração isolada:

bash
# 1. Defina a pasta de config na sua sessão do terminal
export SUI_CONFIG_DIR="/home/buzzeto/HACK/CTF's/Evento/Coding/blockchain/SecondStamp/payload/.sui_config"
# 2. Rode o dry-run da PTB do claim usando os novos IDs gerados (substitua o Package e Challenge se necessário)



sui client ptb \
  --move-call <PACKAGE>::setup::claim @<CHALLENGE_OBJECT> \
  --assign coins \
  --transfer-objects "[coins.0, coins.1]" @<DESTINATION_ADDRESS> \
  --dry-run

sui client ptb \
  --move-call 0x54072d75bfe3aabdcd224962ff9d8ea0be7723d1967bb9949ad5314433d4ce24::setup::claim @0xf50b0d5608623fcd365526b299ef1b15b56031b10960003dc4648ac6263ce68d \
  --assign coins \
  --transfer-objects "[coins.0, coins.1]" @0x8ca3e49d0cb96a3534ca51177630b60ccd0452f053b4097e391bee57b29adaa0 \
  --dry-run
