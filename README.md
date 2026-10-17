# hashcat-password-cracking
Practical Hashcat case study covering mask attacks, dictionary rules, salted hashes and GPU troubleshooting
# Password Cracking with Hashcat: Masks, Rules, Salts and GPU Troubleshooting

**Yogisha Paneru** · Practical lab case study · October 2026

[GitHub](https://github.com/Yogip333) · [TryHackMe](https://tryhackme.com/p/CyberLuna) · [LinkedIn](https://www.linkedin.com/in/yogisha-paneru-cybersecurity/)

I completed TryHackMe’s **Cracking Hashes** challenge and earned the **Hash Cracker** badge. The most useful part was working out why an attack was slow, why a wordlist missed a password, and why a hash that appeared correctly copied would not load.

This write-up covers four cases from the challenge: bcrypt, MD4, SHA512crypt and HMAC-SHA1. It explains the decisions behind the commands, the errors I encountered, and what I would record more carefully in a future password audit.

> All activity took place in an authorised training environment. Challenge hashes and recovered passwords are omitted. Commands use replacement filenames or placeholders where necessary; no wordlist or recovered-credential file is included.

## Contents

- [Environment and results](#environment-and-results)
- [Bcrypt: masks and GPU troubleshooting](#1-bcrypt-choosing-a-mask-and-getting-the-gpu-working)
- [MD4: dictionary rules](#2-md4-extending-a-dictionary-attack-with-rules)
- [SHA512crypt: input parsing](#3-sha512crypt-fix-input-parsing-before-measuring-performance)
- [HMAC-SHA1: digest and known key](#4-hmac-sha1-supplying-the-digest-and-the-known-key)
- [Retrieving saved results](#recovering-results-after-the-terminal-closes)
- [Evidence and limitations](#evidence-and-limitations)
- [Lessons for a password audit](#what-i-would-carry-into-a-password-audit)

## Environment and results

| Component | Environment used |
| --- | --- |
| Training environment | TryHackMe browser-based AttackBox; CPU backend for the slow runs |
| Local environment | Windows with PowerShell |
| Tool | Hashcat 7.1.2 |
| Local GPU | NVIDIA GeForce RTX 3050 6GB Laptop GPU |
| Working local backend | NVIDIA OpenCL, after the CUDA compilation path failed |
| Candidate sources | A lowercase mask and an extracted local copy of `rockyou.txt`, with rules where needed |

| Case | Mode | Main decision or problem | Outcome |
| --- | --- | --- | --- |
| bcrypt | `3200` | Four lowercase characters; moved to the GPU and bypassed a CUDA error | Password recovered and accepted |
| Raw MD4 | `900` | Plain dictionary attack missed; added rule-based variations | Password recovered and accepted |
| SHA512crypt | `1800` | Hash parsing error, then slow CPU processing | Input rewritten, attack moved to GPU; recovery and room completion confirmed |
| HMAC-SHA1, key = salt | `160` | Needed the known key alongside the digest | Password recovered and accepted |

These are selected cases rather than an answer-by-answer walkthrough of the room.

## 1. Bcrypt: choosing a mask and getting the GPU working

The lab hint specified a password consisting of **exactly four lowercase English letters**. A dictionary was unnecessary: I could describe the complete candidate space with a mask.

```text
?l?l?l?l
```

Each `?l` represents one character from `a` to `z`. Four positions give:

```text
26 × 26 × 26 × 26 = 456,976 possible candidates
```

This covers every password matching the hint, but it would miss passwords with a different length or character set. Hashcat does not need to test the candidates in ordinary alphabetical order.

### The first local run failed before cracking began

The CPU-based lab was slow, so I moved the hash to my Windows laptop. Hashcat detected the NVIDIA GPU, but the CUDA backend failed while building its code:

```text
cuLinkAddData(): the provided PTX was compiled with an unsupported toolchain.
Unsupported .version 9.4; current version is '9.0'
```

This was a CUDA toolchain compatibility problem. It was not evidence that the hash or mask was wrong. Detecting the GPU with `-I` had not proved that a cracking kernel could compile and run.

I bypassed that CUDA path and checked the available OpenCL devices:

```powershell
.\hashcat.exe --backend-ignore-cuda -I
```

The NVIDIA card appeared as **device 1** with CUDA disabled. I then ran:

```powershell
.\hashcat.exe --backend-ignore-cuda -d 1 -m 3200 -a 3 .\bcrypt.txt '?l?l?l?l'
```

| Option | Purpose |
| --- | --- |
| `--backend-ignore-cuda` | Skip the failing CUDA backend; use the available OpenCL path |
| `-d 1` | Select the NVIDIA device from this backend listing |
| `-m 3200` | Interpret the input as bcrypt |
| `-a 3` | Generate candidates using a mask |
| `'?l?l?l?l'` | Test exactly four lowercase letters |

Device numbers are environment-specific. I checked them again after changing backends rather than assuming they would stay the same. This workaround let the attack run; it did not repair the underlying CUDA mismatch.

### A busy GPU can still make slow progress

An early status snapshot showed approximately **100 H/s** and **99% GPU utilisation**. The hash used bcrypt cost **12**, making each guess deliberately expensive.

At a constant 100 guesses per second, testing the entire mask would take roughly 76 minutes. This illustrates the relationship between candidate count and speed; it is not a measured completion time.

The password was recovered. A short password still belonged to a small search space, even though bcrypt made searching it more costly.

## 2. MD4: extending a dictionary attack with rules

For another hash, a straight dictionary attack using `rockyou.txt` did not recover the password. That meant the exact candidates tested had not matched. It did not mean the password was impossible to recover.

I used a rule-based dictionary attack to test variations of the wordlist entries. A rule file can change case, append characters, replace letters or apply other transformations, depending on its contents.

The following is a **reconstructed reference command**, using the `best64.rule` file referenced in my notes:

```powershell
.\hashcat.exe --backend-ignore-cuda -d 1 -m 900 -a 0 .\md4.txt .\rockyou.txt -r .\rules\best64.rule
```

The important change was `-r`: the candidate set expanded from exact wordlist entries to the transformations specified by the rule file.

For example, a rule could capitalise a word or append a suffix. These are illustrative transformations, not an attribution of the recovery to a specific rule.

The recovered password was accepted by the lab. I also independently verified that its **MD4 digest matched the target**. Its MD5 digest did not match. This reinforced another lesson: a 32-character hexadecimal string is not enough to identify a hash algorithm by itself.

## 3. SHA512crypt: fix input parsing before measuring performance

The next case used a `$6$`-prefixed **SHA512crypt** string. This is a salted, iterative password-hashing format, rather than raw SHA-512. Its Hashcat mode is `1800`.

Hashcat initially reported a **Token length exception**. At that point, no cracking was taking place: Hashcat had not successfully parsed the input.

I rewrote the complete hash into a clean text file. These templates show the file-writing approach; replace `PASTE_COMPLETE_HASH_HERE` with the full hash, including its dollar signs and embedded salt.

**Linux/Bash:**

```bash
printf '%s\n' 'PASTE_COMPLETE_HASH_HERE' > sha512crypt.txt
```

**Windows PowerShell:**

```powershell
Set-Content -Path .\sha512crypt.txt -Encoding ascii -Value 'PASTE_COMPLETE_HASH_HERE'
```

Single quotes preserve the dollar signs literally in these shells. The salt was already embedded in the SHA512crypt string, so I did not append a second copy.

Rewriting the input addressed the parsing problem. An incorrect mode, missing characters, extra whitespace or encoding issues are possibilities to investigate when this error occurs.

Once the hash loaded, CPU processing was still slow. I moved this attack to my local GPU, using the extracted wordlist:

```powershell
.\hashcat.exe --backend-ignore-cuda -d 1 -m 1800 -a 0 .\sha512crypt.txt 'C:\path\to\rockyou.txt' --status
```

The recovery succeeded, and I subsequently completed the challenge.

## 4. HMAC-SHA1: supplying the digest and the known key

The final case needed two pieces of input: the target digest and a known value described by the lab as a salt.

Unlike plain `SHA1(password)`, this challenge matched:

```text
HMAC-SHA1(key = known lab value, message = candidate password)
```

Hashcat mode **160** uses the supplied salt field as the HMAC key. The file therefore needed a line in this form:

```text
TARGET_DIGEST:KNOWN_KEY
```

Here, **“salt” names Hashcat's input field; the value's role in the calculation is an HMAC key**. A password-storage salt is normally public and unique per password record. A real HMAC key generally needs to remain secret for HMAC to provide authentication. In this lab, the key was deliberately supplied so the unknown password could be tested as the message. This does not make salts and secret keys interchangeable.

The colon separates the fields in Hashcat's input format. It is not part of the password. A reference dictionary command is:

```powershell
.\hashcat.exe --backend-ignore-cuda -d 1 -m 160 -a 0 .\hmacsha1.txt 'C:\path\to\rockyou.txt'
```

The distinction between modes mattered:

| Mode | Construction |
| --- | --- |
| `100` | Plain SHA-1 |
| `150` | HMAC-SHA1 with the candidate password as the key |
| `160` | HMAC-SHA1 with the supplied salt field as the key |

An explanation I initially received reversed the key and message while recommending mode 160. I checked both arrangements independently. Only **known key + recovered password as message** reproduced the target digest.

The accepted answer and this verification confirmed the construction. The broader lesson was to verify the calculation, rather than trust an algorithm label alone.

### Reproduce the verification with synthetic data

This Python example uses invented values, not the challenge answer. It runs with the standard library:

```python
import hashlib
import hmac

known_key = b"portfolio-demo-key"
candidate = b"Cedar42!"

mode_160 = hmac.new(known_key, candidate, hashlib.sha1).hexdigest()
mode_150 = hmac.new(candidate, known_key, hashlib.sha1).hexdigest()

print("key = known value:", mode_160)
print("key = password:   ", mode_150)
assert mode_160 != mode_150
```

Expected output:

```text
key = known value: 540d309a8ac2482e63f80304b5232f91ee806633
key = password:    eed5c0288c6c7ff2400b0aee73954a812ec4504d
```

Swapping the inputs produces different digests. Once a candidate is recovered, calculating the construction independently and comparing its digest with the target provides a useful cross-check. This example demonstrates the calculation; it does not endorse HMAC-SHA1 as a password-storage scheme.

## Recovering results after the terminal closes

After the overnight bcrypt run, I first looked for the password in the original hash file. Hashcat had not written it there: by default, successful recoveries are stored separately in its **potfile**.

From the same Hashcat environment, I retrieved the saved recovery with:

```powershell
.\hashcat.exe -m 3200 .\bcrypt.txt --show
```

For the SHA512crypt case:

```powershell
.\hashcat.exe -m 1800 .\sha512crypt.txt --show
```

`--show` reads saved results; it does not start a new attack. If a custom potfile was used, the same potfile must be selected when retrieving them.

For an interrupted session with saved restore data, I can attempt:

```powershell
.\hashcat.exe --restore
```

I also encountered a separate PowerShell issue when I typed `hashcat` instead of `.\hashcat.exe`. The `.\` prefix explicitly runs the executable in the current directory. That command-launch error did not mean the earlier attack had failed.

## Evidence and limitations

This account is based on retained study notes, terminal excerpts, accepted-answer screenshots and the completion badge. Original terminal excerpts support the CUDA error and the early bcrypt speed snapshot. The laptop later shut down or restarted; final timing logs for bcrypt and SHA512crypt were not retained, so no measured completion-time or CPU-to-GPU speedup claim is made. The original malformed SHA512crypt file and the matching MD4 rule/base word were also not retained, limiting attribution of those causes. Reference commands are labelled or presented as templates, with public-facing filenames substituted. The synthetic HMAC example was executed independently. The badge supports room completion; it is not performance evidence.

## What I would carry into a password audit

- **Identify the construction before choosing a mode.** Raw SHA-512, SHA512crypt and HMAC-SHA1 have different input requirements and costs.
- **Choose candidates deliberately.** A known pattern can justify a mask; word-based passwords may justify dictionary transformations. Neither approach covers every possible password.
- **Separate failure types.** A parsing error, a GPU compilation error and an exhausted candidate set require different fixes.
- **Inspect actual execution.** A detected GPU does not guarantee working kernels; a running process does not guarantee useful candidate coverage.
- **Keep better evidence.** For future runs, I would retain the exact command, device/backend information, wordlist provenance, final status and elapsed time. For rules, I would also record the matching transformation where possible.

From a defensive perspective, the lab showed why fast general-purpose hashes such as MD4 are unsuitable for password storage. **Password-storage salts** are normally public; unique salts make precomputation and reuse across accounts harder, but do not stop guessing against an individual hash. This differs from an HMAC authentication key, which generally must remain secret. A suitable password-hashing scheme with an appropriate work factor increases the cost of every guess; longer, unpredictable passwords expand the candidate space.

This was a focused exercise in offline password recovery. It gave me practical experience choosing attacks, handling salted formats and diagnosing tool failures—skills I can build on in broader penetration-testing labs.

## References

- [Hashcat: hash modes and example formats](https://hashcat.net/wiki/doku.php?id=example_hashes)
- [Hashcat: mask attacks](https://hashcat.net/wiki/doku.php?id=mask_attack)
- [Hashcat: rule-based attacks](https://hashcat.net/wiki/doku.php?id=rule_based_attack)
- [Hashcat: frequently asked questions](https://hashcat.net/wiki/doku.php?id=frequently_asked_questions)
- [Hashcat: command-line options](https://github.com/hashcat/hashcat/blob/v7.1.2/docs/hashcat-help.md)
