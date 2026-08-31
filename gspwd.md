GSP Wireless Display (GSPWD)

Globalized Secure Protocol (GSP)
GSP Wireless Display Module Specification
Abbreviation: GSPWD
Version: 1.0
Status: Experimental
Module Type: Optional

---

Table of Contents

1. Abstract
2. Scope
3. Design Goals
4. Non-Goals
5. Terminology
6. Module Model
7. Architecture
8. Device Roles
9. GSPWD URI
10. Discovery
11. Device Advertisement
12. GSP Session Requirements
13. GSPWD Handshake
14. Capability Negotiation
15. Display Discovery
16. Display Modes
17. Video Pipeline
18. Video Codec Negotiation
19. Frame Model
20. Keyframes
21. Audio Pipeline
22. Audio/Video Synchronization
23. Input Forwarding
24. Input Security
25. Multi-Display
26. Resolution Negotiation
27. Refresh Rate
28. Bitrate
29. Adaptive Streaming
30. Latency Control
31. Packet Loss and Recovery
32. Congestion Control
33. Quality Profiles
34. Stream Control
35. GSPWD Channels
36. GSPWD Message Types
37. GSPWD Frame Format
38. Timestamps
39. Sequence Numbers
40. Session States
41. Keepalive
42. Reconnection
43. Security
44. Authentication
45. Authorization
46. Display Capture Permissions
47. Privacy
48. GPU Acceleration
49. Hardware Encoding and Decoding
50. Software Fallback
51. Terminal Integration
52. Terminal GUI
53. GSPWD CLI
54. CLI Command Reference
55. Configuration
56. Statistics
57. Logging
58. Error Handling
59. Error Codes
60. Exit Codes
61. Compatibility
62. Android
63. Desktop
64. Embedded Systems
65. Resource Requirements
66. Power Management
67. Multiple Receivers
68. Receiver Selection
69. Session Termination
70. Interoperability
71. Version Negotiation
72. Extensibility
73. Implementation Requirements
74. Conformance
75. Example Session
76. Example Packet Flow
77. Reference Architecture
78. Security Considerations
79. Privacy Considerations
80. Future Extensions
81. Summary

---

1. Abstract

GSP Wireless Display (GSPWD) is an optional module of the Globalized Secure Protocol designed for wireless display streaming.

GSPWD allows a compatible device to transmit graphical display content to another compatible device over a GSP connection.

GSPWD can transport:

- Video
- Audio
- Input events
- Display configuration
- Synchronization information
- Stream control information
- Performance feedback

GSPWD is designed for:

- Screen mirroring
- Wireless monitors
- Extended displays
- Remote desktops
- Presentations
- Game streaming
- Application streaming
- Portable displays
- Device-to-device display sharing

GSPWD is an optional GSP module.

A GSP implementation does not need to implement GSPWD to be GSP compliant.

---

2. Scope

This specification defines the behavior and interoperability requirements for GSPWD implementations.

It defines:

- Device roles
- Session establishment
- Discovery
- Capability negotiation
- Display negotiation
- Media transport
- Input transport
- Synchronization
- Stream control
- Security requirements
- Error handling
- CLI behavior
- GUI requirements
- Performance monitoring

This specification does not define a specific physical wireless technology.

---

3. Design Goals

GSPWD is designed with the following priorities:

1. Low latency
2. Secure communication
3. Efficient bandwidth utilization
4. Hardware acceleration
5. Dynamic quality adaptation
6. Reliable session management
7. Cross-platform compatibility
8. Multi-display support
9. Optional audio
10. Optional input forwarding

---

4. Non-Goals

GSPWD does not define:

- A GPU driver
- An operating-system display server
- A physical wireless standard
- A mandatory video codec
- A mandatory audio codec
- A replacement for GSP Core
- A replacement for local display hardware

---

5. Terminology

Term| Definition
Source| Device generating display data
Receiver| Device rendering display data
Controller| Entity controlling a GSPWD session
Display| Logical graphical output
Stream| Continuous GSPWD media data
Frame| Encoded video frame
Keyframe| Independently decodable video frame
Session| Active GSPWD connection
Capability| Feature supported by an endpoint
Sink| Alternative name for Receiver
Capture| Acquisition of display contents
Render| Presentation of received frames

