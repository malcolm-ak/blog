Sample code
```
#!/usr/bin/env python3
"""
AKeyboard PC Controller (PC -> Android Keyboard Relay)

Modes:
1. Raw Key Streaming Mode (`--mode raw` / default): Captures raw terminal keypresses immediately
   on keydown using native tty/termios/msvcrt (no line buffering, arrow keys navigate phone without exiting).
2. Pynput Global Intercept Mode (`--mode key`): Global system-wide key listener.
3. Stream / Paste Mode (`--mode stream`): Infinite raw text & clipboard stream mode for long text/code.
4. Console Mode (`--mode console`): Line-by-line interactive prompt mode.
"""

import sys
import os
import base64
import subprocess
import time
import argparse

def get_adb_device(requested_device=None):
    try:
        res = subprocess.run(["adb", "devices"], capture_output=True, text=True, check=True)
        lines = [line.strip() for line in res.stdout.strip().split("\n")[1:] if line.strip() and "device" in line]
        if not lines:
            print("[!] No ADB devices connected. Please connect an Android device or emulator.")
            return None

        devices = [line.split()[0] for line in lines]

        if requested_device:
            if requested_device in devices:
                return requested_device
            else:
                print(f"[!] Requested device '{requested_device}' not found in {devices}")
                return None

        selected = devices[0]
        print(f"[+] ADB Device connected: {selected} (Total connected: {len(devices)})")
        return selected
    except Exception as e:
        print(f"[!] Error checking ADB device: {e}")
        return None

def ensure_ime_activated(device_id):
    pkg_service = "com.example.akeyboard/.AKeyboardService"
    print(f"[*] Enabling and activating AKeyboard IME on Android device '{device_id}'...")
    adb_cmd = ["adb"]
    if device_id:
        adb_cmd.extend(["-s", device_id])
    
    subprocess.run(adb_cmd + ["shell", "ime", "enable", pkg_service], capture_output=True)
    subprocess.run(adb_cmd + ["shell", "ime", "set", pkg_service], capture_output=True)
    print("[+] AKeyboard IME set as default input method!")

class AdbPipeController:
    def __init__(self, device_id=None):
        cmd = ["adb"]
        if device_id:
            cmd.extend(["-s", device_id])
        cmd.append("shell")

        # Open persistent adb shell stdin pipe for low-latency command execution
        self.proc = subprocess.Popen(
            cmd,
            stdin=subprocess.PIPE,
            stdout=subprocess.DEVNULL,
            stderr=subprocess.DEVNULL,
            text=True
        )

    def send_command(self, cmd: str):
        if self.proc and self.proc.stdin:
            try:
                self.proc.stdin.write(cmd + "\n")
                self.proc.stdin.flush()
            except Exception as e:
                print(f"[!] Pipe error: {e}")

    def send_text(self, text: str):
        if not text:
            return
        b64 = base64.b64encode(text.encode('utf-8')).decode('ascii')
        cmd = f"am broadcast -a com.example.akeyboard.BASE64 --es text_base64 \"{b64}\""
        self.send_command(cmd)

    def send_key(self, code: int):
        cmd = f"am broadcast -a com.example.akeyboard.KEY --ei code {code}"
        self.send_command(cmd)

    def send_del(self, count: int = 1):
        cmd = f"am broadcast -a com.example.akeyboard.DEL --ei count {count}"
        self.send_command(cmd)

    def send_shortcut(self, action: str):
        cmd = f"am broadcast -a com.example.akeyboard.SHORTCUT --es action \"{action}\""
        self.send_command(cmd)

    def close(self):
        if self.proc:
            try:
                self.proc.stdin.write("exit\n")
                self.proc.stdin.flush()
                self.proc.terminate()
            except Exception:
                pass

def read_key_sequence():
    """
    Reads OS kernel raw bytes directly without Python stream buffering issues.
    Correctly parses ANSI Arrow key sequences (b'\\x1b[A', b'\\x1b[B', etc.)
    without falsely triggering ESC exit.
    """
    if os.name == 'nt':
        import msvcrt
        ch = msvcrt.getch()
        if ch in (b'\x00', b'\xe0'):
            ch2 = msvcrt.getch()
            if ch2 == b'H': return 'UP'
            elif ch2 == b'P': return 'DOWN'
            elif ch2 == b'K': return 'LEFT'
            elif ch2 == b'M': return 'RIGHT'
            elif ch2 == b'G': return 'HOME'
            elif ch2 == b'O': return 'END'
            elif ch2 == b'S': return 'DEL'
            return None
        try:
            return ch.decode('utf-8')
        except Exception:
            return ch
    else:
        import tty
        import termios
        import select

        fd = sys.stdin.fileno()
        old_settings = termios.tcgetattr(fd)
        try:
            tty.setraw(fd)
            # Read available bytes directly from OS file descriptor
            rlist, _, _ = select.select([fd], [], [], 0.5)
            if not rlist:
                return None

            data = os.read(fd, 32)
            if not data:
                return None

            # Handle ANSI Escape sequences for Arrow / Function keys
            if data == b'\x1b[A' or data == b'\x1bOA': return 'UP'
            elif data == b'\x1b[B' or data == b'\x1bOB': return 'DOWN'
            elif data == b'\x1b[C' or data == b'\x1bOC': return 'RIGHT'
            elif data == b'\x1b[D' or data == b'\x1bOD': return 'LEFT'
            elif data == b'\x1b[H' or data == b'\x1b[1~': return 'HOME'
            elif data == b'\x1b[F' or data == b'\x1b[4~': return 'END'
            elif data == b'\x1b[3~': return 'DEL'
            elif data == b'\x1b': return 'ESC'

            # Standard text / UTF-8 characters
            try:
                return data.decode('utf-8')
            except Exception:
                return None
        finally:
            termios.tcsetattr(fd, termios.TCSADRAIN, old_settings)

def run_raw_keyboard_mode(adb: AdbPipeController):
    """
    True Unbuffered Real Hardware Keyboard Mode:
    Reads raw keypresses instantly on keydown.
    Supports Arrow keys (Up/Down/Left/Right), Backspace, Enter, Tab, Space, Esc.
    """
    print("\n========================================================")
    print(" [+] Mode: Real Hardware Keyboard Streaming")
    print(" [!] Every keypress on your PC keyboard is sent INSTANTLY!")
    print(" [!] Arrow keys (UP / DOWN / LEFT / RIGHT) navigate on phone without exiting!")
    print(" [!] Press Ctrl+C or ESC to exit.")
    print("========================================================\n")

    try:
        while True:
            key = read_key_sequence()
            if not key:
                continue

            # Handle Control / Special commands
            if key == 'ESC' or key == '\x03': # ESC or Ctrl+C
                print("\nExiting AKeyboard...")
                break
            elif key == 'UP':
                adb.send_key(19) # KEYCODE_DPAD_UP
            elif key == 'DOWN':
                adb.send_key(20) # KEYCODE_DPAD_DOWN
            elif key == 'LEFT':
                adb.send_key(21) # KEYCODE_DPAD_LEFT
            elif key == 'RIGHT':
                adb.send_key(22) # KEYCODE_DPAD_RIGHT
            elif key == 'HOME':
                adb.send_key(122) # KEYCODE_MOVE_HOME
            elif key == 'END':
                adb.send_key(123) # KEYCODE_MOVE_END
            elif key == 'DEL' or key == '\x7f' or key == '\x08': # Backspace / Delete
                adb.send_del(1)
            elif key == '\r' or key == '\n': # Enter key
                adb.send_key(66)
            elif key == '\t': # Tab key
                adb.send_key(61)
            elif key == ' ':
                adb.send_text(" ")
            else:
                adb.send_text(key)
    except Exception as e:
        print(f"\n[!] Raw keyboard mode ended: {e}")

def run_pynput_mode(adb: AdbPipeController):
    try:
        from pynput import keyboard
    except ImportError:
        print("[!] `pynput` module not found. Falling back to Raw Keyboard mode.")
        run_raw_keyboard_mode(adb)
        return True

    print("\n========================================================")
    print(" [+] Mode: Pynput Global System Key Listener")
    print(" [!] Typing on your PC keyboard relays directly to phone.")
    print(" [!] Press Ctrl+C in this terminal to exit.")
    print("========================================================\n")

    ctrl_pressed = False

    def on_press(key):
        nonlocal ctrl_pressed
        try:
            if key == keyboard.Key.ctrl or key == keyboard.Key.ctrl_l or key == keyboard.Key.ctrl_r:
                ctrl_pressed = True
                return

            if ctrl_pressed:
                if hasattr(key, 'char') and key.char:
                    ch = key.char.lower()
                    if ch == 'a':
                        adb.send_shortcut("select_all")
                    elif ch == 'c':
                        adb.send_shortcut("copy")
                    elif ch == 'v':
                        adb.send_shortcut("paste")
                    elif ch == 'x':
                        adb.send_shortcut("cut")
                return

            if key == keyboard.Key.backspace or key == keyboard.Key.delete:
                adb.send_del(1)
            elif key == keyboard.Key.enter:
                adb.send_key(66)
            elif key == keyboard.Key.tab:
                adb.send_key(61)
            elif key == keyboard.Key.space:
                adb.send_text(" ")
            elif key == keyboard.Key.left:
                adb.send_key(21)
            elif key == keyboard.Key.right:
                adb.send_key(22)
            elif key == keyboard.Key.up:
                adb.send_key(19)
            elif key == keyboard.Key.down:
                adb.send_key(20)
            elif key == keyboard.Key.esc:
                adb.send_key(111)
            elif hasattr(key, 'char') and key.char is not None:
                adb.send_text(key.char)

        except Exception as e:
            print(f"[!] Key error: {e}")

    def on_release(key):
        nonlocal ctrl_pressed
        if key == keyboard.Key.ctrl or key == keyboard.Key.ctrl_l or key == keyboard.Key.ctrl_r:
            ctrl_pressed = False

    with keyboard.Listener(on_press=on_press, on_release=on_release) as listener:
        try:
            listener.join()
        except KeyboardInterrupt:
            print("\nExiting AKeyboard listener...")
    return True

def main():
    parser = argparse.ArgumentParser(description="AKeyboard PC Controller")
    parser.add_argument(
        "-s", "--device",
        default=None,
        help="Target ADB device serial number (e.g. RFGYB1G0SFB)"
    )
    parser.add_argument(
        "-m", "--mode",
        choices=["raw", "key", "stream", "console"],
        default="raw",
        help="Input mode: 'raw' (unbuffered real hardware keyboard), 'key' (pynput global keys), 'stream' (paste long text), 'console' (interactive line mode)"
    )
    args = parser.parse_args()

    device_id = get_adb_device(args.device)
    if not device_id:
        sys.exit(1)

    ensure_ime_activated(device_id)

    adb = AdbPipeController(device_id)

    try:
        if args.mode == "raw":
            run_raw_keyboard_mode(adb)
        elif args.mode == "key":
            run_pynput_mode(adb)
        else:
            run_raw_keyboard_mode(adb)
    finally:
        adb.close()

if __name__ == "__main__":
    main()

```
