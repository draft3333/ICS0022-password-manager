# Checkpoint 1: threat model

ICS0022 Secure Programming

This document covers the proposed [password manager architecture](checkpoint-1.md). Mitigations are planned controls, not claims of completed implementation or testing.

## Scope and assumptions

The assets to protect are the master password, derived keys, saved credentials and the integrity and availability of the vault.

The model considers someone who obtains vault files, modifies local data, submits malicious input or gains access to an unattended session. The OS, executable and dependencies are assumed trustworthy. Malware, a keylogger or an administrator controlling the running system can bypass application-level protections.

The main trust boundaries are terminal input entering the application, stored files being loaded and any plaintext leaving application memory for display or copying. Internal modules share one process.

## Master password

**T1. Offline password guessing.** An attacker with copied vault files could try passwords without using the application.

**Intended mitigations:**

- Use Argon2id with a random salt and measured cost settings.
- Encourage a long, unique master password.
- Do not store the plaintext password or derived keys.

A weak password can still be guessed. Delays in the interface cannot stop offline attacks.

**T2. Password exposure during entry or error handling.** Terminal output, command history or logs could reveal the master password.

**Intended mitigations:**

- Read the master password through a hidden interactive prompt, never through command-line arguments.
- Exclude secrets from logs and error messages.
- Clear the input buffer after use.

This does not prevent keylogging on a compromised machine.

## Vault at rest

**T3. Disclosure from stolen files or backups.** Someone could read credentials from accessible vault files.

**Intended mitigations:**

- Encrypt all credential fields with XChaCha20-Poly1305.
- Restrict file permissions to the current OS user.
- Store only encrypted backups and exclude runtime data from Git.

Unlock metadata and file size will remain visible.

**T4. Modification or substitution of vault contents.** An attacker could alter ciphertext, remove records or replace files.

**Intended mitigations:**

- Verify authenticated encryption and authenticate the complete vault structure, including security-relevant metadata.
- Use fresh nonces.
- Reject failed integrity checks before exposing entries.

The exact format remains to be specified. Replacement with an older authentic vault is not prevented by authentication alone.

**T5. Data loss during saving or deletion.** An interrupted write, concurrent save or deleted file could make credentials unavailable.

**Intended mitigations:**

- Plan safe replacement using a temporary encrypted file.
- Preserve the previous file on write failure and prevent conflicting writes.
- Document encrypted backups.

These measures cannot guarantee recovery from every disk failure or deliberate deletion.

**T6. Malicious file fields exhausting resources or confusing the parser.** Forged lengths or excessive key-derivation costs could trigger large allocations, long processing or unsafe memory access.

**Intended mitigations:**

- Bound file sizes, field lengths and supported KDF settings before allocation or derivation.
- Reject unknown versions and malformed data.

Authentication does not remove the need for safe parsing.

## Vault in memory

**T7. Secrets remaining after use.** Unnecessary copies, released buffers or swap files could retain passwords and keys.

**Intended mitigations:**

- Minimize plaintext lifetime.
- Use dedicated buffers with automatic cleanup and libsodium wiping.
- Attempt memory locking with checked results.
- Clear session keys on lock and temporary secrets after use.

OS-managed copies, crash dumps and CPU registers are outside a complete wiping guarantee.

**T8. Access through an unattended unlocked session.** Another person could retrieve credentials without knowing the master password.

**Intended mitigations:**

- Provide explicit locking.
- Release session secrets on lock or normal exit.
- Require successful unlocking before every session's entry operations.

Automatic idle locking remains an open decision. Manual locking depends on user action.

## Interface

**T9. Malicious text or paths.** Oversized input, terminal control characters or path traversal could disrupt the interface or access unintended files.

**Intended mitigations:**

- Validate input lengths, encoding and allowed characters.
- Keep storage paths within the intended data directory.
- Avoid passing user input to shell commands.
- Validate both entered and loaded values.

**T10. Disclosure through output or clipboard use.** Displayed information may remain in terminal history. Copied passwords may be read by other programs.

**Intended mitigations:**

- Display only necessary entry information and keep passwords out of ordinary output and logs.
- If clipboard retrieval is chosen, require an explicit action and plan timed cleanup that preserves newer clipboard content.

Cleanup cannot revoke copies already obtained by another application.

## Planned verification

During implementation, verify that incorrect passwords and modified vaults are rejected, locked sessions cannot access entries, malformed input fails safely, failed saves preserve prior data and application output contains no secrets. Review secret-buffer cleanup separately; passing functional tests will not prove complete memory erasure.

## References

- [OWASP threat modeling guidance](https://cheatsheetseries.owasp.org/cheatsheets/Threat_Modeling_Cheat_Sheet.html)
- [libsodium password hashing](https://doc.libsodium.org/password_hashing/default_phf)
- [libsodium authenticated encryption](https://libsodium.gitbook.io/doc/secret-key_cryptography/aead/chacha20-poly1305/xchacha20-poly1305_construction)
- [libsodium secure memory](https://doc.libsodium.org/memory_management)