---

6. Module Model

GSPWD is layered above GSP Core.

┌────────────────────────────┐
│       Application          │
├────────────────────────────┤
│           GSPWD            │
├────────────────────────────┤
│         GSP Core           │
├────────────────────────────┤
│      GSP Transport         │
├────────────────────────────┤
│      Network / Link        │
└────────────────────────────┘

GSPWD MUST NOT bypass the GSP Core security and session mechanisms.

---

7. Architecture

              SOURCE
┌───────────────────────────────┐
│ Application / Desktop         │
│ Display Capture               │
│ Audio Capture                 │
│ Input Controller              │
│ Video Encoder                 │
│ Audio Encoder                 │
│ Stream Controller             │
└───────────────┬───────────────┘
                │
                ▼
        ┌──────────────┐
        │    GSPWD     │
        └──────┬───────┘
               │
               ▼
        ┌──────────────┐
        │   GSP Core   │
        └──────┬───────┘
               │
          Network
               │
        ┌──────▼───────┐
        │   GSP Core   │
        └──────┬───────┘
               │
               ▼
        ┌──────────────┐
        │    GSPWD     │
        └──────┬───────┘
               │
               ▼
┌───────────────────────────────┐
│ Video Decoder                │
│ Audio Decoder                │
│ Display Renderer             │
│ Input Handler                │
└───────────────────────────────┘
              RECEIVER

---

8. Device Roles

8.1 Source

The Source generates the display stream.

It is responsible for:

- Display capture
- Encoding
- Stream transmission
- Capability advertisement
- Stream adaptation
- Processing Receiver feedback

8.2 Receiver

The Receiver consumes and renders the stream.

It is responsible for:

- Capability advertisement
- Frame reception
- Decoding
- Rendering
- Audio playback
- Input generation

8.3 Controller

A Controller may manage a GSPWD session.

The Controller may:

- Start streaming
- Stop streaming
- Change quality
- Change resolution
- Change FPS
- Select displays
- Enable input
- Disable input

---

9. GSPWD URI

A GSPWD endpoint may use the following URI scheme:

gspwd://<host>[:<port>]/<resource>

Examples:

gspwd://192.168.1.20
gspwd://display.local
gspwd://tablet.local/display/0

The exact transport endpoint is determined by the GSP implementation.

---

10. Discovery

A Receiver may advertise its presence using GSP-compatible discovery mechanisms.

Discovery information may include:

Device ID
Device name
GSP version
GSPWD version
Supported codecs
Maximum resolution
Maximum FPS
Audio support
Input support
HDR support
Multi-display support

Example:

GSPWD DEVICE

Name: Living Room Display
ID: 8F20AC
GSPWD: 1.0

Video:
  H.264
  H.265
  AV1

Audio:
  Opus

Maximum Resolution:
  3840x2160

Maximum Refresh:
  120 Hz

Input:
  Supported

---

11. Device Advertisement

A device advertisement MUST NOT expose private information unnecessarily.

Recommended advertisement fields:

device_id
device_name
gsp_version
gspwd_version
capabilities
display_count

---

12. GSP Session Requirements

Before GSPWD streaming begins:

1. A GSP connection MUST exist.
2. GSP session establishment MUST complete.
3. Authentication MUST complete when required.
4. GSPWD capabilities MUST be negotiated.
5. Display permissions MUST be verified.

GSP Connection
      ↓
Authentication
      ↓
GSP Session
      ↓
GSPWD Negotiation
      ↓
Display Permission
      ↓
Streaming

---

13. GSPWD Handshake

A GSPWD handshake consists of:

GSPWD_INIT
GSPWD_CAPABILITIES
GSPWD_CAPABILITIES_ACK
GSPWD_DISPLAY_INFO
GSPWD_STREAM_CONFIG
GSPWD_READY

Example:

Source → Receiver: GSPWD_INIT
Receiver → Source: GSPWD_CAPABILITIES
Source → Receiver: GSPWD_CAPABILITIES_ACK
Source → Receiver: GSPWD_DISPLAY_INFO
Receiver → Source: GSPWD_STREAM_CONFIG
Receiver → Source: GSPWD_READY

Streaming begins only after "GSPWD_READY".

