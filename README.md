SecureChat — TLS-Encrypted Peer-to-Peer Chat Application
A lightweight, dependency-free Python chat application that establishes an encrypted, socket-based communication channel between two peers using TLS/SSL.

The application is designed as a minimal, self-contained demonstration of secure socket programming, combining Python's built-in `socket` and `ssl` modules with self-signed certificate generation via OpenSSL, multithreaded full-duplex messaging, and a simple server/client command-line interface.

Project status: Development / Educational Project
Version: 1.0
Language: Python 3
Transport Security: TLS (via Python `ssl` module)
Networking Model: Single-connection TCP socket (server/client)

Table of Contents
Overview
Key Features
Technology Stack
Application Architecture
Project Structure
Requirements
Installation
SSL Certificate Generation
Running the Application
Command-Line Usage Reference
Communication Workflow
Threading Model
Session Termination
Troubleshooting
Configuration Notes
Security Notes
Recommended .gitignore
Future Improvements
Disclaimer
License

Overview
SecureChat is a single-file Python application (`secure_chat.py`) that allows two parties to exchange real-time text messages over an encrypted TCP connection. It operates in one of two modes — server or client — and wraps a standard TCP socket in a TLS layer using Python's `ssl` module, so that traffic between the two peers is encrypted in transit.

The application supports:

Self-signed SSL/TLS certificate generation via OpenSSL
A TLS-wrapped TCP server that listens for a single incoming connection
A TLS TCP client that connects to a remote server
Simultaneous bidirectional messaging using independent send/receive threads
Graceful session termination via an `exit` command
Command-line argument parsing for server and client modes
Because it relies only on Python's standard library (`socket`, `ssl`, `threading`, `argparse`, `subprocess`) plus a local OpenSSL installation, the project has no external Python package dependencies.

Key Features
TLS-Encrypted Transport
All messages are transmitted over a socket wrapped with Python's `ssl` module rather than a raw TCP socket. This encrypts the communication channel between the server and client, protecting message contents from passive network eavesdropping.

The server loads a certificate/key pair and presents it during the TLS handshake. The client wraps its socket using a client-purpose SSL context to negotiate the encrypted session.

Self-Signed Certificate Generation
The application can generate its own SSL certificate and private key using OpenSSL, invoked as a subprocess:

A 2048-bit RSA key pair is generated
A self-signed X.509 certificate valid for 365 days is created
The certificate uses the subject `/CN=SecureChat`
Output files are `cert.pem` and `key.pem`
If both files already exist, generation is skipped automatically.

Full-Duplex Messaging
Once a TLS session is established, the application spawns two daemon threads per peer:

A send thread that reads input from the local terminal and transmits it
A receive thread that listens for incoming data and prints it to the terminal
This allows both sides of the conversation to send and receive messages concurrently, without blocking on each other.

Server / Client Modes
The application operates as either:

Server — binds to a local port and waits for exactly one incoming connection
Client — connects to a specified server IP address and port
Mode selection and connection parameters are supplied via command-line arguments.

Graceful Exit Handling
Typing `exit` in either the server or client terminal:

Notifies the remote peer that the session is ending
Closes the local socket cleanly
Terminates the process
The receiving side detects the `exit` signal (or a closed connection) and shuts down its own session in response.

Connection Failure Handling
The application catches common socket-level failures during send and receive operations, including `BrokenPipeError` and `ConnectionResetError`, and exits cleanly with a status message rather than raising an unhandled exception.

Technology Stack
Component | Technology
--- | ---
Programming Language | Python 3
Networking | `socket` (standard library, TCP/IPv4)
Transport Encryption | `ssl` (standard library, TLS)
Certificate Generation | OpenSSL (via `subprocess`)
Concurrency | `threading` (daemon threads for send/receive)
CLI Parsing | `argparse`
Process Interop | `subprocess` (for invoking the `openssl` CLI)
Interface | Terminal / command-line, stdin-driven

Application Architecture
The application follows a simple two-role, single-connection client/server model, with TLS wrapping applied at the socket layer before any application data is exchanged.

                ┌─────────────────────────┐
                │   OpenSSL (subprocess)   │
                │  generates cert.pem /    │
                │        key.pem           │
                └────────────┬─────────────┘
                              │
                              ▼
        ┌──────────────────────────────────────┐
        │              SERVER MODE              │
        │  TCP socket bound to 0.0.0.0:<port>   │
        │  ssl.SSLContext (CLIENT_AUTH purpose) │
        │  loads cert.pem + key.pem             │
        │  accept() → wrap_socket(server_side)  │
        └────────────────┬───────────────────────┘
                          │  TLS Handshake
                          ▼
        ┌──────────────────────────────────────┐
        │              CLIENT MODE              │
        │  TCP socket → connect(ip, port)       │
        │  ssl.SSLContext (SERVER_AUTH purpose) │
        │  wrap_socket(server_hostname=ip)      │
        └────────────────┬───────────────────────┘
                          │
                          ▼
              ┌───────────────────────────┐
              │   Encrypted TLS Channel    │
              └─────────────┬─────────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                              ▼
     ┌──────────────────┐          ┌──────────────────┐
     │  secure_send()    │          │ secure_receive()  │
     │  (daemon thread)  │          │  (daemon thread)   │
     │  stdin → socket    │          │  socket → stdout   │
     └──────────────────┘          └──────────────────┘
