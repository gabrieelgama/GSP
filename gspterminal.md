GSP Terminal Specification

Globalized Secure Protocol (GSP)
GSP Terminal — Terminal Interface Specification
Version: 1.0
Status: Experimental

---

1. Abstract

The GSP Terminal is the official command-line interface for interacting with the Globalized Secure Protocol (GSP).

It provides two independent interface modes:

- Text-only Terminal — lightweight, universal, and compatible with standard terminals.
- Terminal GUI — an advanced graphical terminal interface with real-time panels, session information, traffic statistics, and interactive controls.

The GSP Terminal is an interface layer. It does not replace or redefine the GSP protocol itself.

┌──────────────────────┐
│        USER          │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│     GSP TERMINAL     │
│                      │
│  ┌────────────────┐  │
│  │   Text-only    │  │
│  │ Terminal GUI   │  │
│  └────────────────┘  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│      GSP CLIENT      │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│       GSP CORE       │
└──────────────────────┘

---

2. Interface Modes

2.1 Text-only Terminal

The Text-only Terminal is designed to work in environments where graphical or advanced terminal features are unavailable.

It uses standard:

- "stdin"
- "stdout"
- "stderr"
- TTY input
- Plain text output

The Text-only Terminal must not require:

- A dedicated GPU
- An integrated GPU
- A graphical desktop environment
- Hardware acceleration
- ANSI support
- Unicode support

Example:

$ gsp connect gsp://example.com

GSP Terminal 1.0

Connecting to gsp://example.com...
[OK] Resolving endpoint
[OK] Establishing transport
[OK] GSP negotiation
[OK] Authentication
[OK] Secure session established

Session ID: 7f31a2c9
Protocol: GSP/1.0
Transport: GSPMTP
Encryption: ChaCha20-Poly1305
Compression: LZ4
Latency: 8 ms

gsp>

---

3. Terminal GUI

The Terminal GUI is an advanced graphical interface for the GSP Terminal.

It provides:

- Real-time connection information
- Session monitoring
- Network traffic statistics
- Latency monitoring
- Event logs
- Interactive command input
- Visual connection states
- Graphical controls
- Dynamic terminal panels

3.1 Hardware Requirement

«The Terminal GUI requires an integrated GPU (iGPU) or dedicated GPU (dGPU) to work.»

A compatible GPU and graphics stack are mandatory for the Terminal GUI.

Terminal GUI
     │
     ▼
Graphics Backend
     │
     ▼
Integrated GPU / Dedicated GPU
     │
     ▼
Display

Systems without a usable GPU must use the Text-only Terminal.

The GSP protocol itself does not require a GPU.

Important

GSP Core       → No GPU required
GSP Client     → No GPU required
Text Terminal  → No GPU required
Terminal GUI   → GPU REQUIRED

---

4. Terminal GUI Layout

A typical Terminal GUI implementation may use the following layout:

╭────────────────────────────────────────────────╮
│ GSP TERMINAL                              1.0  │
├────────────────────────────────────────────────┤
│                                                │
│ Connection                                     │
│ ────────────────────────────────────────────── │
│ Endpoint   gsp://example.com                   │
│ Status     ● CONNECTED                         │
│ Session    7f31a2c9                            │
│                                                │
│ Protocol   GSP/1.0                             │
│ Transport  GSPMTP                              │
│ Crypto     ChaCha20-Poly1305                   │
│ Compress   LZ4                                 │
│                                                │
├────────────────────────────────────────────────┤
│ Traffic                                        │
│ ────────────────────────────────────────────── │
│ RX         12.4 MB                             │
│ TX          4.7 MB                             │
│ Latency        8 ms                            │
│                                                │
├────────────────────────────────────────────────┤
│ Event Log                                       │
│ ────────────────────────────────────────────── │
│ [03:12:41] Connected                            │
│ [03:12:42] Handshake complete                  │
│ [03:12:42] Secure session established          │
│                                                │
├────────────────────────────────────────────────┤
│ gsp>                                           │
╰────────────────────────────────────────────────╯

---

5. Command Reference

The following is the complete GSP Terminal command list for version 1.0.

5.1 Global Commands and Options

Command / Option| Description
"gsp"| Start the interactive GSP Terminal
"gsp --help"| Display help
"gsp --version"| Display the GSP Terminal version
"gsp --ui"| Force Terminal GUI mode
"gsp --text"| Force Text-only mode
"gsp --no-color"| Disable terminal colors
"gsp --plain"| Use plain output
"gsp --verbose"| Enable verbose diagnostic output
"gsp --quiet"| Minimize output