---

14. Capability Negotiation

Capabilities MUST be explicitly negotiated.

Possible capabilities include:

video_codecs
audio_codecs
max_resolution
max_fps
max_bitrate
color_formats
hdr
hardware_decode
hardware_encode
input
touch
stylus
gamepad
multi_display

Unsupported capabilities MUST NOT be selected.

---

15. Display Discovery

A Source may expose multiple displays.

Example:

Display 0
1920x1080
60 Hz
Primary

Display 1
2560x1440
144 Hz
Secondary

A Receiver may request a specific display.

---

16. Display Modes

GSPWD supports the following logical modes:

MIRROR
EXTEND
VIRTUAL
REMOTE

MIRROR

The remote display mirrors the Source display.

EXTEND

The Receiver behaves as an additional logical display.

VIRTUAL

The Source creates a virtual display specifically for GSPWD.

REMOTE

The Receiver becomes the primary graphical output for a remote session.

---

17. Video Pipeline

The recommended pipeline is:

Display
  ↓
Capture
  ↓
Color Conversion
  ↓
Video Encoder
  ↓
GSPWD Packetization
  ↓
GSP
  ↓
Network
  ↓
GSP
  ↓
GSPWD Depacketization
  ↓
Video Decoder
  ↓
Renderer
  ↓
Display

---

18. Video Codec Negotiation

GSPWD does not mandate a single codec.

Recommended codecs include:

H.264
H.265 / HEVC
AV1
VP9

A Receiver advertises supported codecs.

The Source selects one mutually supported codec.

Example:

Source:
H.264, H.265, AV1

Receiver:
H.264, H.265

Selected:
H.265

---

19. Frame Model

Encoded frames contain:

Frame Type
Sequence Number
Timestamp
Payload

Frame types may include:

I
P
B

The exact encoding structure is determined by the selected codec.

---

20. Keyframes

A Receiver may request a keyframe.

Example:

Receiver → Source
KEYFRAME_REQUEST

The Source should generate a new keyframe as soon as practical.

Keyframes may be requested after:

- Severe packet loss
- Decoder failure
- Stream recovery
- Resolution change
- Codec configuration change

---

21. Audio Pipeline

Audio is optional.

Audio Capture
      ↓
Audio Encoder
      ↓
GSPWD
      ↓
GSP
      ↓
Receiver
      ↓
Audio Decoder
      ↓
Audio Output

Recommended codecs:

Opus
AAC
PCM

---

22. Audio/Video Synchronization

Audio and video MUST contain timestamps when transmitted together.

The Receiver uses timestamps to synchronize:

Video Clock
     │
     ├─────────┐
     │         │
     ▼         ▼
  Video      Audio
     │         │
     └────┬────┘
          ▼
       Output

The Receiver should minimize drift.

---

23. Input Forwarding

Input forwarding is optional.

Supported input types may include:

Keyboard
Mouse
Touch
Stylus
Gamepad
Remote Control

Input events travel from Receiver to Source.

Receiver
   ↓
Input Event
   ↓
GSPWD
   ↓
GSP
   ↓
Source
   ↓
Operating System

---

24. Input Security

Input forwarding MUST require explicit authorization.

A Receiver must not automatically gain control over the Source.

Possible permission states:

DISABLED
PROMPT
ALLOWED
DENIED

---

25. Multi-Display

GSPWD may support multiple simultaneous displays.

Source
│
├── Display 0
├── Display 1
└── Display 2

Each display may have independent:

Resolution
Refresh rate
Position
Orientation
Scaling
Stream configuration

---

26. Resolution Negotiation

The negotiated resolution MUST NOT exceed endpoint capabilities.

Possible resolutions include:

1280x720
1920x1080
2560x1440
3840x2160

Dynamic resolution changes may occur during an active session.

---

27. Refresh Rate

GSPWD may support:

24 Hz
30 Hz
60 Hz
90 Hz
120 Hz
144 Hz
165 Hz
240 Hz

The selected rate must be supported by the complete rendering pipeline.

---

28. Bitrate

The Source should maintain a configurable target bitrate.

Example:

2 Mbps
5 Mbps
10 Mbps
20 Mbps
50 Mbps
100 Mbps

