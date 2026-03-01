# Encrypted Backup Guide

> TIP: Store this readme next to the encrypted backup in case you forget what you did. No security through obscurity.

---

### Required

- [GPG](https://www.gnupg.org/)
- [VeraCrypt](https://veracrypt.jp/)
- [Diceware](https://www.eff.org/dice)
- Hyper-V or similar VM with Linux Ubuntu (disable network adapter after setup)
- Recommended: alternative or extended diceware wordlist to increase maximum entropy

---
# Preamble.

Security is tough, but we can make it easier by following these steps below.

### Mindset: Assume, Rotate, Move On

Each 6 months, assume all your current credentials are compromised. Mark your calendar.

No need to scour the internet or dark web checking for compromised accounts - simply accept and assume that they are. Checking for compromised accounts on external services leads to `oracular escalation`: you query databases or websites that fingerprint you, correlating your browser/TLS stack + JA3 hash + IP to your account details and email address. These services could be honeypots run by agencies or criminals. Fake breach notification services are trivially easy to set up, rank highly in search, and harvest exactly the data you came to protect.

Don't escalate. Sometimes the most secure action is no action. Assume, rotate, move on. Leave no trace of the process itself.

> WARNING: Panic leads to very bad security decisions. Breach notifications are designed to trigger an emotional response - "MY credentials are exposed" - which bypasses critical thinking. Attackers exploit this deliberately. A calm, scheduled rotation is more secure than any reactive response. Your credentials are not your identity. They are temporary tokens on a rotation schedule. Nothing to panic about.

---

### Precautions

If possible, boot a Hyper-V Ubuntu virtual image without a network adapter as your encryption workspace. Perform all encryption operations inside that VM. This prevents existing spyware, ISP monitoring, malware, keyloggers, and RATs from observing your encryption practice. You might not have them - but it is better to assume they already exist.

One-time network window: You need internet briefly to download VeraCrypt and GPG. Enable the network adapter, download, verify hashes, then remove the adapter permanently. Total window: under 10 minutes. The VM never needs internet again.

---

### Every 6 Months

Take a weekend and re-run this guide, then rotate everything:

| Item                  | Action                                        |
|-----------------------|-----------------------------------------------|
| All passwords         | Rotate entirely                               |
| Diceware passphrases  | Generate fresh ones                           |
| SSH keys              | Regenerate                                    |
| API keys              | Revoke and reissue                            |
| GPG keys              | Consider rotation                             |
| VeraCrypt containers  | Re-encrypt with new passphrase                |
| Cold storage          | New container, new GPG wrap, new passphrases  |
| 2FA seeds             | Audit which services have it enabled          |
| Password manager      | Consider rotating master passphrase           |

---

### Overview

This guide documents a two-layer encryption strategy for secure cold storage backups. The VeraCrypt container holds sensitive files. GPG wraps the container as an independent outer layer. Both must be broken simultaneously for data to be exposed - using different tools, different algorithms, and different diceware passphrases.

This prevents:

- Supply chain / MITM compromise
- Compromised VeraCrypt instance or update
- Compromised GPG instance or update
- Code injection requires both software simultaneously (very unlikely)
- Both software requires simultaneous zerodays (very unlikely)
- XZ-style social engineering attacks
- Fresh weakness in one package - the other buffers it
- Post-quantum `store now, decrypt later` practices (already happening)

Dual diceware passphrases are the cherry on the cake. If long, >= 20 words is recommended.

20 words gives ~257 bits of entropy - comparable to counting every atom in the observable universe. If some future mathematical breakthrough reduced entropy estimates dramatically, 257 bits has enormous margin to absorb that. This also defeats post-quantum attacks: Grover's algorithm halves effective bits, leaving ~128 bits - still unbreakable.

> NOTE: Precautions are good. 15 years ago a 12-character password was considered secure. Today that advice is laughable. What is laughable in 2040 is being decided by choices made today.

--- 

### Security Model

```
cold-storage.vc.gpg        - what sits on cold storage / cloud
  └── cold-storage.vc      - after GPG decryption
        └── /mnt/vault     - after VeraCrypt mount
              └── your files
```

| Layer | Tool      | Algorithm             | Passphrase            |
|-------|-----------|-----------------------|-----------------------|
| Outer | GPG       | AES-256 + SHA-512 KDF | Diceware passphrase A |
| Inner | VeraCrypt | AES + PIM 1000        | Diceware passphrase B |

Neither passphrase is ever written near the other. Neither tool shares code with the other.

---

### Choose Filesystem

Think about this before creating your container.

| Filesystem | Use case                    | Notes                                              |
|------------|-----------------------------|----------------------------------------------------|
| FAT        | Cross-platform, files < 4GB | Requires `--fs-options="iocharset=utf8"` on mount  |
| exFAT      | Cross-platform, files > 4GB | Better FAT alternative                             |
| ext4       | Linux only                  | Native UTF-8, permissions, no file size limit      |

If your vault lives entirely in Linux - use ext4. No charset issues, no file size limits.

---

### Create VeraCrypt Container

ext4 (Linux only, recommended):
```
veracrypt --text \
  --create cold-storage.vc \
  --encryption AES \
  --hash SHA-512 \
  --filesystem ext4 \
  --pim 1000 \
  --size 500M
```

FAT (cross-platform, files < 4GB):
```
veracrypt --text \
  --create cold-storage.vc \
  --encryption AES \
  --hash SHA-512 \
  --filesystem FAT \
  --pim 1000 \
  --size 500M
```

> `--pim 1000` significantly increases key derivation iterations. Cold storage is opened rarely, so the slower unlock time is an acceptable trade-off for much stronger brute-force resistance. VeraCrypt will prompt securely for the passphrase - never pass it as a command-line argument.

---

### Mount and Unmount

```
# Create mount point (once per fresh install)
cd ~
sudo mkdir /mnt/vault

# Mount (ext4)
veracrypt --text \
  --mount cold-storage.vc /mnt/vault \
  --pim 1000

# Mount (FAT - requires charset flag)
veracrypt --text \
  --mount cold-storage.vc /mnt/vault \
  --pim 1000 \
  --fs-options="iocharset=utf8"

# Test
cd /mnt/vault
echo "this is a test" > test.txt
cat test.txt

# Unmount - must cd out first
cd ~
veracrypt --text --dismount /mnt/vault

# Verify fully unmounted
veracrypt --list

# Remove mount point
sudo rm -rf /mnt/vault
```

> Always dismount immediately after use. You cannot dismount while your terminal is inside the vault directory - always `cd ~` first.

---

### Wrap With GPG (Cold Storage)

```
gpg --symmetric \
  --cipher-algo AES256 \
  --s2k-mode 3 \
  --s2k-count 65011712 \
  --s2k-digest-algo SHA512 \
  cold-storage.vc
```

This produces `cold-storage.vc.gpg`. Verify before deleting the original:

```
# Test decrypt - verify integrity without saving output
gpg --decrypt cold-storage.vc.gpg > /dev/null
```

If no errors - GPG wrap is good. Then securely delete the original:

```
shred -u cold-storage.vc
```

> `--s2k-count 65011712` sets key derivation to maximum iterations. Combined with VeraCrypt's PIM 1000, brute-forcing either layer becomes computationally infeasible with current and foreseeable hardware including quantum computers.

> NOTE: GPG caches your passphrase in the GPG agent during the current session. On a fresh system it will always prompt. Your cold storage is properly protected - the cache is session-only and dies when the VM shuts down.

---

### Emergency Restore Sequence

Step 1 - Decrypt GPG outer layer
```
gpg --output cold-storage.vc \
    --decrypt cold-storage.vc.gpg
```

Step 2 - Create mount point and mount VeraCrypt
```
cd ~
sudo mkdir /mnt/vault

veracrypt --text \
  --mount cold-storage.vc /mnt/vault \
  --pim 1000
```

Step 3 - Copy files out
```
cp -r /mnt/vault/* /destination/
```

Step 4 - Dismount and clean up
```
cd ~
veracrypt --text --dismount /mnt/vault
veracrypt --list
sudo rm -rf /mnt/vault
```

Step 5 - Securely delete decrypted container
```
shred -u cold-storage.vc
```

> Step 5 is critical. The decrypted `.vc` file sitting on disk after restoration is a liability. Shred it immediately once files are safely copied out.

---

### Passphrase Notes

- Generate both passphrases with Diceware (6 words minimum, 20 words recommended)
- Never store both passphrases in the same location
- Never pass passphrases as command-line arguments - let tools prompt interactively
- Store passphrases in different physical locations - e.g. one memorised, one in a sealed envelope offsite
- Rotate both passphrases every 6 months with new containers

---

### Backup Storage - 3-2-1 Rule

| # | Location                  | Media                               |
|---|---------------------------|-------------------------------------|
| 1 | Local (e.g. external SSD) | SSD or USB                          |
| 2 | Second local device       | Different media type                |
| 3 | Offsite                   | Trusted location or encrypted cloud |

Test restores periodically. A backup never tested is not a backup.

---

### Pro-Tips

> Pro-tip 1 - Software Escrow: Store a copy of both GPG and VeraCrypt installers separately together with their known hashes. If a package gets compromised at the maintainer level, you can fall back on the older uncompromised copy. Always check and compare hashes before use. Verify against multiple independent sources - official website, GitHub releases, and your stored hash should all match. Store these copies on your cold storage media alongside your encrypted containers.

> Pro-tip 2 - Source builds: If paranoid, build both tools from source. However this introduces new attack angles - you must download many additional packages and build dependencies, each a potential risk. Only do this if you understand the full dependency chain. A safer middle ground: build from source, then run the binary in an isolated VM for several days and monitor all network traffic. VeraCrypt and GPG should produce zero network connections - any outbound traffic is suspicious. (Expert mode only.)

> Pro-tip 3 - Post-compromise buffer: If a fresh vulnerability is discovered in one package, the other layer buffers it completely. Combined with your software escrow copies, you can fall back to a known-good version and re-encrypt at the next rotation cycle - without ever being exposed. If encryption already happened before the vulnerability was discovered, existing containers are unaffected entirely.