Both the server and client instantiate their own pair of send/receive threads once the TLS handshake completes, giving each side an independent, concurrent read/write loop over the same encrypted socket.

Project Structure
SecureChat-Project/
│
├── secure_chat.py        # Main application — server, client, and CLI entry point
├── cert.pem              # Self-signed TLS certificate (generated, not committed)
├── key.pem               # Private key for the certificate (generated, not committed)
└── README.md             # Project documentation
As a single-file application, all functionality — key generation, server logic, client logic, and threaded messaging — is implemented within `secure_chat.py`.

Function | Responsibility
--- | ---
`generate_ssl_keys()` | Generates `cert.pem` / `key.pem` via OpenSSL if they don't already exist
`secure_send(conn)` | Reads terminal input and sends it over the encrypted connection
`secure_receive(conn)` | Listens for incoming data and prints it to the terminal
`start_server(port)` | Binds, listens, accepts one connection, and wraps it in TLS as the server
`start_client(ip, port)` | Connects to a remote server and wraps the connection in TLS as the client
`main (argparse block)` | Parses CLI arguments and dispatches to server or client mode

Requirements
Before running SecureChat, install:

Python 3.7 or newer (for the `ssl` and `socket` standard library features used)
OpenSSL command-line tool, available on the system `PATH`
Check Python:

python3 --version
Check OpenSSL:

openssl version
No third-party Python packages are required — the project relies entirely on the standard library.

Installation
1. Clone or download the project
git clone https://github.com/<your-username>/SecureChat-Project.git
Move into the project directory:

cd SecureChat-Project
2. (Optional) Create a virtual environment
Although no external packages are required, a virtual environment can still be used for isolation:

python3 -m venv venv
source venv/bin/activate      # Linux / macOS
venv\Scripts\activate         # Windows
3. Verify OpenSSL is available
openssl version
If this command is not recognized, install OpenSSL for your platform before continuing.

SSL Certificate Generation
Before starting the server for the first time, generate a certificate and private key:

python secure_chat.py server 5555 --generate-keys
Or generate keys independently of starting a session:

python secure_chat.py --generate-keys server 5555
This creates:

cert.pem   — self-signed X.509 certificate (365-day validity, CN=SecureChat)
key.pem    — 2048-bit RSA private key
If `cert.pem` and `key.pem` already exist in the working directory, the application detects them and skips regeneration, printing a confirmation message instead.

Only the server needs `cert.pem` and `key.pem` present in its working directory. The client does not require its own certificate, since the client's SSL context is configured for server verification rather than presenting a certificate of its own.

Running the Application
Starting the Server
python secure_chat.py server <port>
Example:

python secure_chat.py server 5555
The server will:

Bind to `0.0.0.0` on the specified port
Wait for exactly one incoming connection
Perform a TLS handshake once a client connects
Start the send/receive threads
Starting the Client
python secure_chat.py client <server_ip> <port>
Example:

python secure_chat.py client 127.0.0.1 5555
The client will:

Connect to the given IP address and port
Perform a TLS handshake with the server
Start the send/receive threads
Once connected, either side can type messages directly into the terminal and press Enter to send them.

Command-Line Usage Reference
Command | Description
--- | ---
`python secure_chat.py --generate-keys` | Generates `cert.pem` / `key.pem` and exits
`python secure_chat.py server <port>` | Starts the application in server mode on the given port
`python secure_chat.py client <ip> <port>` | Starts the application in client mode, connecting to `<ip>:<port>`
Argument | Applies To | Description
--- | --- | ---
`mode` | Required | Either `server` or `client`
`ip_port` | Required | For server: `<port>`. For client: `<server_ip> <port>`
`--generate-keys` | Optional flag | Generates SSL keys and exits without starting a session

Communication Workflow
A typical session between two peers proceeds as follows:

Peer A (Server)                         Peer B (Client)
      │                                        │
      ▼                                        │
generate_ssl_keys()                            │
(cert.pem / key.pem)                           │
      │                                        │
      ▼                                        │
start_server(port)                             │
  bind → listen → accept()                     │
      │                                        ▼
      │                              start_client(ip, port)
      │                                connect(ip, port)
      │                                        │
      └──────────── TLS Handshake ─────────────┘
                          │
                          ▼
        secure_send()  ⇄  secure_receive()   (both peers)
                          │
                          ▼
              Messages exchanged in real time
                          │
                          ▼
             One side types "exit" → session ends
                for both peers

Threading Model
Each peer (server and client) runs two daemon threads once the TLS connection is established:

send_thread → runs secure_send(conn)    — blocks on terminal input, writes to socket
recv_thread → runs secure_receive(conn) — blocks on socket recv(), writes to terminal
Both threads are marked as daemon threads, so they terminate automatically if the main process exits. The main thread joins on both threads and stays alive for the duration of the chat session.

