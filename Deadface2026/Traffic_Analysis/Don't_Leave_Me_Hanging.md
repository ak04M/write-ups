# Don't Leave Me Hanging — Write-up

**CTF:** DEADFACE CTF  
**Category:** Traffic Analysis (PCAP / Programming)  
**Points:** 500  
**Author:** @Cyb0rgSw0rd  
**Flag:** `deadface{Pr355_0n3_f0r_D34df4c3}`

## Challenge Description

> Sounds like Deadface are trying to sneak some data out of Aurora Health... but the security is pretty locked down. We suspect that they might be trying to work it from the inside out. The head of security noticed that the traffic for the help desk subnet was flagging for anomalous activity and set the packet capture to us for a set of fresh eyes on the situation.
>
> Submit the answer as `deadface{deadface alias}`.
>
> Download PCAP (80 MB), SHA256: `93bf96711676fa3675d0bd8729ccdf00ffc80ef4ddcc4e1892dfa1a45f11e6f2`

## TL;DR

1. An FTP session revealed that the attacker, `10.20.200.60`, logged in as `mkowalski` and downloaded an Asterisk PBX diagnostics bundle.
2. The bundle contained a TLS key log that allowed us to decrypt SIP signalling traffic.
3. The decrypted SIP messages exposed SDES SRTP keys, allowing us to decrypt the voice calls.
4. One call to extension 2003 contained DTMF tones hidden inside the audio. These tones represented ASCII decimal values that decoded to the flag.

## Tools Used

- `tshark`
- `capinfos`
- `tar`
- Python 3 with `pylibsrtp`
- `faster-whisper` for speech-to-text
- `sox`
- A custom Python DTMF tone detector using the Goertzel algorithm

## Walkthrough

### 1. Inspecting the PCAP

First, I verified the PCAP's SHA256 hash and examined the protocol hierarchy and network conversations.

```bash
shasum -a 256 DontLeaveMeHanging.pcap
tshark -r DontLeaveMeHanging.pcap -q -z io,phs
tshark -r DontLeaveMeHanging.pcap -q -z conv,ip | head -30
```

The capture contained approximately 256,000 frames. Most of the traffic consisted of UDP packets exchanged between the help desk hosts (`10.20.200.21` to `10.20.200.24`) and `10.20.200.5`.

Although Wireshark classified much of this UDP traffic as generic data, it turned out to be SRTP voice traffic that could not be identified properly without the SIP signalling context.

The capture also contained around 12,000 TLS frames, SSH traffic, some HTTP traffic, and 34 FTP frames.

Since FTP was one of the few cleartext protocols in the capture, I investigated it first.

### 2. Finding the Stolen Diagnostics Bundle

I filtered the capture for FTP commands, responses, and file transfers.

```bash
tshark -r DontLeaveMeHanging.pcap -Y "ftp || ftp-data" -T fields \
  -e frame.number -e tcp.stream -e ip.src -e ip.dst \
  -e ftp.request.command -e ftp.request.arg \
  -e ftp.response.code -e ftp.response.arg
```

The results revealed that client `10.20.200.60` connected to the vsFTPd server at `10.20.200.5`, which was the Asterisk PBX server.

The client attempted to log in as `mkowalski`. The first attempt failed with the password `Aur0ra!Helpdesk25`, but the second attempt succeeded with `Aur0ra!Helpdesk26`.

After logging in, the client changed to the directory `/var/log/asterisk/diag` and downloaded the file `pbx-diag-20261008.tar.gz`.

The archive was 8,593 bytes and was transferred over FTP data stream 150.

I extracted the transferred file using:

```bash
tshark -r DontLeaveMeHanging.pcap --export-objects ftp-data,out
```

This gave me the diagnostics bundle needed for further investigation.

### 3. Examining the Diagnostics Bundle

The extracted archive contained the following files:

```text
pbx-diag-20261008/
├── README.txt
├── pjsip_endpoints.txt
├── messages.log
├── tls-keys.log
└── tls-keys.log.1
```

The important file was `tls-keys.log`. It contained TLS session secrets exported from the Asterisk PJSIP TLS transport.

The README explained that the key log had been exported to help a vendor inspect SIP signalling in a packet capture. It also instructed the recipient to delete the bundle after confirming receipt, but the bundle had clearly remained accessible.

The second file, `tls-keys.log.1`, belonged to an aborted run and was not useful.

Next, I examined `pjsip_endpoints.txt`. It showed four help desk extensions, numbered 2001 through 2004, associated with hosts `10.20.200.21` through `10.20.200.24`.

All four extensions used SIP over TLS on port 5061 and configured media encryption with SDES.

However, extension 2003 had a different setting:

- Extension 2003 used `dtmf_mode: inband`.
- The other three extensions used `rfc4733`.

This difference later helped identify the call containing the hidden message.

### 4. Decrypting SIP Signalling

With the TLS key log available, I could decrypt the SIP signalling traffic in the capture.

I configured `tshark` to use the key log and treat TCP port 5061 as TLS:

```bash
tshark -r DontLeaveMeHanging.pcap \
  -o tls.keylog_file:$KEY \
  -d tcp.port==5061,tls \
  -Y sip
```

This revealed 3,245 SIP messages:

- 1,235 OPTIONS messages
- 274 REGISTER messages
- 32 INVITE messages
- 28 BYE messages
- 3 CANCEL messages
- 3 harmless message-waiting NOTIFY messages

All 32 calls were incoming calls from external numbers to the four help desk extensions.

Of these calls, 28 were answered, three were cancelled while ringing, and one returned a `486 Busy Here` response.

Decrypting SIP was important because the signalling messages contained the information needed to decrypt the voice traffic.