The actual bitrate may vary depending on network conditions.

---

29. Adaptive Streaming

GSPWD should monitor:

Bandwidth
RTT
Jitter
Packet loss
Decoder performance
Dropped frames

The Source may dynamically modify:

Bitrate
Resolution
FPS
Encoder parameters

Example:

Network degradation
       ↓
Bitrate ↓
       ↓
Resolution ↓
       ↓
FPS ↓

---

30. Latency Control

GSPWD prioritizes interactive latency.

Latency consists of:

Capture
+
Encode
+
Network
+
Decode
+
Render

Implementations should minimize buffering.

Interactive applications should prefer low-latency encoder configurations.

---

31. Packet Loss and Recovery

A Receiver SHOULD detect missing sequence numbers.

Example:

100
101
102
104

Frame "103" is missing.

The Receiver may:

Request retransmission
Request keyframe
Continue decoding

The selected recovery method depends on the negotiated transport behavior.

---

32. Congestion Control

GSPWD should avoid saturating the network unnecessarily.

Congestion feedback may contain:

Available bandwidth
RTT
Packet loss
Jitter
Receiver queue size

The Source uses this information for stream adaptation.

---

33. Quality Profiles

GSPWD defines recommended profiles:

LOW
BALANCED
QUALITY
ULTRA
CUSTOM

LOW

Optimized for constrained networks.

BALANCED

Balances image quality and latency.

QUALITY

Prioritizes image quality.

ULTRA

Prioritizes maximum quality and resolution.

CUSTOM

Allows explicit configuration.

---

34. Stream Control

The following logical operations are supported:

START
STOP
PAUSE
RESUME
CONFIGURE
KEYFRAME_REQUEST
QUALITY_CHANGE
DISPLAY_CHANGE

---

35. GSPWD Channels

A GSPWD session may use separate logical channels:

CONTROL
VIDEO
AUDIO
INPUT
FEEDBACK

Example:

GSPWD Session
│
├── CONTROL
├── VIDEO
├── AUDIO
├── INPUT
└── FEEDBACK

The actual underlying transport mapping is implementation-defined.

---

36. GSPWD Message Types

Initial message types:

GSPWD_INIT
GSPWD_CAPABILITIES
GSPWD_CAPABILITIES_ACK
GSPWD_DISPLAY_INFO
GSPWD_STREAM_CONFIG
GSPWD_READY
GSPWD_START
GSPWD_STOP
GSPWD_PAUSE
GSPWD_RESUME
GSPWD_VIDEO
GSPWD_AUDIO
GSPWD_INPUT
GSPWD_FEEDBACK
GSPWD_KEYFRAME_REQUEST
GSPWD_QUALITY_CHANGE
GSPWD_DISPLAY_CHANGE
GSPWD_PING
GSPWD_PONG
GSPWD_ERROR
GSPWD_CLOSE

---

37. GSPWD Frame Format

A logical GSPWD frame contains:

+----------------+----------------+
| Version        | Type           |
+----------------+----------------+
| Flags          | Channel        |
+----------------+----------------+
| Sequence Number                 |
+---------------------------------+
| Timestamp                       |
+---------------------------------+
| Payload Length                  |
+---------------------------------+
| Payload                         |
+---------------------------------+

Field sizes are implementation-specific unless explicitly defined by a future wire-format revision.

---

38. Timestamps

Media data SHOULD use monotonic timestamps.

Timestamps are required for:

- A/V synchronization
- Frame ordering
- Latency measurement
- Jitter analysis

---

39. Sequence Numbers

Video and control messages SHOULD use sequence numbers.

Sequence numbers allow detection of:

- Missing packets
- Reordered packets
- Duplicate packets

---

40. Session States

A GSPWD session uses:

IDLE
DISCOVERING
CONNECTING
NEGOTIATING
INITIALIZING
READY
STREAMING
PAUSED
RECOVERING
STOPPING
CLOSED
ERROR

Typical flow:

IDLE
 ↓
DISCOVERING
 ↓
CONNECTING
 ↓
NEGOTIATING
 ↓
INITIALIZING
 ↓
READY
 ↓
STREAMING
 ↓
STOPPING
 ↓
CLOSED

---

41. Keepalive

