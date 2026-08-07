# ForcePush
## Resolvido por Cesar B

```bash
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# ls                       
CHANGELOG.md            dangling_blobs.txt    docs      pyproject.toml  requirements-dev.txt  tests
CONTRIBUTING.md         dangling_commits.txt  examples  pytest.ini      scripts               tox.ini
crownspire              dangling_diffs.txt    LICENSE   README.md       SECURITY.md
dangling_blob_hits.txt  deploy.sh             Makefile  reflog.txt      setup.cfg
 ```
```                                                                                                                             
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# git fsck \
  --full \
  --no-reflogs \
  --unreachable \
  --lost-found 2>&1 |
tee fsck.txt
unreachable commit 3c8803d7146cd07c75325d6b555116200f2569ee
unreachable blob 12b14971d38c09ee73fed80613951dfdd3562291
unreachable tree 1fe5a4a75df03ff80a3743b93b8df188ee48c06c
```
```                                                                                                                 
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# grep -E \
'^(unreachable|dangling) (commit|blob|tree|tag) ' \
fsck.txt |
sort
unreachable blob 12b14971d38c09ee73fed80613951dfdd3562291
unreachable commit 3c8803d7146cd07c75325d6b555116200f2569ee
unreachable tree 1fe5a4a75df03ff80a3743b93b8df188ee48c06c
```
```                                                                                                                              
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# awk '
($1 == "unreachable" || $1 == "dangling") &&
$2 == "commit" {
    print $3
}' fsck.txt |
sort -u |
tee hidden_commits.txt
3c8803d7146cd07c75325d6b555116200f2569ee
```

```                                                                                                                              
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# wc -l hidden_commits.txt
1 hidden_commits.txt
```

```                                                                                                                              
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# while read -r commit; do
    git show -s \
      --date=iso \
      --format='%H | %ad | %an <%ae> | %s' \
      "$commit"
done < hidden_commits.txt |
sort -k2 |
tee hidden_commit_log.txt
3c8803d7146cd07c75325d6b555116200f2569ee | 2026-05-27 23:47:19 +0000 | Doran Ash <d.ash@crownspire.valyssar> | temp: add reliquary.creds to debug 403 on manifest push (REVERT ME)
```

                                                                                                                              
```                                                                                                                              
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# while read -r commit; do
    echo
    echo "================ $commit ================"

    git show \
      --format=fuller \
      --find-renames \
      --find-copies \
      "$commit"
done < hidden_commits.txt |
tee hidden_diffs.txt

================ 3c8803d7146cd07c75325d6b555116200f2569ee ================
commit 3c8803d7146cd07c75325d6b555116200f2569ee
Author:     Doran Ash <d.ash@crownspire.valyssar>
AuthorDate: Wed May 27 23:47:19 2026 +0000
Commit:     Doran Ash <d.ash@crownspire.valyssar>
CommitDate: Wed May 27 23:47:19 2026 +0000

    temp: add reliquary.creds to debug 403 on manifest push (REVERT ME)

diff --git a/reliquary.creds b/reliquary.creds
new file mode 100644
index 0000000..12b1497
--- /dev/null
+++ b/reliquary.creds
@@ -0,0 +1,6 @@
+# Crownspire reliquary -- production warden's key. DO NOT COMMIT.
+RELIQUARY_ENDPOINT=https://reliquary.crownspire.valyssar:9000
+RELIQUARY_BUCKET=crownspire-reliquary-prod
+AWS_ACCESS_KEY_ID=AKIACROWNSPIRE7WARD3N
+AWS_SECRET_ACCESS_KEY=HTB{th3_r3l1qu4ry_n3v3r_f0rg3ts}
+WARDEN_SIGNING_KEY=astrael-relic-sigil-2f9c
 ```