### 5. Decrypting the Voice Calls

The endpoints used SDES for SRTP encryption. With this configuration, the SDP sections of SIP messages contain encryption keys in lines similar to:

```text
a=crypto:1 AES_CM_128_HMAC_SHA1_80 inline:<base64>
```

These keys protect the media sent by the corresponding endpoint.

For every answered call, I extracted the encryption keys and RTP ports from the INVITE and `200 OK` messages. I then used `tshark` to isolate the relevant UDP traffic and Python's `pylibsrtp` library to decrypt the SRTP packets.

The decrypted G.711 µ-law audio was converted into WAV files. I also checked for RFC 4733 `telephone-event` packets, which carry DTMF digits separately from the audio.

To verify that the key mapping was correct, I tested the first call, `73dabecd`. Both directions decrypted successfully, with `decrypt_fail=0`, and the call contained approximately 92 seconds of audio.

After confirming the results, I repeated the process for all 28 answered calls.

### 6. Listening to the Calls

The first decrypted call, `73dabecd`, was the only call containing RFC 4733 DTMF events. The recorded digits were:

```text
40917#
```

I listened to the call and used Whisper to transcribe the conversation. The caller made a prank call, claiming to have been shot in the buttocks and asking whether the hospital accepted Blue Cross Blue Shield.

The call was a decoy, and the digits `40917#` did not lead to the flag.

I then transcribed all 28 answered calls using Whisper. Most contained unrelated conversations, including pizza orders, a haunted printer, jokes about HIPAA, flirting with the help desk, and riddles.

Some conversations mentioned names that sounded like DEADFACE aliases, including:

- Lamia
- Spooky Boy
- Sunstalker

I tested the following submissions:

```text
deadface{Lamia}
deadface{Spooky Boy}
deadface{Sunstalker}
```

All three were rejected.

These names were red herrings. The hidden message was not spoken aloud, so speech transcription alone could not reveal it.

### 7. Finding the Hidden DTMF Message

The key clue was the endpoint configuration discovered earlier.

Extension 2003 used `dtmf_mode: inband`. Unlike RFC 4733, which sends DTMF digits in separate RTP telephone-event packets, inband DTMF represents each digit using audio tones within the voice stream itself.

This explained why the earlier DTMF packet analysis had not found the message.

One call to extension 2003, identified as `06826701`, looked suspicious. Whisper produced an empty transcript for the agent's side, while the caller's audio had an RMS value of approximately 2950. In other words, the audio contained a signal even though there was no meaningful speech to transcribe.

I wrote a Python tone detector based on the Goertzel algorithm to identify DTMF frequencies in the decrypted WAV files.

DTMF uses pairs of frequencies: one from a low-frequency group and one from a high-frequency group.

```python
L = [697, 770, 852, 941]
H = [1209, 1336, 1477, 1633]

# Analyse each 320-sample window.
# Find the strongest low and high frequencies.
# Accept the digit when the two-tone energy ratio exceeds 0.35.
```

The detector analysed the audio in 40 ms windows, identified the strongest low and high frequencies, and matched each valid frequency pair to a DTMF digit.

I ran the detector against every decrypted WAV file. Only `call_06826701_caller.wav`, corresponding to extension 2003, produced a clear, long sequence of digits.

The file was approximately 25 seconds long. Other files produced no meaningful results, apart from a false positive involving `87` and `7` in a veterinary-related call, which was caused by speech.

The detected digit sequence was:

```text
100101097100102097099101123080114051053053095048110051095102048114095068051052100102052099051125
```

This sequence was the hidden message.

### 8. Decoding the ASCII Values

The detected sequence contained 96 digits. Splitting it into groups of three gave 32 decimal values, each representing an ASCII character.

The values were:

```text
100 101 097 100 102 097 099 101 123 080 114 051 053 053 095 048
110 051 095 102 048 114 095 068 051 052 100 102 052 099 051 125
```

I converted each decimal value into its corresponding ASCII character using Python:

```bash
python3 -c "s='100101097100102097099101123080114051053053095048110051095102048114095068051052100102052099051125'; print(''.join(chr(int(s[i:i+3])) for i in range(0,len(s),3)))"
```

The decoded output was:

```text
deadface{Pr355_0n3_f0r_D34df4c3}
```

The leetspeak message reads **“Press one for Deadface.”**

## Flag

```text
deadface{Pr355_0n3_f0r_D34df4c3}
```

## Takeaways

- **The attacker was already inside the network.** The host `10.20.200.60` used the `mkowalski` FTP account and successfully logged in after a failed attempt with a password differing by one digit. This supports the challenge's hint about working from the inside out.

- **Leaked TLS key logs can defeat encryption.** The stolen `tls-keys.log` file exposed the session secrets needed to decrypt SIP signalling. A diagnostics bundle intended for temporary use should not have remained accessible to an attacker.

- **SDES depends on the security of the signalling channel.** Once the TLS session secrets were recovered, the SDES keys embedded in SIP messages became available, allowing the encrypted SRTP calls to be decrypted.

- **Configuration differences can reveal hidden attack paths.** Extension 2003 used inband DTMF, unlike the other extensions. Checking this unusual setting led directly to the hidden message.

- **Speech recognition cannot detect every signal.** Whisper could transcribe spoken conversations but could not interpret the DTMF tones embedded in the audio. A dedicated tone detector was needed.

- **Not every suspicious clue is the answer.** The prank call, its DTMF digits, and the DEADFACE-sounding names were decoys. The actual flag was encoded as ASCII decimal values transmitted through audio tones.

The key to solving the challenge was combining packet analysis, credential and file extraction, TLS decryption, SRTP decryption, and audio signal processing.
