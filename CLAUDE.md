# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**flrig** is a C++ transceiver control program for Amateur Radio. It does not use any third-party transceiver control libraries — each transceiver is encapsulated in its own class using polymorphism to reuse code across related models.

## Build System

Uses GNU Autotools (Autoconf/Automake).

```bash
# First-time or after configure.ac changes
autoreconf -fi
./configure

# Common configure options
./configure --enable-debug       # Enable debug build
./configure --enable-static      # Static linking

# Build
make -j$(nproc)

# Install
sudo make install

# macOS bundle
make appbundle
```

**Dependencies:**
- Required: FLTK, X11 (Linux)
- Optional: `libflxmlrpc >= 1.0.1` (falls back to internal `xmlrpcpp/` if absent), `libgpiod >= 2.2.1` (GPIO PTT support)

The configure summary at the end shows which optional libs were detected.

## Architecture

### Class Hierarchy

All transceivers inherit from `rigbase` (defined in `src/include/rigbase.h`, implemented in `src/rigs/rigbase.cxx`). The base class defines the interface: frequency get/set, mode, bandwidth, PTT, metering, split, filters, etc.

Concrete rig classes override only what differs from the base or from a shared series class. For example, Yaesu FT-9xx models share a common parent before specializing.

Vendor directories under `src/rigs/`:
- `yaesu/`, `icom/`, `kenwood/`, `elecraft/`, `tentec/`, `elad/`, `xiegu/`, `qrp_labs/`, `lab599/`, `other/`

Matching header subdirectories live under `src/include/`.

`src/rigs/rigs.cxx` registers all rig classes into the selectable rig list.

### I/O Layer

Communication with transceivers happens via:
- **Serial** (`src/include/serial.h`): RS-232 CAT control
- **TCP/IP socket** (`src/include/socket.h`, `socket_io.h`): networked rigs
- **TCI** (`src/include/tci_io.h`): Transceiver Control Interface (Expert Electronics)
- **HID** (`src/include/hidapi.h`, `hid_lin.h`, etc.): USB HID devices

`src/include/rig_io.h` abstracts over these transport types.

### XML-RPC Server

`src/server/` implements an XML-RPC server (`src/include/xml_server.h`, `xmlrpc_rig.h`) that exposes rig control to external programs (primarily fldigi). Uses either the internal `src/xmlrpcpp/` library or system `libflxmlrpc`.

### UI

Built with FLTK. UI dialogs are in `src/UI/`. Custom widgets (frequency spinner, signal bar, sliders) are in `src/widgets/` with headers in `src/include/` (`FreqControl.h`, `Fl_SigBar.h`, `flslider2.h`, `ValueSlider.h`).

FLUID (FLTK's UI designer) generates some `.cxx`/`.h` pairs from `.fl` files when available.

### PTT / GPIO

PTT (Push-To-Talk) is handled in `src/include/ptt.h`. GPIO PTT via `libgpiod` is in `src/gpio/` (`src/include/gpio.h`, `gpio_ptt.h`).

### State & Configuration

- `src/include/status.h` — global program state struct
- `src/include/flrigrc.h` — configuration file read/write
- `src/main.cxx` — program entry point, initialization, main loop

### Adding a New Transceiver

1. Create `src/rigs/<vendor>/ModelName.cxx` and `src/include/<vendor>/ModelName.h`
2. Inherit from `rigbase` (or an existing series base class in that vendor directory)
3. Override only the methods that differ from the base
4. Register the new class in `src/rigs/rigs.cxx`
5. Add the source file to `src/Makefile.am`
