# `tg-send` – Send Telegram messages & files from the command line

`tg-send` is a lightweight Bash script that lets you send text messages and files to a Telegram bot directly from your terminal.  
It is designed for **shell automation**, **piping**, and **scripting**, and integrates smoothly with **NixOS** and **Nix** package management.

---

## Features

- Send plain text, command output, or the content of files.
- Attach any file as a Telegram document (with an optional caption).
- Read configuration from **system‑wide** and **user‑specific** files.
- Override settings via **environment variables** or **command‑line flags**.
- Support for Telegram parse modes: `HTML`, `Markdown`, `MarkdownV2`.
- Returns a non‑zero exit code on failure – perfect for scripts.
- Minimal dependencies: only `curl` and `jq` (both wrapped by the Nix package).

---

## Installation

### Using Nix (recommended)

Add `tg-send` to your NixOS or home‑manager configuration.

**From your personal channel** (e.g., a Git repository with a flake or `default.nix`):

```bash
# If using a flake
nix profile install github:yourusername/your-channel#tg-send

# Or via nix-env with a channel
nix-channel --add https://your-channel-url mychannel
nix-channel --update
nix-env -iA mychannel.tg-send
```

**Local build** (from source directory):

```bash
nix-build   # creates ./result
nix-env -i ./result
# or nix profile install ./result
```

---

## Configuration

`tg-send` reads settings from the following locations (in order of precedence, **lowest to highest**):

1. **System‑wide** – `/etc/tg-send.conf`  
2. **User** – `~/.config/tg-send/config` (or `$XDG_CONFIG_HOME/tg-send/config`)  
3. **Environment variables** – `TG_TOKEN` and `TG_CHAT_ID`  
4. **Command‑line options** – `-t` and `-c`

A sample configuration file is installed at `$out/share/doc/tg-send/config.example`.  
Copy it to your preferred location and edit it:

```bash
# System-wide (requires root)
sudo cp $(nix-build --no-out-link)/share/doc/tg-send/config.example /etc/tg-send.conf
sudo chmod 600 /etc/tg-send.conf
sudo vim /etc/tg-send.conf

# User-specific (recommended)
mkdir -p ~/.config/tg-send
cp $(nix-build --no-out-link)/share/doc/tg-send/config.example ~/.config/tg-send/config
chmod 600 ~/.config/tg-send/config
vim ~/.config/tg-send/config
```

The config file is a simple shell snippet – you can set:

```bash
TOKEN="123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11"
CHAT_ID="123456789"
PARSE_MODE="HTML"          # optional
```

> **Security**: Always keep your token private – set strict permissions (`600`) on the config file.

---

## Usage

```
tg-send [options] [message]

Options:
  -t TOKEN      Bot token (overrides all other sources)
  -c CHAT_ID    Chat ID (overrides all other sources)
  -m MESSAGE    Send this text message
  -f FILE       Send this file (as a document)
  -p MODE       Parse mode for text (HTML, Markdown, MarkdownV2)
  -h, --help    Show help
```

- If **no message** is provided via `-m` or as an argument, `tg-send` reads from **stdin** – **but only if stdin is not a terminal** (i.e., when piping).
- When `-f FILE` is used, the file is sent as a document; any text is used as the **caption**.
- If both a message and a file are given, the message becomes the caption.

---

## Examples

### Basic text messages

```bash
# Direct argument
tg-send "Hello from NixOS"

# Read from stdin (piped)
echo "System load: $(uptime)" | tg-send

# Multi-line input
dmesg | tg-send -p HTML        # parse as HTML
```

### Sending files

```bash
# Send a log file with a caption
tg-send -f /var/log/syslog -m "Today's logs"

# Send a file from process substitution
some-command | tg-send -f <(cat) -m "Captured output"

# Send a screenshot (or any binary)
tg-send -f ~/screenshot.png
```

### Overriding settings temporarily

```bash
# Use a different bot for one message
tg-send -t "OTHER_TOKEN" -c "OTHER_CHAT" "Emergency alert"
```

### In scripts (error handling)

```bash
#!/bin/sh
if ! backup_command; then
    echo "Backup failed at $(date)" | tg-send
    exit 1
fi
tg-send "Backup completed successfully"
```

### Using parse modes

```bash
# MarkdownV2 (escaping required for special chars)
tg-send -p MarkdownV2 "*bold* _italic_ [link](https://example.com)"

# HTML
tg-send -p HTML "<b>bold</b> <i>italic</i>"
```

---

## Advanced: NixOS integration

You can manage the system‑wide config file declaratively by adding a small module to your `configuration.nix`:

```nix
{ config, lib, pkgs, ... }:
{
  environment.etc."tg-send.conf".text = ''
    TOKEN="your_token"
    CHAT_ID="your_chat_id"
  '';
  environment.systemPackages = [ pkgs.tg-send ];
}
```

Or, if you use home‑manager, place the config in `home.file.".config/tg-send/config"`.

---

## Troubleshooting

| Symptom | Possible cause | Solution |
|---------|----------------|----------|
| `Error: Bot token and chat ID must be set...` | No config found and no environment/options. | Create a config file or set `TG_TOKEN`/`TG_CHAT_ID`. |
| `jq: error ...` or `Error: Telegram API request failed` | Invalid token, chat ID, or network issue. | Verify your token/chat ID and internet connectivity. |
| File not sent | File too large (Telegram limit: 50 MB). | Compress or split the file. |
| Parse mode not working | Incorrect syntax or unsupported mode. | Use `HTML`, `Markdown`, or `MarkdownV2` (case‑sensitive). |