Active sessions SHOULD use keepalive messages.

GSPWD_PING
GSPWD_PONG

Keepalive prevents idle connections from being incorrectly considered dead.

---

42. Reconnection

A temporary network interruption SHOULD NOT immediately terminate the GSPWD session.

The implementation may attempt:

Network recovery
GSP reconnection
Session restoration
Stream reinitialization
Keyframe request

If recovery fails, the session transitions to "ERROR" or "CLOSED".

---

43. Security

GSPWD MUST operate through an authenticated GSP session.

Display data MUST NOT be transmitted through an unauthenticated GSPWD path.

Security hierarchy:

GSPWD
  ↓
GSP Security
  ↓
Encrypted GSP Session
  ↓
Transport

---

44. Authentication

Authentication is handled by GSP Core.

GSPWD MUST NOT independently replace GSP authentication.

An implementation may additionally authenticate a specific display device.

---

45. Authorization

The Source MUST authorize:

Display capture
Audio capture
Input forwarding
Remote control

Each permission may be independently controlled.

---

46. Display Capture Permissions

The operating system or application MUST provide appropriate permission mechanisms.

Example:

GSP Wireless Display wants to share your screen.

[ Allow ]   [ Deny ]

The implementation MUST NOT silently capture protected display content without the required authorization.

---

47. Privacy

GSPWD should clearly indicate active sharing.

Example:

● GSPWD DISPLAY SHARING ACTIVE

Stopping the session MUST stop display capture.

The Receiver should not retain captured display data unless explicitly configured to do so.

---

48. GPU Acceleration

GSPWD should use GPU acceleration whenever available.

Source:

Display
 ↓
GPU
 ↓
Hardware Encoder
 ↓
GSPWD

Receiver:

GSPWD
 ↓
Hardware Decoder
 ↓
GPU
 ↓
Display

---

49. Hardware Encoding and Decoding

The implementation should prefer hardware codecs when available.

Examples include platform-specific:

Hardware H.264
Hardware H.265
Hardware AV1

Hardware capabilities MUST be negotiated.

---

50. Software Fallback

If hardware acceleration is unavailable, an implementation MAY use software encoding or decoding.

Software fallback may increase:

- CPU usage
- Power consumption
- Encoding latency
- Decoding latency

---

51. Terminal Integration

GSPWD may be controlled through the GSP Terminal.

Example:

gspwd connect <device>
gspwd stream
gspwd status

---

52. Terminal GUI

The GSP Terminal GUI may provide a dedicated GSPWD panel.

Example:

╭──────────────────────────────────────────────╮
│ GSP WIRELESS DISPLAY                         │
├──────────────────────────────────────────────┤
│ Receiver: Tablet                             │
│ Status:   ● STREAMING                        │
│                                              │
│ Resolution: 2560x1600                        │
│ FPS:        120                              │
│ Bitrate:    28.4 Mbps                        │
│ Latency:    11 ms                            │
│ Codec:      AV1                              │
│                                              │
│ Video:      12.4 GB                          │
│ Audio:      842 MB                           │
│ Dropped:    12 frames                        │
╰──────────────────────────────────────────────╯

The GSP Terminal GUI itself requires:

«An integrated GPU (iGPU) or dedicated GPU (dGPU) is required for the Terminal GUI.»

This requirement applies to the GUI interface and does not apply to:

- GSP Core
- GSPWD protocol
- Text-only GSP Terminal

---

53. GSPWD CLI

The GSPWD CLI provides command-line control.

Basic syntax:

gspwd <command> [arguments] [options]

---

54. CLI Command Reference

Complete Command List

gspwd
gspwd --help
gspwd --version
gspwd --text
gspwd --ui
gspwd --verbose
gspwd --quiet
gspwd --no-color

gspwd discover
gspwd list
gspwd info <device>
gspwd capabilities
gspwd connect <device>
gspwd disconnect
gspwd reconnect
gspwd status

gspwd displays
gspwd select <display>

gspwd stream
gspwd stop
gspwd pause
gspwd resume

gspwd quality <mode>
gspwd resolution <resolution>
gspwd fps <rate>
gspwd bitrate <rate>

gspwd audio on
gspwd audio off

gspwd input on
gspwd input off

