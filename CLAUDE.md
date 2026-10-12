# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

NFCMusicPlayer is an Arduino-based portable MP3 player that uses NFC tags to control playback. When an NFC tag is placed on the device, it plays the associated song from the SD card. The project includes custom PCB hardware, 3D-printed enclosure, ESP32 firmware, and a web-based configuration interface.

## Hardware Platform

- **Microcontroller**: ESP32 WROOM-32E Module (4MB)
- **NFC Reader**: PN532 module (SPI interface)
- **Audio**: MAX98357A I2S Class D amplifier
- **Storage**: SD card (SPI interface)
- **Power**: 18650 battery with TP4056 USB-C charger

## Firmware Architecture

The firmware is split across three Arduino (.ino) files that work together:

### NFCMusicPlayer.ino (main)
- **setup()**: Configures pins (LEDs, navigation buttons, power control on GPIO 22 set HIGH), then initializes hardware in sequence: SD card, mappings, PN532 (incl. RF tuning for battery operation), WiFi AP, webserver, audio library, EQ settings from SD card, NFC task
- **loop()**: Checks auto-shutdown timeout, runs `audio.loop()`, updates volume from the potentiometer, and polls the navigation buttons
- **nfcTaskCode()**: Runs as FreeRTOS task, polls for NFC tags every 2 seconds, starts/stops playback based on tag presence
- **audio_eof_mp3()**: End-of-song callback, clears `playing` and the PLAY_LED
- **loadSettings() / saveSettings()**: Read/write EQ settings (`/settings.json`) on the SD card
- **checkNavigationButtons()**: Debounced forward/backward buttons that seek within the current song
- **Error signaling**: Uses STATUS_LED to indicate errors (1s blink = SD card failure, 200ms blink = PN532 failure); STATUS_LED stays on once the player is ready

### MappingsFile.ino
Handles persistent storage of NFC tag-to-song mappings:
- **readMappings()**: Loads mappings from `/mappings.txt` on SD card at startup
- **saveMappings()**: Writes current mappings back to SD card
- **Format**: Plain text, one mapping per line: `AA:BB:CC:DD|songname.mp3`

### Webserver.ino
Provides REST API and web UI for configuration:
- Serves static web UI files from `/web/` folder on SD card
- Global middleware resets the auto-shutdown timer (`lastActivityTime`) on every HTTP request

**REST API Endpoints:**
- **GET /tagid**: Returns currently detected NFC tag UID
- **GET /songs**: Lists all MP3 files in `/music/` folder with metadata (name, size, timestamp, mapped status)
- **GET /mappings**: Returns all configured tag-to-song mappings
- **GET /download?file=**: Downloads/streams an MP3 file
- **POST /addmapping**: Adds new mapping (max 20 slots)
- **POST /delmapping**: Removes mapping by tag ID
- **POST /upload**: Handles chunked file upload for MP3 files
- **POST /deletefile**: Deletes an MP3 file (checks if mapped first)
- **POST /renamefile**: Renames an MP3 file (updates mappings automatically)
- **GET /settings**: Returns EQ settings (`bass`, `mid`, `treble`)
- **POST /settings**: Updates EQ settings (each clamped to -40..+6 dB), applies them via `audio.setTone()` and saves them to `/settings.json`

Filenames passed to `/download`, `/deletefile` and `/renamefile` are rejected if they contain `..` or `/`.

### NFCMusicPlayer.h
Central header file containing:
- All `#include` statements for required libraries
- Pin definitions for SPI buses, I2S audio, LEDs, analog inputs, navigation buttons, and power control
- Configuration constants (intervals, file paths, WiFi credentials, volume smoothing, shutdown timeout, button timing, seek steps)
- Global objects (SPI, NFC, Audio, WebServer)
- Global variables (playback state, tag ID, volume control, activity timer, button debounce state, EQ values)
- `mapping` struct definition and `List<mapping>` for tag storage

## File System Structure (SD Card)

The SD card must contain:
- `/mappings.txt` - NFC tag to song mappings (pipe-delimited: `AA:BB:CC:DD|songname.mp3`)
- `/settings.json` - EQ settings (`{"bass":0,"mid":0,"treble":0}`), created on first save; defaults to 0 dB if missing
- `/music/` - Directory containing MP3 files (not included in `sdcard/` of this repo, must be created)
- `/web/` - Web UI files:
  - `index.html` - Main web interface (Tailwind CSS dark theme)
  - `index.js` - JavaScript application logic
  - `tailwind.css` - Tailwind CSS utilities (offline-capable, ~12KB)
  - `index.css` - Minimal custom CSS (scrollbar, smooth scrolling)
  - `jquery.js` - jQuery library for AJAX and DOM manipulation

## Development Commands

### Building and Uploading Firmware with PlatformIO

Navigate to the firmware directory:
```bash
cd firmware/NFCMusicPlayer
```

**Build the firmware:**
```bash
pio run
```

