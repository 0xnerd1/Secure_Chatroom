# SecureChat — SSL-Encrypted Peer-to-Peer Chat over TCP Sockets

A lightweight, terminal-based, end-to-end encrypted chat application built with Python's native `socket` and `ssl` modules. SecureChat establishes a TLS-wrapped TCP connection between exactly two peers — a **server** and a **client** — and allows them to exchange real-time text messages over an encrypted channel, using a locally generated self-signed certificate.

The application is designed as an academic/security-learning project demonstrating TLS socket programming, certificate generation via OpenSSL, and full-duplex threaded messaging.

**Project status:** Academic / Demonstration Project
**Version:** 1.0
**Language:** Python 3
**Transport Security:** TLS (via Python `ssl` module, OpenSSL-backed)
**Networking Model:** Single TCP socket, full-duplex, multi-threaded

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Technology Stack](#technology-stack)
- [Application Architecture](#application-architecture)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Installation](#installation)
- [Generating SSL Certificates](#generating-ssl-certificates)
- [Running the Application](#running-the-application)
- [Command-Line Reference](#command-line-reference)
- [Connection & Messaging Workflow](#connection--messaging-workflow)
- [Message Exchange Protocol](#message-exchange-protocol)
- [Exiting a Chat Session](#exiting-a-chat-session)
- [Troubleshooting](#troubleshooting)
- [Security Notes](#security-notes)
- [Recommended .gitignore](#recommended-gitignore)
- [Limitations](#limitations)
- [Future Improvements](#future-improvements)
- [Disclaimer](#disclaimer)
- [License](#license)

---

## Overview

SecureChat is a minimal, dependency-free chat tool that demonstrates how to secure a raw TCP socket connection using TLS. One instance of the script runs in **server mode** and listens for an incoming connection; a second instance runs in **client mode** and connects to the server's IP address and port. Once the TLS handshake completes, both peers can send and receive messages simultaneously using separate sender and receiver threads.

The project supports:

- One-time generation of a self-signed TLS certificate and private key
- TLS-wrapped TCP server and client modes
- Full-duplex, multi-threaded messaging (send and receive concurrently)
- Graceful session termination via an `exit` command
- Basic connection-loss and error handling on both ends

The application is a single Python script and requires no external Python packages — only a working OpenSSL installation on the host system for certificate generation.

---

## Key Features

### TLS-Encrypted Transport
All traffic between the server and client is wrapped in a TLS session using Python's `ssl` module, backed by a locally generated `cert.pem` / `key.pem` pair. Messages are not sent in plaintext over the wire.

### Self-Signed Certificate Generation
The script can generate its own SSL certificate and private key using an OpenSSL command invoked via `subprocess`, avoiding the need for a manually created certificate before first use.

### Full-Duplex Messaging
Sending and receiving run on independent daemon threads (`secure_send` and `secure_receive`), so either peer can type and send a message at any time without waiting for the other side to finish receiving.

### Server / Client Modes
A single script serves both roles:

- **Server mode** binds to `0.0.0.0` on a specified port and waits for a single incoming connection.
- **Client mode** connects out to a specified server IP and port.

### Graceful Exit Handling
Typing `exit` on either side sends an exit signal to the remote peer, closes the local socket, and terminates the process cleanly on both ends.

### Basic Resilience
Both the send and receive loops catch `BrokenPipeError`, `ConnectionResetError`, and generic exceptions, printing a clear status message and exiting rather than crashing with an unhandled traceback.

---

## Technology Stack

| Component | Technology |
|---|---|
| Programming Language | Python 3 |
| Transport | TCP (`socket` module) |
| Encryption | TLS (`ssl` module) |
| Certificate Generation | OpenSSL (invoked via `subprocess`) |
| Concurrency Model | `threading` (daemon threads) |
| CLI Parsing | `argparse` |
| Dependencies | Python standard library only (+ OpenSSL binary on host) |

---

## Application Architecture

```
                     ┌───────────────────────┐
                     │      Peer A (You)      │
                     │   Terminal / Console    │
                     └───────────┬─────────────┘
                                 │
                     secure_send │ secure_receive
                        (thread) │ (thread)
                                 │
                     ┌───────────▼─────────────┐
                     │   ssl.SSLSocket (TLS)   │
                     │  wraps TCP socket conn  │
                     └───────────┬─────────────┘
                                 │
                         Encrypted TCP Channel
                                 │
                     ┌───────────▼─────────────┐
                     │   ssl.SSLSocket (TLS)   │
                     │  wraps TCP socket conn  │
                     └───────────┬─────────────┘
                                 │
                     secure_send │ secure_receive
                        (thread) │ (thread)
                                 │
                     ┌───────────▼─────────────┐
                     │      Peer B (Remote)    │
                     │   Terminal / Console    │
                     └───────────────────────────┘
```

The **server** creates an `ssl.SSLContext` with `ssl.Purpose.CLIENT_AUTH`, loads the certificate chain via `load_cert_chain()`, binds a standard TCP socket, and wraps the accepted connection using `context.wrap_socket(conn, server_side=True)`.

The **client** creates an `ssl.SSLContext` with `ssl.Purpose.SERVER_AUTH`, connects a standard TCP socket to the server, and wraps it using `context.wrap_socket(client_sock, server_hostname=ip)`.

Once wrapped, both sides spin up two daemon threads:

- `secure_send` — reads input from the terminal (`input()`) and sends it over the encrypted socket.
- `secure_receive` — blocks on `conn.recv()` and prints incoming messages as they arrive.

---

## Project Structure

```
SecureChat-Project/
│
├── secure_chat.py        # Entire application: server, client, keygen, CLI
├── cert.pem              # Self-signed TLS certificate (generated, not committed)
├── key.pem               # TLS private key (generated, not committed)
└── README.md
```

The project is intentionally implemented as a single script for simplicity, with clearly separated functions for key generation, sending, receiving, server startup, and client startup.

| Function | Responsibility |
|---|---|
| `generate_ssl_keys()` | Generates `cert.pem` and `key.pem` via OpenSSL if they don't already exist |
| `secure_send(conn)` | Reads terminal input and transmits it over the TLS socket; handles the `exit` command |
| `secure_receive(conn)` | Listens for incoming data and prints it; detects remote `exit` / disconnects |
| `start_server(port)` | Builds the server-side TLS context, binds/listens, accepts one connection, starts I/O threads |
| `start_client(ip, port)` | Builds the client-side TLS context, connects to the server, starts I/O threads |
| `__main__` block | Parses CLI arguments and dispatches to key generation, server mode, or client mode |

---

## Requirements

- Python 3.7 or newer (standard library only — no `pip install` required)
- OpenSSL installed and available on the system `PATH` (required only for `--generate-keys`)
- Two machines (or two terminals on the same machine) that can reach each other over TCP on the chosen port

Check Python:
```bash
python3 --version
```

Check OpenSSL:
```bash
openssl version
```

---

## Installation

### 1. Clone the repository
```bash
git clone https://github.com/<your-username>/SecureChat-Project.git
cd SecureChat-Project
```

No virtual environment or `requirements.txt` is needed — the script relies only on Python's standard library (`socket`, `ssl`, `threading`, `sys`, `os`, `argparse`, `subprocess`).

---

## Generating SSL Certificates

Before the first server run, generate a self-signed certificate and private key:

```bash
python3 secure_chat.py --generate-keys
```

This runs the following OpenSSL command internally:

```bash
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes -subj "/CN=SecureChat"
```

This produces two files in the project directory:

| File | Purpose |
|---|---|
| `cert.pem` | Self-signed public certificate presented by the server during the TLS handshake |
| `key.pem` | Private key used by the server to establish the TLS session |

If both files already exist, the script skips generation and prints a confirmation message instead of overwriting them.

> Only the **server** needs `cert.pem` and `key.pem` present in its working directory. The client does not require its own certificate/key pair for this configuration.

---

## Running the Application

### Start the server
On the machine that will listen for the connection:
```bash
python3 secure_chat.py server <port>
```
Example:
```bash
python3 secure_chat.py server 5555
```

### Start the client
On the machine that will connect to the server:
```bash
python3 secure_chat.py client <server_ip> <port>
```
Example:
```bash
python3 secure_chat.py client 192.168.1.10 5555
```

Once the client connects and the TLS handshake succeeds, both terminals display a connection confirmation, and either side can begin typing messages.

---

## Command-Line Reference

| Argument | Applies To | Description |
|---|---|---|
| `mode` | required | `server` or `client` |
| `ip_port` | required | Server mode: `<port>` only. Client mode: `<server_ip> <port>` |
| `--generate-keys` | optional flag | Generates `cert.pem` / `key.pem` and exits immediately, without starting a connection |

Usage errors (wrong number of positional arguments) print a short usage hint and exit with a non-zero status code.

---

## Connection & Messaging Workflow

```
Server                                   Client
  │                                         │
  │ python secure_chat.py server <port>    │
  │ bind + listen on 0.0.0.0:<port>        │
  │                                         │
  │                     python secure_chat.py client <ip> <port>
  │◄────────────── TCP connect ────────────┤
  │                                         │
  │──────────── TLS handshake ────────────►│
  │◄─────────── TLS handshake ─────────────│
  │                                         │
  │  wrap_socket(server_side=True)   wrap_socket(server_hostname=ip)
  │                                         │
  │  start secure_send / secure_receive threads (both sides)
  │                                         │
  │◄══════════ encrypted messages ════════►│
```

1. The server binds to `0.0.0.0:<port>` and calls `listen(1)`, accepting exactly one connection at a time.
2. The client connects to the given server IP and port over a plain TCP socket.
3. Both sides wrap their socket in a TLS context — the server presents its certificate, and the client (with `verify_mode = ssl.CERT_NONE`) accepts it without validating a certificate authority chain.
4. Once the encrypted socket is established, each side launches its `secure_send` and `secure_receive` threads as daemon threads.
5. Messages typed into either terminal are sent immediately and printed on the remote terminal, prefixed with `[📩] Message:`.

---

## Message Exchange Protocol

SecureChat does not implement a structured message protocol (no JSON, headers, or framing) — it sends and receives **raw UTF-8-encoded text** directly over the TLS socket, read in a single `recv(4096)` call per message.

| Direction | Behavior |
|---|---|
| Outgoing | `input()` → `.encode()` → `conn.sendall()` |
| Incoming | `conn.recv(4096)` → `.decode()` → printed to terminal |
| Exit signal | The literal string `"exit"` is sent and interpreted by the receiving side as a disconnect request |

Because there is no length-prefixing or delimiter framing, messages are assumed to fit within a single `recv(4096)` call; very large messages may be split across multiple receive calls and are not currently reassembled.

---

## Exiting a Chat Session

Either peer can end the session by typing:

```
exit
```

What happens:

1. The local `secure_send` thread detects the `exit` keyword.
2. It sends the literal bytes `b"exit"` to the remote peer.
3. It closes the local socket and calls `sys.exit(0)`.
4. On the remote side, `secure_receive` detects the incoming `"exit"` string, prints a disconnect notice, closes its socket, and exits as well.

An unexpected network drop (rather than a typed `exit`) is instead caught as a `BrokenPipeError` or `ConnectionResetError` and reported as a connection loss.

---

## Troubleshooting

### `FileNotFoundError` / OpenSSL errors during `--generate-keys`
Confirm OpenSSL is installed and reachable:
```bash
openssl version
```
If it's missing, install it via your system's package manager (e.g. `apt install openssl`, `brew install openssl`).

### `ssl.SSLError: [SSL] PEM lib` or certificate load failure
This usually means `cert.pem` or `key.pem` is missing, empty, or corrupted in the server's working directory. Regenerate them:
```bash
rm -f cert.pem key.pem
python3 secure_chat.py --generate-keys
```

### `OSError: [Errno 98] Address already in use`
The chosen port is already bound by another process. Either stop that process or start the server on a different port:
```bash
python3 secure_chat.py server 6001
```

### Client connects but the handshake fails
Ensure the server was started with valid certificate files present in its working directory, and that the client is pointed at the correct IP and port. Firewalls or NAT between the two machines can also block the TCP handshake before TLS is ever attempted.

### Nothing happens after `client` connects
Confirm both peers are actually on the same reachable network segment (or that appropriate port forwarding is configured), and that no firewall rule is silently dropping traffic on the chosen port.

---

## Security Notes

This project is intended primarily for **academic and educational purposes** — demonstrating TLS socket programming — and should **not** be treated as a production-grade secure messaging tool in its current form.

Before considering any real-world use, review and address the following:

- **Certificate verification is disabled on the client.** `verify_mode = ssl.CERT_NONE` and `check_hostname = False` mean the client accepts *any* certificate presented by the server, without validating it against a trusted CA or expected hostname. This makes the connection vulnerable to man-in-the-middle attacks unless certificate pinning or proper CA-based verification is added.
- **No authentication of peers.** There is no username/password, pre-shared key, or identity verification — anyone who can reach the listening port and complete a TLS handshake can chat.
- **No message integrity framing beyond TLS.** TLS itself provides confidentiality and integrity for the transport, but the application layer has no additional signing, replay protection, or structured message validation.
- **Single-connection server.** The server accepts exactly one connection and then stops listening; it is not designed for multiple concurrent clients.
- **Private key handling.** `key.pem` is generated with `-nodes` (no passphrase). Treat it as a sensitive file — do not commit it to version control or share it.
- **No rate limiting or input sanitization.** Messages are printed to the terminal as-is; no filtering is performed on received content.
- **Self-signed certificates only.** There is no support for CA-issued certificates or certificate rotation in the current implementation.

---

## Recommended `.gitignore`

```gitignore
# Python
__pycache__/
*.pyc

# TLS material — never commit private keys or generated certs
cert.pem
key.pem

# OS files
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
```

If `key.pem` or `cert.pem` have already been committed to Git history, remove them from tracking and rotate the key pair:
```bash
git rm --cached cert.pem key.pem
git add .gitignore
git commit -m "Remove TLS material from repository"
```

---

## Limitations

- Supports exactly **two peers** per session (one server, one client) — no group chat or multi-client broadcast.
- No message history, persistence, or logging.
- No GUI — terminal/console only.
- No file transfer support.
- No reconnection logic if the connection drops mid-session.
- Messages larger than a single `recv(4096)` buffer are not reassembled.

---

## Future Improvements

Potential enhancements for extending this project beyond its current academic scope:

- Proper certificate verification on the client (CA-signed certs or certificate pinning)
- Peer authentication (pre-shared key, username/password, or public-key identity)
- Message framing/protocol (length-prefixed or JSON-based) to support larger and structured payloads
- Support for multiple simultaneous client connections on the server
- Encrypted local message logging with configurable retention
- File transfer support
- Automatic reconnection with exponential backoff
- A simple GUI (e.g. Tkinter or a web-based front end)
- Unit tests for the send/receive and handshake logic
- Cross-platform packaging (PyInstaller / standalone executable)

---



---

