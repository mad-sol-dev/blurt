# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Blurt** is a GNOME Shell extension that provides speech-to-text input using whisper.cpp. It allows users to dictate text into any editable field by pressing a keyboard shortcut, recording speech, and transcribing it via local or remote Whisper inference.

- **Primary Language**: JavaScript (ES6+ modules for GNOME Shell)
- **Orchestrator Script**: Zsh/Bash (`wsi`) for audio capture and transcription
- **UI Framework**: GNOME Shell API (GObject Introspection bindings: Gio, St, Meta, Shell, Clutter)
- **Configuration**: GSettings with XML schemas

## Development Commands

### Schema Compilation
When modifying `schemas/org.gnome.shell.extensions.blurt.gschema.xml`, recompile with:
```bash
cd ~/.local/share/gnome-shell/extensions/blurt@quantiusbenignus.local/
glib-compile-schemas schemas/
```

### Extension Management
```bash
# Enable the extension
gnome-extensions enable blurt@quantiusbenignus.local

# Disable the extension
gnome-extensions disable blurt@quantiusbenignus.local

# List all extensions
gnome-extensions list

# View extension logs (useful for debugging)
journalctl -f -o cat /usr/bin/gnome-shell
```

### Manual Installation
```bash
# Copy extension to GNOME extensions directory
cp -r . ~/.local/share/gnome-shell/extensions/blurt@quantiusbenignus.local

# Install and make wsi script executable
cp wsi ~/.local/bin/
chmod +x ~/.local/bin/wsi

# Compile schemas
cd ~/.local/share/gnome-shell/extensions/blurt@quantiusbenignus.local/
glib-compile-schemas schemas/

# Reload GNOME Shell (X11 only - press Alt+F2, type 'r', press Enter)
# For Wayland, log out and log back in
```

## Architecture

### Three-Layer Design

1. **GNOME Extension Layer** (`extension.js`, 324 lines)
   - Manages top panel UI with indicator (Ḅ character)
   - Handles keyboard shortcuts (Ctrl+Alt+A to start, Ctrl+Alt+Z to stop)
   - Provides left-click toggle and right-click preferences
   - Visual status feedback via CSS style classes (white=normal, yellow=recording, red=error)
   - Spawns `wsi` subprocess with `Gio.Subprocess`
   - Manages state with `_isRecording` flag to prevent race conditions

2. **Preferences Layer** (`prefs.js`, 110 lines)
   - Adwaita (GTK4) preferences window
   - Configures: wsi path, clipboard vs PRIMARY selection, server IP/port
   - Uses `Gio.Settings` to persist configuration

3. **Orchestrator Script** (`wsi`, 173 lines, zsh/bash)
   - Records audio with sox (16kHz WAV format)
   - Auto-detects silence for stopping: `silence 1 0.1 3% 1 2.0 6%`
   - Supports four transcription backends (priority order):
     1. **Voxtral API** - Mistral cloud transcription (if `VOXTRAL_ENABLED=true` and `MISTRAL_API_KEY` set)
     2. **whisper.cpp server** - Network/localhost API (if `-n` flag or IP:PORT provided)
     3. **Whisperfile** - Portable executable with embedded model (if `$WHISPERFILE` set)
     4. **whisper.cpp local** - Direct `transcribe` executable (fallback)
   - Handles both X11 (`xsel`) and Wayland (`wl-copy`) clipboard
   - Stores temporary files in `/dev/shm` (RAM) for performance

### Settings Schema (`schemas/org.gnome.shell.extensions.blurt.gschema.xml`)
Key configuration keys:
- `speech-input` (array of strings): Keyboard shortcut to start recording
- `stop-recording` (array of strings): Keyboard shortcut to stop recording
- `whisper-path` (string): Relative path from $HOME to wsi script
- `whisper-ip` (string): Server hostname/IP for network mode
- `whisper-port` (string): Server port for network mode
- `use-api` (boolean): Whether to use server mode vs local transcribe
- `use-voxtral` (boolean): Whether to use Voxtral API for transcription
- `mistral-api-key` (string): Mistral API key for Voxtral access
- `use-clipboard` (boolean): Clipboard (Ctrl+V) vs PRIMARY selection (middle-click)

## Key Implementation Details

### State Management in extension.js
- **Recording state**: `_isRecording` boolean prevents overlapping recordings
- **Style classes**: Applied to `_iconLabel` for visual feedback
  - `STYLE_NORMAL = 'si-label'` - Default white indicator
  - `STYLE_RECORDING = 'i-label'` - Yellow during recording
  - `STYLE_ERROR = 'e-label'` - Red on error
- **Settings reload**: `_reloadSettings()` called on settings changes via `_settingsChangedId` signal

### Subprocess Communication
- `wsi` is invoked via `Gio.Subprocess` with stdout/stderr capture
- Command-line arguments passed from settings:
  - `--clipboard` or no flag (PRIMARY selection)
  - `-n` for network mode with hardcoded IP/port in wsi
  - `IP:PORT` for network mode with custom server address
