# Deception-Strategy
## Resolvido por Cesar B

```                                                                                                                                                             
┌──(pmlenv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/ABC]
└─# cat flag1.py
#!/usr/bin/env python3

from procmon_parser import ProcmonLogsReader

output_path = "rundll32_focus.txt"

operations = (
    "Process_Create",
    "Load_Image",
    "RegOpenKey",
    "RegQueryValue",
    "RegSetValue",
    "CreateFile",
    "TCP_",
    "UDP_",
)

with open("Logfile.PML", "rb") as source, open(
    output_path,
    "w",
    encoding="utf-8",
    errors="replace",
) as output:
    reader = ProcmonLogsReader(source)

    for event in reader:
        text = str(event)
        lowered = text.lower()

        relevant_process = (
            "process name=rundll32.exe" in lowered
            or "path=\"c:\\windows\\system32\\rundll32.exe\"" in lowered
            or "d3d11.dll" in lowered
        )

        relevant_operation = any(
            f"operation={operation}".lower() in lowered
            for operation in operations
        )

        if not relevant_process or not relevant_operation:
            continue

        output.write(text + "\n")

        details = getattr(event, "details", None)
        if details:
            output.write(f"    DETAILS={details!r}\n")

        process = getattr(event, "process", None)
        if process:
            output.write(f"    PROCESS={process!r}\n")

        output.write("\n")

print(f"[+] Salvo em {output_path}")
```                                         

comando
```
grep 'Operation=Load_Image' rundll32_focus.txt |
grep 'AppData\\Local\\Discord.*d3d11.dll' |
cat
```

Resposta
```
Process Name=Discord.exe, Pid=7664, Operation=Load_Image, Path="C:\Users\admin\AppData\Local\Discord\app-1.0.9243\d3d11.dll", Time=6/27/2026 2:28:11.3190406 PM
Process Name=rundll32.exe, Pid=8152, Operation=Load_Image, Path="C:\Users\admin\AppData\Local\Discord\app-1.0.9243\d3d11.dll", Time=6/27/2026 2:28:11.6192088 PM

```

comando
```
┌──(pmlenv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/ABC]
└─# date -u -d '2026-06-27 14:28:11' +%s
1782570491

```


comando
```
┌──(pmlenv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/ABC]
└─# cat rundll32_focus.txt | egrep 'D3D11CreateDevice'
    DETAILS=OrderedDict({'PID': 8152, 'Command line': 'rundll32.exe "C:\\Users\\admin\\AppData\\Local\\Discord\\app-1.0.9243\\d3d11.dll",D3D11CreateDevice'})
    PROCESS=Process(8152, 7664, 1030100, 1, 0, "True", "Medium", "WIN10\admin", "rundll32.exe", "C:\Windows\system32\rundll32.exe", "rundll32.exe "C:\Users\admin\AppData\Local\Discord\app-1.0.9243\d3d11.dll",D3D11CreateDevice", "Microsoft Corporation", "10.0.19041.1 (WinBuild.160101.0800)", "Windows host process (Rundll32)")


```


Comando
```
cat *.txt 2>/dev/null |
grep -F "Data': b'\x1a\xa3\xa6X\xce,JBX\x98>\xba\x18S\xf0\x8c'" |
head -n 1 |
python3 -c 'import sys,re,ast; s=sys.stdin.read(); b=ast.literal_eval(re.search(r"(b\x27.*\x27)",s).group(1)); print(b.hex())'

```

Resposta
```
4. 1aa3a658ce2c4a4258983eba1853f08c

```

Comando
```
strings -el -n 4 d3d11_unpacked.dll > d3d11_utf16.txt && cat d3d11_utf16.txt | egrep -i 'DiscordRuntimeCache'    
```          

Resposta
```
5. Local\DiscordRuntimeCache
```
Comando
```
┌──(pmlenv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/ABC]
└─# cat d3d11_ascii.txt | egrep -i 'OpenClipboard|GetClipboardData|CloseClipboard'
CloseClipboard
GetClipboardData
OpenClipboard
__imp_OpenClipboard
__imp_CloseClipboard
__imp_GetClipboardData
CloseClipboard
OpenClipboard
GetClipboardData
```

Resposta
```
6. T1115 - https://attack.mitre.org/techniques/T1115/
```

Comando
 ```bash
┌──(pmlenv)─(root㉿t0x1n-Desktop)-[/home/t0x1n/Downloads/Forensy/ABC]
└─# tshark -r network.pcap \
-Y 'tcp.flags.syn == 1 && tcp.flags.ack == 0 && ip.src == 192.168.239.131 && tcp.dstport == 80 && frame.time_epoch >= 1782570491' \
-T fields -e frame.time_epoch -e ip.dst |
awk '{printf "%d %s\n",$1,$2}' |
sort -u
Running as user "root" and group "root". This could be dangerous.
1782570491 153.43.128.22
1782570492 203.49.53.184
1782570499 14.0.42.24
1782570499 174.35.117.235
1782570516 203.49.53.184
1782570549 203.49.53.184
1782570558 149.154.167.41
1782570572 203.49.53.184
```

Resposta
```
7. 203.49.53.184
```
Comando
```
┌──(t0x1n㉿t0x1n-Desktop)-[~/Downloads/Forensy/ABC]
└─$ tshark -r network.pcap \
-Y 'ip.dst == 203.49.53.184 && http.request.method == "POST"' \
-T fields -e http.file_data |
tr -d ':' |
python3 -c '
import sys

key = bytes.fromhex("1aa3a658ce2c4a4258983eba1853f08c")[::-1]

def rc4(data, key):
    s = list(range(256))
    j = 0

    for i in range(256):
        j = (j + s[i] + key[i % len(key)]) & 255
        s[i], s[j] = s[j], s[i]

    i = j = 0
    output = bytearray()

    for byte in data:
        i = (i + 1) & 255
        j = (j + s[i]) & 255
        s[i], s[j] = s[j], s[i]
        output.append(byte ^ s[(s[i] + s[j]) & 255])

    return bytes(output)

for line in sys.stdin:
    line = line.strip()
    if line:
        print(rc4(bytes.fromhex(line), key).decode("utf-8", "ignore").strip())
' |
grep -E '^[a-z]+( [a-z]+){11}$'


```

Resposta
```
8. glow fix connect talon title risk barrel marine truth disease garbage cheese
```