Session Termination
A session can end in one of the following ways:

Typing exit — The local side sends an `"exit"` payload to the remote peer, closes its socket, and calls `sys.exit(0)`.
Remote peer sends exit or disconnects — `secure_receive()` detects an empty payload or the literal string `"exit"` and closes the connection on its own side.
Connection-level failure — A `BrokenPipeError` or `ConnectionResetError` during send/receive is caught, a status message is printed, and the process exits.
Because the server only calls `accept()` once, a closed session requires restarting the server process to accept a new connection.

Troubleshooting
`openssl: command not found`
OpenSSL is not installed or not on your system `PATH`. Install it via your OS package manager (e.g., `apt install openssl`, `brew install openssl`) and re-run `python secure_chat.py --generate-keys`.

`FileNotFoundError` / SSL context fails to load certificate
The server could not find `cert.pem` and `key.pem` in the current working directory. Run:

python secure_chat.py --generate-keys
Then start the server again from the same directory.

`Address already in use`
The chosen port is already bound by another process. Either stop the other process or choose a different port:

python secure_chat.py server 6000
Client Cannot Connect
Confirm the server is running and listening on the expected port.
Confirm the IP address is reachable (use the server's LAN IP rather than `127.0.0.1` if connecting from a different machine).
Confirm no firewall is blocking the chosen port.
Connection Drops Unexpectedly
This typically surfaces as a `BrokenPipeError` or `ConnectionResetError`, both of which are caught and reported. This generally indicates the remote peer closed the connection or the network path was interrupted.

Configuration Notes
Setting | Location | Default
--- | --- | ---
Certificate file | `CERT_FILE` constant | `cert.pem`
Private key file | `KEY_FILE` constant | `key.pem`
Server bind address | `start_server()` | `0.0.0.0` (all interfaces)
Listen backlog | `start_server()` | `1` (single connection only)
Receive buffer size | `secure_receive()` | `4096` bytes
Certificate validity | `generate_ssl_keys()` | `365` days
Certificate key size | `generate_ssl_keys()` | RSA 2048-bit
These values are currently hardcoded and would need to be edited directly in `secure_chat.py` to change.

Security Notes
This project is intended primarily for educational and development purposes, as a demonstration of TLS socket programming in Python. Before relying on it for any sensitive communication, be aware of the following:

Client does not verify the server certificate. The client sets `check_hostname = False` and `verify_mode = ssl.CERT_NONE`, which disables certificate validation entirely. This makes the connection vulnerable to man-in-the-middle attacks, since the client will accept any certificate presented by the server, self-signed or otherwise.
Self-signed certificates provide encryption, not authentication. Without a trusted certificate authority or a pinned/known certificate fingerprint, a self-signed certificate proves the traffic is encrypted but not that you are talking to the intended peer.
No message authentication or integrity signing beyond TLS. The application relies entirely on the TLS layer for confidentiality and integrity; there is no additional application-level message signing.
No authentication or access control. Any client that can reach the server's IP and port can attempt to connect; there is no username/password or key-based peer authentication.
Single connection only. The server's listen backlog is set to accept only one connection at a time, and only one connection is ever accepted, so it is not resilient to multiple or repeated connection attempts.
Private key handling. `key.pem` is generated with no passphrase (`-nodes`), meaning the private key is stored unencrypted on disk. Protect the file with appropriate filesystem permissions.
Before using this project beyond local testing or a lab environment:

Enable proper certificate verification on the client (`check_hostname = True`, `verify_mode = ssl.CERT_REQUIRED`) with a trusted CA or a pinned certificate.
Consider adding authentication (e.g., pre-shared keys, mutual TLS with client certificates).
Avoid transmitting sensitive data over untrusted networks without additional review.
Restrict file permissions on `key.pem`.
Recommended .gitignore
Before pushing this project to a public repository, exclude generated key material:

# Generated TLS certificate and private key
cert.pem
key.pem

# Python
__pycache__/
*.pyc

# Virtual environment
venv/
.venv/

# OS files
.DS_Store
Thumbs.db
If `key.pem` has ever been committed to version control, removing it from future commits alone is not sufficient — treat the key as compromised, regenerate a new certificate/key pair, and scrub the file from repository history if the repository is or was public.

Future Improvements
Potential future development areas include:

Client-side certificate verification (removing `CERT_NONE`)
Mutual TLS (client certificate authentication)
Support for multiple simultaneous client connections
Message framing/length-prefixing instead of raw `recv()` buffering
Encrypted message logging or session transcripts
A configuration file for host, port, and certificate paths
Graphical or web-based front end
Passphrase-protected private keys
Reconnection handling on dropped connections
Unit and integration tests for the networking layer

Disclaimer
SecureChat is an educational/demonstration project illustrating TLS socket programming concepts in Python. It has not undergone formal security review or penetration testing and should not be used to transmit sensitive or production data without addressing the items listed in Security Notes.

License
No explicit license is currently defined for this project.

If you intend to make the repository open source, add an appropriate LICENSE file and specify the license here.
