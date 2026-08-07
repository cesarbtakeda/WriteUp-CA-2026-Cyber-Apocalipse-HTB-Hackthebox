# The-Ledger-Beneath-the-Hull
## Resolvida por JoaoV1t


```
Lord Damas Marrowcairn controla o navio ASHEN MERCY através de várias empresas. O objetivo é usar os registros do painel para identificar o proprietário, o gerente, o operador, o afretador e a empresa que está por trás de toda a estrutura.
```

```
Primeiro, no Maritime Registry, pesquisamos pelo IMO 9724418 ou pelo arquivo OIR-9724418-2026. O registro mostra que a Thirteenth Tide Shipping Ltd é a proprietária do navio e a Morrow Fleet Management SA é responsável pela gestão técnica.
```

<img width="1906" height="680" alt="image" src="https://github.com/user-attachments/assets/817bc234-d987-4fa2-bf04-a8d9e2c66db3" />

```
Aqui já encontramos as duas primeiras empresas. Depois, pesquisando o mesmo IMO no P&I Directory, aparecem as outras partes envolvidas:
```
```bash

Registered Owner: Thirteenth Tide Shipping Ltd
ISM Manager: Morrow Fleet Management SA
Commercial Operator: Eastreach Maritime Coordination PLC
Time Charterer: Gilded Knife Commodities Ltd
```
```
Agora é só pesquisar essas empresas no Companies Register e seguir os campos de acionista e empresa controladora. A Eastreach Maritime Coordination PLC possui Damas Marrowcairn como diretor, e tanto ela quanto a Gilded Knife Commodities Ltd pertencem à Marrowcairn Strategic Holdings PLC.
```

```
Com isso, as respostas do formulário ficam:
```
```bash
Thirteenth Tide Shipping Ltd
Morrow Fleet Management SA
Eastreach Maritime Coordination PLC
Gilded Knife Commodities Ltd
Marrowcairn Strategic Holdings PLC
```
<img width="932" height="742" alt="image" src="https://github.com/user-attachments/assets/96feccba-ad9b-4933-836f-50632270cd28" />

<img width="845" height="636" alt="image" src="https://github.com/user-attachments/assets/94176420-7117-4724-81e8-7e878e189268" />

<img width="845" height="636" alt="image" src="https://github.com/user-attachments/assets/8a20c479-88c1-41bf-bdff-1fb317c75e6a" />

```
Depois de acertar as cinco respostas, o painel mostra a frase:
```

```The ink has been divided among five names.```

```
Agora e só transformar a frase no formato da flag:
```

```HTB{THE_INK_WAS_DIVIDED_AMONG_FIVE_NAMES}```

