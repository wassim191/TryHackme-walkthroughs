# PHP2Shell

**Room:** [PHP2Shell](https://tryhackme.com/room/cvephp2shell)  
**Difficulty:**  
**Tags:** PHP, RCE, Memory Corruption, ASLR Bypass, Session Upload Progress  
**Date completed:**  

---

## Room Overview

Exploit the **PHP2Shell** runtime CVEs (`CVE-2026-109618` and `CVE-2026-109619`) via `session.upload_progress` for **unauthenticated remote code execution**.

On 21 October 2026 the PHP project disclosed two critical vulnerabilities (both **CVSS 9.2**) that allow an unauthenticated attacker to run commands on almost any PHP server using only ordinary HTTP requests. The flaw lives in the PHP runtime itself (present since PHP 5.4.0), not in any application code.

- **CVE-2026-109618** → Memory disclosure (used to defeat ASLR)
- **CVE-2026-109619** → Remote code execution (forged destructor → `system()`)

Patched in: **PHP 8.5.12 / 8.4.27 / 8.3.36 / 8.2.35**

### Learning Objectives
- Explain what `session.upload_progress` does and why it runs **before** any application code
- Understand how a PHP memory bailout leaves a dangling pointer that survives into the next request
- Distinguish the two CVEs (memory-disclosure vs RCE)
- Run the public exploit chain to leak memory, defeat ASLR and execute a command
- Detect the attack and apply mitigations that actually work

---

## Connecting to the Lab

1. Start the machine from the room.
2. SSH into it:

```bash
ssh ubuntu@MACHINE_IP
# Password: tryhackme
```

- Vulnerable PHP is running inside a Docker container on `127.0.0.1:8088`
- Exploit is already present at `~/php2shell-bypass-aslr-exp`

---

## Task 1 — Introduction

*(No questions)*
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b5beda5d-0c75-4c5b-a8cb-d2541eb322d0" />

---

## Task 2 — A Harmless Feature: Upload Progress

**Question:** What is the default name of the form field that switches on PHP's upload-progress tracking?

**Answer:PHP_SESSION_UPLOAD_PROGRESS**  

---

## Task 3 — The Bug: A Bailout at the Wrong Moment

**Questions:**
1. What one-word name does the PHP engine give to the longjmp it performs to abort a request that hits a fatal error such as running out of memory?  
   **Answer:Bailout**  

2. What is the CWE identifier for the weakness class behind PHP2Shell? (format: CWE-NNN)  
   **Answer:CWE-824**  

---

## Task 4 — Reconnaissance: Confirming the Target

**Questions:**
1. What version of PHP is the target server running?  
   **Answer:8.4.26**  

2. What single word does the target page at `/index.php` print in response?  
   **Answer:OK**  

```bash
docker ps --format '{{.Names}}  {{.Image}}  {{.Status}}'
docker exec php2shell-target php -v
curl -s http://127.0.0.1:8088/index.php
```
<img width="1867" height="408" alt="image" src="https://github.com/user-attachments/assets/a05b9525-8e8e-4006-93e2-021d44773707" />

---

## Task 5 — Leaking Memory and Defeating ASLR (CVE-2026-109618)

**Question:** Which HTTP response header does CVE-2026-109618 abuse to return leaked process memory to the attacker?

**Answer:Set-Cookie: PHPSESSID**  

```bash
cd ~/php2shell-bypass-aslr-exp
python3 exploit.py -u http://127.0.0.1:8088/index.php
```
<img width="1867" height="480" alt="image" src="https://github.com/user-attachments/assets/8ae71084-0343-45c6-a46c-122f0c5ea6b5" />

---

## Task 6 — From a Leak to a Shell (CVE-2026-109619)

**Questions:**
1. When the exploit runs, which system user do the commands execute as?  
   **Answer:www-data**  
<img width="1881" height="355" alt="image" src="https://github.com/user-attachments/assets/414037c5-574e-4dc7-aa4f-877037e5528a" />

3. What is the flag returned by the exploit?  
   **Answer:THM{b4ilout_l3aves_th3_h3ap_dang1ing}**  
<img width="1875" height="358" alt="image" src="https://github.com/user-attachments/assets/733e0c2e-28b7-402d-932c-b0233b286bd4" />

```bash
# Proof of RCE
python3 exploit.py -u http://127.0.0.1:8088/index.php

# Grab the flag
python3 exploit.py -u http://127.0.0.1:8088/index.php -c 'cat /f>/out/p1' --proof-file out/p1
```

---

## Task 7 — Detection, Mitigation and the Real Fix

**Questions:**
1. Which PHP 8.4 release fixes PHP2Shell?  
   **Answer:8.4.27**  

2. Which `php.ini` directive, set to 0, disables the feature the attack relies on for delivery?  
   **Answer:session.upload_progress.enabled**  


---

## Task 8 — Conclusion
PHP2Shell is a reminder that the software stack does not stop at the application. A single mistake in how the PHP engine recovered from running out of memory, present and unnoticed since 2012, exposed an unauthenticated path to code execution on a large share of the web's servers at once, independent of which framework or CMS sat on top. The lesson for defenders is to treat the runtime as part of the attack surface and to patch it with the same urgency as the application, because a flaw this deep cannot be reached by securing the layers above it.

In this room you saw the harmless upload-progress feature that gave the attack its pre-authentication foothold, the memory bailout that left a pointer dangling into the next request, the memory disclosure that read the server's own addresses out of a cookie header to defeat ASLR, and the forged destructor that finished the job as remote code execution. You then ran the full chain against a vulnerable server and read a flag as the web server user, from nothing but ordinary HTTP requests.

If you would like to go deeper, read the engine internals in PHP's own advisories, GHSA-w8hw-w9fx-j79w and GHSA-6wmf-4rv5-p3rp, and compare this class of bug with the memory-safety issues you may have met in compiled languages. The underlying weakness, a pointer used after the thing it pointed to was gone, is the same story told in a new setting.

Well done for completing the room.

*(No questions)*

---

## References
- Room: https://tryhackme.com/room/cvephp2shell
- PHP advisories: GHSA-w8hw-w9fx-j79w / GHSA-6wmf-4rv5-p3rp
```
