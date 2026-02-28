# EKEEChat

> **End-to-End Encrypted Messenger** — Powered by EKEE (RSA-4096 + double AES-256).  
> Password-protected servers. Zero server-side logging. Zero decryption on the relay.

EKEEChat is a Python-based encrypted chat system where the server never sees your messages — it only forwards encrypted bytes it cannot read. Every message is encrypted on your machine before it leaves, and decrypted only on the recipient's machine.

---

## How It Works

```
Alice                      Relay Server                Bob
  |                             |                        |
  |  EKEE-encrypted packet -->  |  --> forwards bytes --> |
  |                             |                        |
  |                        (server sees                  |
  |                     only ciphertext,                 |
  |                      stores nothing)                 |
```

The server is a **blind relay**. It cannot decrypt messages. It stores no logs. It cannot tell what is being said, only that packets are moving between clients.

---

## Features

- **EKEE encryption** — RSA-4096 + double AES-256-GCM with keystream mixing per message
- **Password-protected servers** — challenge-response auth, password never sent over the wire
- **Ephemeral session keys** — fresh EKEE keypair per session, wiped on exit
- **Multi-room support** — multiple isolated chat rooms on one server
- **Automatic key exchange** — public keys exchanged on first message, no manual setup
- **Fingerprint verification** — verify peer identity out-of-band to defeat MITM
- **Direct messages** — `/dm` sends encrypted messages to one specific peer
- **Bore / ngrok compatible** — works over any TCP tunnel for remote connections
- **Single file, one dependency** — `pip install ekee` and you're done

---

## Installation

```bash
pip install ekee
```

---

## Quick Start

### Running a server

```bash
python ekeechat.py
```

Select option `1` (Server), set a port, and optionally set a **server password**. The password protects your relay — only people who know it can connect.

```
  Port [9999]:
  Server password: ••••••••
  Confirm        : ••••••••
  Password set. Clients must enter this to connect.
```

### Connecting as a client

```bash
python ekeechat.py
```

Select option `2` (Client), enter the server IP, port, your alias, room name, and the server password.

```
  Server IP: 192.168.1.100
  Port [9999]:
  Your alias: alice
  Room [default]: ops
  Server password: ••••••••
  Session passphrase: ••••••••
```

### Server operator also chatting

When setting up a server, choose `Also join as client? [y/n]: y` — the server starts in the background and you drop straight into the chat as a participant.

---

## Tunneling Over the Internet (Bore)

EKEEChat works over any TCP tunnel. Using [bore](https://github.com/ekzhang/bore):

**Server side:**
```bash
python ekeechat.py   # run on port 9999
bore local 9999 --to bore.pub
# bore gives you: bore.pub:XXXXX
```

**Client side:**
```bash
# Server IP: bore.pub
# Port: XXXXX (the port bore gave you)
python ekeechat.py
```

Works the same with ngrok, frp, or any TCP forwarder.

---

## Chat Commands

| Command | Description |
|---------|-------------|
| `/help` | Show all commands |
| `/who` | Your fingerprint + list of known peers |
| `/verify <alias>` | Show a peer's EKEE fingerprint |
| `/dm <alias> <message>` | Send an encrypted direct message to one person |
| `/quit` | Disconnect and wipe session keys |

---

## Authentication — How Server Passwords Work

Passwords are never sent over the network. EKEEChat uses **challenge-response authentication**:

```
1. Client connects
2. Server sends: random 32-byte challenge
3. Client computes: SHA3-256(KDF(password) + challenge + "EKEEChat-auth")
4. Server verifies the response
5. PASS -> connection accepted
   FAIL -> connection dropped immediately
```

**What this means:**
- An eavesdropper on the network sees only the challenge and a hash response — useless without the password
- Replay attacks don't work — the challenge is different every time
- The KDF uses 100,000 iterations — brute-forcing the password is computationally expensive
- Wrong password = instant disconnect, no information leaked

---

## Fingerprint Verification

When a peer sends their first message, EKEEChat automatically exchanges public keys and displays a fingerprint:

```
  * Key exchange with alice -- fingerprint: 3a4f:9c12:7e81:bb43:...
  * Use /verify alice to confirm fingerprint out-of-band
```

To confirm there is no man-in-the-middle:

1. Both users run `/verify <each_other>`
2. Call or meet the other person
3. Read your fingerprints out loud
4. If they match — connection is secure and authentic
5. If they don't match — someone is intercepting — do not trust the connection

This is the same model used by Signal (Safety Numbers) and WhatsApp (Security Codes).

---

## Session Security

| Property | Detail |
|----------|--------|
| Key lifetime | One session — generated on connect, wiped on `/quit` |
| Key storage | Temp folder, deleted on exit |
| Server storage | Nothing — relay only, no logs |
| Message storage | Nothing — no history anywhere |
| Server knowledge | Encrypted packet sizes and timing only |

---

## Cryptographic Details

| Component | Algorithm |
|-----------|----------|
| Message encryption | EKEE: RSA-4096 + double AES-256-GCM |
| Key exchange | RSA-4096 public key embedded in first message |
| Server auth | SHA3-256 challenge-response with 100k-iteration KDF |
| Session key derivation | Alternating SHA3-512 / BLAKE2b, 100k iterations |
| Nonces | 12 bytes random per AES-GCM operation |

### The double AES layer (EKEE)

Every message goes through two AES-256-GCM passes with a keystream mix between them:

```
plaintext
    ↓ AES-256-GCM (key 1)
    ↓ XOR keystream mix (SHA3-512 derived)
    ↓ AES-256-GCM (key 2)
ciphertext
```

Breaking a message requires defeating both AES layers independently.

---

## Architecture

```
ekeechat.py
├── EKEESession      — EKEE keypair management, per-peer encryption
├── Server           — TCP relay, challenge-response auth, room management
│   ├── _accept_loop — accepts incoming connections
│   └── _handle      — auth handshake + message relay per client
└── Client           — connects, authenticates, sends/receives
    ├── _recv_loop   — background thread: receives + decrypts messages
    └── _send_loop   — main thread: reads input + encrypts + sends
```

---

## Threat Model

**Protected against:**
- Passive eavesdroppers on the network (messages are encrypted)
- Unauthorized users connecting to your server (challenge-response auth)
- Server compromise (server never has decryption keys)
- Replay attacks (fresh challenge per connection)
- Man-in-the-middle (detectable via fingerprint verification)

**Not protected against:**
- An attacker who has physical access to your machine while chatting
- An attacker who compromises your machine and steals your session passphrase
- Metadata analysis (connection timing, packet sizes, who is talking to whom)
- A compromised server operator redirecting your tunnel (use fingerprint verification)

For metadata protection, run over Tor or a VPN in addition to EKEEChat.

---

## Requirements

- Python 3.8+
- `ekee` (`pip install ekee`)
- Windows / Linux / macOS

---

## License

MIT License. Use freely, modify freely, contribute back if you improve it.

---

## Part of the Forge Suite

EKEEChat is part of a collection of security tools:

| Tool | Purpose |
|------|---------|
| **EKEE** | RSA-4096 + double AES-256 file encryption |
| **CipherForge** | Triple-layer encryption with vault |
| **SentinelForge** | Blue team workspace protection agent |
| **SignForge** | Code signing and verification |
| **EKEEChat** | End-to-end encrypted messenger |
| **HybridForge** | Post-quantum hybrid RSA + Kyber-1024 encryption |
