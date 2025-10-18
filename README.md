# Gpt-verecem-dedi
[Youtube Hotkey.py](https://github.com/user-attachments/files/22981664/Youtube.Hotkey.py)
# youtube_hotkey_gui.py
import asyncio
import threading
import time
import json
import requests
import websockets
import os
from pathlib import Path
import re
import tkinter as tk
from tkinter import font as tkfont
from datetime import datetime

# optional libs for tray & icon
try:
    from PIL import Image, ImageDraw, Image
except Exception:
    Image = None

try:
    import pystray
except Exception:
    pystray = None

try:
    import keyboard
except Exception:
    keyboard = None

# -------------------------
# Config (USER-SPECIFIC)
# -------------------------
DEBUG_URL = "http://localhost:9222/json"
ICON_PATH = Path(r"C:\Users\Metehan\Desktop\Hotkey_2.0\asset\icon.ico")  # senin verdiğin yol
FONT_NAME = "Cascadia Code"
BG_COLOR = "#000000"
TEXT_COLOR = "#838b8b"
TITLE_BAR_BG = "#101214"
TITLE_TEXT_COLOR = "#cfcfcf"

# renk sözlüğü (dilediğin gibi değiştir)
LOG_TAG_COLORS = {
    "ok": "#6bdc6b",       # [✓]
    "num": "#A0A0A0",      # numbers in brackets
    "bracket": "#B0B0B0",  # the [] themselves
    "warn": "#dc9e6b",
    "event": "#6b9edc",
    "action": TEXT_COLOR,
    "timestamp": "#A0A0A0"
}

LOOP_POLL_INTERVAL = 5
KEEPALIVE_INTERVAL = 10

# -------------------------
# State
# -------------------------
youtube_pages = []
ws_connections = []
page_initialized = False
loop = None

# tracking sets/lists to avoid spammy logs
connected_slots = set()      # slots we already logged "connected" for
disconnected_slots = set()   # slots we detected disconnected
previous_titles = []         # previous title per slot (for change detection)

# -------------------------
# GUI globals
# -------------------------
root = None
log_text = None

# -------------------------
# Helper functions for GUI logging with colored tags
# -------------------------
def gui_timestamp():
    return datetime.now().strftime("%H:%M:%S")

def safe_insert_with_tags(textwidget, message):
    if not textwidget:
        print(message)
        return
    
    timestamp = gui_timestamp()
    # timestamp'i ekle
    textwidget.insert("end", f"[{timestamp}] ", ("timestamp",))  # timestamp tag kullanılıyor
    
    pos = 0
    for m in re.finditer(r'\[\d+\]', message):
        start_idx = m.start()
        end_idx = m.end()
        if start_idx > pos:
            pre = message[pos:start_idx]
            textwidget.insert("end", pre, ("action",))
        token = m.group(0)
        textwidget.insert("end", "[", ("bracket",))
        num = token[1:-1]
        textwidget.insert("end", num, ("num",))
        textwidget.insert("end", "]", ("bracket",))
        pos = end_idx

    if pos < len(message):
        textwidget.insert("end", message[pos:], ("action",))
    
    textwidget.insert("end", "\n")
    textwidget.see("end")

def gui_log_with_label(label, message, label_tag="ok"):
    """
    Insert a label like [✓] or [!] with its color, then a timestamp (separately colored),
    then the message (parsed for [n] tokens).
    """
    if not log_text:
        # fallback to console
        print(f"{label} {gui_timestamp()} {message}")
        return
    log_text.configure(state="normal")
    # label with its tag
    start = log_text.index("end-1c")
    log_text.insert("end", f"{label} ", (label_tag,))
    end = log_text.index("end-1c")
    log_text.tag_add(label, start, end)
    # timestamp with timestamp tag
    ts = gui_timestamp()
    start = log_text.index("end-1c")
    log_text.insert("end", f"[{ts}] ", ("timestamp",))
    end = log_text.index("end-1c")
    log_text.tag_add("timestamp", start, end)
    # message (may contain [...]) handled by safe_insert_with_tags
    safe_insert_with_tags(log_text, message)
    log_text.configure(state="disabled")

def gui_log_ok(msg):
    gui_log_with_label("[✓]", msg, "ok")

def gui_log_warn(msg):
    gui_log_with_label("[!]", msg, "warn")

def gui_log_event(msg):
    gui_log_with_label("[→]", msg, "event")

def gui_log_numbered_list(items):
    """
    Print a numbered list like:
    [1] Title 1
    [2] Title 2
    with bracket/number coloring.
    """
    if not log_text:
        for i, it in enumerate(items, start=1):
            print(f"[{i}] {it}")
        return
    log_text.configure(state="normal")
    for i, it in enumerate(items, start=1):
        log_text.insert("end", "[", ("bracket",))
        log_text.insert("end", str(i), ("num",))
        log_text.insert("end", "] ", ("bracket",))
        log_text.insert("end", f"{it}\n", ("action",))
    log_text.see("end")
    log_text.configure(state="disabled")

# -------------------------
# Async websocket tasks (improved: no spam)
# -------------------------
async def update_pages():
    """
    - On first detection of found pages, connect and log once per slot.
    - On subsequent polls, only log when title for a slot changed.
    - If no pages found, clear state and log once.
    """
    global youtube_pages, ws_connections, page_initialized, connected_slots, previous_titles
    while True:
        try:
            resp = requests.get(DEBUG_URL, timeout=3).json()
            found = []
            for p in resp:
                title = p.get("title", "")
                url = p.get("url", "")
                if "youtube.com/watch" in url or "YouTube" in title:
                    found.append(p)

            if found:
                # if number of tabs changed, reinit arrays (but try to preserve previous_titles where possible)
                if not page_initialized or len(found) != len(youtube_pages):
                    youtube_pages = found
                    ws_connections = [None] * len(youtube_pages)
                    # ensure previous_titles fits
                    if len(previous_titles) != len(youtube_pages):
                        previous_titles = [None] * len(youtube_pages)
                    connected_slots.clear()
                    disconnected_slots.clear()
                    # connect initial
                    for i, page in enumerate(youtube_pages):
                        ws_url = page.get("webSocketDebuggerUrl")
                        title = page.get("title", "-")
                        if ws_url:
                            try:
                                ws = await websockets.connect(ws_url)
                                await ws.send(json.dumps({"id": 0, "method": "Runtime.enable"}))
                                ws_connections[i] = ws
                                connected_slots.add(i)
                                previous_titles[i] = title
                                gui_log_ok(f"Slot {i+1} bağlandı: {title}")
                            except Exception as e:
                                gui_log_warn(f"WS bağlanamadı (slot {i+1}): {e}")
                        else:
                            gui_log_warn(f"Slot {i+1}: webSocketDebuggerUrl eksik.")
                    page_initialized = True
                else:
                    # same number of pages; update titles and log only on change
                    youtube_pages = found
                    for i, page in enumerate(youtube_pages):
                        title = page.get("title", "-")
                        if i >= len(previous_titles):
                            previous_titles.append(title)
                        else:
                            prev = previous_titles[i]
                            if prev != title:
                                gui_log_event(f"Slot {i+1} video değişti: {title}")
                                previous_titles[i] = title
                    # optionally show current list once when requested or when titles changed — we just log events above
            else:
                # no pages found
                if page_initialized:
                    gui_log_warn("YouTube sekmesi kayboldu. Tekrar aranacak...")
                youtube_pages = []
                ws_connections = []
                page_initialized = False
                connected_slots.clear()
                disconnected_slots.clear()
                previous_titles = []

        except Exception as e:
            gui_log_warn(f"Update Pages Hatası: {e}")

        await asyncio.sleep(LOOP_POLL_INTERVAL)

async def keepalive_ws():
    """
    - Send light eval to keep WS alive.
    - Only log when connection actually transitions disconnected->connected or connected->disconnected.
    """
    global ws_connections, youtube_pages, disconnected_slots, connected_slots
    while True:
        for i in range(len(ws_connections)):
            ws = ws_connections[i] if i < len(ws_connections) else None
            if not ws:
                # attempt to reconnect if we have page info
                if i < len(youtube_pages):
                    page = youtube_pages[i]
                    ws_url = page.get("webSocketDebuggerUrl")
                    if ws_url:
                        try:
                            ws_new = await websockets.connect(ws_url)
                            await ws_new.send(json.dumps({"id": 0, "method": "Runtime.enable"}))
                            ws_connections[i] = ws_new
                            if i in disconnected_slots:
                                disconnected_slots.remove(i)
                                connected_slots.add(i)
                                gui_log_ok(f"Slot {i+1} yeniden bağlandı (keepalive).")
                            else:
                                # first-time connect handled elsewhere
                                connected_slots.add(i)
                        except Exception:
                            # still down; mark disconnected if not already
                            if i not in disconnected_slots:
                                disconnected_slots.add(i)
                                if i in connected_slots:
                                    connected_slots.remove(i)
                                    gui_log_warn(f"Slot {i+1} bağlantısı koptu (keepalive).")
                continue
            try:
                await ws.send(json.dumps({"id": 999, "method": "Runtime.evaluate", "params": {"expression": "1+1"}}))
                await ws.recv()
                # if we previously flagged as disconnected, now it's back — log once
                if i in disconnected_slots:
                    disconnected_slots.remove(i)
                    connected_slots.add(i)
                    gui_log_ok(f"Slot {i+1} yeniden bağlandı (keepalive).")
            except Exception:
                # connection problem
                if i not in disconnected_slots:
                    disconnected_slots.add(i)
                    if i in connected_slots:
                        connected_slots.remove(i)
                    gui_log_warn(f"Slot {i+1} bağlantısı koptu. Yeniden deneniyor...")
                # attempt reconnect in next iterations (no spam here)
        await asyncio.sleep(KEEPALIVE_INTERVAL)

async def send_js(slot, js):
    slot_index = slot - 1
    if slot_index >= len(ws_connections) or not ws_connections[slot_index]:
        gui_log_warn(f"Slot {slot} için YouTube sekmesi bulunamadı!")
        return
    try:
        ws = ws_connections[slot_index]
        msg = json.dumps({"id": int(time.time()), "method": "Runtime.evaluate", "params": {"expression": js}})
        await ws.send(msg)
        await ws.recv()
    except Exception as e:
        gui_log_warn(f"Slot {slot} JS gönderilemedi: {e} — yeniden bağlanıyor...")
        # reconnect attempt (will be handled by keepalive as well)
        if slot_index < len(youtube_pages):
            page = youtube_pages[slot_index]
            ws_url = page.get("webSocketDebuggerUrl")
            if ws_url:
                try:
                    ws_new = await websockets.connect(ws_url)
                    await ws_new.send(json.dumps({"id": 0, "method": "Runtime.enable"}))
                    ws_connections[slot_index] = ws_new
                    gui_log_ok(f"Slot {slot} yeniden bağlandı, JS tekrar gönderiliyor.")
                    await send_js(slot, js)
                except Exception as e2:
                    gui_log_warn(f"Slot {slot} tekrar bağlanamadı: {e2}")

async def toggle_video(slot):
    js = 'var v=document.querySelector("video"); if(v){if(v.paused){v.play(); "PLAYED"}else{v.pause(); "PAUSED"}}'
    await send_js(slot, js)
    gui_log_event(f"Slot {slot} toggle isteği gönderildi.")

async def restart_video(slot):
    js = 'var v=document.querySelector("video"); if(v){v.currentTime=0;v.play(); "RESTARTED"}'
    await send_js(slot, js)
    gui_log_event(f"Slot {slot} restart isteği gönderildi.")

async def next_video(slot):
    js = 'var n=document.querySelector(".ytp-next-button"); if(n){n.click(); "NEXT"}'
    await send_js(slot, js)
    gui_log_event(f"Slot {slot} next isteği gönderildi.")

async def prev_video(slot):
    js = 'var p=document.querySelector(".ytp-prev-button"); if(p){p.click(); "PREV"}'
    await send_js(slot, js)
    gui_log_event(f"Slot {slot} prev isteği gönderildi.")

# -------------------------
# Async loop runner
# -------------------------
def start_async_loop():
    global loop
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    tasks = [update_pages(), keepalive_ws()]
    loop.run_until_complete(asyncio.gather(*tasks))

def run_async(coro):
    if not loop:
        gui_log_warn("Async loop hazır değil.")
        return
    return asyncio.run_coroutine_threadsafe(coro, loop)

# -------------------------
# Hotkeys
# -------------------------
last_press = {"alt+1": 0, "alt+2": 0, "alt+3": 0, "alt+4": 0}
double_delay = 0.4

def on_hotkey(slot, keyname, single_func, double_func):
    now = time.time()
    if now - last_press.get(keyname, 0) < double_delay:
        run_async(double_func(slot))
    else:
        run_async(single_func(slot))
    last_press[keyname] = now

def setup_hotkeys():
    if not keyboard:
        gui_log_warn("keyboard kütüphanesi yüklü değil; global hotkeys çalışmayacak.")
        return
    keyboard.add_hotkey("alt+1", lambda: on_hotkey(1, "alt+1", toggle_video, restart_video))
    keyboard.add_hotkey("alt+2", lambda: on_hotkey(2, "alt+2", toggle_video, restart_video))
    keyboard.add_hotkey("alt+3", lambda: on_hotkey(1, "alt+3", next_video, prev_video))
    keyboard.add_hotkey("alt+4", lambda: on_hotkey(2, "alt+4", next_video, prev_video))
    gui_log_event("Hotkey sistemi aktif: Alt+1/2/3/4 kayıtlı.")

# -------------------------
# Tray & window
# -------------------------
tray_icon = None

def create_image_for_tray(size=(64,64)):
    if Image:
        img = Image.new("RGBA", size, (0,0,0,0))
        d = ImageDraw.Draw(img)
        radius = min(size)//2 - 6
        d.ellipse((6,6,size[0]-6,size[1]-6), fill=(40,40,40,255))
        d.text((size[0]//4, size[1]//6), "YT", fill=(180,180,180,255))
        return img
    return None

def on_tray_open(icon, item):
    show_window()

def on_tray_quit(icon, item):
    gui_log_event("Çıkış seçildi. Program kapanıyor.")
    if pystray and icon:
        icon.stop()
    if loop:
        try:
            loop.stop()
        except:
            pass
    try:
        root.quit()
    except:
        pass
    os._exit(0)

def init_tray():
    global tray_icon
    if not pystray:
        gui_log_warn("pystray yüklü değil; sistem tepsisi olmayacak.")
        return
    # load icon image if available
    img = None
    try:
        if ICON_PATH.exists() and Image:
            img = Image.open(str(ICON_PATH))
    except Exception:
        img = None
    if not img:
        img = create_image_for_tray()
    menu = pystray.Menu(
        pystray.MenuItem("Göster", on_tray_open),
        pystray.MenuItem("Kapat", on_tray_quit)
    )
    tray_icon = pystray.Icon("youtube_hotkey", img, "YouTube Hotkey", menu)
    threading.Thread(target=tray_icon.run, daemon=True).start()

def hide_window():
    try:
        root.withdraw()
    except:
        pass
    gui_log_event("[i] Pencere gizlendi — görev çubuğunda çalışıyor.")

def show_window():
    try:
        root.deiconify()
        root.lift()
        root.focus_force()
    except:
        pass
    gui_log_event("[i] Pencere gösterildi (tepsi -> göster).")

# -------------------------
# GUI
# -------------------------
def make_gui():
    global root, log_text
    root = tk.Tk()

    # Windows görev çubuğu ikon düzeltmesi (AppID → iconbitmap'ten önce)
    if os.name == "nt":
        import ctypes
        myappid = u"com.metehan.youtubehotkey"  # benzersiz ID
        ctypes.windll.shell32.SetCurrentProcessExplicitAppUserModelID(myappid)

    root.title("YouTube Hotkey")
    root.configure(bg=BG_COLOR)
    root.geometry("720x420")
    root.minsize(480, 260)

    # icon (AppID sonrası)
    try:
        if ICON_PATH.exists():
            root.iconbitmap(str(ICON_PATH))
    except Exception:
        pass

    # custom title bar
    root.overrideredirect(True)
    topbar = tk.Frame(root, bg=TITLE_BAR_BG, relief="raised", bd=0)
    topbar.pack(fill="x")
    title_lbl = tk.Label(topbar, text=" YouTube Hotkey", bg=TITLE_BAR_BG, fg=TITLE_TEXT_COLOR)
    title_lbl.pack(side="left", padx=6, pady=4)

    # ✕ tam kapatma, ⛔ pencereyi gizle
    def on_close_btn():  # ✕
        gui_log_event("[✗] Program kapatılıyor...")
        try:
            if tray_icon:
                tray_icon.stop()
        except:
            pass
        try:
            root.destroy()
        except:
            pass
        os._exit(0)

    def on_hide_btn():  # ⛔
        hide_window()

    btn_close = tk.Button(topbar, text="✕", command=on_close_btn, bg=TITLE_BAR_BG, fg=TITLE_TEXT_COLOR, bd=0, padx=6)
    btn_close.pack(side="right", padx=4, pady=2)
    btn_exit = tk.Button(topbar, text="⛔", command=on_hide_btn, bg=TITLE_BAR_BG, fg=TITLE_TEXT_COLOR, bd=0, padx=6)
    btn_exit.pack(side="right", padx=4, pady=2)

    # move window
    def start_move(event):
        root.x = event.x
        root.y = event.y
    def stop_move(event):
        root.x = None
        root.y = None
    def do_move(event):
        dx = event.x - getattr(root, "x", 0)
        dy = event.y - getattr(root, "y", 0)
        x = root.winfo_x() + dx
        y = root.winfo_y() + dy
        root.geometry(f"+{x}+{y}")
    topbar.bind("<Button-1>", start_move)
    topbar.bind("<ButtonRelease-1>", stop_move)
    topbar.bind("<B1-Motion>", do_move)

    content = tk.Frame(root, bg=BG_COLOR)
    content.pack(fill="both", expand=True)

    log_text = tk.Text(content, bg=BG_COLOR, fg=TEXT_COLOR, bd=0, wrap="word", state="disabled")
    log_text.pack(fill="both", expand=True, padx=8, pady=(6,8))

    # tags
    log_text.tag_configure("[✓]", foreground=LOG_TAG_COLORS["ok"], font=(FONT_NAME, 10, "bold"))
    log_text.tag_configure("[!]", foreground=LOG_TAG_COLORS["warn"], font=(FONT_NAME, 10, "bold"))
    log_text.tag_configure("num", foreground=LOG_TAG_COLORS["num"], font=(FONT_NAME, 10, "bold"))
    log_text.tag_configure("bracket", foreground=LOG_TAG_COLORS["bracket"], font=(FONT_NAME, 10, "bold"))
    log_text.tag_configure("ok", foreground=LOG_TAG_COLORS["ok"])
    log_text.tag_configure("warn", foreground=LOG_TAG_COLORS["warn"])
    log_text.tag_configure("event", foreground=LOG_TAG_COLORS["event"])
    log_text.tag_configure("action", foreground=TEXT_COLOR)
    log_text.tag_configure("timestamp", foreground=LOG_TAG_COLORS["timestamp"], font=(FONT_NAME, 9))

    # font
    try:
        available = tkfont.families()
        if FONT_NAME in available:
            log_text.configure(font=(FONT_NAME, 11))
            title_lbl.configure(font=(FONT_NAME, 11, "bold"))
        else:
            log_text.configure(font=("Consolas", 11))
            title_lbl.configure(font=("Consolas", 11, "bold"))
    except Exception:
        pass

    gui_log_ok("Program başladı.")
    gui_log_event("Sekmeler taranıyor...")

    # window events: minimize -> hide
    def on_window_state(event=None):
        try:
            if root.state() == "iconic":
                hide_window()
        except:
            pass

    root.bind("<Unmap>", on_window_state)

    # X tuşu -> tam kapatma
    root.protocol("WM_DELETE_WINDOW", on_close_btn)

    # start tray & hotkeys
    init_tray()
    threading.Thread(target=setup_hotkeys, daemon=True).start()

    return root

# -------------------------
# Entry point
# -------------------------
def main():
    t = threading.Thread(target=start_async_loop, daemon=True)
    t.start()
    r = make_gui()
    r.mainloop()

if __name__ == "__main__":
    main()
