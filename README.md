# Encrypted Backup Guide

> TIP: Store this readme next to the encrypted backup. In case your forget what you did. No security through obscurity.

### Required

- GPG
- Veracrypt
- HyperV or similar VM container with Linux Ubuntu. (disable network adapter)
- Diceware; https://www.eff.org/dice
- Would recommend a good (alternative) wordlist to increase maximum entropy.

Each 6 months, assume your all your current credentials are compromised. (mark your calendar each 6 months)

Then proceed below:

### Precautions

TIP: If possible, boot up a `HyperV` virtual image without network-adapter, as encryption workspace. And proceed all encryption in that VM instance. This prevents existing hidden spyware, ISP snapshots, malware, loggers, rats from spying on your encryption practice. You might not have them, but it is better to assume they already exist.

### Every 6 months

Take a weekend and re-run this readme, and then also:

```
| Item                   | Action                                         |
|------------------------|------------------------------------------------|
| All passwords          | Rotate entirely                                |
| Diceware passphrases   | Generate fresh ones                            |
| SSH keys               | Regenerate                                     |
| API keys               | Revoke and reissue                             |
| GPG keys               | Consider rotation                              |
| VeraCrypt containers   | Re-encrypt with new passphrase                 |
| Cold storage           | New container, new GPG wrap, new passphrases   |
| 2FA seeds              | Audit which services have it enabled           |
| Passsword manager      | Consider rotating                              |
```

### Overview

This guide documents a two-layer encryption strategy for secure cold storage backups.
The VeraCrypt container holds sensitive files. GPG wraps the container as an independent
outer layer. Both must be broken simultaneously for data to be exposed - using different
tools, different algorithms, and different (diceware) passphrases.

This prevents:

- Supply chain / MITM compromise
- Compromised Veracrypt instance/update
- Compromised GPG instance/update
- Code needs to be injected in both software (very unlikley)
- Software needs to have simultaneous zerodays (very unlikely)
- XZ-style social engineering
- Fresh weakness found in one software package. (the other buffers it)
- Post-quantum `store now, decrypt later` practices. (already happening)

Dual diceware passwords is the cherry on the cake. If long, it would indicate >= 20 words. 

20 words gives ~257 bits of entropy. Good luck!

257 bits is very strong, because if some future mathematical breakthrough reduced entropy estimates dramatically, 257 bits has enormous margin to absorb that.

> NOTE: precautions are good, because 15 years ago, it was acceptable to use a 12 char password. Today that advise is laughable. 

---

### Security Model

```
cold-storage.vc.gpg        - what sits on cold storage / cloud
  └── cold-storage.vc      - after GPG decryption
        └── /mnt/vault     - after VeraCrypt mount
              └── your files
```

```
| Layer | Tool       | Algorithm             | Passphrase            |
|-------|------------|-----------------------|-----------------------|
| Outer | GPG        | AES-256 + SHA-512 KDF | Diceware passphrase A |
| Inner | VeraCrypt  | AES                   | Diceware passphrase B |
```

Neither passphrase is ever written near the other. Neither tool shares code with the other.

---

### Create VeraCrypt Container

```
veracrypt --text \
  --create cold-storage.vc \
  --encryption AES \
  --hash SHA-512 \
  --filesystem FAT \
  --pim 1000 \
  --size 500M
```

> `--pim 1000` significantly increases key derivation iterations. Cold storage is opened
> rarely, so the slower unlock time is an acceptable trade-off for much stronger brute-force
> resistance. VeraCrypt will prompt securely for the passphrase - do not pass it as an
> argument.

---

### Mount and Unmount

```
# Mount
veracrypt --text \
  --mount cold-storage.vc /mnt/vault \
  --pim 1000

# Unmount when done
veracrypt --text --dismount /mnt/vault
```

> Always dismount immediately after use. Do not leave the container mounted unattended.

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

This produces `cold-storage.vc.gpg`. The original `cold-storage.vc` can be deleted
after verifying the GPG file decrypts correctly.

> `--s2k-count 65011712` sets key derivation to maximum iterations. Combined with
> VeraCrypt's high PIM, brute-forcing either layer becomes computationally infeasible
> with current and foreseeable hardware.

---

### Emergency Restore Sequence

Step 1 - Decrypt GPG outer layer

```
gpg --output cold-storage.vc \
    --decrypt cold-storage.vc.gpg
```

Step 2 - Mount VeraCrypt inner container

```
veracrypt --text \
  --mount cold-storage.vc /mnt/vault \
  --pim 1000
```

Step 3 - Copy files out

```
cp -r /mnt/vault/* /destination/
```

Step 4 - Dismount

```
veracrypt --text --dismount /mnt/vault
```

Step 5 - Securely delete the decrypted container

```
shred -u cold-storage.vc
```

> Step 5 is critical. The decrypted `.vc` file sitting on disk after restoration
> is a liability. Shred it immediately once files are safely copied.

---

### Passphrase Notes

- Both passphrases should be generated with Diceware (6 words minimum, 10 words recomended)
- Never store both passphrases in the same location
- Never pass passphrases as command-line arguments - let tools prompt interactively
- Passphrases stored in different physical locations (e.g. one memorised, one in a sealed envelope offsite)

---

### Backup Storage - 3-2-1 Rule

| # | Location                  | Media                                |
|---|---------------------------|--------------------------------------|
| 1 | Local (e.g. external SSD) | SSD or USB                           |
| 2 | Second local device       | Different media type                 |
| 3 | Offsite                   | Trusted location or encrypted cloud  |

Test restores periodically. A backup never tested is not a backup

> Pro-tip: Software Escrow. Store a copy of both GPG and Veracrypt software seperately together with known hashes. If a package does get compromised on the maintainer, you can fall back on the older uncompromised copy. But first always check and compare.

> Pro-tip 2: if paranoid, build both software from source. Although, this could introduce new angles of attack as you have to download many additional packages that could contain risks such as unvetted code. Only do this if you understand the risks. Its better to run a build, and then let it sit in a isolated vm container for a few days, and log internet traffic to see calls. (expert mode)