gspwd stats
gspwd ping
gspwd version
gspwd help
gspwd exit
gspwd quit

---

55. Configuration

Configuration may contain:

Default receiver
Default codec
Default resolution
Default FPS
Default bitrate
Audio enabled
Input enabled
Quality profile
Reconnect behavior
Logging level

Example:

quality=balanced
codec=auto
resolution=auto
fps=60
audio=true
input=false

---

56. Statistics

"gspwd stats" should expose:

Connection state
Resolution
FPS
Bitrate
Codec
Latency
Jitter
Packet loss
Dropped frames
Decode time
Encode time
RX bytes
TX bytes

Example:

GSPWD Statistics
────────────────────────

State:          STREAMING
Codec:          H.265
Resolution:     1920x1080
FPS:            60
Bitrate:        18.4 Mbps

Latency:        11 ms
Jitter:         1.8 ms
Packet Loss:    0.02%
Dropped Frames: 2

Encode:         3.4 ms
Decode:         2.7 ms

TX:             1.42 GB
RX:             24.1 MB

---

57. Logging

GSPWD logging levels:

TRACE
DEBUG
INFO
WARN
ERROR
FATAL

Example:

[INFO] GSPWD receiver discovered
[INFO] GSP session established
[INFO] Capabilities negotiated
[INFO] Video codec selected: AV1
[INFO] Stream started

---

58. Error Handling

Errors MUST be reported clearly.

Example:

[GSPWD-004] Unsupported video codec

The implementation SHOULD provide a human-readable explanation.

---

59. Error Codes

Initial GSPWD error codes:

Code| Meaning
"GSPWD-001"| Receiver not found
"GSPWD-002"| Receiver unavailable
"GSPWD-003"| Capability negotiation failed
"GSPWD-004"| Unsupported codec
"GSPWD-005"| Unsupported resolution
"GSPWD-006"| Encoder initialization failed
"GSPWD-007"| Decoder initialization failed
"GSPWD-008"| Display capture failed
"GSPWD-009"| Audio initialization failed
"GSPWD-010"| Input forwarding unavailable
"GSPWD-011"| Stream interrupted
"GSPWD-012"| GSP session unavailable
"GSPWD-013"| Permission denied
"GSPWD-014"| Invalid configuration
"GSPWD-015"| Receiver rejected session
"GSPWD-016"| Protocol version mismatch
"GSPWD-017"| Session timeout
"GSPWD-018"| GPU initialization failed
"GSPWD-019"| Hardware codec unavailable
"GSPWD-020"| Stream recovery failed

---

60. Exit Codes

Recommended CLI exit codes:

Code| Meaning
"0"| Success
"1"| General error
"2"| Invalid arguments
"3"| Receiver unavailable
"4"| Permission denied
"5"| GSP connection failure
"6"| Capability negotiation failure
"7"| Codec failure
"8"| Display capture failure
"9"| Streaming failure
"10"| Invalid session

---

61. Compatibility

GSPWD may be implemented on:

Desktop
Laptop
Tablet
Smartphone
Smart TV
Embedded devices
Portable displays
Game devices
Servers with display hardware

Actual functionality depends on:

- Operating system
- GPU
- Codec support
- Display stack
- Network capabilities

---

62. Android

GSPWD may be implemented on Android devices.

Possible Source:

Android phone
Android tablet

Possible Receiver:

Android tablet
Android TV
Phone
Portable display

Android implementations MUST respect the operating system's display-capture and input permissions.

---

63. Desktop

Desktop implementations may use native display capture APIs.

Supported platforms may include:

Linux
Windows
macOS

Platform-specific implementations may provide hardware acceleration.

---

64. Embedded Systems

GSPWD may be implemented on embedded Linux or other systems capable of running GSP.

A minimal Receiver may only support:

GSP
GSPWD
One video codec
One display
No input

---

65. Resource Requirements

GSPWD resource requirements depend on stream configuration.

Resources may include:

CPU
GPU
RAM
Network bandwidth
Video encoder
Video decoder
Display output

4K/120 FPS streaming requires significantly more resources than 720p/30 FPS streaming.

---

66. Power Management

Mobile implementations SHOULD adapt streaming based on power conditions.

Possible behavior:

Battery low
   ↓