**Upload to ESP32:**
```bash
pio run --target upload
```

**Monitor serial output:**
```bash
pio device monitor
```

**Build, upload, and monitor in one command:**
```bash
pio run --target upload && pio device monitor
```

**Clean build files:**
```bash
pio run --target clean
```

### Programming the ESP32
- Requires 3.3V USB-to-serial adapter (never use 5V - it will damage the ESP32)
- To enter programming mode: Press BOOT button while pressing RESET
- PlatformIO will automatically handle upload timing in most cases

## Required Libraries

PlatformIO automatically installs these dependencies from platformio.ini:
- ESP32-audioI2S 2.0.6 (from GitHub tag; README still lists 2.0.0)
- Adafruit_PN532 1.3.4
- AsyncTCP 3.4.0
- ESP Async WebServer 3.7.7
- ArduinoJson 7.4.1
- Arduino List 3.0.1

No manual library installation required when using PlatformIO.

## Key Design Patterns

### NFC Polling Strategy
- NFC reading runs in separate FreeRTOS task to avoid blocking audio playback
- Polls every 2 seconds (NFC_READ_INTERVAL)
- Tag presence triggers playback start, tag removal triggers stop
- Tag UID stored as colon-separated hex string: "AA:BB:CC:DD"

### Volume Control
- Uses analog potentiometer on GPIO 34, read on every loop iteration
- Smooths readings with an exponential moving average (VOLUME_SMOOTHING = 0.1)
- Updates the volume at most every 50 ms (VOLUME_UPDATE_MS), only when the value changes
- Logarithmic curve (exponent 2.5) maps the 0-4095 ADC range to volume 0-20
- `audio.setVolumeSteps()` is not available in ESP32-audioI2S 2.0.6+, so the library's default step range is used

### Audio Playback
- Uses ESP32-audioI2S library with I2S interface
- Streams MP3 files directly from SD card
- Callback `audio_eof_mp3()` handles end-of-song events
- Song repeats while the NFC tag remains present: at song end `playing` is cleared, and the next NFC poll restarts it
- Tone/EQ applied with `audio.setTone(bass, mid, treble)` at startup and on every `/settings` POST

### Equalizer
- Three bands: Bass, Mid, Treble, each -40 to +6 dB (UI labels: 500 Hz, 3 kHz, 6 kHz)
- Stored in `/settings.json` on the SD card, loaded in `setup()`

### Navigation Buttons
- FORWARD_BTN (GPIO 16) seeks +30 s, BACKWARD_BTN (GPIO 17) seeks -10 s via `audio.setTimeOffset()`
- Inputs use INPUT_PULLUP, active LOW
- 50 ms debounce (DEBOUNCE_DELAY), at least 300 ms between triggers (REPEAT_DELAY)
- Only active while a song is playing; each press resets the auto-shutdown timer

### Auto-Shutdown
- POWER_PIN (GPIO 22) is set HIGH in `setup()` to keep an external power circuit on
- After 2 minutes (SHUTDOWN_TIMEOUT_MS) without activity and while not playing, POWER_PIN goes LOW so the external circuit cuts power
- Activity = song playing, navigation button press, or any HTTP request (webserver middleware)

### WiFi Configuration
- Always runs as Access Point (no station mode)
- SSID: "NFCMusicPlayer"
- Password: "MyMusicPlayer"
- Static IP assigned by ESP32 softAP

## Web UI Architecture

### Design Philosophy
- **Dark Theme**: Minimalist slate-based color scheme optimized for readability
- **Offline-First**: All assets served locally from SD card (no CDN dependencies)
- **Mobile-Responsive**: Tailwind CSS utilities for adaptive layouts
- **Modern UX**: Toast notifications, smooth transitions, visual feedback

### Technology Stack
- **Tailwind CSS**: Custom minimal build (~12KB) with only used utility classes
- **jQuery**: AJAX requests and DOM manipulation
- **HTML5 Audio API**: In-browser MP3 preview playback
- **Vanilla JavaScript**: Toast notifications, file upload, table sorting

### UI Sections

**Equalizer**
- Three range sliders (Bass, Mid, Treble, -40 to +6 dB) with live dB display
- Loads current values from `GET /settings` on page load
- Saves automatically via `POST /settings` (300 ms debounce), confirms with a toast
- "Reset (Flat)" button sets all bands to 0 dB

**File Management**
- Unified table showing all MP3 files with integrated NFC tag management
- **Columns**: Play preview | Filename | Size | Upload Date | NFC Tag | Actions
- **Features**:
  - Sortable columns (name, size, date) - click headers to toggle
  - Default sort: newest files first
  - Audio preview with HTML5 Audio API (play/pause toggle)
  - **Inline NFC tag management**:
    - Unmapped files: "🔍 Scan & Assign Tag" button - one-click scan and assign
    - Mapped files: Tag ID badge with remove (✕) button
    - Automatic duplicate tag detection
  - File actions: Download, Rename, Delete
