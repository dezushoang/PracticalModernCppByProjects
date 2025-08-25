# **Software Requirements Specification (SRS)**  
**Project:** Serial Port Monitor  
**Platform:** Ubuntu Desktop + Embedded Linux (e.g., Raspberry Pi)  
**Language Standard:** C++23  
**Author:** Dezus  
**Date:** 20 Aug 2025  

---

## **1. Overview**
This project develops a lightweight, real‑time serial data monitor that connects to a microcontroller (MCU) via UART, receives structured sensor data, and visualizes it in a responsive GUI. The app should be portable between desktop Ubuntu and embedded Linux boards without heavy dependencies.

---

## **2. Functional Requirements**
### **2.1 Serial Communication**
- **UART frame**:
  - Per‑byte UART frame (idle high, LSB first):
    ```
    Idle(1) ─ Start(0) ─ D0 ─ D1 ─ D2 ─ D3 ─ D4 ─ D5 ─ D6 ─ D7 ─ [Parity?] ─ Stop(1[,1]) ─ Idle(1)...
    ```
- **Protocol**: UART, 9600 to 115200 baud. Defaults: 8N1, no flow control. User‑configurable: Parity [None, Even, Odd]; Stop bits [1, 2]; Flow control [None, RTS/CTS (hardware), XON/XOFF (software)].
- **Library**: Boost.Asio (async mode with C++23 coroutines).
- Detect available serial ports and list them in GUI.
- Allow manual port selection and connection/disconnection at runtime.
- Automatic reconnect with exponential backoff when the device is unplugged/replugged.
- **Data Format**: CSV string terminated by `;`. Example: `1692543123456,12.34,56.78;`
  - Fields: `MCU_Timestamp,Sensor1,Sensor2`
  - Timestamp unit: ms since epoch, 64‑bit.
  - Decimal separator: dot (`.`). No thousands separators. No quotes. No embedded delimiters.
  - Whitespace around fields is ignored.
- **Framing & Resynchronization**:
  - `;` is the exclusive end‑of‑message delimiter. Newlines may be present but are ignored.
  - On malformed frames or missing delimiter, discard bytes until the next `;` and attempt to resync.
  - Maximum frame size: 128 bytes. Larger frames are dropped and counted as errors.
- Robust error handling for malformed or incomplete data with error counters exposed in the UI.

### **2.2 Data Parsing**
- Parse each message into:
  - `MCU Timestamp` (int64, ms; invalid or out‑of‑range values are rejected)
  - `Sensor 1 Value` (float)
  - `Sensor 2 Value` (float)
- If critical parse errors occur, drop records and increment an error counter.
- Backpressure: cap the ingest queue to N records (default: 50k). On overflow, drop oldest or newest (configurable; default: drop oldest) and count drops.

### **2.3 GUI & Visualization**
- **Framework**: Dear ImGui for UI, ImPlot for charts, OpenGL backend.
- **Windowing**: GLFW (fallback: SDL2 if required on target).
- **Theme**: Dark mode by default.
- Two **separate chart panels**, one per sensor channel.
- Each panel supports:
  - Real‑time scrolling with a rolling history window (default: last 60 seconds).
  - Zoom and pan to review history.
  - Data decimation for rendering when sample count exceeds threshold to maintain frame rate.
- **Connection Setup Panel**:
  - Dropdown to select serial port.
  - Dropdown to select baud rate (9600 to 115200).
  - Connect/Disconnect button.
  - Connection status indicator: Green when connected, Gray when disconnected.
  - Refresh button to rescan ports.
- Display connection status, port/baud info, data rate (messages/s averaged over 1s), and error counters.
- **Pause Plotting** control: freezes charts while continuing to log.
- **Data Log Panel**:
  - Enable/Disable logging toggle.
  - Select target folder for logs.
  - Show current log file path and size.

### **2.4 Logging**
- Save incoming data to a fresh CSV file each session.
- Filename auto‑includes date/time stamp (`SerialPortMonitor-YYYY‑MM‑DD_HH‑MM‑SS.csv`).
- CSV header:
  ```
  MCU_Timestamp,Sensor1,Sensor2
  ```
  - `MCU_Timestamp`: ISO8601 with milliseconds in local time or UTC (specify; default: UTC).
- Logs saved to a configurable directory; create directory if missing.
- Flush policy: line‑buffered or flush every N records (default: every 100 records) to balance durability and performance.
- Disk‑full or write errors: surface error in UI and disable further logging.

### **2.5 Testing / Simulation**
- Provide separate CLI simulator tool:
  - Generates or replays CSV‑style test data over a pseudo‑tty.
  - Linux: creates a pty pair and prints the slave path for the GUI to open.
  - Supports rate control (messages/s), burst patterns, and replay from a CSV file.
  - Optional noise injection (drop, duplicate, corrupt frames) for parser robustness testing.

---

## **3. Non‑Functional Requirements**
- **Performance**:
  - Sustain continuous streaming at 115 200 baud with <100 ms end‑to‑end display latency (median), <150 ms (p95).
  - CPU budget: <15% of a single core on desktop; <30% on Raspberry Pi 4 during steady‑state.
  - Memory budget: <200 MB RSS with 60s history.
- **Portability**:
  - Compile and run without modification on Ubuntu x86_64 and Raspberry Pi (ARM).
  - OpenGL 3.3+ (desktop) or OpenGL ES 3.0+ (RPi). GLFW 3.3+. Boost ≥ 1.82 (coroutines compatible) or standalone Asio ≥ 1.28.
- **Usability**: Minimal learning curve for first‑time use. Persist last used port, baud, and log directory in a config file.
- **Maintainability**: Code organized into modules: serial I/O, framing/parser, data model/queues, logging, UI, rendering, configuration, simulator, and platform abstractions. Unit tests for parser and framing. 
- **Scalability**: Easy to add more sensor channels or change chart layout. (Future: support dynamic channel count with schema negotiation.)

---

## **4. Assumptions & Dependencies**
- User has permission to access serial devices; udev rule and group membership documented.
- MCU sends well‑formed CSV records at a maximum of 2000 msgs/min (or specify).
- Third‑party: Dear ImGui, ImPlot, GLFW, Boost/Asio, OpenGL.

## **5. Acceptance Criteria**
- Connect to a chosen serial port, receive and parse data at 115 200 baud for 10 minutes with no crashes, p95 latency <150 ms.
- Two charts render continuously with 60s rolling history and support zoom/pan.
- Error counters increment on injected malformed frames; UI displays data rate and errors.
- Logging produces a CSV with header and both timestamps; file is readable and complete upon exit.
- Simulator can generate and replay data; GUI can connect to its PTY device.

---