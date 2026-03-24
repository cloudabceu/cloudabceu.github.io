---
layout: post
title: GPG Key Management - Create, Backup, Import, and Edit User ID
categories: [GPG, Security, Linux]
toc: true
date: 2026-03-10
---

This post documents the process of creating a new GPG key, backing up a GPG key, importing it to a new machine, and editing the user ID (UID) associated with the key. This is useful for administrators who need to migrate GPG keys, update key metadata, or generate new keys for secure communication.

This post documents the process of backing up a GPG key, importing it to a new machine, and editing the user ID (UID) associated with the key. This is useful for administrators who need to migrate GPG keys or update key metadata.

<!--more-->

## 0. Create a New GPG Key

To generate a new GPG key pair (public and private key):

```bash
gpg --full-generate-key
```

You will be prompted for:
- Key type (default RSA and RSA is fine)
- Key size (2048 or 4096 recommended)
- Expiry (choose as needed)
- Real name, email, and optional comment

Example:

```
Please select what kind of key you want:
   (1) RSA and RSA (default)
Your selection? 1
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 4096
Key is valid for? (0) 0
Real name: CloudABC
Email address: blog@cloudabc.eu
Comment:
You selected this USER-ID:
    "CloudABC <blog@cloudabc.eu>"
Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
```

After creation, list your keys with:

```bash
gpg --list-secret-keys
```

## 1. Backup the Public and Private Key

On the source machine, export both the public and private keys:

```bash
# Import the public key if not already present
gpg --import cloudabc-public.gpg

# Export the private key in ASCII-armored format
gpg --armor --export-secret-keys BCEEF176584DFFFF > cloudabc-private.gpg
```

## 2. Import the Keys on the New Machine

Transfer the exported key files to the new machine, then import them:

```bash
gpg --import cloudabc-private.gpg
gpg --import cloudabc-public.gpg

gpg --list-secret-keys
```

Example output:

```
/root/.gnupg/pubring.kbx
------------------------
sec   rsa2048 2025-10-24 [SC]
      72030CA118C1A27568B137C4BCEEF176584DFFFF
uid           [ unknown] Dummy <dummy@cloudabc.eu>
ssb   rsa2048 2025-10-24 [E]
```

## 3. Add a New User ID (UID) to the Key

Edit the key to add a new UID:

```bash
gpg --edit-key BCEEF176584DFFFF
```

At the `gpg>` prompt:

```
gpg> uid
gpg> adduid
Real name: CloudABC
Email address: blog@cloudabc.eu
Comment: [leave blank]
You selected this USER-ID:
    "CloudABC <blog@cloudabc.eu>"
Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
```

You should now see both the old and new UIDs:

```
gpg> uid
[ unknown] (1). Dummy <dummy@cloudabc.eu>
[ unknown] (2)  CloudABC <blog@cloudabc.eu>
```

## 4. Delete the Old User ID

Select the old UID and delete it:

```
gpg> uid 1
gpg> deluid
Really remove this user ID? (y/N) y
```

Verify only the new UID remains:

```
gpg> uid
[ unknown] (1)  CloudABC <blog@cloudabc.eu>
```

Save and exit:

```
gpg> save
```

## Summary

Always backup both public and private keys before making changes.
You can import/export keys between machines using GPG's import/export commands.
Use `gpg --edit-key` to add or remove user IDs as needed.

Some tutorials say you must set a UID as "sign" or "primary" when adding or changing user IDs, but in my testing, these steps are not needed for basic key management and usage.

This process ensures your GPG key is portable and up-to-date with the correct user information.