---

5.2 Connection Commands

Command| Description
"gsp connect <URI>"| Connect to a GSP endpoint
"gsp disconnect"| Disconnect the current session
"gsp reconnect"| Reconnect to the current endpoint
"gsp status"| Display the current connection status

Examples:

gsp connect gsp://example.com

gsp disconnect

gsp reconnect

gsp status

---

5.3 Session Commands

Command| Description
"gsp session list"| List active sessions
"gsp session show <ID>"| Display information about a session
"gsp session close <ID>"| Close a specific session
"gsp session current"| Display the current session

Examples:

gsp session list

gsp session show 7f31a2c9

gsp session close 7f31a2c9

---

5.4 Data Commands

Command| Description
"gsp send <data>"| Send data through the current GSP session
"gsp receive"| Receive data
"gsp receive --follow"| Continuously receive incoming data
"gsp send --file <file>"| Send a file
"gsp receive --file <file>"| Receive data into a file

Examples:

gsp send "Hello GSP"

gsp receive

gsp receive --follow

---

5.5 Network Commands

Command| Description
"gsp ping <URI>"| Measure GSP round-trip latency
"gsp trace <URI>"| Trace the GSP connection path
"gsp stats"| Display network statistics

Example:

gsp ping gsp://example.com

Possible output:

GSP PING example.com

64 bytes
RTT: 7.82 ms

64 bytes
RTT: 8.11 ms

64 bytes
RTT: 7.63 ms

--- statistics ---
min: 7.63 ms
avg: 7.85 ms
max: 8.11 ms

---

5.6 Information Commands

Command| Description
"gsp info"| Display GSP information
"gsp version"| Display protocol and client versions
"gsp capabilities"| Display supported GSP capabilities
"gsp transport"| Display transport information
"gsp crypto"| Display cryptographic configuration
"gsp compression"| Display compression configuration

---

5.7 Terminal Commands

These commands are available inside the interactive shell:

Command| Description
"help"| Display available commands
"clear"| Clear the terminal
"status"| Display connection status
"version"| Display version information
"history"| Display command history
"exit"| Exit the GSP Terminal
"quit"| Exit the GSP Terminal

Example:

gsp> help
gsp> status
gsp> history
gsp> clear
gsp> exit

---

6. Interactive Shell

Running:

gsp

without a command starts the interactive shell.

GSP Terminal 1.0

Type 'help' for available commands.

gsp>

Example session:

gsp> connect gsp://example.com

[OK] Endpoint resolved
[OK] Transport established
[OK] GSP handshake completed
[OK] Authentication completed
[OK] Secure session established

gsp> status

State:       CONNECTED
Endpoint:    gsp://example.com
Session:     7f31a2c9
Transport:   GSPMTP
Encryption:  ChaCha20-Poly1305
Compression: LZ4
Latency:     8 ms

gsp> send "Hello GSP"

[OK] 9 bytes transmitted

gsp> ping

RTT: 8 ms

gsp> disconnect

[OK] Session closed

gsp> exit

---

7. Connection States

The GSP Terminal must support the following connection states:

DISCONNECTED
     │
     ▼
CONNECTING
     │
     ▼
NEGOTIATING
     │
     ▼
AUTHENTICATING
     │
     ▼
CONNECTED

On failure:

CONNECTING
     │
     ▼
ERROR
     │
     ▼
DISCONNECTED

---

8. Status Indicators

Text-only

The Text-only Terminal uses:

[OK]
[INFO]
[WARN]
[ERROR]

Example:

[OK] Transport established
[OK] GSP negotiation completed
[OK] Secure session established

Terminal GUI

The GUI may use graphical status indicators:

● CONNECTED
● SECURE
● SYNCHRONIZED

Possible states:

○ DISCONNECTED
◌ CONNECTING
● CONNECTED
× ERROR

---

9. Security Information

The terminal may display non-sensitive cryptographic information.

Example:

Encryption: ChaCha20-Poly1305
Key Exchange: Ephemeral
Session ID: 7f31a2c9

The terminal MUST NOT display:

- Private keys
- Session secrets
- Encryption keys
- Authentication secrets
- Other sensitive cryptographic material

Even when using:

gsp --verbose

sensitive material must remain hidden.

---

10. Machine-readable Output

GSP Terminal supports machine-readable output for automation.

Example:

gsp status --format json

Example output:

{
  "state": "connected",
  "endpoint": "gsp://example.com",
  "protocol": "GSP/1.0",
  "transport": "GSPMTP",
  "latency_ms": 8
}