```                                                                                                                             
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# grep -aEin -C 12 \
'PRIVATE KEY|OPENSSH|BEGIN RSA|BEGIN EC|BEGIN PRIVATE|warden|reliquary|production|prod|secret|token|HTB\{' \
hidden_diffs.txt
1-
2-================ 3c8803d7146cd07c75325d6b555116200f2569ee ================
3-commit 3c8803d7146cd07c75325d6b555116200f2569ee
4-Author:     Doran Ash <d.ash@crownspire.valyssar>
5-AuthorDate: Wed May 27 23:47:19 2026 +0000
6-Commit:     Doran Ash <d.ash@crownspire.valyssar>
7-CommitDate: Wed May 27 23:47:19 2026 +0000
8-
9:    temp: add reliquary.creds to debug 403 on manifest push (REVERT ME)
10-
11:diff --git a/reliquary.creds b/reliquary.creds
12-new file mode 100644
13-index 0000000..12b1497
14---- /dev/null
15:+++ b/reliquary.creds
16-@@ -0,0 +1,6 @@
17:+# Crownspire reliquary -- production warden's key. DO NOT COMMIT.
18:+RELIQUARY_ENDPOINT=https://reliquary.crownspire.valyssar:9000
19:+RELIQUARY_BUCKET=crownspire-reliquary-prod
20-+AWS_ACCESS_KEY_ID=AKIACROWNSPIRE7WARD3N
21:+AWS_SECRET_ACCESS_KEY=HTB{th3_r3l1qu4ry_n3v3r_f0rg3ts}
22:+WARDEN_SIGNING_KEY=astrael-relic-sigil-2f9c
```
```                                                                                                                             
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# while read -r commit; do
    echo
    echo "========== $commit =========="

    git ls-tree -r --full-tree "$commit"
done < hidden_commits.txt |
tee hidden_trees.txt

========== 3c8803d7146cd07c75325d6b555116200f2569ee ==========
100644 blob 918e62e9fd29db665e9a4c5dca6fd458320d1fc8    .editorconfig
100644 blob 1ddc62ee296cafce8e9c802cf267dfa71bbc2f19    .gitattributes
100644 blob 968a09c0e4a728c6aa981b87ec1e4403dc2c2c52    .github/CODEOWNERS
100644 blob 4e3060d9a4952f38e5a6e9e80bb4aaf97eabba15    .github/ISSUE_TEMPLATE/bug_report.md
100644 blob 3cd83ad8651cd0bc96e651fefd711a135f3751e6    .github/PULL_REQUEST_TEMPLATE.md
100644 blob 3b18786a5ef13aa4aea642ee1ba72e7d53db9fcd    .github/workflows/ci.yml
100644 blob 9245fa17c67fa6d83c820beababb90421a7119c6    .github/workflows/deploy.yml
100644 blob 326c45df1f228f2a4702b6f8cbdd54687ff8000a    .gitignore
100644 blob 64d5a60932fdb441482211a2c05ff3d751eece47    CHANGELOG.md
100644 blob 67e869134e166f2c6b4f9e193e5cb9d17c164c1e    CONTRIBUTING.md
100644 blob 51392efd721155415dee662b6390713f4ae4dd16    LICENSE
100644 blob ec26575c15849943404768198cb65c551783bf11    Makefile
100644 blob 1c3e55c1c16b59f6ae0f9f270f0b126c423d4d03    README.md
100644 blob 0f7c37d45778676884a7be69c99a56b116f802ac    SECURITY.md
100644 blob cea1b7f0686826c35b96462e42dc28dc5194df2e    crownspire/__init__.py
100644 blob bfdcd0c1158dff4fcc25157b317419f0a89dd33f    crownspire/__main__.py
100644 blob b9067c2663dcc43dee6c3b7d7ccc82b7978dbc1b    crownspire/altar.py
100644 blob aeffcd876109691020300ff022f0c70534293cf2    crownspire/cli.py
100644 blob b4de25446d60fc49831915f28bd705419aa28e42    crownspire/config.py
100644 blob 29af7a0803167ba751f331f15ad82665085e00f1    crownspire/dotenv.py
100644 blob bfc54a537c53982b19411f9eb39e041d04ba7674    crownspire/errors.py
100644 blob 22176fd3161ae73f8daf7014bc99750c4c324dd0    crownspire/logconfig.py
100644 blob 88b4a753f17b2e3e1fbea6d9181770e21bab43ef    crownspire/manifest.py
100644 blob c42488e06d90809c3a60fa119b55812a5b3f1cc9    crownspire/reliquary.py
100644 blob c8ff25059e0cc6921a38a7403b5a2e702aeb9bbd    crownspire/signing.py
100644 blob ce0a3e162dc613c40b1f210e8ea8c17b0f75a866    crownspire/utils.py
100644 blob 1479c725608c9f770fba96172fb0ca1988116394    deploy.sh
100644 blob 2a6b6eea56370032b955f1ec0b1ce8f95ccc7d20    docs/architecture.md
100644 blob 645c99c4630722abb8b498102d9324d95f982e80    docs/deploy.md
100644 blob 960157cfed3783a859bd1d11560a4a4361e8a481    docs/manifest-format.md
100644 blob 80c15e015b33ded6799ba313dd6af620f4096b45    examples/binding-rite.json
100644 blob 7d517d4a6390a4ff380c0c2452681c818ed5af52    examples/dawn-rite.json
100644 blob 9d8778bec8196c9d4d46c74d3d8cd9724ef618ab    examples/ward-rite.json
100644 blob 88b15376b8930be6ff3b319dbf441878ed9e1a87    pyproject.toml
100644 blob 95df8d92f64026f8459cb11cb13361452d892ef5    pytest.ini
100644 blob 12b14971d38c09ee73fed80613951dfdd3562291    reliquary.creds
100644 blob 423a625813545bd7591102be9187b0d9b6117d63    requirements-dev.txt
100644 blob 85d1c9eccae80c2d0b4232b133241be4404897bb    scripts/new-manifest.py
100644 blob 8903865916d689d82b81d2c15d6911b234082197    scripts/rotate-key.sh
100644 blob 3939587ffb0e8a741cbb5399237d0d02171a1aa5    scripts/verify-all.sh
100644 blob 1e8863ebb821ffcb0f041d9ad3cb2495100c1c06    setup.cfg
100644 blob b9a3efeb09cdfe29e3b3dd5dd7dd098b7908135b    tests/conftest.py
100644 blob 039609b0d832f9e4db11b192c251b53cbc7f2b0a    tests/test_altar.py
100644 blob 1848e4d7d0a812ececa9b73be0c0b0422ea3cd6d    tests/test_cli.py
100644 blob a547a37e31f53fea73def57cfb27fe2adf6105ae    tests/test_config.py
100644 blob 74e91c8a89c94a52c0e5d21537af3f82a91974b1    tests/test_dotenv.py
100644 blob 67097b96fcbfa6748410d332b93ac651856698b4    tests/test_manifest.py
100644 blob cfb19755a507e1adae5a5f7e430bd4b64f5d4140    tests/test_reliquary.py
100644 blob 78fac98939d43a0331bfa3530f039ec954716fe9    tests/test_signing.py
100644 blob fe9d29ea27edd959a39a3300b606e420b89b009b    tests/test_utils.py
100644 blob 82cf02221ce0d9111e2db6c661d06740aad1cb75    tox.ini
```
```                                                                                                                              
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# grep -iE \
'key|pem|rsa|ssh|warden|reliquary|prod|secret|token|credential|env|vault|flag' \
hidden_trees.txt
100644 blob 29af7a0803167ba751f331f15ad82665085e00f1    crownspire/dotenv.py
100644 blob c42488e06d90809c3a60fa119b55812a5b3f1cc9    crownspire/reliquary.py
100644 blob 12b14971d38c09ee73fed80613951dfdd3562291    reliquary.creds
100644 blob 8903865916d689d82b81d2c15d6911b234082197    scripts/rotate-key.sh
100644 blob 74e91c8a89c94a52c0e5d21537af3f82a91974b1    tests/test_dotenv.py
100644 blob cfb19755a507e1adae5a5f7e430bd4b64f5d4140    tests/test_reliquary.py
```
```                                                                                                                              
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# mkdir -p recovered_blobs

git cat-file \
  --batch-all-objects \
  --batch-check='%(objectname) %(objecttype)' |
awk '$2 == "blob" {print $1}' |
sort -u > all_blobs.txt
```

 ```                                                                                                                             
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# TMP="$(mktemp)"

while read -r blob; do
    git cat-file blob "$blob" > "$TMP" 2>/dev/null || continue

    if grep -aEqi \
      -- '-----BEGIN [A-Z0-9 ]*PRIVATE KEY-----|OPENSSH PRIVATE KEY|HTB\{|warden|reliquary|Cinderbound|production[_ -]?key|WARDEN[_ -]?KEY' \
      "$TMP" 
    then
        cp "$TMP" "recovered_blobs/$blob"

        echo
        echo "========== BLOB $blob =========="

        grep -aEin -C 8 \
          -- '-----BEGIN [A-Z0-9 ]*PRIVATE KEY-----|OPENSSH PRIVATE KEY|HTB\{|warden|reliquary|Cinderbound|production[_ -]?key|WARDEN[_ -]?KEY' \
          "$TMP" 
    fi
done < all_blobs.txt |
tee blob_hits.txt

rm -f "$TMP"

========== BLOB 039609b0d832f9e4db11b192c251b53cbc7f2b0a ==========
1-import json
2-
3-from crownspire.altar import all_ok, verify_dir
4-from crownspire.signing import write_signature
5-
6:KEY = b"test-warden-key"
7-
8-
9-def _write_manifest(path, name="dawn-rite"):
10-    data = {
11-        "name": name,
12-        "realm": "crownspire",
13-        "revision": 1,
14-        "sigils": [{"id": "ember", "order": 1, "binding": "oath"}],

========== BLOB 07ad97512bbd8b89f3e3eb8ee264bdf4a7474d91 ==========
1:"""crownspire-deploy: sign and publish sigil manifests to the reliquary."""
2-
3-__version__ = "0.2.0"

========== BLOB 0f7c37d45778676884a7be69c99a56b116f802ac ==========
1-# Security policy
2-
3-## Reporting
4-
5:Found something? Mail the wardens at `security@crownspire.valyssar`. Do not open
6:a public issue for anything that touches the reliquary or the signing key.
7-
8-## Handling credentials
9-
10:- The warden's signing key and the reliquary access keys are **secrets**. They
11:  live in the warden's vault, in CI repository secrets, and (locally, for
12-  development only) in an untracked `.env`.
13-- `.env` and `*.creds` are gitignored. Never `git add -f` them.
14-- If a secret ever lands in a commit:
15-  1. **Rotate it immediately** -- assume it is burned the moment it is pushed.
16-  2. Scrub it from history (`git filter-repo` / BFG) and force-push.
17-  3. Remember that rewriting history does **not** delete the old objects until
18-     a `gc`/`prune` runs. A pushed secret must be rotated regardless.
19-

========== BLOB 12b14971d38c09ee73fed80613951dfdd3562291 ==========
1:# Crownspire reliquary -- production warden's key. DO NOT COMMIT.
2:RELIQUARY_ENDPOINT=https://reliquary.crownspire.valyssar:9000
3:RELIQUARY_BUCKET=crownspire-reliquary-prod
4-AWS_ACCESS_KEY_ID=AKIACROWNSPIRE7WARD3N
5:AWS_SECRET_ACCESS_KEY=HTB{th3_r3l1qu4ry_n3v3r_f0rg3ts}
6:WARDEN_SIGNING_KEY=astrael-relic-sigil-2f9c

========== BLOB 1479c725608c9f770fba96172fb0ca1988116394 ==========
1-#!/bin/sh
2:# publish signed sigil manifests to the Crownspire reliquary
3-set -e
4:echo "sealing manifests -> $RELIQUARY_BUCKET"
5:aws --endpoint-url "$RELIQUARY_ENDPOINT" s3 sync ./build "s3://$RELIQUARY_BUCKET/"

========== BLOB 1848e4d7d0a812ececa9b73be0c0b0422ea3cd6d ==========
12-    rc = main(["validate", "nope.json"])
13-    err = capsys.readouterr().err
14-    assert rc == 1
15-    assert "error:" in err
16-
17-
18-def test_sign_and_verify(manifest_file, monkeypatch, tmp_path, capsys):
19-    env = {
20:        "RELIQUARY_ENDPOINT": "https://x:9000",
21:        "RELIQUARY_BUCKET": "b",
22-        "AWS_ACCESS_KEY_ID": "k",
23-        "AWS_SECRET_ACCESS_KEY": "s",
24:        "WARDEN_SIGNING_KEY": "sigil-key",
25-    }
26-    for k, v in env.items():
27-        monkeypatch.setenv(k, v)
28-
29-    sig_path = str(tmp_path / "m.sig")
30-    assert main(["sign", manifest_file, "-o", sig_path]) == 0
31-    assert main(["verify", manifest_file, sig_path]) == 0

========== BLOB 1c3e55c1c16b59f6ae0f9f270f0b126c423d4d03 ==========
1-# crownspire-deploy
2-
3:Cinderbound tooling to sign and publish **sigil manifests** to the Crownspire
4:reliquary before a binding rite. The reliquary is an S3-compatible store; every
5:manifest is signed with the wardens' key so the priesthood can verify
6-provenance at the altar.
7-
8-## Install
9-
10-    python -m pip install -e ".[dev]"
11-
12-## Usage
13-
--
18-See `docs/manifest-format.md` for the schema and `docs/deploy.md` for the
19-deploy flow.
20-
21-## Credentials
22-
23-Credentials never live in the repo. In CI they come from the secret store
24-(see `.github/workflows/deploy.yml`); locally they come from an untracked
25-`.env` (copy `.env.example`). If you ever see a `.env` or `*.creds` file
26:tracked here, something has gone wrong -- rotate the warden's key immediately
27-and scrub it from history.

========== BLOB 218e2ebfe98ed7355527ad049466a40465b5795a ==========
1-"""Command-line entry point for crownspire-deploy.
2-
3-Subcommands:
4-
5-    crownspire validate <manifest>         check a manifest parses + validates
6-    crownspire sign <manifest> [-o SIG]    write a detached HMAC signature
7-    crownspire verify <manifest> <sig>     verify a manifest against a signature
8:    crownspire publish <build-dir>         sign + push signed manifests to the reliquary
9-
10-Credentials come from the environment (see crownspire.config).
11-"""
12-from __future__ import annotations
13-
14-import argparse
15-import os
16-import sys
17-from typing import List, Optional
18-
19-from . import __version__
20-from .config import Config
21-from .errors import CrownspireError
22-from .manifest import load_manifest
23:from .reliquary import Reliquary
24-from .signing import verify_manifest, write_signature
25-
26-
27-def _cmd_validate(args: argparse.Namespace) -> int:
28-    manifest = load_manifest(args.manifest)
29-    print(f"ok: {manifest.name} rev {manifest.revision} "
30-          f"({len(manifest.sigils)} sigils, realm {manifest.realm})")
31-    return 0
--
50-        print("signature OK")
51-        return 0
52-    print("signature MISMATCH", file=sys.stderr)
53-    return 2
54-
55-
56-def _cmd_publish(args: argparse.Namespace) -> int:
57-    config = Config.from_env()
58:    reliquary = Reliquary(config)
59-    # sign every manifest in the build dir, then sync the whole tree up
60-    signed = 0
61-    for name in sorted(os.listdir(args.build_dir)):
62-        if not name.endswith(".json"):
63-            continue
64-        path = os.path.join(args.build_dir, name)
65-        manifest = load_manifest(path)
66-        write_signature(manifest, config.signing_key_bytes, path + ".sig")
67-        signed += 1
68-    print(f"signed {signed} manifest(s), syncing -> {config.bucket}")
69:    reliquary.sync(args.build_dir, prefix=args.prefix)
70-    print("done")
71-    return 0
72-
73-
74-def build_parser() -> argparse.ArgumentParser:
75-    parser = argparse.ArgumentParser(prog="crownspire",
76:                                     description="Cinderbound reliquary deployer")
77-    parser.add_argument("--version", action="version",
78-                        version=f"crownspire {__version__}")
79-    sub = parser.add_subparsers(dest="command", required=True)
80-
81-    p_validate = sub.add_parser("validate", help="validate a manifest")
82-    p_validate.add_argument("manifest")
83-    p_validate.set_defaults(func=_cmd_validate)
84-

========== BLOB 2a6b6eea56370032b955f1ec0b1ce8f95ccc7d20 ==========
9-               |            bytes            |
10-               v                             v
11-        +-------------+              <manifest>.sig
12-        |   cli.py    |
13-        +------+------+
14-               | publish
15-               v
16-        +-------------+   aws s3 sync  +----------------+
17:        | reliquary.py| -------------> | Crownspire     |
18:        +-------------+                | reliquary (S3) |
19-                                       +----------------+
20-                                               |
21-                                        altar re-verifies
22-                                       (altar.py) before a rite
23-```
24-
25-## Modules
26-
27-| module          | responsibility                                            |
28-|-----------------|-----------------------------------------------------------|
29-| `manifest.py`   | model + validation + canonical byte serialization         |
30-| `signing.py`    | HMAC-SHA256 sign / verify / write detached signature       |
31-| `config.py`     | read + validate required environment                       |
32-| `dotenv.py`     | load a local `.env` for development                        |
33:| `reliquary.py`  | thin wrapper over the `aws` CLI (S3-compatible endpoint)   |
34-| `altar.py`      | re-verify a published set (the last gate before binding)   |
35-| `cli.py`        | argument parsing + command dispatch                        |
36-| `utils.py`      | small filesystem/hash helpers                              |
37-
38-## Why the canonical form
39-
40-Signing the raw file bytes would make a signature break every time someone
41-re-indented the JSON. Instead we sign a canonical representation (sorted keys,

========== BLOB 51392efd721155415dee662b6390713f4ae4dd16 ==========
1-MIT License
2-
3:Copyright (c) 2026 The Cinderbound, Crownspire
4-
5-Permission is hereby granted, free of charge, to any person obtaining a copy
6-of this software and associated documentation files (the "Software"), to deal
7-in the Software without restriction, including without limitation the rights
8-to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
9-copies of the Software, and to permit persons to whom the Software is
10-furnished to do so, subject to the following conditions:
11-

========== BLOB 612be6c0d6ca7be48fb18c525a52eac240f261a6 ==========
1-# crownspire-deploy
2-
3:Cinderbound tooling to sign and publish **sigil manifests** to the Crownspire
4:reliquary before a binding rite. The reliquary is an S3-compatible store; every
5:manifest is signed with the wardens' key so the priesthood can verify
6-provenance at the altar.
7-
8-## Install
9-
10-    python -m pip install -e ".[dev]"
11-
12-## Usage
13-
--
23-See `docs/manifest-format.md` for the schema, `docs/deploy.md` for the deploy
24-flow, and `docs/architecture.md` for how the pieces fit together.
25-
26-## Credentials
27-
28-Credentials never live in the repo. In CI they come from the secret store
29-(see `.github/workflows/deploy.yml`); locally they come from an untracked
30-`.env`. If you ever see a `.env` or `*.creds` file tracked here, something has
31:gone wrong -- rotate the warden's key immediately (`scripts/rotate-key.sh`) and
32-scrub it from history. See `SECURITY.md`.

========== BLOB 645c99c4630722abb8b498102d9324d95f982e80 ==========
1-# Deploying manifests
2-
3-## Local
4-
5:1. `cp .env.example .env` and fill in the warden's key material. **Never commit
6-   `.env`.** It is gitignored; keep it that way.
7-2. Build your manifests into `build/` (one `.json` per rite).
8-3. `python -m crownspire publish build/` -- signs each manifest and syncs the
9:   directory to the reliquary.
10-
11-## CI
12-
13-Tagging a release (`git tag v1.2.3 && git push --tags`) triggers
14-`.github/workflows/deploy.yml`. All credentials come from repository secrets:
15-
16-| secret                | maps to env                 |
17-|-----------------------|-----------------------------|
18:| `RELIQUARY_ENDPOINT`  | `RELIQUARY_ENDPOINT`        |
19:| `RELIQUARY_BUCKET`    | `RELIQUARY_BUCKET`          |
20:| `RELIQUARY_KEY_ID`    | `AWS_ACCESS_KEY_ID`         |
21:| `RELIQUARY_SECRET`    | `AWS_SECRET_ACCESS_KEY`     |
22:| `WARDEN_SIGNING_KEY`  | `WARDEN_SIGNING_KEY`        |
23-
24-## The 403 on first push
25-
26:After a warden key rotation the gateway returns `403` on the very first request
27-while it warms its cache. `deploy.sh` retries once after a short pause. If it
28-keeps failing, the key is genuinely wrong -- rotate again from the vault, do not
29-paste keys into the repo to test.

========== BLOB 64d5a60932fdb441482211a2c05ff3d751eece47 ==========
5-### Added
6-- `crownspire altar` re-verifies a published set the way the altar does
7-- local `.env` support via a tiny built-in dotenv loader (`--env-file`)
8-- `docs/architecture.md`, `SECURITY.md`, `CODEOWNERS`
9-- `scripts/rotate-key.sh` and an `ashmarch` example manifest
10-
11-### Changed
12-- `deploy.sh` loads credentials from the environment and retries once on a 403
13:- signing key and reliquary creds now come solely from CI secrets / local `.env`
14-
15-### Fixed
16:- `deploy.sh` used `RELIQUARY_URL`; corrected to `RELIQUARY_ENDPOINT`
17-
18-## [0.2.0] - 2026-05-24
19-
20-### Added
21-- CI workflow (lint + test matrix) and tag-triggered publish workflow
22-- `docs/manifest-format.md` and `docs/deploy.md`
23-- `scripts/verify-all.sh`, `scripts/new-manifest.py`
24-- example manifests under `examples/`
25-
26-## [0.1.0] - 2026-05-19
27-
28-### Added
29:- initial reliquary deployer skeleton
30-- manifest model with validation
31-- HMAC signing + detached signature files
32-- `crownspire` CLI (`validate`, `sign`, `verify`, `publish`)
33:- reliquary client over the `aws` CLI

========== BLOB 67e869134e166f2c6b4f9e193e5cb9d17c164c1e ==========
1-# Contributing
2-
3-Small repo, few rules, but please hold to them:
4-
5:1. **Never commit secrets.** Credentials live in the warden's vault, in CI
6-   secrets, or in your untracked `.env`. `.env` and `*.creds` are gitignored --
7-   do not `git add -f` them "just to test something".
8-2. Run `make lint test` before opening a PR.
9-3. Keep commits small and message them in the imperative ("add", "fix", not
10-   "added").
11-4. Manifest schema changes need a matching update to `docs/manifest-format.md`
12-   and a bump to the example manifests.
13-

========== BLOB 6e8d34dad2dd90d0130badc7a9d8f71884855aa2 ==========
1-# crownspire-deploy
2-
3:Cinderbound tooling to sign and publish **sigil manifests** to the Crownspire
4:reliquary before a binding rite.
5-
6-Early days -- see `docs/` as it fills in.

========== BLOB 700daa2ca7f3738a0d4b9ff54971916c8115cb10 ==========
1-#!/bin/sh
2:# publish signed sigil manifests to the Crownspire reliquary
3-set -e
4:echo "sealing manifests -> $RELIQUARY_BUCKET"
5:aws --endpoint-url "$RELIQUARY_URL" s3 sync ./build "s3://$RELIQUARY_BUCKET/"

========== BLOB 88b15376b8930be6ff3b319dbf441878ed9e1a87 ==========
1-[build-system]
2-requires = ["setuptools>=61.0"]
3-build-backend = "setuptools.build_meta"
4-
5-[project]
6-name = "crownspire-deploy"
7:description = "Cinderbound tooling to sign and publish sigil manifests to the reliquary"
8-readme = "README.md"
9-requires-python = ">=3.9"
10-license = { file = "LICENSE" }
11-authors = [
12-    { name = "Kestra Vayne", email = "k.vayne@crownspire.valyssar" },
13-]
14-dynamic = ["version"]
15-

========== BLOB 88b4a753f17b2e3e1fbea6d9181770e21bab43ef ==========
1-"""Sigil manifest model, loading, and validation.
2-
3:A *sigil manifest* is the little JSON document the Cinderbound publish to the
4:reliquary before a binding rite. It lists the sigils that make up a rite along
5-with the realm and a monotonic revision number. We keep the schema deliberately
6-small -- the priesthood edit these by hand.
7-"""
8-from __future__ import annotations
9-
10-import json
11-from dataclasses import dataclass, field
12-from typing import Any, Dict, List

========== BLOB 8903865916d689d82b81d2c15d6911b234082197 ==========
1-#!/bin/sh
2:# Rotate the warden's signing key.
3-#
4-# This does NOT touch the repo. It rotates the key in the vault and re-signs the
5-# currently published manifests so the altar keeps accepting them. Run it after
6-# any suspected key exposure -- a key is burned the instant it is pushed.
7-set -eu
8-
9:if [ -z "${WARDEN_SIGNING_KEY:-}" ]; then
10:    echo "WARDEN_SIGNING_KEY not set (source your .env or CI secrets)" >&2
11-    exit 1
12-fi
13-
14-build_dir="${1:-build}"
15-
16:echo "re-signing manifests in $build_dir with the current warden key"
17-for m in "$build_dir"/*.json; do
18-    [ -f "$m" ] || continue
19-    python -m crownspire sign "$m"
20-done
21-
22-echo "done. remember to publish the new signatures:"
23-echo "    python -m crownspire publish $build_dir"

========== BLOB 894501d1ce14d35635eaf8818442c8da9b29da69 ==========
1-# Changelog
2-
3-## [Unreleased]
4-
5-### Added
6:- initial reliquary deployer skeleton
7-- manifest model with validation
8-- HMAC signing + detached signature files
9-- `crownspire` CLI (`validate`, `sign`, `verify`, `publish`)
10:- reliquary client over the `aws` CLI

========== BLOB 9245fa17c67fa6d83c820beababb90421a7119c6 ==========
9-    steps:
10-      - uses: actions/checkout@v4
11-      - uses: actions/setup-python@v5
12-        with:
13-          python-version: "3.11"
14-      - run: python -m pip install -e .
15-      - name: publish signed manifests
16-        env:
17:          RELIQUARY_ENDPOINT: ${{ secrets.RELIQUARY_ENDPOINT }}
18:          RELIQUARY_BUCKET: ${{ secrets.RELIQUARY_BUCKET }}
19:          AWS_ACCESS_KEY_ID: ${{ secrets.RELIQUARY_KEY_ID }}
20:          AWS_SECRET_ACCESS_KEY: ${{ secrets.RELIQUARY_SECRET }}
21:          WARDEN_SIGNING_KEY: ${{ secrets.WARDEN_SIGNING_KEY }}
22-        run: |
23-          make sign
24-          ./deploy.sh prod

========== BLOB 960157cfed3783a859bd1d11560a4a4361e8a481 ==========
20-|------------|--------|---------------------------------------------------|
21-| `name`     | string | rite name, kebab-case                             |
22-| `realm`    | string | one of `valyssar`, `crownspire`, `ashmarch`       |
23-| `revision` | int    | monotonic, starts at 1                            |
24-| `sigils`   | array  | each has `id`, `order` (unique), `binding`        |
25-
26-## Signing
27-
28:The signature is `HMAC-SHA256(warden_key, canonical_bytes)` in hex, where
29-`canonical_bytes` is the manifest serialized with sorted keys and sigils sorted
30-by `order`. Because the canonical form is order-independent, re-formatting the
31-JSON on disk never changes the signature.
32-
33-Signatures are written to a detached `<manifest>.sig` next to the manifest and
34-uploaded alongside it. The altar refuses to bind a rite whose signature does
35-not verify.

========== BLOB 968a09c0e4a728c6aa981b87ec1e4403dc2c2c52 ==========
1-# default owner for everything
2-*               @kvayne
3-
4:# deploy + reliquary paths need a devops review
5-/deploy.sh      @dash
6:/crownspire/reliquary.py  @dash
7-/.github/       @dash
8-
9-# docs
10-/docs/          @btoll

========== BLOB a40d4b0be5e3a06725615c8de45496e9013e9939 ==========
1-# crownspire-deploy
2-
3:Cinderbound tooling to sign and publish **sigil manifests** to the Crownspire
4:reliquary before a binding rite. The reliquary is an S3-compatible store; every
5:manifest is signed with the wardens' key so the priesthood can verify
6-provenance at the altar.
7-
8-## Install
9-
10-    python -m pip install -e ".[dev]"
11-
12-## Usage
13-

========== BLOB a547a37e31f53fea73def57cfb27fe2adf6105ae ==========
1-import pytest
2-
3-from crownspire.config import Config
4-from crownspire.errors import ConfigError
5-
6-FULL_ENV = {
7:    "RELIQUARY_ENDPOINT": "https://reliquary.crownspire.valyssar:9000",
8:    "RELIQUARY_BUCKET": "crownspire-reliquary-prod",
9-    "AWS_ACCESS_KEY_ID": "AKIATEST",
10-    "AWS_SECRET_ACCESS_KEY": "shhh",
11:    "WARDEN_SIGNING_KEY": "sigil-key",
12-}
13-
14-
15-def test_from_env_ok():
16-    config = Config.from_env(FULL_ENV)
17:    assert config.bucket == "crownspire-reliquary-prod"
18-    assert config.signing_key_bytes == b"sigil-key"
19-
20-
21-def test_missing_one_var():
22-    env = dict(FULL_ENV)
23:    del env["WARDEN_SIGNING_KEY"]
24-    with pytest.raises(ConfigError) as exc:
25-        Config.from_env(env)
26:    assert "WARDEN_SIGNING_KEY" in str(exc.value)
27-
28-
29-def test_empty_var_counts_as_missing():
30-    env = dict(FULL_ENV, AWS_SECRET_ACCESS_KEY="")
31-    with pytest.raises(ConfigError):
32-        Config.from_env(env)

========== BLOB a875d5531b7a2d7b2219974775b6ee4c32251f7d ==========
5-### Added
6-- CI workflow (lint + test matrix) and tag-triggered publish workflow
7-- `docs/manifest-format.md` and `docs/deploy.md`
8-- `scripts/verify-all.sh`, `scripts/new-manifest.py`
9-- example manifests under `examples/`
10-
11-### Changed
12-- `deploy.sh` loads credentials from the environment and retries once on a 403
13:- signing key and reliquary creds now come solely from CI secrets / local `.env`
14-
15-### Fixed
16:- `deploy.sh` used `RELIQUARY_URL`; corrected to `RELIQUARY_ENDPOINT`
17-
18-## [0.1.0] - 2026-05-19
19-
20-### Added
21:- initial reliquary deployer skeleton
22-- manifest model with validation
23-- HMAC signing + detached signature files
24-- `crownspire` CLI (`validate`, `sign`, `verify`, `publish`)
25:- reliquary client over the `aws` CLI

========== BLOB aeffcd876109691020300ff022f0c70534293cf2 ==========
1-"""Command-line entry point for crownspire-deploy.
2-
3-Subcommands:
4-
5-    crownspire validate <manifest>         check a manifest parses + validates
6-    crownspire sign <manifest> [-o SIG]    write a detached HMAC signature
7-    crownspire verify <manifest> <sig>     verify a manifest against a signature
8:    crownspire publish <build-dir>         sign + push signed manifests to the reliquary
9-    crownspire altar <build-dir>           re-verify a published set the way the altar does
10-
11-Credentials come from the environment (see crownspire.config).
12-"""
13-from __future__ import annotations
14-
15-import argparse
16-import os
--
18-from typing import List, Optional
19-
20-from . import __version__
21-from .altar import all_ok, verify_dir
22-from .config import Config
23-from .dotenv import load_dotenv
24-from .errors import CrownspireError
25-from .manifest import load_manifest
26:from .reliquary import Reliquary
27-from .signing import verify_manifest, write_signature
28-
29-
30-def _cmd_validate(args: argparse.Namespace) -> int:
31-    manifest = load_manifest(args.manifest)
32-    print(f"ok: {manifest.name} rev {manifest.revision} "
33-          f"({len(manifest.sigils)} sigils, realm {manifest.realm})")
34-    return 0
--
53-        print("signature OK")
54-        return 0
55-    print("signature MISMATCH", file=sys.stderr)
56-    return 2
57-
58-
59-def _cmd_publish(args: argparse.Namespace) -> int:
60-    config = Config.from_env()
61:    reliquary = Reliquary(config)
62-    # sign every manifest in the build dir, then sync the whole tree up
63-    signed = 0
64-    for name in sorted(os.listdir(args.build_dir)):
65-        if not name.endswith(".json"):
66-            continue
67-        path = os.path.join(args.build_dir, name)
68-        manifest = load_manifest(path)
69-        write_signature(manifest, config.signing_key_bytes, path + ".sig")
70-        signed += 1
71-    print(f"signed {signed} manifest(s), syncing -> {config.bucket}")
72:    reliquary.sync(args.build_dir, prefix=args.prefix)
73-    print("done")
74-    return 0
75-
76-
77-def _cmd_altar(args: argparse.Namespace) -> int:
78-    config = Config.from_env()
79-    results = verify_dir(args.build_dir, config.signing_key_bytes)
80-    for r in results:
--
84-        print("altar: all manifests verified, rite may bind")
85-        return 0
86-    print("altar: refusing to bind, one or more manifests failed", file=sys.stderr)
87-    return 2
88-
89-
90-def build_parser() -> argparse.ArgumentParser:
91-    parser = argparse.ArgumentParser(prog="crownspire",
92:                                     description="Cinderbound reliquary deployer")
93-    parser.add_argument("--version", action="version",
94-                        version=f"crownspire {__version__}")
95-    parser.add_argument("--env-file", default=".env",
96-                        help="dotenv file to load before running (default: .env)")
97-    sub = parser.add_subparsers(dest="command", required=True)
98-
99-    p_validate = sub.add_parser("validate", help="validate a manifest")
100-    p_validate.add_argument("manifest")

========== BLOB b4de25446d60fc49831915f28bd705419aa28e42 ==========
9-
10-import os
11-from dataclasses import dataclass
12-from typing import Mapping, Optional
13-
14-from .errors import ConfigError
15-
16-REQUIRED_ENV = (
17:    "RELIQUARY_ENDPOINT",
18:    "RELIQUARY_BUCKET",
19-    "AWS_ACCESS_KEY_ID",
20-    "AWS_SECRET_ACCESS_KEY",
21:    "WARDEN_SIGNING_KEY",
22-)
23-
24-
25-@dataclass
26-class Config:
27-    endpoint: str
28-    bucket: str
29-    access_key_id: str
--
38-    def from_env(cls, env: Optional[Mapping[str, str]] = None) -> "Config":
39-        env = os.environ if env is None else env
40-        missing = [k for k in REQUIRED_ENV if not env.get(k)]
41-        if missing:
42-            raise ConfigError(
43-                "missing required environment variables: " + ", ".join(missing)
44-            )
45-        return cls(
46:            endpoint=env["RELIQUARY_ENDPOINT"],
47:            bucket=env["RELIQUARY_BUCKET"],
48-            access_key_id=env["AWS_ACCESS_KEY_ID"],
49-            secret_access_key=env["AWS_SECRET_ACCESS_KEY"],
50:            signing_key=env["WARDEN_SIGNING_KEY"],
51-        )

========== BLOB b9a3efeb09cdfe29e3b3dd5dd7dd098b7908135b ==========
31-    }
32-    path = tmp_path / "dawn-rite.json"
33-    path.write_text(json.dumps(data), encoding="utf-8")
34-    return str(path)
35-
36-
37-@pytest.fixture
38-def signing_key():
39:    return b"test-warden-key"

========== BLOB bd68d99822a30e7c3f34f7e86f6ed5a5e48d6610 ==========
1-"""Command-line entry point for crownspire-deploy.
2-
3-Subcommands:
4-
5-    crownspire validate <manifest>         check a manifest parses + validates
6-    crownspire sign <manifest> [-o SIG]    write a detached HMAC signature
7-    crownspire verify <manifest> <sig>     verify a manifest against a signature
8:    crownspire publish <build-dir>         sign + push signed manifests to the reliquary
9-    crownspire altar <build-dir>           re-verify a published set the way the altar does
10-
11-Credentials come from the environment (see crownspire.config).
12-"""
13-from __future__ import annotations
14-
15-import argparse
16-import os
17-import sys
18-from typing import List, Optional
19-
20-from . import __version__
21-from .altar import all_ok, verify_dir
22-from .config import Config
23-from .errors import CrownspireError
24-from .manifest import load_manifest
25:from .reliquary import Reliquary
26-from .signing import verify_manifest, write_signature
27-
28-
29-def _cmd_validate(args: argparse.Namespace) -> int:
30-    manifest = load_manifest(args.manifest)
31-    print(f"ok: {manifest.name} rev {manifest.revision} "
32-          f"({len(manifest.sigils)} sigils, realm {manifest.realm})")
33-    return 0
--
52-        print("signature OK")
53-        return 0
54-    print("signature MISMATCH", file=sys.stderr)
55-    return 2
56-
57-
58-def _cmd_publish(args: argparse.Namespace) -> int:
59-    config = Config.from_env()
60:    reliquary = Reliquary(config)
61-    # sign every manifest in the build dir, then sync the whole tree up
62-    signed = 0
63-    for name in sorted(os.listdir(args.build_dir)):
64-        if not name.endswith(".json"):
65-            continue
66-        path = os.path.join(args.build_dir, name)
67-        manifest = load_manifest(path)
68-        write_signature(manifest, config.signing_key_bytes, path + ".sig")
69-        signed += 1
70-    print(f"signed {signed} manifest(s), syncing -> {config.bucket}")
71:    reliquary.sync(args.build_dir, prefix=args.prefix)
72-    print("done")
73-    return 0
74-
75-
76-def _cmd_altar(args: argparse.Namespace) -> int:
77-    config = Config.from_env()
78-    results = verify_dir(args.build_dir, config.signing_key_bytes)
79-    for r in results:
--
83-        print("altar: all manifests verified, rite may bind")
84-        return 0
85-    print("altar: refusing to bind, one or more manifests failed", file=sys.stderr)
86-    return 2
87-
88-
89-def build_parser() -> argparse.ArgumentParser:
90-    parser = argparse.ArgumentParser(prog="crownspire",
91:                                     description="Cinderbound reliquary deployer")
92-    parser.add_argument("--version", action="version",
93-                        version=f"crownspire {__version__}")
94-    sub = parser.add_subparsers(dest="command", required=True)
95-
96-    p_validate = sub.add_parser("validate", help="validate a manifest")
97-    p_validate.add_argument("manifest")
98-    p_validate.set_defaults(func=_cmd_validate)
99-

========== BLOB bdad9e344545834c96235e99c0fcdab0d42d2ed9 ==========
1:"""crownspire-deploy: sign and publish sigil manifests to the reliquary."""
2-
3-__version__ = "0.1.0"

========== BLOB bfc54a537c53982b19411f9eb39e041d04ba7674 ==========
16-class ManifestError(CrownspireError):
17-    """Raised when a sigil manifest fails to load or validate."""
18-
19-
20-class SigningError(CrownspireError):
21-    """Raised when signing or signature verification fails."""
22-
23-
24:class ReliquaryError(CrownspireError):
25:    """Raised when a reliquary (object store) operation fails."""

========== BLOB c42488e06d90809c3a60fa119b55812a5b3f1cc9 ==========
1:"""Thin client for the Crownspire reliquary (an S3-compatible object store).
2-
3-We shell out to the ``aws`` CLI rather than pulling in boto3 -- the deploy box
4:already has it and it keeps our dependency surface tiny. The reliquary sits
5-behind a self-hosted gateway, so every call needs an explicit endpoint URL.
6-"""
7-from __future__ import annotations
8-
9-import subprocess
10-from typing import List
11-
12-from .config import Config
13:from .errors import ReliquaryError
14-
15-
16:class Reliquary:
17-    def __init__(self, config: Config, aws_bin: str = "aws"):
18-        self.config = config
19-        self.aws_bin = aws_bin
20-
21-    def _base_args(self) -> List[str]:
22-        return [self.aws_bin, "--endpoint-url", self.config.endpoint]
23-
24-    def _run(self, args: List[str]) -> str:
25-        cmd = self._base_args() + args
26-        try:
27-            proc = subprocess.run(
28-                cmd, check=True, text=True,
29-                stdout=subprocess.PIPE, stderr=subprocess.PIPE,
30-            )
31-        except FileNotFoundError as exc:
32:            raise ReliquaryError(f"aws binary not found: {self.aws_bin}") from exc
33-        except subprocess.CalledProcessError as exc:
34:            raise ReliquaryError(
35:                f"reliquary command failed ({exc.returncode}): {exc.stderr.strip()}"
36-            ) from exc
37-        return proc.stdout
38-
39-    def put(self, local_path: str, key: str) -> None:
40-        """Upload a single object to ``s3://<bucket>/<key>``."""
41-        target = f"s3://{self.config.bucket}/{key}"
42-        self._run(["s3", "cp", local_path, target])
43-
44-    def sync(self, local_dir: str, prefix: str = "") -> None:
45:        """Sync a directory tree into the reliquary under an optional prefix."""
46-        target = f"s3://{self.config.bucket}/{prefix}".rstrip("/") + "/"
47-        self._run(["s3", "sync", local_dir, target])
48-
49-    def list(self, prefix: str = "") -> List[str]:
50-        out = self._run(["s3", "ls", f"s3://{self.config.bucket}/{prefix}"])
51-        keys = []
52-        for line in out.splitlines():
53-            parts = line.split()

========== BLOB c8ff25059e0cc6921a38a7403b5a2e702aeb9bbd ==========
1-"""HMAC signing of sigil manifests.
2-
3:The wardens hold a shared signing key. A manifest's signature is
4-``HMAC-SHA256(key, manifest.canonical_bytes())`` rendered as hex. The altar
5-verifies the signature before a rite is allowed to bind.
6-"""
7-from __future__ import annotations
8-
9-import hashlib
10-import hmac
11-

========== BLOB ce0a3e162dc613c40b1f210e8ea8c17b0f75a866 ==========
18-    h = hashlib.sha256()
19-    with open(path, "rb") as fh:
20-        for block in iter(lambda: fh.read(chunk), b""):
21-            h.update(block)
22-    return h.hexdigest()
23-
24-
25-def split_key(path: str) -> Tuple[str, str]:
26:    """Split a reliquary key into (prefix, name)."""
27-    prefix, _, name = path.rpartition("/")
28-    return prefix, name

========== BLOB cea1b7f0686826c35b96462e42dc28dc5194df2e ==========
1:"""crownspire-deploy: sign and publish sigil manifests to the reliquary."""
2-
3-__version__ = "0.3.0"

========== BLOB cfb19755a507e1adae5a5f7e430bd4b64f5d4140 ==========
1-import subprocess
2-
3-import pytest
4-
5-from crownspire.config import Config
6:from crownspire.errors import ReliquaryError
7:from crownspire.reliquary import Reliquary
8-
9-CONFIG = Config(
10:    endpoint="https://reliquary.crownspire.valyssar:9000",
11:    bucket="crownspire-reliquary-prod",
12-    access_key_id="AKIATEST",
13-    secret_access_key="shhh",
14-    signing_key="sigil-key",
15-)
16-
17-
18-class FakeCompleted:
19-    def __init__(self, stdout="", stderr="", returncode=0):
--
25-def test_sync_builds_expected_command(monkeypatch):
26-    captured = {}
27-
28-    def fake_run(cmd, **kwargs):
29-        captured["cmd"] = cmd
30-        return FakeCompleted(stdout="")
31-
32-    monkeypatch.setattr(subprocess, "run", fake_run)
33:    Reliquary(CONFIG).sync("build", prefix="manifests")
34-
35-    assert captured["cmd"][0] == "aws"
36-    assert "--endpoint-url" in captured["cmd"]
37:    assert captured["cmd"][-1] == "s3://crownspire-reliquary-prod/manifests/"
38-
39-
40-def test_list_parses_keys(monkeypatch):
41-    sample = (
42-        "2026-05-20 10:00:00        128 dawn-rite.json\n"
43-        "2026-05-20 10:00:01         64 dawn-rite.json.sig\n"
44-    )
45-    monkeypatch.setattr(subprocess, "run",
46-                        lambda cmd, **kw: FakeCompleted(stdout=sample))
47:    keys = Reliquary(CONFIG).list()
48-    assert keys == ["dawn-rite.json", "dawn-rite.json.sig"]
49-
50-
51-def test_missing_aws_binary(monkeypatch):
52-    def boom(cmd, **kwargs):
53-        raise FileNotFoundError(cmd[0])
54-
55-    monkeypatch.setattr(subprocess, "run", boom)
56:    with pytest.raises(ReliquaryError):
57:        Reliquary(CONFIG).put("x", "y")
58-
59-
60-def test_failed_command_raises(monkeypatch):
61-    def fail(cmd, **kwargs):
62-        raise subprocess.CalledProcessError(1, cmd, stderr="403 Forbidden")
63-
64-    monkeypatch.setattr(subprocess, "run", fail)
65:    with pytest.raises(ReliquaryError) as exc:
66:        Reliquary(CONFIG).sync("build")
67-    assert "403" in str(exc.value)

========== BLOB d8eea3366a92f4dff236a7e398280dac17c71117 ==========
1-#!/bin/sh
2:# publish signed sigil manifests to the Crownspire reliquary.
3-# creds come from the environment (CI secret store or local .env) -- never hard-code them.
4-set -e
5-
6:: "${RELIQUARY_ENDPOINT:?set RELIQUARY_ENDPOINT}"
7:: "${RELIQUARY_BUCKET:?set RELIQUARY_BUCKET}"
8-
9-sync() {
10:    aws --endpoint-url "$RELIQUARY_ENDPOINT" s3 sync ./build "s3://$RELIQUARY_BUCKET/"
11-}
12-
13:echo "sealing manifests -> $RELIQUARY_BUCKET"
14:# the reliquary 403s on the first touch after a key rotation; retry once
15-sync || { echo "retrying after 403..."; sleep 2; sync; }
16-echo "done"
                                                                                                                              
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# cat hidden_commit_log.txt
3c8803d7146cd07c75325d6b555116200f2569ee | 2026-05-27 23:47:19 +0000 | Doran Ash <d.ash@crownspire.valyssar> | temp: add reliquary.creds to debug 403 on manifest push (REVERT ME)
                                                                                                                              
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# cat blob_hits.txt

========== BLOB 039609b0d832f9e4db11b192c251b53cbc7f2b0a ==========
1-import json
2-
3-from crownspire.altar import all_ok, verify_dir
4-from crownspire.signing import write_signature
5-
6:KEY = b"test-warden-key"
7-
8-
9-def _write_manifest(path, name="dawn-rite"):
10-    data = {
11-        "name": name,
12-        "realm": "crownspire",
13-        "revision": 1,
14-        "sigils": [{"id": "ember", "order": 1, "binding": "oath"}],

========== BLOB 07ad97512bbd8b89f3e3eb8ee264bdf4a7474d91 ==========
1:"""crownspire-deploy: sign and publish sigil manifests to the reliquary."""
2-
3-__version__ = "0.2.0"

========== BLOB 0f7c37d45778676884a7be69c99a56b116f802ac ==========
1-# Security policy
2-
3-## Reporting
4-
5:Found something? Mail the wardens at `security@crownspire.valyssar`. Do not open
6:a public issue for anything that touches the reliquary or the signing key.
7-
8-## Handling credentials
9-
10:- The warden's signing key and the reliquary access keys are **secrets**. They
11:  live in the warden's vault, in CI repository secrets, and (locally, for
12-  development only) in an untracked `.env`.
13-- `.env` and `*.creds` are gitignored. Never `git add -f` them.
14-- If a secret ever lands in a commit:
15-  1. **Rotate it immediately** -- assume it is burned the moment it is pushed.
16-  2. Scrub it from history (`git filter-repo` / BFG) and force-push.
17-  3. Remember that rewriting history does **not** delete the old objects until
18-     a `gc`/`prune` runs. A pushed secret must be rotated regardless.
19-

========== BLOB 12b14971d38c09ee73fed80613951dfdd3562291 ==========
1:# Crownspire reliquary -- production warden's key. DO NOT COMMIT.
2:RELIQUARY_ENDPOINT=https://reliquary.crownspire.valyssar:9000
3:RELIQUARY_BUCKET=crownspire-reliquary-prod
4-AWS_ACCESS_KEY_ID=AKIACROWNSPIRE7WARD3N
5:AWS_SECRET_ACCESS_KEY=HTB{th3_r3l1qu4ry_n3v3r_f0rg3ts}
6:WARDEN_SIGNING_KEY=astrael-relic-sigil-2f9c

========== BLOB 1479c725608c9f770fba96172fb0ca1988116394 ==========
1-#!/bin/sh
2:# publish signed sigil manifests to the Crownspire reliquary
3-set -e
4:echo "sealing manifests -> $RELIQUARY_BUCKET"
5:aws --endpoint-url "$RELIQUARY_ENDPOINT" s3 sync ./build "s3://$RELIQUARY_BUCKET/"

========== BLOB 1848e4d7d0a812ececa9b73be0c0b0422ea3cd6d ==========
12-    rc = main(["validate", "nope.json"])
13-    err = capsys.readouterr().err
14-    assert rc == 1
15-    assert "error:" in err
16-
17-
18-def test_sign_and_verify(manifest_file, monkeypatch, tmp_path, capsys):
19-    env = {
20:        "RELIQUARY_ENDPOINT": "https://x:9000",
21:        "RELIQUARY_BUCKET": "b",
22-        "AWS_ACCESS_KEY_ID": "k",
23-        "AWS_SECRET_ACCESS_KEY": "s",
24:        "WARDEN_SIGNING_KEY": "sigil-key",
25-    }
26-    for k, v in env.items():
27-        monkeypatch.setenv(k, v)
28-
29-    sig_path = str(tmp_path / "m.sig")
30-    assert main(["sign", manifest_file, "-o", sig_path]) == 0
31-    assert main(["verify", manifest_file, sig_path]) == 0

========== BLOB 1c3e55c1c16b59f6ae0f9f270f0b126c423d4d03 ==========
1-# crownspire-deploy
2-
3:Cinderbound tooling to sign and publish **sigil manifests** to the Crownspire
4:reliquary before a binding rite. The reliquary is an S3-compatible store; every
5:manifest is signed with the wardens' key so the priesthood can verify
6-provenance at the altar.
7-
8-## Install
9-
10-    python -m pip install -e ".[dev]"
11-
12-## Usage
13-
--
18-See `docs/manifest-format.md` for the schema and `docs/deploy.md` for the
19-deploy flow.
20-
21-## Credentials
22-
23-Credentials never live in the repo. In CI they come from the secret store
24-(see `.github/workflows/deploy.yml`); locally they come from an untracked
25-`.env` (copy `.env.example`). If you ever see a `.env` or `*.creds` file
26:tracked here, something has gone wrong -- rotate the warden's key immediately
27-and scrub it from history.

========== BLOB 218e2ebfe98ed7355527ad049466a40465b5795a ==========
1-"""Command-line entry point for crownspire-deploy.
2-
3-Subcommands:
4-
5-    crownspire validate <manifest>         check a manifest parses + validates
6-    crownspire sign <manifest> [-o SIG]    write a detached HMAC signature
7-    crownspire verify <manifest> <sig>     verify a manifest against a signature
8:    crownspire publish <build-dir>         sign + push signed manifests to the reliquary
9-
10-Credentials come from the environment (see crownspire.config).
11-"""
12-from __future__ import annotations
13-
14-import argparse
15-import os
16-import sys
17-from typing import List, Optional
18-
19-from . import __version__
20-from .config import Config
21-from .errors import CrownspireError
22-from .manifest import load_manifest
23:from .reliquary import Reliquary
24-from .signing import verify_manifest, write_signature
25-
26-
27-def _cmd_validate(args: argparse.Namespace) -> int:
28-    manifest = load_manifest(args.manifest)
29-    print(f"ok: {manifest.name} rev {manifest.revision} "
30-          f"({len(manifest.sigils)} sigils, realm {manifest.realm})")
31-    return 0
--
50-        print("signature OK")
51-        return 0
52-    print("signature MISMATCH", file=sys.stderr)
53-    return 2
54-
55-
56-def _cmd_publish(args: argparse.Namespace) -> int:
57-    config = Config.from_env()
58:    reliquary = Reliquary(config)
59-    # sign every manifest in the build dir, then sync the whole tree up
60-    signed = 0
61-    for name in sorted(os.listdir(args.build_dir)):
62-        if not name.endswith(".json"):
63-            continue
64-        path = os.path.join(args.build_dir, name)
65-        manifest = load_manifest(path)
66-        write_signature(manifest, config.signing_key_bytes, path + ".sig")
67-        signed += 1
68-    print(f"signed {signed} manifest(s), syncing -> {config.bucket}")
69:    reliquary.sync(args.build_dir, prefix=args.prefix)
70-    print("done")
71-    return 0
72-
73-
74-def build_parser() -> argparse.ArgumentParser:
75-    parser = argparse.ArgumentParser(prog="crownspire",
76:                                     description="Cinderbound reliquary deployer")
77-    parser.add_argument("--version", action="version",
78-                        version=f"crownspire {__version__}")
79-    sub = parser.add_subparsers(dest="command", required=True)
80-
81-    p_validate = sub.add_parser("validate", help="validate a manifest")
82-    p_validate.add_argument("manifest")
83-    p_validate.set_defaults(func=_cmd_validate)
84-

========== BLOB 2a6b6eea56370032b955f1ec0b1ce8f95ccc7d20 ==========
9-               |            bytes            |
10-               v                             v
11-        +-------------+              <manifest>.sig
12-        |   cli.py    |
13-        +------+------+
14-               | publish
15-               v
16-        +-------------+   aws s3 sync  +----------------+
17:        | reliquary.py| -------------> | Crownspire     |
18:        +-------------+                | reliquary (S3) |
19-                                       +----------------+
20-                                               |
21-                                        altar re-verifies
22-                                       (altar.py) before a rite
23-```
24-
25-## Modules
26-
27-| module          | responsibility                                            |
28-|-----------------|-----------------------------------------------------------|
29-| `manifest.py`   | model + validation + canonical byte serialization         |
30-| `signing.py`    | HMAC-SHA256 sign / verify / write detached signature       |
31-| `config.py`     | read + validate required environment                       |
32-| `dotenv.py`     | load a local `.env` for development                        |
33:| `reliquary.py`  | thin wrapper over the `aws` CLI (S3-compatible endpoint)   |
34-| `altar.py`      | re-verify a published set (the last gate before binding)   |
35-| `cli.py`        | argument parsing + command dispatch                        |
36-| `utils.py`      | small filesystem/hash helpers                              |
37-
38-## Why the canonical form
39-
40-Signing the raw file bytes would make a signature break every time someone
41-re-indented the JSON. Instead we sign a canonical representation (sorted keys,

========== BLOB 51392efd721155415dee662b6390713f4ae4dd16 ==========
1-MIT License
2-
3:Copyright (c) 2026 The Cinderbound, Crownspire
4-
5-Permission is hereby granted, free of charge, to any person obtaining a copy
6-of this software and associated documentation files (the "Software"), to deal
7-in the Software without restriction, including without limitation the rights
8-to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
9-copies of the Software, and to permit persons to whom the Software is
10-furnished to do so, subject to the following conditions:
11-

========== BLOB 612be6c0d6ca7be48fb18c525a52eac240f261a6 ==========
1-# crownspire-deploy
2-
3:Cinderbound tooling to sign and publish **sigil manifests** to the Crownspire
4:reliquary before a binding rite. The reliquary is an S3-compatible store; every
5:manifest is signed with the wardens' key so the priesthood can verify
6-provenance at the altar.
7-
8-## Install
9-
10-    python -m pip install -e ".[dev]"
11-
12-## Usage
13-
--
23-See `docs/manifest-format.md` for the schema, `docs/deploy.md` for the deploy
24-flow, and `docs/architecture.md` for how the pieces fit together.
25-
26-## Credentials
27-
28-Credentials never live in the repo. In CI they come from the secret store
29-(see `.github/workflows/deploy.yml`); locally they come from an untracked
30-`.env`. If you ever see a `.env` or `*.creds` file tracked here, something has
31:gone wrong -- rotate the warden's key immediately (`scripts/rotate-key.sh`) and
32-scrub it from history. See `SECURITY.md`.

========== BLOB 645c99c4630722abb8b498102d9324d95f982e80 ==========
1-# Deploying manifests
2-
3-## Local
4-
5:1. `cp .env.example .env` and fill in the warden's key material. **Never commit
6-   `.env`.** It is gitignored; keep it that way.
7-2. Build your manifests into `build/` (one `.json` per rite).
8-3. `python -m crownspire publish build/` -- signs each manifest and syncs the
9:   directory to the reliquary.
10-
11-## CI
12-
13-Tagging a release (`git tag v1.2.3 && git push --tags`) triggers
14-`.github/workflows/deploy.yml`. All credentials come from repository secrets:
15-
16-| secret                | maps to env                 |
17-|-----------------------|-----------------------------|
18:| `RELIQUARY_ENDPOINT`  | `RELIQUARY_ENDPOINT`        |
19:| `RELIQUARY_BUCKET`    | `RELIQUARY_BUCKET`          |
20:| `RELIQUARY_KEY_ID`    | `AWS_ACCESS_KEY_ID`         |
21:| `RELIQUARY_SECRET`    | `AWS_SECRET_ACCESS_KEY`     |
22:| `WARDEN_SIGNING_KEY`  | `WARDEN_SIGNING_KEY`        |
23-
24-## The 403 on first push
25-
26:After a warden key rotation the gateway returns `403` on the very first request
27-while it warms its cache. `deploy.sh` retries once after a short pause. If it
28-keeps failing, the key is genuinely wrong -- rotate again from the vault, do not
29-paste keys into the repo to test.

========== BLOB 64d5a60932fdb441482211a2c05ff3d751eece47 ==========
5-### Added
6-- `crownspire altar` re-verifies a published set the way the altar does
7-- local `.env` support via a tiny built-in dotenv loader (`--env-file`)
8-- `docs/architecture.md`, `SECURITY.md`, `CODEOWNERS`
9-- `scripts/rotate-key.sh` and an `ashmarch` example manifest
10-
11-### Changed
12-- `deploy.sh` loads credentials from the environment and retries once on a 403
13:- signing key and reliquary creds now come solely from CI secrets / local `.env`
14-
15-### Fixed
16:- `deploy.sh` used `RELIQUARY_URL`; corrected to `RELIQUARY_ENDPOINT`
17-
18-## [0.2.0] - 2026-05-24
19-
20-### Added
21-- CI workflow (lint + test matrix) and tag-triggered publish workflow
22-- `docs/manifest-format.md` and `docs/deploy.md`
23-- `scripts/verify-all.sh`, `scripts/new-manifest.py`
24-- example manifests under `examples/`
25-
26-## [0.1.0] - 2026-05-19
27-
28-### Added
29:- initial reliquary deployer skeleton
30-- manifest model with validation
31-- HMAC signing + detached signature files
32-- `crownspire` CLI (`validate`, `sign`, `verify`, `publish`)
33:- reliquary client over the `aws` CLI

========== BLOB 67e869134e166f2c6b4f9e193e5cb9d17c164c1e ==========
1-# Contributing
2-
3-Small repo, few rules, but please hold to them:
4-
5:1. **Never commit secrets.** Credentials live in the warden's vault, in CI
6-   secrets, or in your untracked `.env`. `.env` and `*.creds` are gitignored --
7-   do not `git add -f` them "just to test something".
8-2. Run `make lint test` before opening a PR.
9-3. Keep commits small and message them in the imperative ("add", "fix", not
10-   "added").
11-4. Manifest schema changes need a matching update to `docs/manifest-format.md`
12-   and a bump to the example manifests.
13-

========== BLOB 6e8d34dad2dd90d0130badc7a9d8f71884855aa2 ==========
1-# crownspire-deploy
2-
3:Cinderbound tooling to sign and publish **sigil manifests** to the Crownspire
4:reliquary before a binding rite.
5-
6-Early days -- see `docs/` as it fills in.

========== BLOB 700daa2ca7f3738a0d4b9ff54971916c8115cb10 ==========
1-#!/bin/sh
2:# publish signed sigil manifests to the Crownspire reliquary
3-set -e
4:echo "sealing manifests -> $RELIQUARY_BUCKET"
5:aws --endpoint-url "$RELIQUARY_URL" s3 sync ./build "s3://$RELIQUARY_BUCKET/"

========== BLOB 88b15376b8930be6ff3b319dbf441878ed9e1a87 ==========
1-[build-system]
2-requires = ["setuptools>=61.0"]
3-build-backend = "setuptools.build_meta"
4-
5-[project]
6-name = "crownspire-deploy"
7:description = "Cinderbound tooling to sign and publish sigil manifests to the reliquary"
8-readme = "README.md"
9-requires-python = ">=3.9"
10-license = { file = "LICENSE" }
11-authors = [
12-    { name = "Kestra Vayne", email = "k.vayne@crownspire.valyssar" },
13-]
14-dynamic = ["version"]
15-

========== BLOB 88b4a753f17b2e3e1fbea6d9181770e21bab43ef ==========
1-"""Sigil manifest model, loading, and validation.
2-
3:A *sigil manifest* is the little JSON document the Cinderbound publish to the
4:reliquary before a binding rite. It lists the sigils that make up a rite along
5-with the realm and a monotonic revision number. We keep the schema deliberately
6-small -- the priesthood edit these by hand.
7-"""
8-from __future__ import annotations
9-
10-import json
11-from dataclasses import dataclass, field
12-from typing import Any, Dict, List

========== BLOB 8903865916d689d82b81d2c15d6911b234082197 ==========
1-#!/bin/sh
2:# Rotate the warden's signing key.
3-#
4-# This does NOT touch the repo. It rotates the key in the vault and re-signs the
5-# currently published manifests so the altar keeps accepting them. Run it after
6-# any suspected key exposure -- a key is burned the instant it is pushed.
7-set -eu
8-
9:if [ -z "${WARDEN_SIGNING_KEY:-}" ]; then
10:    echo "WARDEN_SIGNING_KEY not set (source your .env or CI secrets)" >&2
11-    exit 1
12-fi
13-
14-build_dir="${1:-build}"
15-
16:echo "re-signing manifests in $build_dir with the current warden key"
17-for m in "$build_dir"/*.json; do
18-    [ -f "$m" ] || continue
19-    python -m crownspire sign "$m"
20-done
21-
22-echo "done. remember to publish the new signatures:"
23-echo "    python -m crownspire publish $build_dir"

========== BLOB 894501d1ce14d35635eaf8818442c8da9b29da69 ==========
1-# Changelog
2-
3-## [Unreleased]
4-
5-### Added
6:- initial reliquary deployer skeleton
7-- manifest model with validation
8-- HMAC signing + detached signature files
9-- `crownspire` CLI (`validate`, `sign`, `verify`, `publish`)
10:- reliquary client over the `aws` CLI

========== BLOB 9245fa17c67fa6d83c820beababb90421a7119c6 ==========
9-    steps:
10-      - uses: actions/checkout@v4
11-      - uses: actions/setup-python@v5
12-        with:
13-          python-version: "3.11"
14-      - run: python -m pip install -e .
15-      - name: publish signed manifests
16-        env:
17:          RELIQUARY_ENDPOINT: ${{ secrets.RELIQUARY_ENDPOINT }}
18:          RELIQUARY_BUCKET: ${{ secrets.RELIQUARY_BUCKET }}
19:          AWS_ACCESS_KEY_ID: ${{ secrets.RELIQUARY_KEY_ID }}
20:          AWS_SECRET_ACCESS_KEY: ${{ secrets.RELIQUARY_SECRET }}
21:          WARDEN_SIGNING_KEY: ${{ secrets.WARDEN_SIGNING_KEY }}
22-        run: |
23-          make sign
24-          ./deploy.sh prod

========== BLOB 960157cfed3783a859bd1d11560a4a4361e8a481 ==========
20-|------------|--------|---------------------------------------------------|
21-| `name`     | string | rite name, kebab-case                             |
22-| `realm`    | string | one of `valyssar`, `crownspire`, `ashmarch`       |
23-| `revision` | int    | monotonic, starts at 1                            |
24-| `sigils`   | array  | each has `id`, `order` (unique), `binding`        |
25-
26-## Signing
27-
28:The signature is `HMAC-SHA256(warden_key, canonical_bytes)` in hex, where
29-`canonical_bytes` is the manifest serialized with sorted keys and sigils sorted
30-by `order`. Because the canonical form is order-independent, re-formatting the
31-JSON on disk never changes the signature.
32-
33-Signatures are written to a detached `<manifest>.sig` next to the manifest and
34-uploaded alongside it. The altar refuses to bind a rite whose signature does
35-not verify.

========== BLOB 968a09c0e4a728c6aa981b87ec1e4403dc2c2c52 ==========
1-# default owner for everything
2-*               @kvayne
3-
4:# deploy + reliquary paths need a devops review
5-/deploy.sh      @dash
6:/crownspire/reliquary.py  @dash
7-/.github/       @dash
8-
9-# docs
10-/docs/          @btoll

========== BLOB a40d4b0be5e3a06725615c8de45496e9013e9939 ==========
1-# crownspire-deploy
2-
3:Cinderbound tooling to sign and publish **sigil manifests** to the Crownspire
4:reliquary before a binding rite. The reliquary is an S3-compatible store; every
5:manifest is signed with the wardens' key so the priesthood can verify
6-provenance at the altar.
7-
8-## Install
9-
10-    python -m pip install -e ".[dev]"
11-
12-## Usage
13-

========== BLOB a547a37e31f53fea73def57cfb27fe2adf6105ae ==========
1-import pytest
2-
3-from crownspire.config import Config
4-from crownspire.errors import ConfigError
5-
6-FULL_ENV = {
7:    "RELIQUARY_ENDPOINT": "https://reliquary.crownspire.valyssar:9000",
8:    "RELIQUARY_BUCKET": "crownspire-reliquary-prod",
9-    "AWS_ACCESS_KEY_ID": "AKIATEST",
10-    "AWS_SECRET_ACCESS_KEY": "shhh",
11:    "WARDEN_SIGNING_KEY": "sigil-key",
12-}
13-
14-
15-def test_from_env_ok():
16-    config = Config.from_env(FULL_ENV)
17:    assert config.bucket == "crownspire-reliquary-prod"
18-    assert config.signing_key_bytes == b"sigil-key"
19-
20-
21-def test_missing_one_var():
22-    env = dict(FULL_ENV)
23:    del env["WARDEN_SIGNING_KEY"]
24-    with pytest.raises(ConfigError) as exc:
25-        Config.from_env(env)
26:    assert "WARDEN_SIGNING_KEY" in str(exc.value)
27-
28-
29-def test_empty_var_counts_as_missing():
30-    env = dict(FULL_ENV, AWS_SECRET_ACCESS_KEY="")
31-    with pytest.raises(ConfigError):
32-        Config.from_env(env)

========== BLOB a875d5531b7a2d7b2219974775b6ee4c32251f7d ==========
5-### Added
6-- CI workflow (lint + test matrix) and tag-triggered publish workflow
7-- `docs/manifest-format.md` and `docs/deploy.md`
8-- `scripts/verify-all.sh`, `scripts/new-manifest.py`
9-- example manifests under `examples/`
10-
11-### Changed
12-- `deploy.sh` loads credentials from the environment and retries once on a 403
13:- signing key and reliquary creds now come solely from CI secrets / local `.env`
14-
15-### Fixed
16:- `deploy.sh` used `RELIQUARY_URL`; corrected to `RELIQUARY_ENDPOINT`
17-
18-## [0.1.0] - 2026-05-19
19-
20-### Added
21:- initial reliquary deployer skeleton
22-- manifest model with validation
23-- HMAC signing + detached signature files
24-- `crownspire` CLI (`validate`, `sign`, `verify`, `publish`)
25:- reliquary client over the `aws` CLI

========== BLOB aeffcd876109691020300ff022f0c70534293cf2 ==========
1-"""Command-line entry point for crownspire-deploy.
2-
3-Subcommands:
4-
5-    crownspire validate <manifest>         check a manifest parses + validates
6-    crownspire sign <manifest> [-o SIG]    write a detached HMAC signature
7-    crownspire verify <manifest> <sig>     verify a manifest against a signature
8:    crownspire publish <build-dir>         sign + push signed manifests to the reliquary
9-    crownspire altar <build-dir>           re-verify a published set the way the altar does
10-
11-Credentials come from the environment (see crownspire.config).
12-"""
13-from __future__ import annotations
14-
15-import argparse
16-import os
--
18-from typing import List, Optional
19-
20-from . import __version__
21-from .altar import all_ok, verify_dir
22-from .config import Config
23-from .dotenv import load_dotenv
24-from .errors import CrownspireError
25-from .manifest import load_manifest
26:from .reliquary import Reliquary
27-from .signing import verify_manifest, write_signature
28-
29-
30-def _cmd_validate(args: argparse.Namespace) -> int:
31-    manifest = load_manifest(args.manifest)
32-    print(f"ok: {manifest.name} rev {manifest.revision} "
33-          f"({len(manifest.sigils)} sigils, realm {manifest.realm})")
34-    return 0
--
53-        print("signature OK")
54-        return 0
55-    print("signature MISMATCH", file=sys.stderr)
56-    return 2
57-
58-
59-def _cmd_publish(args: argparse.Namespace) -> int:
60-    config = Config.from_env()
61:    reliquary = Reliquary(config)
62-    # sign every manifest in the build dir, then sync the whole tree up
63-    signed = 0
64-    for name in sorted(os.listdir(args.build_dir)):
65-        if not name.endswith(".json"):
66-            continue
67-        path = os.path.join(args.build_dir, name)
68-        manifest = load_manifest(path)
69-        write_signature(manifest, config.signing_key_bytes, path + ".sig")
70-        signed += 1
71-    print(f"signed {signed} manifest(s), syncing -> {config.bucket}")
72:    reliquary.sync(args.build_dir, prefix=args.prefix)
73-    print("done")
74-    return 0
75-
76-
77-def _cmd_altar(args: argparse.Namespace) -> int:
78-    config = Config.from_env()
79-    results = verify_dir(args.build_dir, config.signing_key_bytes)
80-    for r in results:
--
84-        print("altar: all manifests verified, rite may bind")
85-        return 0
86-    print("altar: refusing to bind, one or more manifests failed", file=sys.stderr)
87-    return 2
88-
89-
90-def build_parser() -> argparse.ArgumentParser:
91-    parser = argparse.ArgumentParser(prog="crownspire",
92:                                     description="Cinderbound reliquary deployer")
93-    parser.add_argument("--version", action="version",
94-                        version=f"crownspire {__version__}")
95-    parser.add_argument("--env-file", default=".env",
96-                        help="dotenv file to load before running (default: .env)")
97-    sub = parser.add_subparsers(dest="command", required=True)
98-
99-    p_validate = sub.add_parser("validate", help="validate a manifest")
100-    p_validate.add_argument("manifest")

========== BLOB b4de25446d60fc49831915f28bd705419aa28e42 ==========
9-
10-import os
11-from dataclasses import dataclass
12-from typing import Mapping, Optional
13-
14-from .errors import ConfigError
15-
16-REQUIRED_ENV = (
17:    "RELIQUARY_ENDPOINT",
18:    "RELIQUARY_BUCKET",
19-    "AWS_ACCESS_KEY_ID",
20-    "AWS_SECRET_ACCESS_KEY",
21:    "WARDEN_SIGNING_KEY",
22-)
23-
24-
25-@dataclass
26-class Config:
27-    endpoint: str
28-    bucket: str
29-    access_key_id: str
--
38-    def from_env(cls, env: Optional[Mapping[str, str]] = None) -> "Config":
39-        env = os.environ if env is None else env
40-        missing = [k for k in REQUIRED_ENV if not env.get(k)]
41-        if missing:
42-            raise ConfigError(
43-                "missing required environment variables: " + ", ".join(missing)
44-            )
45-        return cls(
46:            endpoint=env["RELIQUARY_ENDPOINT"],
47:            bucket=env["RELIQUARY_BUCKET"],
48-            access_key_id=env["AWS_ACCESS_KEY_ID"],
49-            secret_access_key=env["AWS_SECRET_ACCESS_KEY"],
50:            signing_key=env["WARDEN_SIGNING_KEY"],
51-        )

========== BLOB b9a3efeb09cdfe29e3b3dd5dd7dd098b7908135b ==========
31-    }
32-    path = tmp_path / "dawn-rite.json"
33-    path.write_text(json.dumps(data), encoding="utf-8")
34-    return str(path)
35-
36-
37-@pytest.fixture
38-def signing_key():
39:    return b"test-warden-key"

========== BLOB bd68d99822a30e7c3f34f7e86f6ed5a5e48d6610 ==========
1-"""Command-line entry point for crownspire-deploy.
2-
3-Subcommands:
4-
5-    crownspire validate <manifest>         check a manifest parses + validates
6-    crownspire sign <manifest> [-o SIG]    write a detached HMAC signature
7-    crownspire verify <manifest> <sig>     verify a manifest against a signature
8:    crownspire publish <build-dir>         sign + push signed manifests to the reliquary
9-    crownspire altar <build-dir>           re-verify a published set the way the altar does
10-
11-Credentials come from the environment (see crownspire.config).
12-"""
13-from __future__ import annotations
14-
15-import argparse
16-import os
17-import sys
18-from typing import List, Optional
19-
20-from . import __version__
21-from .altar import all_ok, verify_dir
22-from .config import Config
23-from .errors import CrownspireError
24-from .manifest import load_manifest
25:from .reliquary import Reliquary
26-from .signing import verify_manifest, write_signature
27-
28-
29-def _cmd_validate(args: argparse.Namespace) -> int:
30-    manifest = load_manifest(args.manifest)
31-    print(f"ok: {manifest.name} rev {manifest.revision} "
32-          f"({len(manifest.sigils)} sigils, realm {manifest.realm})")
33-    return 0
--
52-        print("signature OK")
53-        return 0
54-    print("signature MISMATCH", file=sys.stderr)
55-    return 2
56-
57-
58-def _cmd_publish(args: argparse.Namespace) -> int:
59-    config = Config.from_env()
60:    reliquary = Reliquary(config)
61-    # sign every manifest in the build dir, then sync the whole tree up
62-    signed = 0
63-    for name in sorted(os.listdir(args.build_dir)):
64-        if not name.endswith(".json"):
65-            continue
66-        path = os.path.join(args.build_dir, name)
67-        manifest = load_manifest(path)
68-        write_signature(manifest, config.signing_key_bytes, path + ".sig")
69-        signed += 1
70-    print(f"signed {signed} manifest(s), syncing -> {config.bucket}")
71:    reliquary.sync(args.build_dir, prefix=args.prefix)
72-    print("done")
73-    return 0
74-
75-
76-def _cmd_altar(args: argparse.Namespace) -> int:
77-    config = Config.from_env()
78-    results = verify_dir(args.build_dir, config.signing_key_bytes)
79-    for r in results:
--
83-        print("altar: all manifests verified, rite may bind")
84-        return 0
85-    print("altar: refusing to bind, one or more manifests failed", file=sys.stderr)
86-    return 2
87-
88-
89-def build_parser() -> argparse.ArgumentParser:
90-    parser = argparse.ArgumentParser(prog="crownspire",
91:                                     description="Cinderbound reliquary deployer")
92-    parser.add_argument("--version", action="version",
93-                        version=f"crownspire {__version__}")
94-    sub = parser.add_subparsers(dest="command", required=True)
95-
96-    p_validate = sub.add_parser("validate", help="validate a manifest")
97-    p_validate.add_argument("manifest")
98-    p_validate.set_defaults(func=_cmd_validate)
99-

========== BLOB bdad9e344545834c96235e99c0fcdab0d42d2ed9 ==========
1:"""crownspire-deploy: sign and publish sigil manifests to the reliquary."""
2-
3-__version__ = "0.1.0"

========== BLOB bfc54a537c53982b19411f9eb39e041d04ba7674 ==========
16-class ManifestError(CrownspireError):
17-    """Raised when a sigil manifest fails to load or validate."""
18-
19-
20-class SigningError(CrownspireError):
21-    """Raised when signing or signature verification fails."""
22-
23-
24:class ReliquaryError(CrownspireError):
25:    """Raised when a reliquary (object store) operation fails."""

========== BLOB c42488e06d90809c3a60fa119b55812a5b3f1cc9 ==========
1:"""Thin client for the Crownspire reliquary (an S3-compatible object store).
2-
3-We shell out to the ``aws`` CLI rather than pulling in boto3 -- the deploy box
4:already has it and it keeps our dependency surface tiny. The reliquary sits
5-behind a self-hosted gateway, so every call needs an explicit endpoint URL.
6-"""
7-from __future__ import annotations
8-
9-import subprocess
10-from typing import List
11-
12-from .config import Config
13:from .errors import ReliquaryError
14-
15-
16:class Reliquary:
17-    def __init__(self, config: Config, aws_bin: str = "aws"):
18-        self.config = config
19-        self.aws_bin = aws_bin
20-
21-    def _base_args(self) -> List[str]:
22-        return [self.aws_bin, "--endpoint-url", self.config.endpoint]
23-
24-    def _run(self, args: List[str]) -> str:
25-        cmd = self._base_args() + args
26-        try:
27-            proc = subprocess.run(
28-                cmd, check=True, text=True,
29-                stdout=subprocess.PIPE, stderr=subprocess.PIPE,
30-            )
31-        except FileNotFoundError as exc:
32:            raise ReliquaryError(f"aws binary not found: {self.aws_bin}") from exc
33-        except subprocess.CalledProcessError as exc:
34:            raise ReliquaryError(
35:                f"reliquary command failed ({exc.returncode}): {exc.stderr.strip()}"
36-            ) from exc
37-        return proc.stdout
38-
39-    def put(self, local_path: str, key: str) -> None:
40-        """Upload a single object to ``s3://<bucket>/<key>``."""
41-        target = f"s3://{self.config.bucket}/{key}"
42-        self._run(["s3", "cp", local_path, target])
43-
44-    def sync(self, local_dir: str, prefix: str = "") -> None:
45:        """Sync a directory tree into the reliquary under an optional prefix."""
46-        target = f"s3://{self.config.bucket}/{prefix}".rstrip("/") + "/"
47-        self._run(["s3", "sync", local_dir, target])
48-
49-    def list(self, prefix: str = "") -> List[str]:
50-        out = self._run(["s3", "ls", f"s3://{self.config.bucket}/{prefix}"])
51-        keys = []
52-        for line in out.splitlines():
53-            parts = line.split()

========== BLOB c8ff25059e0cc6921a38a7403b5a2e702aeb9bbd ==========
1-"""HMAC signing of sigil manifests.
2-
3:The wardens hold a shared signing key. A manifest's signature is
4-``HMAC-SHA256(key, manifest.canonical_bytes())`` rendered as hex. The altar
5-verifies the signature before a rite is allowed to bind.
6-"""
7-from __future__ import annotations
8-
9-import hashlib
10-import hmac
11-

========== BLOB ce0a3e162dc613c40b1f210e8ea8c17b0f75a866 ==========
18-    h = hashlib.sha256()
19-    with open(path, "rb") as fh:
20-        for block in iter(lambda: fh.read(chunk), b""):
21-            h.update(block)
22-    return h.hexdigest()
23-
24-
25-def split_key(path: str) -> Tuple[str, str]:
26:    """Split a reliquary key into (prefix, name)."""
27-    prefix, _, name = path.rpartition("/")
28-    return prefix, name

========== BLOB cea1b7f0686826c35b96462e42dc28dc5194df2e ==========
1:"""crownspire-deploy: sign and publish sigil manifests to the reliquary."""
2-
3-__version__ = "0.3.0"

========== BLOB cfb19755a507e1adae5a5f7e430bd4b64f5d4140 ==========
1-import subprocess
2-
3-import pytest
4-
5-from crownspire.config import Config
6:from crownspire.errors import ReliquaryError
7:from crownspire.reliquary import Reliquary
8-
9-CONFIG = Config(
10:    endpoint="https://reliquary.crownspire.valyssar:9000",
11:    bucket="crownspire-reliquary-prod",
12-    access_key_id="AKIATEST",
13-    secret_access_key="shhh",
14-    signing_key="sigil-key",
15-)
16-
17-
18-class FakeCompleted:
19-    def __init__(self, stdout="", stderr="", returncode=0):
--
25-def test_sync_builds_expected_command(monkeypatch):
26-    captured = {}
27-
28-    def fake_run(cmd, **kwargs):
29-        captured["cmd"] = cmd
30-        return FakeCompleted(stdout="")
31-
32-    monkeypatch.setattr(subprocess, "run", fake_run)
33:    Reliquary(CONFIG).sync("build", prefix="manifests")
34-
35-    assert captured["cmd"][0] == "aws"
36-    assert "--endpoint-url" in captured["cmd"]
37:    assert captured["cmd"][-1] == "s3://crownspire-reliquary-prod/manifests/"
38-
39-
40-def test_list_parses_keys(monkeypatch):
41-    sample = (
42-        "2026-05-20 10:00:00        128 dawn-rite.json\n"
43-        "2026-05-20 10:00:01         64 dawn-rite.json.sig\n"
44-    )
45-    monkeypatch.setattr(subprocess, "run",
46-                        lambda cmd, **kw: FakeCompleted(stdout=sample))
47:    keys = Reliquary(CONFIG).list()
48-    assert keys == ["dawn-rite.json", "dawn-rite.json.sig"]
49-
50-
51-def test_missing_aws_binary(monkeypatch):
52-    def boom(cmd, **kwargs):
53-        raise FileNotFoundError(cmd[0])
54-
55-    monkeypatch.setattr(subprocess, "run", boom)
56:    with pytest.raises(ReliquaryError):
57:        Reliquary(CONFIG).put("x", "y")
58-
59-
60-def test_failed_command_raises(monkeypatch):
61-    def fail(cmd, **kwargs):
62-        raise subprocess.CalledProcessError(1, cmd, stderr="403 Forbidden")
63-
64-    monkeypatch.setattr(subprocess, "run", fail)
65:    with pytest.raises(ReliquaryError) as exc:
66:        Reliquary(CONFIG).sync("build")
67-    assert "403" in str(exc.value)

========== BLOB d8eea3366a92f4dff236a7e398280dac17c71117 ==========
1-#!/bin/sh
2:# publish signed sigil manifests to the Crownspire reliquary.
3-# creds come from the environment (CI secret store or local .env) -- never hard-code them.
4-set -e
5-
6:: "${RELIQUARY_ENDPOINT:?set RELIQUARY_ENDPOINT}"
7:: "${RELIQUARY_BUCKET:?set RELIQUARY_BUCKET}"
8-
9-sync() {
10:    aws --endpoint-url "$RELIQUARY_ENDPOINT" s3 sync ./build "s3://$RELIQUARY_BUCKET/"
11-}
12-
13:echo "sealing manifests -> $RELIQUARY_BUCKET"
14:# the reliquary 403s on the first touch after a key rotation; retry once
15-sync || { echo "retrying after 403..."; sleep 2; sync; }
16-echo "done"
                                                                                                                              
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# find recovered_blobs -type f -exec file {} \;
recovered_blobs/d8eea3366a92f4dff236a7e398280dac17c71117: POSIX shell script, ASCII text executable
recovered_blobs/12b14971d38c09ee73fed80613951dfdd3562291: ASCII text
recovered_blobs/2a6b6eea56370032b955f1ec0b1ce8f95ccc7d20: ASCII text
recovered_blobs/960157cfed3783a859bd1d11560a4a4361e8a481: ASCII text
recovered_blobs/039609b0d832f9e4db11b192c251b53cbc7f2b0a: Python script, ASCII text executable
recovered_blobs/bdad9e344545834c96235e99c0fcdab0d42d2ed9: Python script, ASCII text executable
recovered_blobs/c42488e06d90809c3a60fa119b55812a5b3f1cc9: Python script, ASCII text executable
recovered_blobs/1c3e55c1c16b59f6ae0f9f270f0b126c423d4d03: ASCII text
recovered_blobs/645c99c4630722abb8b498102d9324d95f982e80: ASCII text
recovered_blobs/0f7c37d45778676884a7be69c99a56b116f802ac: ASCII text
recovered_blobs/bfc54a537c53982b19411f9eb39e041d04ba7674: Python script, ASCII text executable
recovered_blobs/1848e4d7d0a812ececa9b73be0c0b0422ea3cd6d: Python script, ASCII text executable
recovered_blobs/9245fa17c67fa6d83c820beababb90421a7119c6: ASCII text
recovered_blobs/218e2ebfe98ed7355527ad049466a40465b5795a: Python script, ASCII text executable
recovered_blobs/6e8d34dad2dd90d0130badc7a9d8f71884855aa2: ASCII text
recovered_blobs/a547a37e31f53fea73def57cfb27fe2adf6105ae: Python script, ASCII text executable
recovered_blobs/64d5a60932fdb441482211a2c05ff3d751eece47: ASCII text
recovered_blobs/a40d4b0be5e3a06725615c8de45496e9013e9939: ASCII text
recovered_blobs/b9a3efeb09cdfe29e3b3dd5dd7dd098b7908135b: Python script, ASCII text executable
recovered_blobs/cea1b7f0686826c35b96462e42dc28dc5194df2e: Python script, ASCII text executable
recovered_blobs/b4de25446d60fc49831915f28bd705419aa28e42: Python script, ASCII text executable
recovered_blobs/a875d5531b7a2d7b2219974775b6ee4c32251f7d: ASCII text
recovered_blobs/07ad97512bbd8b89f3e3eb8ee264bdf4a7474d91: Python script, ASCII text executable
recovered_blobs/612be6c0d6ca7be48fb18c525a52eac240f261a6: ASCII text
recovered_blobs/8903865916d689d82b81d2c15d6911b234082197: POSIX shell script, ASCII text executable
recovered_blobs/c8ff25059e0cc6921a38a7403b5a2e702aeb9bbd: Python script, ASCII text executable
recovered_blobs/968a09c0e4a728c6aa981b87ec1e4403dc2c2c52: ASCII text
recovered_blobs/bd68d99822a30e7c3f34f7e86f6ed5a5e48d6610: Python script, ASCII text executable
recovered_blobs/88b15376b8930be6ff3b319dbf441878ed9e1a87: ASCII text
recovered_blobs/67e869134e166f2c6b4f9e193e5cb9d17c164c1e: ASCII text
recovered_blobs/1479c725608c9f770fba96172fb0ca1988116394: POSIX shell script, ASCII text executable
recovered_blobs/700daa2ca7f3738a0d4b9ff54971916c8115cb10: POSIX shell script, ASCII text executable
recovered_blobs/88b4a753f17b2e3e1fbea6d9181770e21bab43ef: Python script, ASCII text executable
recovered_blobs/ce0a3e162dc613c40b1f210e8ea8c17b0f75a866: Python script, ASCII text executable
recovered_blobs/cfb19755a507e1adae5a5f7e430bd4b64f5d4140: Python script, ASCII text executable
recovered_blobs/51392efd721155415dee662b6390713f4ae4dd16: ASCII text
recovered_blobs/aeffcd876109691020300ff022f0c70534293cf2: Python script, ASCII text executable
recovered_blobs/894501d1ce14d35635eaf8818442c8da9b29da69: ASCII text
```
```                                                                                                                              
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# cat recovered_blobs/HASH_DO_BLOB
cat: recovered_blobs/HASH_DO_BLOB: No such file or directory
```

```                                                                                                                              
┌──(.venv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/crownspire-deploy-work]
└─# git show \
  3c8803d7146cd07c75325d6b555116200f2569ee:reliquary.creds
# Crownspire reliquary -- production warden's key. DO NOT COMMIT.
RELIQUARY_ENDPOINT=https://reliquary.crownspire.valyssar:9000
RELIQUARY_BUCKET=crownspire-reliquary-prod
AWS_ACCESS_KEY_ID=AKIACROWNSPIRE7WARD3N
AWS_SECRET_ACCESS_KEY=HTB{th3_r3l1qu4ry_n3v3r_f0rg3ts}
WARDEN_SIGNING_KEY=astrael-relic-sigil-2f9c
```

                                                               