- **Upload Section**:
  - Visual file selection feedback
  - Real-time progress bar with percentage
  - Sequential file upload for reliability
  - Validates MP3 format before upload

### NFC Tag Workflow
1. User places NFC tag on the reader
2. User clicks "🔍 Scan & Assign Tag" button in the desired file's row
3. Button shows "Scanning..." loading state
4. System automatically:
   - Scans for tag via `/tagid` endpoint
   - Checks if tag is already mapped to another file
   - Assigns tag to the selected file via `/addmapping` endpoint
5. Table updates to show tag badge with remove button
6. Toast notification confirms success or reports errors

### Toast Notification System
Replaces browser `alert()` dialogs with modern, non-intrusive notifications:
- **Success** (green checkmark): File uploaded, mapping added, etc.
- **Error** (red X): Upload failed, invalid file, etc.
- **Warning** (amber triangle): No tag detected, validation issues
- **Info** (blue circle): General information
- Auto-dismiss after 3 seconds with slide-in animation

### Client-Side File Management
- **Upload**: Sequential chunked upload with progress tracking
- **Download**: Direct download via `/download?file=` endpoint
- **Delete**: Server validates file is not mapped before deletion
- **Rename**: Server automatically updates all mappings with new filename
- **Preview**: Streams file via `/download` endpoint to HTML5 Audio element

### Responsive Breakpoints
- **Below 768px**: Size and Upload Date columns hidden, EQ sliders stacked in one column
- **768px and up (`md`)**: Size column visible, EQ sliders in three columns
- **1024px and up (`lg`)**: Upload Date column visible as well

### State Management
- **Current Sort**: initialized as `{ column: 'date', ascending: false }`. Note: `refreshFileList()` calls `sortTable(currentSort.column)`, which toggles the direction, so the first load shows oldest first and every refresh flips the order (see Known Issues)
- **Current Audio**: Tracks playing preview, ensures only one plays at a time
- File list refreshes after: upload, delete, rename, mapping changes

## Pin Assignments

Critical SPI pin configuration (non-standard):
- MOSI: GPIO 23
- MISO: GPIO 19
- SCK: GPIO 18
- SD_CS: GPIO 13
- PN532_CS: GPIO 5

I2S audio pins:
- DOUT: GPIO 27
- BCLK: GPIO 26
- LRC: GPIO 25

Other pins:
- PLAY_LED: GPIO 2 (on while playing)
- STATUS_LED: GPIO 4 (error blink codes, on when ready)
- VOLUME_PIN: GPIO 34 (potentiometer, ADC)
- BATTERY_PIN: GPIO 35 (defined but not used yet)
- POWER_PIN: GPIO 22 (power control for auto-shutdown)
- FORWARD_BTN: GPIO 16, BACKWARD_BTN: GPIO 17 (navigation buttons)

## Hardware Constraints

- Maximum 20 tag mappings (MAX_SLOTS)
- MCP1826S regulator provides up to 1A at 3.3V
- Brownout detector stays enabled (protects the ESP32); the `soc/rtc_cntl_reg.h` include for disabling it is still in the header but unused
- 18650 battery provides approximately 10 hours playback

## Known Issues

- `/songs` serializes into a fixed `StaticJsonDocument<2000>`; with many files or long filenames the list gets truncated
- Upload area says "Drag and drop", but no drop handler is implemented; only click-to-browse works
- Uploading a file with an existing name overwrites it without warning
- File list sort direction flips on every `refreshFileList()` (see State Management)
- `index.js` still contains empty leftover "Refresh songs dropdown" blocks (`$.getJSON("/songs", ...)` with empty loops); there is no dropdown anymore
- `documentation/error-codes.md` has a heading only, no content
- `BATTERY_PIN` is not read, so there is no battery level indication

## Common Modifications

When modifying pin assignments:
- Update defines in NFCMusicPlayer.h
- Ensure SPI pins don't conflict with I2S
- Verify pin capabilities (ADC, output, etc.)

When adding new web endpoints:
- Add handler in Webserver.ino `initWebserver()`
- Use AsyncResponseStream with ArduinoJson for responses
- Keep JSON document sizes reasonable for ESP32 memory limits
- Update corresponding JavaScript in index.js to consume the endpoint
- Add toast notifications for user feedback

When modifying the web UI:
- **Styling**: Add utility classes to `tailwind.css` if needed (keep file minimal)
- **Custom CSS**: Use `index.css` only for browser-specific overrides (scrollbar, etc.)
- **Colors**: Dark theme uses slate-* color palette (slate-900 background, slate-100 text)
- **Responsive**: Use Tailwind breakpoints: sm (640px), md (768px), lg (1024px)
- **User Feedback**: Always use `showToast()` instead of `alert()` for notifications
- **File Operations**: Refresh the file list (`refreshFileList()`) after modifications
- **Offline Requirement**: Never use CDN links - all assets must be local on SD card