Supported formats may include:

text
json

Machine-readable output must not contain decorative UI elements.

---

11. Standard Streams

The GSP Terminal follows standard command-line conventions:

stdin  → Input
stdout → Normal output
stderr → Errors and diagnostics

Example:

gsp status > status.txt

Normal output is redirected to "status.txt".

Errors remain available through "stderr".

---

12. Exit Codes

The following exit codes are defined:

Code| Meaning
"0"| Success
"1"| General error
"2"| Invalid arguments
"3"| Connection failure
"4"| Authentication failure
"5"| Protocol error
"6"| Timeout
"7"| Permission error
"8"| Invalid session

Example:

gsp connect gsp://example.com
echo $?

Successful execution:

0

---

13. Logging

The GSP Terminal defines the following logging levels:

TRACE
DEBUG
INFO
WARN
ERROR
FATAL

Default level:

INFO

Example:

[INFO] Connecting to endpoint
[DEBUG] Transport selected: GSPMTP
[INFO] Negotiation completed
[INFO] Secure session established

---

14. Quiet Mode

Quiet mode is intended for shell scripts and automation.

gsp --quiet ping gsp://example.com

Possible output:

7.82

Example:

LATENCY=$(gsp --quiet ping gsp://example.com)

---

15. Non-interactive Mode

The GSP Terminal must detect when standard input is not connected to a TTY.

Example:

echo "Hello GSP" | gsp send gsp://example.com

The terminal must not start an interactive shell in non-interactive mode.

---

16. Terminal Compatibility

The Text-only Terminal should work in:

- Linux
- Android / Termux
- macOS
- Windows Terminal
- SSH sessions
- Containers
- CI/CD environments
- BusyBox-based environments
- Embedded Linux environments

The Terminal GUI has additional graphics requirements.

---

17. Android Compatibility

The Text-only Terminal may be used through:

Termux
ADB shell
Linux containers
proot environments

Example:

gsp --text status

The Terminal GUI may only start when a compatible graphics environment and GPU are available.

---

18. Architecture

The recommended architecture is:

                 ┌──────────────────┐
                 │   GSP Terminal   │
                 └────────┬─────────┘
                          │
              ┌───────────┴───────────┐
              │                       │
       ┌──────▼──────┐        ┌───────▼───────┐
       │ Text-only   │        │ Terminal GUI  │
       │ Interface   │        │ Interface     │
       └──────┬──────┘        └───────┬───────┘
              │                       │
              └───────────┬───────────┘
                          ▼
                   ┌──────────────┐
                   │ GSP Client   │
                   └──────┬───────┘
                          ▼
                   ┌──────────────┐
                   │   GSP Core   │
                   └──────────────┘

The UI layer MUST NOT implement the GSP protocol directly.

Both interfaces should communicate with the same GSP Client API.

---

19. GPU Architecture for Terminal GUI

The Terminal GUI should use a graphics abstraction layer:

┌──────────────────────┐
│    Terminal GUI      │
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│   Graphics Backend   │
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│ GPU Driver / API     │
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│ iGPU / dGPU          │
└──────────────────────┘

The implementation may use an appropriate graphics API/backend depending on the target platform.

A GPU is a mandatory requirement for the Terminal GUI.

If GPU initialization fails:

Terminal GUI
     │
     ▼
GPU initialization
     │
     ├── SUCCESS → Start GUI
     │
     └── FAILURE → Use Text-only

---

20. Minimum GSP Terminal 1.0 Implementation

A compliant implementation should provide:

[✓] Interactive shell
[✓] Text-only mode
[✓] Terminal GUI
[✓] GPU requirement detection for GUI
[✓] connect
[✓] disconnect
[✓] reconnect
[✓] status
[✓] session management
[✓] send
[✓] receive
[✓] ping
[✓] statistics
[✓] help
[✓] version
[✓] exit
[✓] Standard streams
[✓] Exit codes
[✓] JSON output
[✓] Logging
[✓] Non-interactive operation

---

21. Core Design Principle

The GSP Terminal follows one fundamental principle:

«The interface must never be required for the GSP protocol itself to operate.»

Therefore:

GSP Core
   │
   └── No GPU required

GSP Client
   │
   └── No GPU required

Text-only Terminal
   │
   └── No GPU required

Terminal GUI
   │
   └── Integrated GPU or Dedicated GPU REQUIRED

This separation allows GSP to remain lightweight, scriptable, portable, and usable across servers, desktops, Android devices, containers, and highly constrained environments while still providing a full graphical experience when appropriate hardware is available.