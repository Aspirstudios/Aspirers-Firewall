# Aspirers Firewall

**A per-application firewall for Windows with a simple graphical interface.**

Aspirers Firewall controls network traffic on a **per-application basis**, rather than by port, and does **not** use the built-in Windows Firewall.

> **If the program closes or crashes, traffic is allowed to flow again. The firewall fails open, so it can never leave you without an internet connection.**

---

## Table of Contents

* [What It Does](#what-it-does)
* [Features](#features)
* [Requirements](#requirements)
* [Quick Start](#quick-start)
* [Building the EXE](#building-the-exe)
* [Using It](#using-it)
* [Configuration File](#configuration-file)
* [Command-Line Options](#command-line-options)
* [How It Works](#how-it-works)
* [Limitations](#limitations)
* [Troubleshooting](#troubleshooting)
* [Credits](#credits)
* [License](#license)

---

## What It Does

Aspirers Firewall captures TCP and UDP packets using **WinDivert**, determines which program owns each connection, and then allows or blocks the packet according to your configured application list.

You can choose between two modes:

| Mode          | Behavior                                                          |
| ------------- | ----------------------------------------------------------------- |
| **Whitelist** | Blocks all traffic except traffic from applications on your list. |
| **Blacklist** | Allows all traffic except traffic from applications on your list. |

---

## Features

* **Two filtering modes**: whitelist (block everything except listed applications) or blacklist (allow everything except listed applications).
* **Live process browser**: view running programs, the number of processes for each program, and their active connections. Search, sort, and toggle applications with a double-click.
* **Custom application list**: add any `.exe` manually or remove entries with a single click.
* **Allow DNS**: optionally allow any application to perform DNS lookups on port 53.
* **Allow local network traffic**: optionally allow communication with private, link-local, and multicast addresses.
* **Watch-only mode**: logs traffic that would be blocked without actually blocking it. Ideal for safely testing a new configuration.
* **System tray support**: closing or minimizing the window keeps the firewall running in the system tray.
* **Live logging**: blocked traffic is summarized every few seconds and also written to `blocked.log`.
* **Hot reload**: changes made to `config.txt` are detected and applied automatically while the firewall is running.
* **DHCP always allowed**: DHCP traffic is always permitted so Windows can retain its IP address.
* **Fail-open design**: if an internal error occurs, the packet is allowed through rather than being blocked.

---

## Requirements

* **Windows 10 or 11 (64-bit)**
* **Administrator privileges** — the application requests elevation when launched.
* **Python 3.9 or later**, if running from source.
* The following Python packages, which are installed automatically on first launch:

  * `psutil`
  * `pydivert`
  * `pystray`
  * `pillow`

---

## Quick Start

### Run from Source

Run:

```bash
python "Aspirers Firewall.pyw"
```

You can also double-click the file. Windows will request administrator privileges, and any missing Python packages will be installed automatically using `pip`.

### Run the Compiled Version

Alternatively, run the compiled `AspirersFirewall.exe` directly. See [Building the EXE](#building-the-exe) below for instructions.

---

## Building the EXE

1. Place the script and `logo.ico` in the same folder.
2. Install the required dependencies and PyInstaller:

```bash
pip install psutil pydivert pystray pillow pyinstaller
```

3. Build the executable:

```bash
python -m PyInstaller --onefile --noconsole --uac-admin --icon logo.ico --name AspirersFirewall --collect-all pydivert --collect-all pystray "Aspirers Firewall.pyw"
```

The resulting executable will be located at:

```text
dist\AspirersFirewall.exe
```

| Option                   | Purpose                                                                |
| ------------------------ | ---------------------------------------------------------------------- |
| `--onefile`              | Packages the application into a single executable.                     |
| `--noconsole`            | Prevents a console window from appearing.                              |
| `--uac-admin`            | Requests administrator privileges when the application starts.         |
| `--collect-all pydivert` | **Required.** Includes the WinDivert driver files (`.dll` and `.sys`). |
| `--collect-all pystray`  | Includes the required pystray backend files.                           |

> **Tip:** If `pyinstaller` is not recognized as a command, use `python -m PyInstaller` (or `py -m PyInstaller`) instead.

---

## Using It

1. Open the application and go to the **Running Processes** tab.
2. Double-click the applications you want to add to your list. Listed applications are highlighted.
3. Select a filtering **mode** at the top of the window.
4. Click **Start**.

Before enabling whitelist mode, it is strongly recommended to enable **Watch Only** first and check the **Log** tab. This lets you see what would be blocked before applying the rules for real.

### Tabs

* **Running Processes**: displays a live list of running applications, with search, sorting, and automatic refresh.
* **My List**: displays all applications currently in your configuration, with options to add or remove entries.
* **Log**: displays real-time firewall activity and mirrors the information to `blocked.log`.

> **Warning:** In whitelist mode, an empty list means **all traffic will be blocked**, including your internet connection. The application warns you before starting in this configuration.

---

## Configuration File

Settings are stored in `config.txt` next to the application. You can edit the configuration either from the application interface or manually. Changes are detected and reloaded automatically.

Example:

```text
# Firewall configuration. Edit it from the app or manually; changes are reloaded automatically.
@mode=whitelist
@dns
@lan
C:\Program Files\Mozilla Firefox\firefox.exe
chrome.exe
```

| Entry                | Meaning                                                            |
| -------------------- | ------------------------------------------------------------------ |
| `@mode=whitelist`    | Block all traffic except traffic from applications on the list.    |
| `@mode=blacklist`    | Allow all traffic except traffic from applications on the list.    |
| `@dns`               | Allow any application to use DNS (port 53).                        |
| `@lan`               | Allow any application to communicate with the local network.       |
| Full path            | Matches the specified `.exe` using its complete path.              |
| File name only       | Matches any executable with that file name, such as `firefox.exe`. |
| `#` at the beginning | Marks the line as a comment.                                       |

For backward compatibility, the older Spanish keywords (`@modo=blanca` and `@modo=negra`) and the previous `permitidos.txt` file format are still supported.

---

## Command-Line Options

| Option     | Effect                                                              |
| ---------- | ------------------------------------------------------------------- |
| `--start`  | Starts traffic filtering immediately when the application launches. |
| `--hidden` | Starts the application minimized to the system tray.                |

For example, to start the firewall with Windows:

```text
AspirersFirewall.exe --start --hidden
```

---

## How It Works

1. **WinDivert** captures non-loopback TCP and UDP packets.
2. **psutil** builds a table mapping local ports to the processes that own them.
3. For each packet, the filtering engine evaluates the following rules in order:

   1. DHCP traffic is always allowed.
   2. DNS traffic is allowed if `@dns` is enabled.
   3. Local network traffic is allowed if `@lan` is enabled.
   4. Otherwise, the packet's owning application is checked against the configured list and the active filtering mode.
4. Allowed packets are reinjected into the network stack. Blocked packets are simply dropped.

A short-term cache keeps track of recently closed connections. This helps ensure that final packets in a connection, such as FIN and ACK packets, can still be associated with the correct application.

---

## Limitations

* Filtering decisions are based on **local port lookups**. A connection that opens and closes faster than the process table can be refreshed may be attributed to an unknown application.
* Traffic from processes that cannot be identified is treated as **unlisted**.
* Only **TCP and UDP** traffic is filtered.
* This is a **hobby-grade firewall**, not a replacement for a full commercial security or endpoint-protection suite.
* Some antivirus products may flag PyInstaller executables that load a packet-capture driver. Adding an exclusion for the application folder will usually resolve the issue.

---

## Troubleshooting

### The application will not start

Make sure you are running it with administrator privileges. If you have antivirus or endpoint-security software installed, it may be preventing the WinDivert driver from loading.

### I lost my internet connection

Close the application or stop the firewall from the system tray. Traffic should immediately be allowed again.

If you are using whitelist mode, make sure your browser is included in the application list and that **Allow DNS** is enabled if required.

### `pyinstaller` is not recognized

Use:

```bash
python -m PyInstaller ...
```

instead of:

```bash
pyinstaller ...
```

### An application is being blocked even though it is on my list

Try specifying the application's full executable path instead of just its file name, or vice versa.

Check the **Log** tab to see the exact executable path associated with the blocked traffic.

---

## Credits

* [WinDivert](https://reqrypt.org/windivert.html) — packet capture and reinjection.
* [PyDivert](https://github.com/ffalcinelli/pydivert) — Python bindings for WinDivert.
* [psutil](https://github.com/giampaolo/psutil) — process and system information.
* [pystray](https://github.com/moses-palmer/pystray) — system tray integration.
* [Pillow](https://python-pillow.org/) — image processing and icon support.

---

