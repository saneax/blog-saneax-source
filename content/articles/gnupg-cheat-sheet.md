---
title: "GnuPG (GPG) Command Cheat Sheet: Encryption & Signing Guide"
date: "2026-04-30T08:06:59+00:00"
draft: false
type: "post"
tags: ["gpg", "security", "encryption", "linux", "cheat-sheet", "privacy", "gnupg"]
categories: ["Security", "DevOps", "Tools"]
description: "A comprehensive GnuPG cheat sheet covering key management, encryption, signing, and advanced features. Includes SEO-friendly commands and best practices for secure communication."
summary: "Quick reference guide for GnuPG commands including key generation, encryption, decryption, signing, and advanced security features."
author: "Sanjay Upadhyay"
toc: true
slug: "gnupg-cheat-sheet"
aliases: ["/gpg-commands/", "/encryption-guide/"]
---

GNU Privacy Guard (GnuPG or GPG) is the standard for secure communication and file encryption on Linux and Unix-like systems. Whether you are signing Git commits, encrypting emails, or securing sensitive files, having a quick reference for commands is essential.

This cheat sheet covers the most commonly used features, as well as lesser-known but powerful utilities, organized by workflow.

> **Note:** On most modern systems, the command is `gpg`. On older systems, it might be `gpg2`. Replace `[KEY_ID]` with the key fingerprint or email, and `[FILE]` with your filename.

## 1. Key Management

Managing your keyring is the first step to secure communication.

| Action | Command | Description |
| :--- | :--- | :--- |
| **Generate Key** | `gpg --full-generate-key` | Interactive key generation (recommended). |
| **List Public Keys** | `gpg --list-keys` | Show all public keys in your keyring. |
| **List Secret Keys** | `gpg --list-secret-keys` | Show all private keys in your keyring. |
| **Export Public Key** | `gpg --armor --export [KEY_ID] > pubkey.asc` | Export to ASCII file for sharing. |
| **Export Secret Key** | `gpg --armor --export-secret-keys [KEY_ID] > seckey.asc` | **⚠️ Backup only!** Keep this safe. |
| **Import Key** | `gpg --import [FILE]` | Import a public or private key file. |
| **Delete Public Key** | `gpg --delete-key [KEY_ID]` | Remove a public key from your ring. |
| **Delete Secret Key** | `gpg --delete-secret-key [KEY_ID]` | Remove a private key from your ring. |
| **Show Fingerprint** | `gpg --fingerprint [KEY_ID]` | Verify key identity (crucial for trust). |

## 2. Encryption & Decryption

Encrypt files for specific recipients or using symmetric passwords.

| Action | Command | Description |
| :--- | :--- | :--- |
| **Encrypt File** | `gpg --encrypt --armor --recipient [EMAIL] [FILE]` | Encrypt for a specific user (ASCII output). |
| **Encrypt (Binary)** | `gpg --encrypt --recipient [EMAIL] [FILE]` | Encrypt for a specific user (binary output). |
| **Decrypt File** | `gpg --decrypt [FILE].gpg > [FILE]` | Decrypt a file (requires private key). |
| **Decrypt & View** | `gpg --decrypt [FILE].gpg` | Decrypt and print content to stdout. |
| **Symmetric Encrypt** | `gpg --symmetric --armor [FILE]` | Encrypt with a password only (no keys). |
| **Symmetric Decrypt** | `gpg --decrypt [FILE].asc` | Decrypt password-protected file. |
| **Encrypt for Self** | `gpg --encrypt --recipient [YOUR_EMAIL] [FILE]` | Encrypt so only you can open it. |

## 3. Signing & Verification

Ensure integrity and authenticity of your files and messages.

| Action | Command | Description |
| :--- | :--- | :--- |
| **Clearsign Text** | `gpg --clearsign [FILE]` | Sign text; content remains readable. |
| **Detach Sign** | `gpg --detach-sign --armor [FILE]` | Create a separate `.sig` file (good for binaries). |
| **Sign & Encrypt** | `gpg --encrypt --sign --recipient [EMAIL] [FILE]` | Both encrypt and sign in one step. |
| **Verify Signature** | `gpg --verify [FILE].sig [FILE]` | Verify a detached signature. |
| **Verify Clearsign** | `gpg --verify [FILE].asc` | Verify a clearsigned document. |

## 4. Key Server Operations

*Note: Key server infrastructure has changed. `keys.openpgp.org` is currently the most reliable.*

| Action | Command | Description |
| :--- | :--- | :--- |
| **Receive Key** | `gpg --keyserver keys.openpgp.org --recv-keys [KEY_ID]` | Download a public key from server. |
| **Send Key** | `gpg --keyserver keys.openpgp.org --send-keys [KEY_ID]` | Upload your public key to server. |
| **Search Key** | `gpg --keyserver keys.openpgp.org --search-keys [EMAIL]` | Search for a key by email/name. |
| **Refresh Keys** | `gpg --keyserver keys.openpgp.org --refresh-keys` | Update existing keys (revocations/expiry). |

## 5. Advanced & Lesser-Known Features

These commands are useful for automation, hardening security, or troubleshooting.

| Feature | Command | Why it's useful |
| :--- | :--- | :--- |
| **Hide Recipient** | `gpg --hidden-recipient [EMAIL] [FILE]` | Hides the target ID in the encrypted packet. |
| **Edit Key** | `gpg --edit-key [KEY_ID]` | Modify expiry, add UIDs, add subkeys. |
| **Gen Revoke Cert** | `gpg --gen-revoke [KEY_ID]` | Create a certificate to invalidate your key if lost. |
| **Update Trust DB** | `gpg --update-trustdb` | Force GPG to recalculate owner trust values. |
| **Clean Keyring** | `gpg --clean-keys` | Remove unused/invalid signatures from keys. |
| **Script Mode** | `gpg --batch --yes --passphrase "PWD" ...` | **⚠️ Risky.** Use for automation (requires `pinentry-mode loopback`). |
| **SSH Support** | `gpg --export-ssh-key [KEY_ID]` | Export GPG key for use in SSH authentication. |

## 6. Configuration Tips (`~/.gnupg/gpg.conf`)

Add these lines to your config file for better defaults and security.

```bash
# Always use armor (ASCII) for easier copy/paste
armor

# Show long key IDs (more secure than short IDs)
keyid-format long

# Preferred algorithms (Modern standards)
personal-cipher-preferences AES256
personal-digest-preferences SHA512
personal-compress-preferences ZLIB BZIP2 ZIP Uncompressed

# Auto-key retrieval (convenient but privacy trade-off)
# auto-key-retrieve
{{< comments >}}