- Extension monitors subprocess completion and displays notifications on errors

### Audio Recording Configuration (in wsi)
Sox command: `rec -t wav $ramf rate 16k silence 1 0.1 3% 1 2.0 6% `
- Records at 16kHz (whisper.cpp requirement)
- Auto-stops after 2 seconds of silence below 6% threshold
- Adjust silence parameters if environment is too noisy or voice fades out

### Transcription Backend Selection (in wsi)
Priority logic:
1. If `VOXTRAL_ENABLED=true` and `MISTRAL_API_KEY` is set, use Voxtral API via curl
2. If `-n` flag or `IP:PORT` argument provided, use whisper.cpp server via curl
3. If `$WHISPERFILE` is set and executable exists, use whisperfile
4. Otherwise, use local `transcribe` executable (fallback)

## Version Compatibility

- **Main branch**: GNOME Shell 46-48
- **ver.44/**: Legacy support for GNOME Shell 44
- **ver.45/**: Legacy support for GNOME Shell 45

When making changes, consider whether they should be backported to legacy versions.

## File Organization

- **extension.js**: Main extension class inheriting from `Extension`
- **prefs.js**: Preferences dialog using Adwaita widgets
- **wsi**: Standalone orchestrator script (works independently)
- **stylesheet.css**: Custom CSS for indicator styling
- **metadata.json**: Extension manifest (UUID, version, shell-version compatibility)
- **schemas/**: GSettings schema definition and compiled binary
- **resources/**: Icons, screenshots, demo video

## Common Development Patterns

### Adding New Settings
1. Add key to `schemas/org.gnome.shell.extensions.blurt.gschema.xml`
2. Recompile schema with `glib-compile-schemas schemas/`
3. Access in extension.js: `this._settings.get_string('key-name')`
4. Add UI in prefs.js using Adwaita widgets
5. Connect change signal: `this._settings.connect('changed', ...)`

### Modifying Keyboard Shortcuts
Edit schema key default value in gschema.xml:
```xml
<key name="speech-input" type="as">
    <default><![CDATA[['<Ctrl><Alt>A']]]></default>
```
Then recompile schema and reload extension.

### Debugging
- Use `log()` for logging to journal: `log('Blurt: message')`
- View logs: `journalctl -f -o cat /usr/bin/gnome-shell`
- Check for errors in "Looking Glass" (Alt+F2, type `lg`)
- Test wsi independently: Run `~/.local/bin/wsi` from terminal

## Network Transcription Mode

Recommended for performance. Requires whisper.cpp server running:
```bash
/path/to/whisper.cpp/build/bin/whisper-server \
  --host 0.0.0.0 --port 58080 -t 8 -nt \
  -m /dev/shm/ggml-base.en.bin &
```

**Performance**: Network mode provides ~3x speedup over local transcribe (90x real-time vs 30x real-time) even on localhost, due to persistent model loading and optimized server architecture.

**Configuration**: Set server IP/port in extension preferences, or use `-n` flag to use hardcoded values in wsi script.

## Voxtral API Mode (RECOMMENDED)

Best option for users with powerful GPUs (RTX 3070+) who don't want to keep models in VRAM 24/7, or for low-end hardware:

**Advantages:**
- **Zero VRAM usage** when idle - no 24/7 model loading
- **Better accuracy** than Whisper in most benchmarks
- **Extremely affordable** - $0.001/min (~$1.50/year for typical use)
- **No installation** - no compiling whisper.cpp or downloading models
- **Multilingual** - auto-detects 9+ languages
- **Privacy-respecting** - Mistral is European (GDPR), open-source focused

**Setup:**
1. Get free API key from https://console.mistral.ai/
2. Configure in extension preferences:
   - Enable "Use Voxtral API" toggle
   - Enter Mistral API key
3. Or configure in wsi script:
   ```bash
   MISTRAL_API_KEY="your-api-key"
   VOXTRAL_ENABLED=true
   ```

**Implementation:**
- Uses curl to POST audio file to Mistral's transcription endpoint
- Requires `jq` for JSON response parsing
- Returns plain text transcription
- Automatic fallback to other backends on failure

**Command-line arguments:**
- `-v` or `--voxtral`: Enable Voxtral mode
- `-k` or `--api-key KEY`: Set API key from command line

## Whisperfile Mode

Alternative to whisper.cpp installation:
- Download from https://huggingface.co/Mozilla/whisperfile
- Portable executable with embedded model (no installation needed)
- Uncomment `$WHISPERFILE` variable in wsi CONFIG section
- Slightly slower than compiled whisper.cpp with GPU support
- Useful for quick setup or distribution

## Important Constraints

- Extension must be compatible with GNOME Shell ES6 module system
- All UI must use GNOME Shell toolkit (St, Clutter), not GTK directly
- Settings changes require extension reload or system signals
- Audio files stored in `/dev/shm` are wiped on reboot (intentional)
- sox requires specific 16kHz WAV format for whisper.cpp compatibility
- Subprocess must handle both X11 and Wayland clipboard mechanisms