Reduce FPS
   ↓
Reduce bitrate
   ↓
Reduce resolution

The user should be able to override automatic power optimization when permitted.

---

67. Multiple Receivers

A Source MAY support multiple simultaneous Receivers.

Example:

                 ┌──► Receiver A
Source ── GSP ───┼──► Receiver B
                 └──► Receiver C

Each Receiver may have an independent stream configuration.

---

68. Receiver Selection

When multiple Receivers are available, the user may select one explicitly.

Example:

$ gspwd list

1. Tablet
2. TV
3. Laptop

$ gspwd connect 2

The implementation MUST NOT connect to an unexpected Receiver without appropriate authorization.

---

69. Session Termination

A session may terminate because of:

User request
Receiver disconnect
Source disconnect
Network failure
Permission revocation
Protocol error
Resource exhaustion

The Source should stop capture after session termination.

---

70. Interoperability

Implementations MUST:

- Follow GSP session requirements
- Respect negotiated capabilities
- Reject unsupported configurations
- Correctly process mandatory messages
- Correctly report protocol errors

Implementations SHOULD support at least one common video codec and one common configuration for interoperability.

---

71. Version Negotiation

GSPWD versions are independent from GSP Core versions.

Example:

GSP:    1.0
GSPWD:  1.0

Future versions may be negotiated.

If no mutually compatible GSPWD version exists:

GSPWD-016: Protocol version mismatch

---

72. Extensibility

GSPWD is designed to allow future extensions.

Possible future modules include:

HDR
Spatial Audio
VR Display
AR Display
3D Streaming
Low-latency Game Mode
Remote GPU Rendering
Display Compression Extensions
Advanced Input

Unknown optional fields MUST NOT cause a compatible implementation to crash.

---

73. Implementation Requirements

A minimal GSPWD Source SHOULD support:

GSP session
GSPWD handshake
Capability negotiation
One display
One video codec
Video streaming
Stream termination
Error handling

A minimal Receiver SHOULD support:

GSP session
GSPWD handshake
Capability negotiation
Video decoding
Display rendering
Stream termination
Error handling

Advanced implementations MAY support:

Audio
Input
HDR
Multi-display
Adaptive bitrate
Hardware encoding
Hardware decoding
Multiple Receivers

---

74. Conformance

A GSPWD implementation is considered GSPWD 1.0 compliant when it correctly implements all mandatory requirements of this specification.

A compliance profile may be declared:

GSPWD-Source
GSPWD-Receiver
GSPWD-Controller

An implementation may support multiple profiles simultaneously.

---

75. Example Session

$ gspwd discover

GSPWD Receivers

1. Galaxy Tablet
   ID: 8F20AC
   Resolution: 2560x1600
   Refresh: 120 Hz
   Codecs: H.265, AV1
   Audio: Yes
   Input: Yes

$ gspwd connect 8F20AC

[OK] Receiver found
[OK] GSP session established
[OK] GSPWD handshake completed
[OK] Capabilities exchanged
[OK] Video codec selected: AV1
[OK] Audio codec selected: Opus
[OK] Display configuration selected
[OK] Session ready

$ gspwd stream

[OK] Display capture started
[OK] Video encoder started
[OK] Audio encoder started
[OK] GSPWD streaming

$ gspwd status

State:       STREAMING
Resolution:  2560x1600
FPS:         120
Codec:       AV1
Audio:       Opus
Bitrate:     28.4 Mbps
Latency:     11 ms

$ gspwd stop

[OK] Streaming stopped
[OK] Display capture stopped

$ gspwd disconnect

[OK] GSPWD session closed

---

76. Example Packet Flow

