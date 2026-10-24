# Second Shift: DEADFACE Relay Write-up

**Category:** Crypto / Forensics  
**Flag:** `deadface{ctr_nonce_reuse_breaks_confidentiality}`

## Challenge

ZeroTrace recovered files from a DEADFACE relay server used by `mort1cia`. One routine transmission was available in two forms: its original plaintext and its encrypted version after relay processing. A second outbound transmission was recovered only in its encrypted form.

The objective was to decrypt the second transmission, understand what DEADFACE was preparing to do, and recover the flag.

## Files

| File | Description |
|---|---|
| `relay-heartbeat.json` | Known plaintext of the routine heartbeat (251 bytes) |
| `relay-heartbeat.enc` | The same heartbeat after relay processing (250 bytes) |
| `relay-tasking.enc` | Second outbound transmission, available only in its processed form (220 bytes) |

The known heartbeat plaintext contains the following JSON:

```json
{"type":"heartbeat","host":"relay-07","user":"mort1cia","status":"ok","queue":3,"channel":"ops","timestamp":"2026-10-16T23:41:08Z","message":"No anomalies detected. Continuing scheduled relay checks.","build":"relay-agent-3.8.14","region":"us-south"}
```

## Analysis

### 1. Identifying the weakness

The first step was to compare the two encrypted files, `relay-heartbeat.enc` and `relay-tasking.enc`.

The first 9 bytes of both ciphertexts are identical:

```text
relay-heartbeat.enc: 16 b7 a1 33 9f ab 35 6a ef 40 4a 77 ...
relay-tasking.enc:   16 b7 a1 33 9f ab 35 6a ef 5c 4e 65 ...
```

Both plaintext messages begin with `{"type":"`. Since the corresponding ciphertext bytes are also identical, this suggests that the same keystream was used for both messages at the beginning.

This is a strong indication of **keystream reuse**, which can happen when a stream cipher or a block cipher in CTR mode uses the same key and nonce for multiple messages.

In CTR mode, a nonce helps generate the keystream used to encrypt a message. If the same key and nonce are reused, the same keystream can be generated again. This creates a serious security weakness because a known plaintext can be used to recover the keystream and decrypt other messages encrypted with it.

### 2. Recovering the keystream

Stream ciphers and CTR mode encryption use XOR to combine the plaintext with the keystream.

The relationship is:

```text
ciphertext = plaintext XOR keystream
```

Because XOR can reverse itself, we can recover the keystream when both the plaintext and ciphertext are known:

```text
keystream = plaintext XOR ciphertext
```

The file `relay-heartbeat.json` provides the original plaintext, while `relay-heartbeat.enc` provides its corresponding ciphertext.

I used **CyberChef** instead of a Python script to perform the XOR operation and recover the keystream.

The recovered keystream was 250 bytes long, which was enough to decrypt the entire 220-byte `relay-tasking.enc` file.

### 3. Decrypting the tasking message

Once the keystream was recovered, the same XOR operation could be applied to the second ciphertext.

The decryption formula is:

```text
tasking_plaintext = tasking_ciphertext XOR keystream
```

In CyberChef, I used the recovered keystream and the tasking ciphertext as inputs to the XOR operation. Since the keystream was reused, this revealed the original contents of the second transmission without needing to know the encryption key.

The decrypted message was:

## Decrypted output

```json
{"type":"tasking","target":"aurora-vpn","action":"stage-access","window":"00:30Z","operator":"mort1cia","note":"Use the backup tunnel. Archive bundle under DEADFACE{ctr_nonce_reuse_breaks_confidentiality} before exfil."}
```

## What DEADFACE was preparing

The decrypted message revealed the following details about the planned operation:

- **Target:** `aurora-vpn` — the VPN infrastructure DEADFACE was targeting.
- **Action:** `stage-access` — prepare access to the target ahead of the main operation.
- **Window:** `00:30Z` — the planned activity window, expressed in UTC.
- **Operator:** `mort1cia` — the operator associated with the transmission.
- **Method:** Use the backup tunnel, archive a data bundle, and then exfiltrate it.

In short, DEADFACE was preparing to stage access to the Aurora VPN and exfiltrate an archived data bundle through a backup tunnel during the 00:30Z window.

## Flag

```text
deadface{ctr_nonce_reuse_breaks_confidentiality}
```

## Notes

- The heartbeat plaintext is 251 bytes, while its encrypted file is 250 bytes. The ciphertext is probably missing a trailing newline that exists in the plaintext file.
- This difference does not affect the solution. CyberChef's XOR operation can process the matching bytes, and the recovered keystream is long enough to decrypt the entire 220-byte tasking message.
- The decrypted message contains the flag in uppercase as `DEADFACE{ctr_nonce_reuse_breaks_confidentiality}`. The flag is submitted in the lowercase format required by the challenge: `deadface{ctr_nonce_reuse_breaks_confidentiality}`.

## Takeaway

Reusing a nonce in CTR mode, or reusing a keystream in a stream cipher, can completely compromise confidentiality.

When the same key and nonce generate the same keystream for multiple messages, an attacker who knows one plaintext and its corresponding ciphertext can recover the keystream bytes. Those bytes can then be used to decrypt other messages encrypted with the same keystream, without brute-forcing the key or recovering the secret key itself.

The proper fix is to use a unique nonce for every encryption under the same key. Authenticated encryption methods such as AES-GCM and ChaCha20-Poly1305 should also be used with correct nonce management to protect both confidentiality and integrity.