SOURCE                                  RECEIVER

   │                                        │
   │──── GSPWD_INIT ──────────────────────►│
   │                                        │
   │◄─── GSPWD_CAPABILITIES ───────────────│
   │                                        │
   │──── CAPABILITIES_ACK ────────────────►│
   │                                        │
   │──── DISPLAY_INFO ────────────────────►│
   │                                        │
   │◄─── STREAM_CONFIG ────────────────────│
   │                                        │
   │◄─── READY ────────────────────────────│
   │                                        │
   │──── START ───────────────────────────►│
   │                                        │
   │──── VIDEO ───────────────────────────►│
   │──── VIDEO ───────────────────────────►│
   │──── AUDIO ───────────────────────────►│
   │──── VIDEO ───────────────────────────►│
   │                                        │
   │◄─── FEEDBACK ─────────────────────────│
   │                                        │
   │──── VIDEO ───────────────────────────►│
   │                                        │
   │◄─── KEYFRAME_REQUEST ─────────────────│
   │                                        │
   │──── KEYFRAME ────────────────────────►│
   │                                        │
   │──── STOP ────────────────────────────►│
   │                                        │
   │◄─── CLOSE ───────────────────────────│

---

77. Reference Architecture

                    GSPWD SOURCE
┌───────────────────────────────────────────────┐
│                                               │
│  Desktop / Application                        │
│          │                                    │
│          ▼                                    │
│  Display Capture                              │
│          │                                    │
│          ├──────────────► Audio Capture       │
│          │                                    │
│          ▼                                    │
│  Video Encoder                                │
│          │                                    │
│          ├──────────────► Audio Encoder       │
│          │                                    │
│          ▼                                    │
│  GSPWD Stream Controller                      │
│          │                                    │
└──────────┼────────────────────────────────────┘
           │
           ▼
      ┌───────────┐
      │ GSP Core  │
      └─────┬─────┘
            │
         Network
            │
      ┌─────▼─────┐
      │ GSP Core  │
      └─────┬─────┘
            │
┌───────────┼────────────────────────────────────┐
│           ▼                                    │
│  GSPWD Receiver                                │
│                                                │
│  Stream Controller                             │
│       │                                        │
│       ├──► Video Decoder ──► Renderer ──► GPU │
│       │                                        │
│       └──► Audio Decoder ──► Audio Output      │
│                                                │
│  Input Handler ──────────────────────────────► │
│                                                │
└────────────────────────────────────────────────┘

---

78. Security Considerations

GSPWD introduces several security-sensitive operations.

These include:

- Display capture
- Audio capture
- Input forwarding
- Remote control
- Multi-device discovery

Implementations MUST protect these operations through appropriate authorization mechanisms.

The GSPWD module MUST NOT expose cryptographic secrets.

Private keys and session secrets MUST NOT appear in:

CLI output
GUI
Logs
Debug output
Statistics

---

79. Privacy Considerations

A GSPWD implementation should clearly communicate:

When display sharing is active
Which display is being shared
Whether audio is being transmitted
Whether input forwarding is enabled
Which Receiver is connected

Example:

GSPWD

Display sharing: ACTIVE
Audio sharing:   ACTIVE
Input control:   DISABLED

Receiver:
Galaxy Tablet
8F20AC

---

80. Future Extensions

Possible future GSPWD specifications may define:

GSPWD-HDR
GSPWD-4K
GSPWD-8K
GSPWD-VR
GSPWD-AR
GSPWD-Game
GSPWD-LowLatency
GSPWD-MultiDisplay
GSPWD-SpatialAudio
GSPWD-RemoteGPU

These extensions must remain compatible with the modular GSP architecture.

---

81. Summary

GSPWD provides an optional wireless display system built on top of GSP.

                    GSP
                     │
                     ▼
                   GSPWD
                     │
       ┌─────────────┼─────────────┐
       │             │             │
       ▼             ▼             ▼
     Video         Audio         Input
       │             │             │
       └─────────────┼─────────────┘
                     │
                     ▼
              Wireless Display

GSPWD provides:

- Secure wireless display streaming
- Low-latency video
- Optional audio
- Optional input forwarding
- Adaptive bitrate
- Dynamic resolution
- Dynamic FPS
- Hardware acceleration
- Multi-display support
- Multiple quality profiles
- Session recovery
- Device discovery
- Capability negotiation
- Terminal integration
- Terminal GUI integration

GSPWD remains optional and does not modify the fundamental requirements of GSP Core.

The core principle is:

«GSP provides the secure communication foundation; GSPWD provides the display-streaming functionality on top of it.»

The GSP Terminal GUI is an interface component and requires:

«An integrated GPU (iGPU) or dedicated GPU (dGPU) to work.»

The GSP Core, Text-only GSP Terminal do not require a GPU.