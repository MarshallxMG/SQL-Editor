#!/usr/bin/env python3
"""
HRMS SQL Editor — Enhanced (full file)

Features:
- Light-mode startup (ttkbootstrap "flatly" if available)
- Font-size zoom slider for SQL editors
- Autocomplete (SQL keywords + schema table names)
- Results / History / Messages tabs
- Collapsible left panel (Connections / Schema)
- Context menus for result grid (copy cell/row/column, export)
- EER diagram, plotting, connection manager, workspace autosave
"""
import platform
import os
import sys
import json
import csv
import re
import time
import math
import threading
from datetime import datetime
import argparse
import tkinter as tk
from tkinter import ttk, filedialog, messagebox, simpledialog
import tkinter.font as tkfont
import warnings

warnings.filterwarnings("ignore", message="Pick support for Wedge")

try:
    import ttkbootstrap as tb  # type: ignore[import-not-found]
    from ttkbootstrap.constants import *  # type: ignore[import-not-found]
    from ttkbootstrap.icons import Icon  # type: ignore[import-not-found]
except Exception:
    tb = None
    # Fallbacks so static analysis doesn't flag undefined names when ttkbootstrap is unavailable
    SUCCESS = "success"
    DANGER = "danger"

# 🔹 Gemini AI (optional)
_HAS_GENAI = False
try:
    import google.generativeai as genai  # type: ignore[import-not-found]
    _HAS_GENAI = True
except Exception:
    genai = None  # type: ignore[assignment]

if _HAS_GENAI:
    # ✅ Configure Gemini with API key securely
    api_key = os.getenv("GEMINI_API_KEY")
    if not api_key:
        raise RuntimeError(
            "❌ Missing GEMINI_API_KEY. Please set it in your environment variables:\n"
            "   Windows (CMD):   setx GEMINI_API_KEY \"your_api_key_here\"\n"
        )
    genai.configure(api_key=api_key)

_TRIM = False
parser = argparse.ArgumentParser()
parser.add_argument("--trim", action="store_true", help="Disable optional features")
ns_args = parser.parse_args()
_TRIM = ns_args.trim

# Optional features flags
_HAS_OPENPYXL = False
_HAS_MATPLOTLIB = False
_HAS_MPLCURSORS = False
_HAS_CRYPTO = False
_HAS_GRAPHVIZ = False
_HAS_PIL = False

if not _TRIM:
    try:
        import openpyxl  # type: ignore[import-not-found]
        from openpyxl.utils import get_column_letter  # type: ignore[import-not-found]
        _HAS_OPENPYXL = True
    except Exception:
        _HAS_OPENPYXL = False

    try:
        import matplotlib  # type: ignore[import-not-found]
        matplotlib.use("TkAgg")
        from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg  # type: ignore[import-not-found]
        import matplotlib.pyplot as plt  # type: ignore[import-not-found]
        _HAS_MATPLOTLIB = True
    except Exception:
        _HAS_MATPLOTLIB = False

    try:
        import mplcursors  # type: ignore[import-not-found]
        _HAS_MPLCURSORS = True
    except Exception:
        _HAS_MPLCURSORS = False

    try:
        from cryptography.fernet import Fernet  # type: ignore[import-not-found]
        _HAS_CRYPTO = True
    except Exception:
        _HAS_CRYPTO = False

    try:
        import graphviz  # type: ignore[import-not-found]
        _HAS_GRAPHVIZ = True
    except Exception:
        _HAS_GRAPHVIZ = False

    try:
        from PIL import Image, ImageTk  # type: ignore[import-not-found]
        _HAS_PIL = True
    except Exception:
        _HAS_PIL = False

try:
    import pymysql
except Exception:
    pymysql = None

# ---- Constants ----
CONFIG_FILE = "db_config.json"
WORKSPACE_FILE = "workspace.json"
DRAFTS_DIR = ".drafts"
PAGE_SIZE = 100
AUTOSAVE_INTERVAL_MS = 15_000
COL_MIN_WIDTH = 40
COL_MAX_WIDTH = 1600
HIGHLIGHT_DEBOUNCE_MS = 150

SQL_KEYWORDS = [
    "SELECT", "FROM", "WHERE", "INSERT", "UPDATE", "DELETE", "JOIN", "LEFT", "RIGHT",
    "INNER", "OUTER", "GROUP BY", "ORDER BY", "LIMIT", "AS", "ON", "AND", "OR", "NOT", "IN", "IS", "NULL",
    "CREATE", "DROP", "ALTER", "TABLE", "DATABASE", "INDEX", "VIEW", "USE"
]

ICON_DIR = os.path.join(os.path.dirname(os.path.abspath(__file__)), "icons")


# ---- Utility: ToolTip ----
class ToolTip(object):
    def __init__(self, widget, text, delay=500):
        self.widget = widget
        self.text = text
        self.delay = delay
        self._id = None
        self._win = None
        try:
            self.widget.bind("<Enter>", self._enter, add="+")
            self.widget.bind("<Leave>", self._leave, add="+")
            self.widget.bind("<Motion>", self._motion, add="+")
        except Exception:
            pass

    def _enter(self, event=None):
        self._schedule()

    def _leave(self, event=None):
        self._unschedule()
        self._hide()

    def _motion(self, event=None):
        pass

    def _schedule(self):
        self._unschedule()
        try:
            self._id = self.widget.after(self.delay, self._show)
        except Exception:
            self._id = None

    def _unschedule(self):
        if self._id:
            try:
                self.widget.after_cancel(self._id)
            except Exception:
                pass
            self._id = None

    def _show(self):
        if self._win:
            return
        try:
            x = self.widget.winfo_rootx() + 20
            y = self.widget.winfo_rooty() + self.widget.winfo_height() + 1
            self._win = tw = tk.Toplevel(self.widget)
            tw.wm_overrideredirect(True)
            try:
                tw.attributes("-topmost", True)
            except Exception:
                pass
            label = tk.Label(tw, text=self.text, justify=tk.LEFT,
                             background="#ffffe0", relief=tk.SOLID, borderwidth=1,
                             font=("Arial", 9))
            label.pack(ipadx=4, ipady=2)
            tw.wm_geometry("+%d+%d" % (x, y))
        except Exception:
            pass

    def _hide(self):
        if self._win:
            try:
                self._win.destroy()
            except Exception:
                pass
            self._win = None


def show_toast(parent, text, kind="info", duration_ms=2200):
    try:
        tw = tk.Toplevel(parent)
        tw.wm_overrideredirect(True)
        try:
            tw.attributes("-topmost", True)
        except Exception:
            pass
        bg = {"info": "#263238", "success": "#1b5e20", "error": "#7f0000", "warn": "#4e342e"}.get(kind, "#263238")
        fg = "#ffffff"
        frm = tk.Frame(tw, bg=bg)
        frm.pack(fill=tk.BOTH, expand=True)
        lbl = tk.Label(frm, text=text, bg=bg, fg=fg, padx=12, pady=8)
        lbl.pack()
        # position bottom-right of parent
        parent.update_idletasks()
        x = parent.winfo_rootx() + parent.winfo_width() - 320
        y = parent.winfo_rooty() + parent.winfo_height() - 80
        tw.geometry(f"300x40+{max(0,x)}+{max(0,y)}")

        # animated snackbar: slide-up + fade
        try:
            tw.attributes("-alpha", 0.0)
        except Exception:
            pass
        start_y = y + 40
        end_y = y

        def animate(step=0, steps=10):
            try:
                alpha = min(1.0, step/float(steps))
                curr_y = int(start_y - (start_y - end_y) * (step/float(steps)))
                try:
                    tw.attributes("-alpha", alpha)
                except Exception:
                    pass
                tw.geometry(f"300x40+{max(0,x)}+{max(0,curr_y)}")
                if step < steps:
                    tw.after(22, lambda: animate(step+1, steps))
                else:
                    tw.after(duration_ms, start_fade_out)
            except Exception:
                try:
                    tw.destroy()
                except Exception:
                    pass

        def start_fade_out(step=10):
            try:
                if step <= 0:
                    tw.destroy()
                    return
                tw.attributes("-alpha", max(0.0, step/10.0))
                tw.after(28, lambda: start_fade_out(step-1))
            except Exception:
                try:
                    tw.destroy()
                except Exception:
                    pass

        animate(0, 10)
    except Exception:
        pass

def load_icon(name, size=(18, 18)):
    """Load icons from disk to keep colors fixed across themes."""
    if not name:
        return None

    path = os.path.join(ICON_DIR, name)
    if _HAS_PIL:
        try:
            if not os.path.exists(path):
                return None
            img = Image.open(path)
            img = img.resize(size, Image.Resampling.LANCZOS)
            return ImageTk.PhotoImage(img)
        except Exception:
            return None
    else:
        try:
            if not os.path.exists(path):
                return None
            return tk.PhotoImage(file=path)
        except Exception:
            return None

# ---- SQLEditor with autocomplete and line numbers ----
class SQLEditor(tk.Frame):
    def __init__(self, parent, font=None, dark=False, **kwargs):
        super().__init__(parent, **kwargs)
        self.dark = dark
        self.font = font or ("Consolas", 12)
        self._highlight_after_id = None
        self._numbers_after_id = None
        self.modified = False
        self.schema_tables = []
        self._autocomplete_window = None
        self._build_ui()

    def _build_ui(self):
        """Build the SQL editor UI with collapsible Find/Replace bar."""

        # --- Line number panel ---
        self.linenumber = tk.Text(
            self, width=5, padx=4, takefocus=0, border=0,
            background="#f0f0f0", state=tk.DISABLED, wrap="none",
            font=self.font
        )
        self.linenumber.pack(side=tk.LEFT, fill=tk.Y)

        # --- Editor container ---
        center = ttk.Frame(self)
        center.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

        vscroll = ttk.Scrollbar(center, orient=tk.VERTICAL)
        vscroll.pack(side=tk.RIGHT, fill=tk.Y)

        hscroll = ttk.Scrollbar(self, orient=tk.HORIZONTAL)
        hscroll.pack(side=tk.BOTTOM, fill=tk.X)

        bg = "#1e1e1e" if self.dark else "white"
        fg = "#dcdcdc" if self.dark else "black"

        self.text = tk.Text(
            center, wrap="none", undo=True, font=self.font,
            yscrollcommand=vscroll.set, xscrollcommand=hscroll.set,
            bg=bg, fg=fg, insertbackground=fg
        )
        self.text.pack(fill=tk.BOTH, expand=True, side=tk.LEFT)

        vscroll.config(command=self._on_vscroll)
        hscroll.config(command=self.text.xview)

        # --- Collapsible Find/Replace bar (hidden by default) ---
        self.findbar = ttk.Frame(self)
        self.findbar.pack(fill=tk.X, side=tk.BOTTOM)
        self.findbar.pack_forget()  # hide initially

        tk.Label(self.findbar, text="Find:").pack(side=tk.LEFT, padx=2)
        self.search_entry = tk.Entry(self.findbar)
        self.search_entry.pack(side=tk.LEFT, padx=2)

        tk.Label(self.findbar, text="Replace:").pack(side=tk.LEFT, padx=2)
        self.replace_entry = tk.Entry(self.findbar)
        self.replace_entry.pack(side=tk.LEFT, padx=2)

        tk.Button(self.findbar, text="Find", command=self._do_search).pack(side=tk.LEFT, padx=2)
        tk.Button(self.findbar, text="Replace", command=self._do_replace).pack(side=tk.LEFT, padx=2)

        # --- Event bindings ---
        self.text.bind("<<Modified>>", self._on_modified)
        self.text.bind("<KeyRelease>", lambda e: self._schedule_update_numbers())
        self.text.bind("<MouseWheel>", lambda e: self._schedule_update_numbers())
        self.text.bind("<Button-1>", lambda e: self._schedule_update_numbers())
        self.text.bind("<Return>", lambda e: self._schedule_update_numbers())
        self.text.bind("<FocusIn>", lambda e: self._schedule_update_numbers())

        # Shortcuts
        self.text.bind("<Control-f>", lambda e: self._toggle_findbar())
        self.text.bind("<Escape>", lambda e: self._hide_findbar())
        self.text.bind("<F3>", lambda e: self._do_search())

        self.update_line_numbers()
        self.highlight_syntax()

    def _toggle_findbar(self):
        """Show or hide the inline Find bar."""
        if hasattr(self, "findbar") and self.findbar.winfo_exists():
            # If already visible, close it
            self.findbar.destroy()
            return

        # --- Create Find bar ---
        self.findbar = tk.Frame(self, bg="#f0f0f0")
        self.findbar.pack(fill=tk.X, side=tk.TOP, padx=2, pady=1)

        tk.Label(self.findbar, text="Find:").pack(side=tk.LEFT, padx=4)

        self.find_var = tk.StringVar()
        find_entry = tk.Entry(self.findbar, textvariable=self.find_var)
        find_entry.pack(side=tk.LEFT, fill=tk.X, expand=True, padx=4)

        def do_find():
            target = self.find_var.get()
            if not target:
                return
            start = self.text.search(target, "insert", stopindex=tk.END)
            if start:
                end = f"{start}+{len(target)}c"
                self.text.tag_remove("find_highlight", "1.0", tk.END)
                self.text.tag_add("find_highlight", start, end)
                self.text.tag_config("find_highlight", background="yellow")
                self.text.mark_set("insert", end)
                self.text.see("insert")

        tk.Button(self.findbar, text="Find", command=do_find).pack(side=tk.LEFT, padx=4)

        # Focus the entry when opening
        find_entry.focus_set()

    def _show_findbar(self, event=None):
        # If already open, don't create again
        if hasattr(self, "findbar") and self.findbar.winfo_exists():
            return  

        # Floating window (no border, hover style)
        self.findbar = tk.Toplevel(self.text)
        self.findbar.wm_overrideredirect(True)
        self.findbar.configure(bg="#2d2d2d")

        # Position top-right corner of editor
        x = self.text.winfo_rootx() + self.text.winfo_width() - 320
        y = self.text.winfo_rooty() + 10
        self.findbar.geometry(f"300x70+{x}+{y}")

        # ---- UI ----
        # Find
        tk.Label(self.findbar, text="Find:", fg="white", bg="#2d2d2d").pack(anchor="w", padx=4, pady=(4,0))
        self.search_entry = tk.Entry(self.findbar, width=25, bg="#1e1e1e", fg="white", insertbackground="white")
        self.search_entry.pack(fill="x", padx=4)
        self.search_entry.focus_set()

        # Replace
        tk.Label(self.findbar, text="Replace:", fg="white", bg="#2d2d2d").pack(anchor="w", padx=4, pady=(4,0))
        self.replace_entry = tk.Entry(self.findbar, width=25, bg="#1e1e1e", fg="white", insertbackground="white")
        self.replace_entry.pack(fill="x", padx=4)

        # Buttons
        btn_frame = tk.Frame(self.findbar, bg="#2d2d2d")
        btn_frame.pack(pady=4)
        tk.Button(btn_frame, text="Find Next", command=self._do_search).pack(side=tk.LEFT, padx=2)
        tk.Button(btn_frame, text="Replace", command=self._do_replace).pack(side=tk.LEFT, padx=2)
        tk.Button(btn_frame, text="Replace All", command=self._do_replace_all).pack(side=tk.LEFT, padx=2)

        # Bindings
        self.search_entry.bind("<Return>", lambda e: self._do_search())
        self.findbar.bind("<Escape>", lambda e: self._hide_findbar())

    def _hide_findbar(self, event=None):
        if hasattr(self, "findbar") and self.findbar.winfo_exists():
            self.findbar.destroy()
            self.findbar = None

    def _do_search(self):
        query = self.search_entry.get()
        if not query:
            return
        start = self.text.search(query, "insert", tk.END)
        if start:
            end = f"{start}+{len(query)}c"
            self.text.tag_remove("sel", "1.0", tk.END)
            self.text.tag_add("sel", start, end)
            self.text.mark_set("insert", end)
            self.text.see(start)

    def _do_replace(self):
        if not hasattr(self, "search_entry") or not hasattr(self, "replace_entry"):
            return
        query = self.search_entry.get()
        repl = self.replace_entry.get()
        if not query:
            return
        start = self.text.search(query, "insert", tk.END)
        if start:
            end = f"{start}+{len(query)}c"
            self.text.delete(start, end)
            self.text.insert(start, repl)
            self.text.mark_set("insert", f"{start}+{len(repl)}c")
            self._do_search()  # move to next

    def _do_replace_all(self):
        query = self.search_entry.get()
        repl = self.replace_entry.get()
        if not query:
            return
        idx = "1.0"
        while True:
            start = self.text.search(query, idx, tk.END)
            if not start:
                break
            end = f"{start}+{len(query)}c"
            self.text.delete(start, end)
            self.text.insert(start, repl)
            idx = f"{start}+{len(repl)}c"

    def _on_vscroll(self, *args):
        self.text.yview(*args)
        try:
            self.linenumber.yview(*args)
        except Exception:
            pass

    def _schedule_update_numbers(self):
        if self._numbers_after_id:
            try:
                self.after_cancel(self._numbers_after_id)
            except Exception:
                pass
        self._numbers_after_id = self.after(25, self.update_line_numbers)

    def _schedule_highlight(self):
        if self._highlight_after_id:
            try:
                self.after_cancel(self._highlight_after_id)
            except Exception:
                pass
        self._highlight_after_id = self.after(HIGHLIGHT_DEBOUNCE_MS, self.highlight_syntax)

    def update_line_numbers(self):
        try:
            self.linenumber.configure(state=tk.NORMAL)
            self.linenumber.delete("1.0", tk.END)
            end_line = int(self.text.index("end-1c").split(".")[0])
            nums = "\n".join(str(i) for i in range(1, end_line + 1))
            self.linenumber.insert("1.0", nums)
            self.linenumber.configure(state=tk.DISABLED)
            self.linenumber.yview_moveto(self.text.yview()[0])
        except Exception:
            pass

    def get(self):
        return self.text.get("1.0", tk.END)

    def insert(self, index, text):
        self.text.insert(index, text)
        self.modified = True
        self._schedule_update_numbers()
        self._schedule_highlight()

    def delete(self, start, end):
        self.text.delete(start, end)
        self.modified = True
        self._schedule_update_numbers()
        self._schedule_highlight()

    def load_file(self, path):
        with open(path, "r", encoding="utf-8") as f:
            content = f.read()
        self.text.delete("1.0", tk.END)
        self.text.insert("1.0", content)
        self.update_line_numbers()
        self.modified = False
        self.highlight_syntax()

    def _on_modified(self, event=None):
        try:
            self.modified = self.text.edit_modified()
            self.text.edit_modified(False)
        except Exception:
            pass

    def highlight_syntax(self):
        try:
            text = self.text.get("1.0", tk.END)
            for tag in ("kw", "str", "cmt", "num"):
                self.text.tag_remove(tag, "1.0", tk.END)
            for kw in SQL_KEYWORDS:
                for m in re.finditer(r"\b" + re.escape(kw) + r"\b", text, re.IGNORECASE):
                    start = "1.0+%dc" % m.start()
                    end = "1.0+%dc" % m.end()
                    self.text.tag_add("kw", start, end)
            for m in re.finditer(r"'[^']*'", text):
                start = "1.0+%dc" % m.start()
                end = "1.0+%dc" % m.end()
                self.text.tag_add("str", start, end)
            for m in re.finditer(r"--.*", text):
                start = "1.0+%dc" % m.start()
                end = "1.0+%dc" % m.end()
                self.text.tag_add("cmt", start, end)
            for m in re.finditer(r"\b\d+\b", text):
                start = "1.0+%dc" % m.start()
                end = "1.0+%dc" % m.end()
                self.text.tag_add("num", start, end)
            if self.dark:
                self.text.tag_configure("kw", foreground="#569CD6")
                self.text.tag_configure("str", foreground="#CE9178")
                self.text.tag_configure("cmt", foreground="#6A9955")
                self.text.tag_configure("num", foreground="#B5CEA8")
            else:
                self.text.tag_configure("kw", foreground="blue")
                self.text.tag_configure("str", foreground="green")
                self.text.tag_configure("cmt", foreground="gray")
                self.text.tag_configure("num", foreground="purple")
        except Exception:
            pass

    def selection_get(self):
        try:
            return self.text.get(tk.SEL_FIRST, tk.SEL_LAST)
        except Exception:
            return self.get()

    # --- Autocomplete setup and helpers ---
    def _setup_autocomplete(self, schema_tables=None):
        self._autocomplete_window = None
        self.schema_tables = schema_tables or []
        try:
            self.text.bind("<KeyRelease>", self._show_autocomplete, add="+")
        except Exception:
            pass

    def _show_autocomplete(self, event=None):
        try:
            word = self.get_current_word()
            if not word or len(word) < 2:
                self._hide_autocomplete()
                return

            suggestions = [kw for kw in SQL_KEYWORDS if kw.lower().startswith(word.lower())]
            suggestions += [t for t in self.schema_tables if t.lower().startswith(word.lower())]

            if not suggestions:
                self._hide_autocomplete()
                return

            if self._autocomplete_window:
                self._autocomplete_window.destroy()

            bbox = self.text.bbox("insert")
            if not bbox:
                self._hide_autocomplete()
                return
            x, y, _, _ = bbox
            x += self.text.winfo_rootx()
            y += self.text.winfo_rooty() + 20

            self._autocomplete_window = tw = tk.Toplevel(self.text)
            tw.wm_overrideredirect(True)
            try:
                tw.attributes("-topmost", True)
            except Exception:
                pass
            tw.geometry(f"+{x}+{y}")

            lb = tk.Listbox(tw, height=min(6, len(suggestions)))
            lb.pack(fill="both", expand=True)
            for s in suggestions:
                lb.insert(tk.END, s)

            def insert_selection(evt=None):
                try:
                    choice = lb.get(tk.ACTIVE)
                except Exception:
                    choice = None
                if choice:
                    self.replace_current_word(choice)
                self._hide_autocomplete()

            lb.bind("<Double-1>", insert_selection)
            lb.bind("<Return>", insert_selection)
        except Exception:
            self._hide_autocomplete()

    def get_current_word(self):
        try:
            index = self.text.index("insert wordstart")
            return self.text.get(index, "insert")
        except Exception:
            return ""

    def replace_current_word(self, new_text):
        try:
            self.text.delete("insert wordstart", "insert")
            self.text.insert("insert", new_text)
        except Exception:
            pass

    def _hide_autocomplete(self):
        if self._autocomplete_window:
            try:
                self._autocomplete_window.destroy()
            except Exception:
                pass
            self._autocomplete_window = None


# ---- EER helper functions & TableNode (kept from your original) ----
def _create_rounded_rect(canvas, x1, y1, x2, y2, r=8, **opts):
    items = []
    items.append(canvas.create_arc(x1, y1, x1+2*r, y1+2*r, start=90, extent=90, style=tk.PIESLICE, **opts))
    items.append(canvas.create_arc(x2-2*r, y1, x2, y1+2*r, start=0, extent=90, style=tk.PIESLICE, **opts))
    items.append(canvas.create_arc(x2-2*r, y2-2*r, x2, y2, start=270, extent=90, style=tk.PIESLICE, **opts))
    items.append(canvas.create_arc(x1, y2-2*r, x1+2*r, y2, start=180, extent=90, style=tk.PIESLICE, **opts))
    items.append(canvas.create_rectangle(x1 + r, y1, x2 - r, y2, **opts))
    items.append(canvas.create_rectangle(x1, y1 + r, x2, y2 - r, **opts))
    return items

def _create_box_with_shadow(canvas, x1, y1, x2, y2, r=10, fill="#ffffff", outline="#333333", shadow_color="#888888", shadow_offset=(6,6)):
    tag = f"box_{id(canvas)}_{x1}_{y1}_{x2}_{y2}"
    sx, sy = shadow_offset
    shadow_items = _create_rounded_rect(canvas, x1+sx, y1+sy, x2+sx, y2+sy, r=r, fill=shadow_color, outline="", width=0)
    box_items = _create_rounded_rect(canvas, x1, y1, x2, y2, r=r, fill=fill, outline=outline, width=1)
    for it in shadow_items + box_items:
        canvas.addtag_withtag(tag, it)
    return tag

def _draw_header_gradient(canvas, x1, y1, x2, y2, color1="#0078D7", color2="#005A9E", steps=20, tag=None):
    height = y2 - y1
    for i in range(steps):
        r = i / steps
        try:
            c1 = canvas.winfo_rgb(color1)
            c2 = canvas.winfo_rgb(color2)
            blended = "#%02x%02x%02x" % (
                int(c1[0]*(1-r)/256 + c2[0]*r/256),
                int(c1[1]*(1-r)/256 + c2[1]*r/256),
                int(c1[2]*(1-r)/256 + c2[2]*r/256),
            )
        except Exception:
            blended = color1
        y_start = y1 + int(i * height/steps)
        y_end   = y1 + int((i+1) * height/steps)
        canvas.create_rectangle(x1, y_start, x2, y_end, fill=blended, outline="", tags=(tag,))

class TableNode:
    BOX_WIDTH = 220
    HEADER_H = 28
    ROW_H = 20
    PAD = 6
    BOTTOM_PAD = 8

    def __init__(self, canvas, name, cols, x=20, y=20):
        self.canvas = canvas
        self.name = name
        self.cols = cols or []
        self.x, self.y = x, y
        self.width = TableNode.BOX_WIDTH
        self.height = TableNode.HEADER_H + len(self.cols) * TableNode.ROW_H + TableNode.PAD + TableNode.BOTTOM_PAD
        self.tag = None
        self.text_ids = []
        self.draw()

    def draw(self):
        x1, y1 = self.x, self.y
        x2, y2 = x1 + self.width, y1 + self.height

        self.tag = _create_box_with_shadow(
            self.canvas, x1, y1, x2, y2,
            r=10, fill="#fdfdfd", outline="#555555",
            shadow_color="#aaaaaa", shadow_offset=(4, 4)
        )

        _draw_header_gradient(
            self.canvas, x1, y1, x2, y1 + TableNode.HEADER_H,
            color1="#0078D7", color2="#005A9E", tag=self.tag
        )

        title_id = self.canvas.create_text(
            x1 + 10, y1 + TableNode.HEADER_H/2,
            anchor="w", text=self.name,
            font=("Segoe UI", 10, "bold"), fill="white",
            tags=(self.tag,)
        )
        self.text_ids = [title_id]

        for i, c in enumerate(self.cols):
            cy = y1 + TableNode.HEADER_H + i * TableNode.ROW_H + TableNode.ROW_H/2 + 3
            line_y = y1 + TableNode.HEADER_H + (i+1) * TableNode.ROW_H
            if i < len(self.cols)-1:
                self.canvas.create_line(x1 + 5, line_y, x2 - 5, line_y, fill="#e0e0e0", width=1, tags=(self.tag,))
            tid = self.canvas.create_text(x1 + 10, cy, anchor="w", text=c, font=("Segoe UI", 9), fill="#333333", tags=(self.tag,))
            self.text_ids.append(tid)

        # bindings
        self.canvas.tag_bind(self.tag, "<Enter>", self.on_hover_in)
        self.canvas.tag_bind(self.tag, "<Leave>", self.on_hover_out)
        self.canvas.tag_bind(self.tag, "<ButtonPress-1>", self.start_drag)
        self.canvas.tag_bind(self.tag, "<B1-Motion>", self.on_drag)

    def on_hover_in(self, event=None):
        items = self.canvas.find_withtag(self.tag)
        for it in items:
            if self.canvas.type(it) in ("rectangle", "arc"):
                try:
                    self.canvas.itemconfig(it, outline="#00aaff", width=2)
                except Exception:
                    pass

    def on_hover_out(self, event=None):
        items = self.canvas.find_withtag(self.tag)
        for it in items:
            if self.canvas.type(it) in ("rectangle", "arc"):
                try:
                    self.canvas.itemconfig(it, outline="#555555", width=1)
                except Exception:
                    pass

    def start_drag(self, event):
        self.drag_start_x, self.drag_start_y = event.x, event.y

    def on_drag(self, event):
        dx, dy = event.x - self.drag_start_x, event.y - self.drag_start_y
        self.move(dx, dy)
        self.drag_start_x, self.drag_start_y = event.x, event.y

    def move(self, dx, dy):
        self.x += dx
        self.y += dy
        try:
            self.canvas.move(self.tag, dx, dy)
        except Exception:
            pass

    def contains(self, cx, cy):
        return self.x <= cx <= self.x + self.width and self.y <= cy <= self.y + self.height

    def column_at(self, cx, cy):
        if not self.contains(cx, cy):
            return None
        rel_y = cy - (self.y + TableNode.HEADER_H)
        if rel_y < 0:
            return None
        idx = int(rel_y // TableNode.ROW_H)
        if 0 <= idx < len(self.cols):
            return self.cols[idx]
        return None


# ---- Main Application ----
class HRMSQueryGUI(tb.Window if "tb" in globals() else tk.Tk):
    def __init__(self):
        # --- Auto-detect system theme BEFORE parent init ---
        dark_mode = False
        try:
            if sys.platform == "win32":
                import winreg
                key = winreg.OpenKey(
                    winreg.HKEY_CURRENT_USER,
                    r"Software\Microsoft\Windows\CurrentVersion\Themes\Personalize"
                )
                val, _ = winreg.QueryValueEx(key, "AppsUseLightTheme")
                dark_mode = (val == 0)  # 0 = dark, 1 = light
            elif sys.platform == "darwin":
                from subprocess import check_output
                mode = check_output(
                    ["defaults", "read", "-g", "AppleInterfaceStyle"],
                    universal_newlines=True
                ).strip()
                dark_mode = (mode.lower() == "dark")
            else:  # Linux/GNOME etc.
                theme = os.environ.get("GTK_THEME", "").lower()
                dark_mode = "dark" in theme
        except Exception:
            pass

        self.dark_mode = dark_mode  # ✅ store result

        # --- Call parent init with correct theme (default = darkly) ---
        if "tb" not in globals():
            tk.Tk.__init__(self)
            self.title("HRMS SQL Editor — Enhanced")
        else:
            # Always prefer darkly by default; allow override later via selector
            super().__init__(themename="darkly")
            self.dark_mode = True
            self.title("HRMS SQL Editor — Enhanced")

        # 🔽 Fixed toolbar button styles
        self._apply_fixed_button_styles()

        # --- Window setup ---
        # HiDPI-friendly scaling
        try:
            if sys.platform == "win32":
                # modest scale for 125-150% displays
                tk.CallWrapper = tk.CallWrapper
                self.tk.call("tk", "scaling", 1.25)
        except Exception:
            pass

        self.geometry("1300x850")
        self.connections = {}
        self.active_conn_name = None
        self.active_conn = None
        self.schema_cache = {}
        self.editors = []
        self.query_rows = []
        self.current_page = 1
        self.running_thread = None
        self.running_tid = None
        self._history_data = []

        self.left_visible = True
        self.editor_font = ("Consolas", 12)
        self.result_font = ("Arial", 10)

        # boolean var used in UI controls
        self.dark_toggle_var = tk.BooleanVar(value=self.dark_mode)

        # load configs then build UI
        self._load_connections_from_file()
        self._build_ui()

        os.makedirs(DRAFTS_DIR, exist_ok=True)
        self._start_autosave()

        # Create loading overlay (hidden by default) and apply window fade-in
        try:
            self._create_loading_overlay()
        except Exception:
            pass
        try:
            self._fade_in_window()
        except Exception:
            pass
        try:
            self._init_top_bar_indicator()
        except Exception:
            pass
        # Initialize hover detection for menu switching animations
        try:
            self._init_menu_hover_overlay()
        except Exception:
            pass
        try:
            self._start_menu_hover_poll()
        except Exception:
            pass


    def _build_ui(self):
        # menubar
        self._create_menubar()

        # --- Toolbar ---
        toolbar = tb.Frame(self, padding=8)
        toolbar.pack(fill=tk.X)

        # --- Query action buttons ---
        def pill(btn):
            try:
                btn.configure(style=btn.cget("style"))
            except Exception:
                pass
            btn.pack_configure(side=tk.LEFT, padx=5, pady=2)

        # Single Run button
        pill(ttk.Button(toolbar, text="Run", style="Primary.TButton", command=self.run_query))

        pill(ttk.Button(toolbar, text="Cancel", style="Danger.TButton", command=self.cancel_query))
        pill(ttk.Button(toolbar, text="Explain", style="Secondary.TButton", command=self.run_explain))
        pill(ttk.Button(toolbar, text="Visual", style="Secondary.TButton", command=self.open_query_builder_with_schema))
        pill(ttk.Button(toolbar, text="Plot", style="Secondary.TButton", command=self.open_data_viz))
        pill(ttk.Button(toolbar, text="AI → SQL", style="Secondary.TButton", command=self.ask_nl_to_sql))

        tb.Separator(toolbar, orient=tk.VERTICAL).pack(side=tk.LEFT, padx=10, fill=tk.Y)

        # --- Workspace buttons (custom styled) ---
        btn_save = ttk.Button(toolbar, text="Save WS", style="Primary.TButton", command=self.save_workspace)
        btn_save.pack(side=tk.LEFT, padx=3)
        ToolTip(btn_save, "Save workspace (connections, tabs)")
        btn_load = ttk.Button(toolbar, text="Load", style="Secondary.TButton", command=self.load_query)
        btn_load.pack(side=tk.LEFT, padx=3)
        ToolTip(btn_load, "Load SQL from file…")
        btn_new = ttk.Button(toolbar, text="New", style="Primary.TButton", command=self.add_tab)
        btn_new.pack(side=tk.LEFT, padx=3)
        ToolTip(btn_new, "New query tab (Ctrl+T)")
        btn_close = ttk.Button(toolbar, text="Close", style="Danger.TButton", command=self.close_tab)
        btn_close.pack(side=tk.LEFT, padx=3)
        ToolTip(btn_close, "Close current tab (Ctrl+W)")

        # --- Font zoom ---
        tb.Label(toolbar, text="Font Size:").pack(side=tk.LEFT, padx=(16, 6))
        self.font_size_var = tk.IntVar(value=self.editor_font[1])
        tb.Scale(
            toolbar, from_=8, to=28, variable=self.font_size_var,
            orient="horizontal", command=lambda v: self._set_editor_font(int(float(v))),
            length=140, bootstyle="info"
        ).pack(side=tk.LEFT, padx=4)

        # --- Search field ---
        tb.Label(toolbar, text="Search:").pack(side=tk.LEFT, padx=(20, 2))
        self.search_var = tk.StringVar()
        self.search_var.trace_add("write", lambda *_: self.apply_filter())
        search_entry = tb.Entry(toolbar, textvariable=self.search_var, width=28, bootstyle="secondary")
        search_entry.pack(side=tk.LEFT, padx=4)
        ToolTip(search_entry, "Filter history/results")

        # --- Theme selector ---
        themes = ["darkly", "cosmo", "flatly", "cyborg"]
        self.theme_var = tk.StringVar(value="darkly")
        theme_cb = tb.Combobox(
            toolbar, textvariable=self.theme_var,
            values=themes, state="readonly", width=10, bootstyle="dark"
        )
        theme_cb.pack(side=tk.RIGHT, padx=6)
        theme_cb.bind("<<ComboboxSelected>>", lambda e: self._switch_theme(self.theme_var.get()))
        ToolTip(theme_cb, "Switch theme")

        # --- Toggle left panel button ---
        icon = load_icon("toggle.png")
        toggle_btn = tb.Button(
            toolbar,
            image=icon if icon else None,
            text="Toggle Connections" if not icon else "",
            command=self._toggle_left_panel,
            bootstyle="warning-outline"
        )
        if icon:
            toggle_btn.image = icon
        toggle_btn.pack(side=tk.RIGHT, padx=6)
        ToolTip(toggle_btn, "Show/Hide connection panel")

        # --- Main layout (paned window: left, center, right) ---
        self.main_pane = tb.Panedwindow(self, orient=tk.HORIZONTAL)
        self.main_pane.pack(fill=tk.BOTH, expand=True)

        # Left frame (connections + schema)
        self.left_initial_width = 260
        self.left_frame = tb.Frame(self.main_pane, width=self.left_initial_width)
        self.left_frame.pack_propagate(False)
        self._create_left_panel(self.left_frame)
        self.main_pane.add(self.left_frame, weight=0)

        # Center (editor tabs)
        center_frame = tb.Frame(self.main_pane)
        self.tab_control = tb.Notebook(center_frame)
        self.tab_control.enable_traversal()
        self.tab_control.pack(fill=tk.BOTH, expand=True, padx=6, pady=6)
        self.add_tab()
        center_frame.pack(fill=tk.BOTH, expand=True)
        self.main_pane.add(center_frame, weight=3)

        # Notebook tabs polish
        try:
            st = self.style if hasattr(self, "style") else ttk.Style()
            st.configure("TNotebook", tabmargins=[6, 4, 6, 0])
            st.configure("TNotebook.Tab", padding=[14, 6], focusthickness=0)
            st.map("TNotebook.Tab",
                   background=[("selected", "#2b2f33" if self.dark_mode else "#e0e0e0")],
                   foreground=[("selected", "#ffffff" if self.dark_mode else "#000000")])
        except Exception:
            pass

        # Right (results, history, messages)
        right_outer = tb.Frame(self.main_pane)
        self.result_tabs = tb.Notebook(right_outer)

        results_frame = tb.Frame(self.result_tabs)
        self._create_results_panel(results_frame)
        self.result_tabs.add(results_frame, text="Results")

        history_frame = tb.Frame(self.result_tabs)
        self._create_history_panel(history_frame)
        self.result_tabs.add(history_frame, text="History")

        messages_frame = tb.Frame(self.result_tabs)
        self.messages_box = tk.Text(messages_frame, wrap="word", height=8, bg="#f9f9f9", fg="black")
        self.messages_box.pack(fill=tk.BOTH, expand=True, padx=4, pady=4)
        self.result_tabs.add(messages_frame, text="Messages")

        self.result_tabs.pack(fill=tk.BOTH, expand=True)
        right_outer.pack(fill=tk.BOTH, expand=True)
        self.main_pane.add(right_outer, weight=2)

        # --- Status bar ---
        self.status_var = tk.StringVar(value="Ready ✅")
        self.db_status_var = tk.StringVar(value="(DB: none)")

        status_frame = tb.Frame(self)
        status_frame.pack(fill=tk.X, side=tk.BOTTOM)

        self.status_label = tb.Label(
            status_frame,
            textvariable=self.status_var,
            relief=tk.SUNKEN,
            anchor="w",
            padding=4,
            bootstyle="secondary"
        )
        self.status_label.pack(side=tk.LEFT, fill=tk.X, expand=True)

        # Query status indicator (icon-like label)
        self.query_status_var = tk.StringVar(value="● Idle")
        self.query_status_label = tb.Label(
            status_frame,
            textvariable=self.query_status_var,
            relief=tk.SUNKEN,
            anchor="center",
            padding=4,
            bootstyle="secondary"
        )
        self.query_status_label.pack(side=tk.RIGHT)

        self.db_status_label = tb.Label(
            status_frame,
            textvariable=self.db_status_var,
            relief=tk.SUNKEN,
            anchor="e",
            padding=4,
            bootstyle="dark"
        )
        self.db_status_label.pack(side=tk.RIGHT)

        # Progress bar
        self.progress = tb.Progressbar(self, mode="indeterminate", bootstyle="info-striped")
        self.progress.pack(fill=tk.X, side=tk.BOTTOM)
        self.progress.stop()

        # Animate tab switch: subtle editor background pulse on tab change
        try:
            self.tab_control.bind("<<NotebookTabChanged>>", self._on_tab_changed)
        except Exception:
            pass

        # --- Keyboard shortcuts ---
        self.bind_all("<F5>", lambda e: self.run_query())
        self.bind_all("<Control-s>", lambda e: self.save_query())
        self.bind_all("<Control-o>", lambda e: self.load_query())
        self.bind_all("<Control-t>", lambda e: self.add_tab())
        self.bind_all("<Control-w>", lambda e: self.close_tab())

    def _apply_fixed_button_styles(self):
        try:
            style = self.style if hasattr(self, "style") else ttk.Style()
            # Darkly-like fixed palette for toolbar buttons
            style.configure("Primary.TButton", foreground="white", background="#4CAF50", bordercolor="#3d8f40", focusthickness=1, focuscolor="#66bb6a")
            style.map("Primary.TButton", background=[("active", "#45A049"), ("pressed", "#3d8f40")])

            style.configure("Secondary.TButton", foreground="white", background="#2196F3", bordercolor="#1565c0", focusthickness=1, focuscolor="#64b5f6")
            style.map("Secondary.TButton", background=[("active", "#1976D2"), ("pressed", "#1565c0")])

            style.configure("Danger.TButton", foreground="white", background="#F44336", bordercolor="#b71c1c", focusthickness=1, focuscolor="#ef9a9a")
            style.map("Danger.TButton", background=[("active", "#D32F2F"), ("pressed", "#b71c1c")])

            style.configure("Neutral.TButton", foreground="white", background="#607D8B", bordercolor="#37474f", focusthickness=1, focuscolor="#90a4ae")
            style.map("Neutral.TButton", background=[("active", "#455A64"), ("pressed", "#37474f")])
        except Exception:
            pass

    # --- Menubar ---
    def _create_menubar(self):
        menubar = tk.Menu(self)
        self.config(menu=menubar)
        file_menu = tk.Menu(menubar, tearoff=0)
        try:
            file_menu.configure(postcommand=lambda: self._animate_top_bar("file"))
        except Exception:
            pass
        file_menu.add_command(label="New Tab", command=self.add_tab, accelerator="Ctrl+T")
        file_menu.add_command(label="Open SQL...", command=self.load_query, accelerator="Ctrl+O")
        file_menu.add_command(label="Save Query...", command=self.save_query, accelerator="Ctrl+S")
        file_menu.add_command(label="Save Query As...", command=self.save_query_as)
        file_menu.add_separator()
        file_menu.add_command(label="Manage Connections...", command=self.open_connection_manager)
        file_menu.add_command(label="Save Workspace", command=self.save_workspace)
        file_menu.add_command(label="Restore Workspace", command=self.restore_workspace)
        file_menu.add_separator()
        file_menu.add_command(label="Exit", command=self.on_closing)
        menubar.add_cascade(label="File", menu=file_menu)

        tools_menu = tk.Menu(menubar, tearoff=0)
        try:
            tools_menu.configure(postcommand=lambda: self._animate_top_bar("tools"))
        except Exception:
            pass
        tools_menu.add_command(label="Visual Query Builder", command=self.open_query_builder_with_schema)
        tools_menu.add_command(label="Data Visualization", command=self.open_data_viz)
        tools_menu.add_command(label="Server Admin Tools", command=self.open_server_admin)
        menubar.add_cascade(label="Tools", menu=tools_menu)

        view_menu = tk.Menu(menubar, tearoff=0)
        try:
            view_menu.configure(postcommand=lambda: self._animate_top_bar("view"))
        except Exception:
            pass
        view_menu.add_command(label="Toggle Dark Mode", command=self.toggle_dark_mode)
        menubar.add_cascade(label="View", menu=view_menu)

        # Command Palette (Ctrl+Shift+P)
        def open_palette():
            win = tk.Toplevel(self)
            win.title("Command Palette")
            win.geometry("420x360")
            win.transient(self)
            win.grab_set()
            q = tk.StringVar()
            tk.Entry(win, textvariable=q).pack(fill=tk.X, padx=8, pady=8)
            lb = tk.Listbox(win)
            lb.pack(fill=tk.BOTH, expand=True, padx=8, pady=8)
            commands = [
                ("Run", self.run_query),
                ("Explain", self.run_explain),
                ("New Tab", self.add_tab),
                ("Save Workspace", self.save_workspace),
                ("Restore Workspace", self.restore_workspace),
                ("Toggle Left Panel", self._toggle_left_panel),
                ("Toggle Dark Mode", self.toggle_dark_mode),
            ]
            def refresh():
                s = q.get().lower().strip()
                lb.delete(0, tk.END)
                for title, fn in commands:
                    if s in title.lower():
                        lb.insert(tk.END, title)
            def run_selected(evt=None):
                sel = lb.curselection()
                if not sel:
                    return
                title = lb.get(sel[0])
                for t, fn in commands:
                    if t == title:
                        try:
                            fn()
                        except Exception:
                            pass
                        break
                try:
                    win.destroy()
                except Exception:
                    pass
            q.trace_add("write", lambda *_: refresh())
            lb.bind("<Return>", run_selected)
            lb.bind("<Double-1>", run_selected)
            refresh()
        self.bind_all("<Control-Shift-P>", lambda e: open_palette())

    # --- Tab changed animation ---
    def _on_tab_changed(self, event=None):
        try:
            idx = self.tab_control.index(self.tab_control.select())
            editor = self.editors[idx]
            original_bg = editor.text.cget("bg")
            pulse1 = "#eef9ff" if not self.dark_mode else "#26333b"
            pulse2 = "#e6f7ff" if not self.dark_mode else "#2a3b47"

            def seq(step=0):
                if step == 0:
                    editor.text.configure(bg=pulse1)
                    self.after(70, lambda: seq(1))
                elif step == 1:
                    editor.text.configure(bg=pulse2)
                    self.after(70, lambda: seq(2))
                else:
                    editor.text.configure(bg=original_bg)

            seq(0)
        except Exception:
            pass

    # --- Top bar indicator (menu activity) ---
    def _init_top_bar_indicator(self):
        try:
            bar = tk.Frame(self, height=2, bg="#2196F3")
            # Use place to control width for animation; start hidden
            bar.place(x=0, y=0, width=0, height=2)
            self._top_bar = bar
        except Exception:
            self._top_bar = None

    def _animate_top_bar(self, which="file", duration_ms=220, steps=12):
        try:
            if not getattr(self, "_top_bar", None):
                return
            colors = {
                "file": "#42a5f5",
                "tools": "#ab47bc",
                "view": "#ffb74d",
            }
            color = colors.get(which, "#42a5f5")
            self._top_bar.configure(bg=color)
            total_w = max(1, self.winfo_width())
            def step(i=0):
                w = int(total_w * (i / float(steps)))
                try:
                    self._top_bar.place(x=0, y=0, width=w, height=2)
                except Exception:
                    pass
                if i < steps:
                    self.after(max(1, duration_ms // steps), lambda: step(i+1))
                else:
                    # shrink back quickly
                    self.after(120, lambda: self._top_bar.place(x=0, y=0, width=0, height=2))
            step(0)
        except Exception:
            pass

    # --- Window fade-in ---
    def _fade_in_window(self, duration_ms=240, steps=8):
        try:
            self.attributes("-alpha", 0.0)
        except Exception:
            return
        step = 1
        def tick():
            nonlocal step
            try:
                self.attributes("-alpha", min(1.0, step / float(steps)))
            except Exception:
                return
            step += 1
            if step <= steps:
                self.after(max(1, duration_ms // max(1, steps)), tick)
        tick()

    # --- Loading overlay ---
    def _create_loading_overlay(self):
        try:
            overlay = tk.Frame(self, bg="#000000", cursor="watch")
            overlay.place(relx=0, rely=0, relwidth=1, relheight=1)
            overlay.lower()  # hide under widgets initially
            overlay.place_forget()
            inner = tk.Frame(overlay, bg="#000000")
            inner.place(relx=0.5, rely=0.5, anchor="center")
            lbl = tb.Label(inner, text="Working…", padding=8, bootstyle="light")
            lbl.pack(side=tk.TOP, fill=tk.X)
            pb = tb.Progressbar(inner, mode="indeterminate", bootstyle="info-striped")
            pb.pack(fill=tk.X)
            self._overlay_frame = overlay
            self._overlay_progress = pb
        except Exception:
            self._overlay_frame = None
            self._overlay_progress = None

    def _show_loading_overlay(self, message="Working…"):
        try:
            if not getattr(self, "_overlay_frame", None):
                return
            # Update message if label exists
            try:
                for child in self._overlay_frame.winfo_children():
                    for sub in child.winfo_children():
                        if isinstance(sub, ttk.Label) or isinstance(sub, tb.Label if 'tb' in globals() else ttk.Label):
                            sub.configure(text=message)
            except Exception:
                pass
            self._overlay_frame.place(relx=0, rely=0, relwidth=1, relheight=1)
            self._overlay_frame.lift()
            if self._overlay_progress:
                try:
                    self._overlay_progress.start(12)
                except Exception:
                    pass
        except Exception:
            pass

    def _hide_loading_overlay(self):
        try:
            if getattr(self, "_overlay_progress", None):
                try:
                    self._overlay_progress.stop()
                except Exception:
                    pass
            if getattr(self, "_overlay_frame", None):
                self._overlay_frame.place_forget()
        except Exception:
            pass

    # --- Results refresh flash ---
    def _animate_results_flash(self, cycles=2, period_ms=90):
        try:
            st = self.style if hasattr(self, "style") else ttk.Style()
            base_bg = st.lookup("Treeview", "fieldbackground") or ("#263238" if self.dark_mode else "#ffffff")
            flash_bg1 = "#2e3d44" if self.dark_mode else "#e8f5e9"
            flash_bg2 = "#2b3940" if self.dark_mode else "#e3f2fd"
            seq = [flash_bg1, flash_bg2] * cycles + [base_bg]
            def step(i=0):
                if i >= len(seq):
                    return
                try:
                    st.configure("Treeview", fieldbackground=seq[i])
                except Exception:
                    pass
                self.after(period_ms, lambda: step(i+1))
            step(0)
        except Exception:
            pass

    # --- Editor font zoom helper ---
    def _set_editor_font(self, size):
        self.editor_font = ("Consolas", int(size))
        for ed in self.editors:
            try:
                ed.text.config(font=self.editor_font)
                ed.linenumber.config(font=self.editor_font)
            except Exception:
                pass
    # --- Flash status temporarily ---        
    def flash_status(self, text, duration=2000, steps=10):
        """Show a temporary status message with a fade-out effect."""
        self.status_var.set(text)

        def fade(step=0):
            if step >= steps:
                self.status_var.set("Ready ✅")
                return
            # Compute fade color (from bright green to gray)
            ratio = 1 - (step / steps)
            r = int(0 * ratio + 100 * (1 - ratio))   # start at 0, fade to 100
            g = int(180 * ratio + 100 * (1 - ratio)) # bright green to gray
            b = int(0 * ratio + 100 * (1 - ratio))
            color = f"#{r:02x}{g:02x}{b:02x}"

            # Apply to status label
            try:
                self.status_label.config(foreground=color)
            except Exception:
                pass

            self.after(duration // steps, lambda: fade(step + 1))

        fade()


    # --- Toggle left panel ---
    def _toggle_left_panel(self):
        if self.left_visible:
            self._animate_left_panel_hide()
        else:
            self._animate_left_panel_show()

    def _animate_left_panel_show(self, duration_ms=160, steps=8):
        try:
            panes = self.main_pane.panes()
            if str(self.left_frame) not in panes:
                self.left_frame.configure(width=0)
                self.main_pane.add(self.left_frame, weight=0)

            target = getattr(self, "left_initial_width", 260)
            step_w = max(1, target // steps)

            def step(curr=0):
                new_w = min(target, curr + step_w)
                try:
                    self.main_pane.paneconfigure(self.left_frame, width=new_w)
                except Exception:
                    pass
                if new_w < target:
                    self.after(max(1, duration_ms // steps), lambda: step(new_w))
                else:
                    self.left_visible = True

            step(0)
        except Exception:
            try:
                self.main_pane.add(self.left_frame, weight=0)
            except Exception:
                pass
            self.left_visible = True

    def _animate_left_panel_hide(self, duration_ms=160, steps=8):
        try:
            try:
                info = self.main_pane.pane(self.left_frame)
                curr_w = int(info.get("width", self.left_initial_width))
            except Exception:
                curr_w = self.left_initial_width

            step_w = max(1, curr_w // steps)

            def step(curr=curr_w):
                new_w = max(0, curr - step_w)
                try:
                    self.main_pane.paneconfigure(self.left_frame, width=new_w)
                except Exception:
                    pass
                if new_w > 0:
                    self.after(max(1, duration_ms // steps), lambda: step(new_w))
                else:
                    try:
                        self.main_pane.forget(self.left_frame)
                    except Exception:
                        pass
                    self.left_visible = False

            step(curr_w)
        except Exception:
            try:
                self.main_pane.forget(self.left_frame)
            except Exception:
                pass
            self.left_visible = False

    # --- Theme switcher for ttkbootstrap ---
    def _switch_theme(self, new_theme):
        try:
            if tb is not None:
                tb.Style(new_theme)
                self.theme_var.set(new_theme)
                # keep icons fixed (already file-based), re-style widgets
                self.dark_mode = (new_theme.lower() in ("darkly", "cyborg"))
                # update editor backgrounds, treeviews, etc.
                for ed in self.editors:
                    ed.dark = self.dark_mode
                    try:
                        ed.text.config(bg="#1e1e1e" if self.dark_mode else "white",
                                       fg="#dcdcdc" if self.dark_mode else "black",
                                       insertbackground="#dcdcdc" if self.dark_mode else "black")
                    except Exception:
                        pass
                    ed.highlight_syntax()
                self._apply_treeview_style()
                self.update_result_grid()
                self.status_var.set(f"Theme switched to {new_theme}")
                # Re-apply fixed button colors so they remain darkly-like
                self._apply_fixed_button_styles()
        except Exception as e:
            messagebox.showerror("Theme Error", str(e))

    def toggle_dark_mode(self):
        self.dark_mode = not self.dark_mode
        self.dark_toggle_var.set(self.dark_mode)
        for ed in self.editors:
            ed.dark = self.dark_mode
            try:
                ed.text.config(bg="#1e1e1e" if self.dark_mode else "white",
                               fg="#dcdcdc" if self.dark_mode else "black",
                               insertbackground="#dcdcdc" if self.dark_mode else "black")
            except Exception:
                pass
            ed.highlight_syntax()
        self._apply_treeview_style()
        self.update_result_grid()
        bg = "#2d2d2d" if self.dark_mode else "SystemButtonFace"
        try:
            self.config(bg=bg)
            self.status_var.set("Dark Mode ON" if self.dark_mode else "Dark Mode OFF")
        except Exception:
            pass

    # --- Left panel creation (connections + schema) ---
    def _create_left_panel(self, parent):
        conn_frame = ttk.LabelFrame(parent, text="Connections", padding=6)
        conn_frame.pack(fill=tk.X, padx=6, pady=6)
        ttk.Label(conn_frame, text="Active:").grid(row=0, column=0, sticky="w")
        self.conn_label = ttk.Label(conn_frame, text="(none)")
        self.conn_label.grid(row=0, column=1, sticky="w", padx=(4, 10))
        ttk.Button(conn_frame, text="Manage", command=self.open_connection_manager).grid(row=0, column=2, sticky="e")
        # Button row (Connect / Disconnect side by side)
        btn_frame = ttk.Frame(conn_frame)
        btn_frame.grid(row=1, column=0, columnspan=3, pady=6, sticky="we")

        ttk.Button(btn_frame, text="Connect", command=self.connect_selected).pack(side=tk.LEFT, expand=True, fill=tk.X, padx=3)
        ttk.Button(btn_frame, text="Disconnect", command=self.disconnect_active).pack(side=tk.LEFT, expand=True, fill=tk.X, padx=3)

        # keep column stretch for other widgets
        conn_frame.columnconfigure(0, weight=1)
        conn_frame.columnconfigure(1, weight=1)
        conn_frame.columnconfigure(2, weight=0)


        schema_frame = ttk.LabelFrame(parent, text="Schema Browser", padding=6)
        schema_frame.pack(fill=tk.BOTH, expand=True, padx=6, pady=(0, 6))
        # quick search
        search_box = ttk.Entry(schema_frame)
        search_box.pack(fill=tk.X, padx=2, pady=(0,4))
        self.schema_tree = ttk.Treeview(schema_frame, style="Schema.Treeview")
        self.schema_tree.pack(fill=tk.BOTH, expand=True)
        self.schema_tree.bind("<Double-1>", self._on_schema_double)
        # right-click on schema -> show options (e.g., copy name or open query)
        def schema_right(event):
            sel = self.schema_tree.identify_row(event.y)
            if not sel:
                return
            menu = tk.Menu(self, tearoff=0)
            item_text = self.schema_tree.item(sel, "text")
            parent = self.schema_tree.parent(sel)
            if parent == "":
                # table item
                menu.add_command(label="Copy Table Name", command=lambda: self._clipboard_copy(item_text))
                menu.add_command(label="Select * LIMIT 100", command=lambda: self.add_tab(initial_text=f"SELECT * FROM `{item_text}` LIMIT 100;"))
            else:
                # column item
                col = item_text
                tbl = self.schema_tree.item(parent, "text")
                menu.add_command(label="Copy Column Name", command=lambda: self._clipboard_copy(col))
                menu.add_command(label="Select Column (LIMIT 100)", command=lambda: self.add_tab(initial_text=f"SELECT `{col}` FROM `{tbl}` LIMIT 100;"))
            try:
                menu.tk_popup(event.x_root, event.y_root)
            finally:
                try:
                    menu.grab_release()
                except Exception:
                    pass
        self.schema_tree.bind("<Button-3>", schema_right, add="+")

        # filter handler
        def on_filter(*_):
            q = search_box.get().strip().lower()
            # simple filter: show only matching tables/columns by text
            try:
                for item in self.schema_tree.get_children(""):
                    tbl = self.schema_tree.item(item, "text").lower()
                    visible = (q in tbl) or (q == "")
                    # show/hide table
                    try:
                        self.schema_tree.detach(item) if not visible else self.schema_tree.reattach(item, "", tk.END)
                    except Exception:
                        pass
            except Exception:
                pass
        search_box.bind("<KeyRelease>", lambda e: on_filter())

    def _clipboard_copy(self, txt):
        try:
            self.clipboard_clear()
            self.clipboard_append(txt)
        except Exception:
            pass

    # --- center panel add tab ---
    def add_tab(self, initial_text=""):
        frame = ttk.Frame(self.tab_control)
        editor = SQLEditor(frame, font=self.editor_font, dark=self.dark_mode)
        editor.pack(fill=tk.BOTH, expand=True)

        # enable autocomplete using known schema table names
        try:
            # Subtle background flash to draw attention to the new tab
            original_bg = editor.text.cget("bg")
            flash_bg = "#e6f7ff" if not self.dark_mode else "#2a3b47"

            def animate_flash(step=0, steps=6):
                # toggle between flash and original a few times
                if step >= steps:
                    try:
                        editor.text.configure(bg=original_bg)
                    except Exception:
                        pass
                    return
                try:
                    editor.text.configure(bg=flash_bg if step % 2 == 0 else original_bg)
                except Exception:
                    pass
                self.after(60, lambda: animate_flash(step + 1, steps))

            animate_flash()
        except Exception:
            pass

        # add the tab and select it (single, correct numbering)
        idx = len(self.editors) + 1
        self.tab_control.add(frame, text=f"Query {idx}")
        self.tab_control.select(frame)
        self.editors.append(editor)
        # set initial text if provided
        if initial_text:
            try:
                editor.set(initial_text)
            except Exception:
                try:
                    editor.insert("1.0", initial_text)
                except Exception:
                    pass
        # wire autocomplete with current schema tables
        try:
            editor._setup_autocomplete(list(self.schema_cache.keys()))
        except Exception:
            try:
                editor._setup_autocomplete([])
            except Exception:
                pass

    def close_tab(self):
        if len(self.editors) <= 1:
            if len(self.editors) == 1:
                if messagebox.askyesno("Close Tab", "This is the last tab. Clear its contents instead?"):
                    ed = self.editors[0]
                    ed.delete("1.0", tk.END)
                    ed.modified = False
            return
        idx = self.tab_control.index(self.tab_control.select())
        editor = self.editors[idx]
        if editor.modified:
            if not messagebox.askyesno("Unsaved", "Close despite unsaved changes?"):
                return
        self.tab_control.forget(idx)
        try:
            self.editors.pop(idx)
        except Exception:
            pass

    # --- Results panel build ---
    def _create_results_panel(self, parent):
        # --- Results Treeview ---
        self.result_tree = ttk.Treeview(parent, show="headings", selectmode="extended")  
        # ^ extended allows multi-select (Ctrl/Shift). Use "browse" if you want only one row at a time.
        self.result_tree.pack(fill=tk.BOTH, expand=True, side=tk.LEFT)

        # --- Scrollbars ---
        vscroll = ttk.Scrollbar(parent, orient=tk.VERTICAL, command=self.result_tree.yview)
        vscroll.pack(side=tk.RIGHT, fill=tk.Y)
        self.result_tree.configure(yscrollcommand=vscroll.set)

        hscroll = ttk.Scrollbar(parent, orient=tk.HORIZONTAL, command=self.result_tree.xview)
        hscroll.pack(side=tk.BOTTOM, fill=tk.X)
        self.result_tree.configure(xscrollcommand=hscroll.set)

        # --- Styling ---
        style = ttk.Style()

        if self.dark_mode:  # ✅ Dark theme styling
            style.configure(
                "Treeview",
                rowheight=25,
                font=("Segoe UI", 10),
                background="#1e1e1e",
                fieldbackground="#1e1e1e",
                foreground="white"
            )
            style.map(
                "Treeview",
                background=[
                    ("selected", "#264F78"),   # highlight only when clicked
                    ("active", "#1e1e1e")      # hover = normal background
                ],
                foreground=[
                    ("selected", "white"),
                    ("active", "white")        # keep text visible on hover
                ]
            )
            # Alternate row colors
            self.result_tree.tag_configure("oddrow", background="#252526")
            self.result_tree.tag_configure("evenrow", background="#1e1e1e")

        else:  # ✅ Light theme styling
            style.configure(
                "Treeview",
                rowheight=25,
                font=("Segoe UI", 10),
                background="white",
                fieldbackground="white",
                foreground="black"
            )
            style.map(
                "Treeview",
                background=[
                    ("selected", "#0078D7"),   # Windows blue highlight
                    ("active", "white")        # hover = no change
                ],
                foreground=[
                    ("selected", "white"),
                    ("active", "black")
                ]
            )
            # Alternate row colors
            self.result_tree.tag_configure("oddrow", background="#f9f9f9")
            self.result_tree.tag_configure("evenrow", background="#ffffff")

        # ✅ Auto-resize columns on window resize
        def _resize_columns(event):
            if not self.result_tree["columns"]:
                return
            total_cols = len(self.result_tree["columns"])
            col_width = int(self.result_tree.winfo_width() / max(total_cols, 1))
            for c in self.result_tree["columns"]:
                self.result_tree.column(c, width=col_width)

        self.result_tree.bind("<Configure>", _resize_columns)

        # column reorder support by dragging header
        def enable_column_reorder(tree):
            drag = {"col": None}
            def on_press(event):
                col_id = tree.identify_column(event.x)
                if col_id:
                    drag["col"] = col_id
            def on_release(event):
                if drag["col"]:
                    target_col = tree.identify_column(event.x)
                    if target_col and drag["col"] != target_col:
                        cols = list(tree["columns"])
                        i1 = int(drag["col"].replace("#", "")) - 1
                        i2 = int(target_col.replace("#", "")) - 1
                        if 0 <= i1 < len(cols) and 0 <= i2 < len(cols):
                            cols[i1], cols[i2] = cols[i2], cols[i1]
                            tree["columns"] = cols
                            for c in cols:
                                tree.heading(c, text=c, command=lambda _c=c: self.sort_column(_c, False))
                    drag["col"] = None
            tree.bind("<ButtonPress-1>", on_press, add="+")
            tree.bind("<ButtonRelease-1>", on_release, add="+")
        enable_column_reorder(self.result_tree)

        # header right-click -> filter on column
        def on_header_right_click(event):
            region = self.result_tree.identify_region(event.x, event.y)
            if region != "heading":
                return
            col_id = self.result_tree.identify_column(event.x)
            if not col_id:
                return
            idx = int(col_id.replace("#", "")) - 1
            cols = list(self.result_tree["columns"])
            if idx < 0 or idx >= len(cols):
                return
            col = cols[idx]
            fw = tk.Toplevel(self)
            fw.title(f"Filter {col}")
            fw.geometry("260x100")
            fw.transient(self)
            tk.Label(fw, text=f"Filter {col}:").pack(pady=4)
            entry = ttk.Entry(fw)
            entry.pack(fill=tk.X, padx=6)
            def apply():
                val = entry.get().strip().lower()
                for rid in self.result_tree.get_children():
                    try:
                        self.result_tree.reattach(rid, "", "end")
                    except Exception:
                        pass
                if val:
                    for rid in list(self.result_tree.get_children()):
                        cell = str(self.result_tree.set(rid, col)).lower()
                        if val not in cell:
                            try:
                                self.result_tree.detach(rid)
                            except Exception:
                                pass
                fw.destroy()
            ttk.Button(fw, text="Apply", command=apply).pack(pady=8)
        self.result_tree.bind("<Button-3>", on_header_right_click, add="+")

        # right click on rows -> context menu for copies/exports
        def copy_column_name():
            x = self.result_tree.winfo_pointerx() - self.result_tree.winfo_rootx()
            col_id = self.result_tree.identify_column(x)
            if col_id:
                try:
                    idx = int(col_id.replace("#", "")) - 1
                except Exception:
                    idx = -1
                cols = list(self.result_tree["columns"])
                if 0 <= idx < len(cols):
                    try:
                        self.clipboard_clear()
                        self.clipboard_append(cols[idx])
                    except Exception:
                        pass

        def result_right_click(event):
            # show context menu for clicked row
            item = self.result_tree.identify_row(event.y)
            menu = tk.Menu(self, tearoff=0)
            menu.add_command(label="Copy Cell", command=self.copy_cell)
            menu.add_command(label="Copy Row", command=self.copy_row)
            menu.add_command(label="Copy Column Name", command=copy_column_name)
            # Copy As submenu
            copy_as = tk.Menu(menu, tearoff=0)
            copy_as.add_command(label="CSV", command=lambda: self.copy_selection_as(format="csv"))
            copy_as.add_command(label="TSV", command=lambda: self.copy_selection_as(format="tsv"))
            copy_as.add_command(label="Markdown", command=lambda: self.copy_selection_as(format="md"))
            copy_as.add_command(label="INSERT VALUES", command=lambda: self.copy_selection_as(format="insert"))
            menu.add_cascade(label="Copy As", menu=copy_as)
            menu.add_separator()
            menu.add_command(label="Export CSV", command=self.export_csv)
            if _HAS_OPENPYXL:
                menu.add_command(label="Export Excel", command=self.export_excel)
            else:
                menu.add_command(label="Export Excel (missing openpyxl)", state="disabled")
            try:
                menu.tk_popup(event.x_root, event.y_root)
            finally:
                try:
                    menu.grab_release()
                except Exception:
                    pass
        # We bind to both header and rows; header handler handled earlier
        self.result_tree.bind("<Button-3>", result_right_click, add="+")

        self.result_tree.bind("<Double-1>", self._on_result_double)

        # Auto-fit column width on header double-click
        def on_header_double_click(event):
            region = self.result_tree.identify_region(event.x, event.y)
            if region != "heading":
                return
            col_id = self.result_tree.identify_column(event.x)
            if not col_id:
                return
            col_index = int(col_id.replace("#", "")) - 1
            cols = list(self.result_tree["columns"])
            if col_index < 0 or col_index >= len(cols):
                return
            col = cols[col_index]
            # measure header text
            header = self.result_tree.heading(col, option="text")
            max_w = tkfont.Font().measure(str(header)) + 24
            # measure cell contents in current rows
            for iid in self.result_tree.get_children(""):
                val = self.result_tree.set(iid, col)
                max_w = max(max_w, tkfont.Font().measure(str(val)) + 24)
            self.result_tree.column(col, width=min(max(max_w, 60), 640))

        self.result_tree.bind("<Double-Button-1>", on_header_double_click, add="+")

        # Remove results pulse to avoid flashing other Treeviews
        self._results_pulse = None

    def _apply_treeview_style(self):
        style = ttk.Style()
        # Scoped style for results and schema separately
        results_style = "Results.Treeview"
        schema_style = "Schema.Treeview"
        if self.dark_mode:
            style.configure(
                results_style,
                background="#1e1e1e",
                fieldbackground="#1e1e1e",
                foreground="#f0f0f0",
                rowheight=25
            )
            style.configure(
                schema_style,
                background="#171717",
                fieldbackground="#171717",
                foreground="#e0e0e0",
                rowheight=22
            )
            style.map(
                results_style,
                background=[("selected", "#264F78"), ("active", "#1e1e1e")],
                foreground=[("selected", "#ffffff"), ("active", "#f0f0f0")]
            )
            style.map(
                schema_style,
                background=[("selected", "#2d2d2d"), ("active", "#171717")],
                foreground=[("selected", "#ffffff"), ("active", "#e0e0e0")]
            )
            self.result_tree.tag_configure("oddrow", background="#252526", foreground="#f0f0f0")
            self.result_tree.tag_configure("evenrow", background="#1e1e1e", foreground="#f0f0f0")
        else:
            style.configure(
                results_style,
                background="white",
                fieldbackground="white",
                foreground="black",
                rowheight=25
            )
            style.configure(
                schema_style,
                background="#fafafa",
                fieldbackground="#fafafa",
                foreground="#222",
                rowheight=22
            )
            style.map(
                results_style,
                background=[("selected", "#0078D7"), ("active", "white")],
                foreground=[("selected", "#ffffff"), ("active", "black")]
            )
            style.map(
                schema_style,
                background=[("selected", "#e3f2fd"), ("active", "#fafafa")],
                foreground=[("selected", "#000000"), ("active", "#222")]
            )
            self.result_tree.tag_configure("oddrow", background="#f9f9f9", foreground="black")
            self.result_tree.tag_configure("evenrow", background="#ffffff", foreground="black")

        try:
            self.result_tree.configure(style=results_style)
        except Exception:
            pass

    # --- History panel ---
    def _create_history_panel(self, parent):
        topf = ttk.Frame(parent)
        topf.pack(fill=tk.X, padx=6, pady=6)
        ttk.Label(topf, text="Action History").pack(side=tk.LEFT)
        ttk.Button(topf, text="Refresh", command=self.refresh_history).pack(side=tk.RIGHT, padx=4)
        ttk.Button(topf, text="Clear", command=self.clear_history).pack(side=tk.RIGHT, padx=4)
        self.history_list = tk.Listbox(parent)
        self.history_list.pack(fill=tk.BOTH, expand=True, padx=6, pady=(0, 6))
        self.history_list.bind("<Double-Button-1>", self._history_rerun)
        self.history_list.bind("<Button-3>", self._on_history_right)

    # --- Connection manager / save/load config ---
    def open_connection_manager(self):
        win = tk.Toplevel(self)
        win.title("Connection Manager")
        win.geometry("640x420")
        win.transient(self)
        win.grab_set()

        left = ttk.Frame(win, width=220)
        left.pack(side=tk.LEFT, fill=tk.Y, padx=6, pady=6)
        lb = tk.Listbox(left)
        lb.pack(fill=tk.BOTH, expand=True)
        for nm in self.connections.keys():
            lb.insert(tk.END, nm)

        right = ttk.Frame(win)
        right.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=6, pady=6)

        ttk.Label(right, text="Name:").grid(row=0, column=0, sticky="w")
        name_var = tk.StringVar()
        ttk.Entry(right, textvariable=name_var).grid(row=0, column=1, sticky="we")
        ttk.Label(right, text="Host:").grid(row=1, column=0, sticky="w")
        host_var = tk.StringVar(value="localhost")
        ttk.Entry(right, textvariable=host_var).grid(row=1, column=1, sticky="we")
        ttk.Label(right, text="Port:").grid(row=2, column=0, sticky="w")
        port_var = tk.IntVar(value=3306)
        ttk.Entry(right, textvariable=port_var).grid(row=2, column=1, sticky="we")
        ttk.Label(right, text="User:").grid(row=3, column=0, sticky="w")
        user_var = tk.StringVar(value="root")
        ttk.Entry(right, textvariable=user_var).grid(row=3, column=1, sticky="we")
        ttk.Label(right, text="Password:").grid(row=4, column=0, sticky="w")
        pass_var = tk.StringVar()
        ttk.Entry(right, textvariable=pass_var, show="*").grid(row=4, column=1, sticky="we")
        save_enc_var = tk.BooleanVar(value=False)
        ttk.Checkbutton(right, text="Encrypt & Save Password", variable=save_enc_var).grid(row=5, column=0, columnspan=2, sticky="w")

        def refresh_listbox():
            lb.delete(0, tk.END)
            for nm in self.connections.keys():
                lb.insert(tk.END, nm)

        def on_select(evt):
            sel = lb.curselection()
            if not sel:
                return
            nm = lb.get(sel[0])
            cfg = self.connections.get(nm, {})
            if not isinstance(cfg, dict):
                cfg = {}
            name_var.set(nm)
            host_var.set(cfg.get("host", "localhost"))
            port_var.set(cfg.get("port", 3306))
            user_var.set(cfg.get("user", "root"))
            pw = cfg.get("password", "")
            if pw and _HAS_CRYPTO:
                try:
                    pw = decrypt_password(pw)
                except Exception:
                    pass
            pass_var.set(pw)

        lb.bind("<<ListboxSelect>>", on_select)

        def save_conn():
            nm = name_var.get().strip()
            if not nm:
                messagebox.showwarning("Name required", "Please enter a connection name.")
                return
            pw = pass_var.get().strip()
            if save_enc_var.get() and _HAS_CRYPTO and pw:
                pw_val = encrypt_password(pw)
            else:
                pw_val = pw if save_enc_var.get() else ""
            self.connections[nm] = {"host": host_var.get().strip(), "port": int(port_var.get()), "user": user_var.get().strip(), "password": pw_val}
            self._save_connections_to_file()
            refresh_listbox()
            messagebox.showinfo("Saved", f"Connection '{nm}' saved.")

        def delete_conn():
            sel = lb.curselection()
            if not sel:
                return
            nm = lb.get(sel[0])
            if messagebox.askyesno("Confirm", f"Delete '{nm}'?"):
                self.connections.pop(nm, None)
                self._save_connections_to_file()
                refresh_listbox()

        def select_active():
            nm = name_var.get().strip()
            if not nm or nm not in self.connections:
                messagebox.showwarning("Select", "Choose a saved connection.")
                return
            self.active_conn_name = nm
            self.conn_label.config(text=nm)
            messagebox.showinfo("Selected", f"Connection '{nm}' selected. Click Connect to open it.")

        btnf = ttk.Frame(right)
        btnf.grid(row=7, column=0, columnspan=2, pady=8)
        ttk.Button(btnf, text="Save", style="Primary.TButton", command=save_conn).pack(side=tk.LEFT, padx=6)
        ttk.Button(btnf, text="Delete", style="Danger.TButton", command=delete_conn).pack(side=tk.LEFT, padx=6)
        ttk.Button(btnf, text="Select Active", style="Secondary.TButton", command=select_active).pack(side=tk.LEFT, padx=6)
        ttk.Button(btnf, text="Close", style="Neutral.TButton", command=win.destroy).pack(side=tk.LEFT, padx=6)
        win.wait_window(win)

    def _save_connections_to_file(self):
        try:
            with open(CONFIG_FILE, "w", encoding="utf-8") as f:
                json.dump(self.connections, f, indent=2)
        except Exception:
            pass

    def _load_connections_from_file(self):
        if os.path.exists(CONFIG_FILE):
            try:
                with open(CONFIG_FILE, "r", encoding="utf-8") as f:
                    self.connections = json.load(f)
            except Exception:
                self.connections = {}

    # --- Connect / Disconnect ---
    def connect_selected(self):
        """Connect to the database using the active connection name"""
        if not self.active_conn_name:
            messagebox.showwarning("Select", "Select a saved connection first.")
            return

        try:
            with open(CONFIG_FILE, "r", encoding="utf-8") as f:
                all_cfg = json.load(f)
            if self.active_conn_name not in all_cfg:
                messagebox.showerror("Error", f"Connection '{self.active_conn_name}' not found.")
                return

            cfg = all_cfg[self.active_conn_name]
            host = cfg.get("host", "127.0.0.1")
            port = int(cfg.get("port", 3306))
            user = cfg.get("user", "root")
            pw = cfg.get("password", "")
            db = cfg.get("database", "")

            # ✅ decrypt password if possible
            if pw and _HAS_CRYPTO:
                try:
                    pw = decrypt_password(pw)
                except Exception:
                    pass

            conn = pymysql.connect(
                host=host,
                port=port,
                user=user,
                password=pw,
                database=db,   # ✅ use saved database
                autocommit=True,
                cursorclass=pymysql.cursors.DictCursor,
                connect_timeout=8
            )

            self.active_conn = conn
            self.conn_label.config(text=self.active_conn_name or "(connected)")

            # ✅ Show DB name in status bar
            cur = conn.cursor()
            cur.execute("SELECT DATABASE()")
            row = cur.fetchone()
            dbname = list(row.values())[0] if isinstance(row, dict) else row[0]
            self.db_status_var.set(f"(DB: {dbname})")
            cur.close()

            self.refresh_schema_browser()
            messagebox.showinfo("Connected", f"Connected to {self.active_conn_name} successfully.")

        except Exception as exc:
            import traceback
            traceback.print_exc()
            messagebox.showerror("Connect failed", str(exc))

    def disconnect_active(self):
        if self.active_conn:
            try:
                self.active_conn.close()
            except Exception:
                pass
        self.active_conn = None
        self.conn_label.config(text="(none)")
        self._log_history("Disconnected")
        try:
            self.schema_tree.delete(*self.schema_tree.get_children())
        except Exception:
            pass

        self.db_status_var.set("(DB: none)")

    # --- Save / Load query ---
    def save_query(self):
        self.save_query_as()

    def save_query_as(self):
        if not self.editors:
            messagebox.showwarning("No Editor", "No editor tab is open.")
            return
        ed = self.editors[self.tab_control.index(self.tab_control.select())]
        path = filedialog.asksaveasfilename(defaultextension=".sql", filetypes=[("SQL Files", "*.sql"), ("All Files", "*.*")])
        if not path:
            return
        try:
            with open(path, "w", encoding="utf-8") as f:
                f.write(ed.get())
            ed.modified = False
            self._log_history(f"Saved query to {path}")
        except Exception as exc:
            messagebox.showerror("Save failed", str(exc))

    def load_query(self):
        if not self.editors:
            messagebox.showwarning("No Editor", "No editor tab is open.")
            return
        path = filedialog.askopenfilename(filetypes=[("SQL Files", "*.sql"), ("All Files", "*.*")])
        if not path:
            return
        ed = self.editors[self.tab_control.index(self.tab_control.select())]
        try:
            with open(path, "r", encoding="utf-8") as f:
                content = f.read()
            ed.delete("1.0", tk.END)
            ed.insert("1.0", content)
            ed.modified = False
            self._log_history(f"Loaded query from {path}")
        except Exception as exc:
            messagebox.showerror("Load failed", str(exc))

    # --- Autosave drafts ---
    def _start_autosave(self):
        try:
            self.after(AUTOSAVE_INTERVAL_MS, self._autosave)
        except Exception:
            pass

    def _autosave(self):
        try:
            for i, ed in enumerate(self.editors, start=1):
                txt = ed.get()
                p = os.path.join(DRAFTS_DIR, f"draft_tab_{i}.sql")
                with open(p, "w", encoding="utf-8") as f:
                    f.write(txt)
        except Exception:
            pass
        try:
            self.after(AUTOSAVE_INTERVAL_MS, self._autosave)
        except Exception:
            pass

    # --- History logging and helpers ---
    def _log_history(self, txt):
        ts = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        display = f"{ts}  {txt}"
        try:
            self.history_list.insert(tk.END, display)
        except Exception:
            pass
        if not hasattr(self, "_history_data"):
            self._history_data = []
        self._history_data.append({"status": display, "raw": txt})
        try:
            self.history_list.see(tk.END)
        except Exception:
            pass

    def _add_history(self, preview, status, raw):
        entry = {"preview": preview, "status": status, "raw": raw, "ts": datetime.now().isoformat()}
        display = f"{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}  {status}"
        entry["display"] = display
        self._history_data.append(entry)
        try:
            self.history_list.insert(tk.END, display)
            self.history_list.see(tk.END)
        except Exception:
            pass

    def _history_rerun(self, event):
        sel = self.history_list.curselection()
        if not sel:
            return
        idx = sel[0]
        data = self._history_data[idx]
        if messagebox.askyesno("Re-run", f"Re-run this query?\n\n{data.get('preview','')}"):
            self.add_tab(initial_text=data.get('raw',''))
            self.run_query()

    def refresh_history(self):
        try:
            self.history_list.delete(0, tk.END)
            for h in self._history_data:
                self.history_list.insert(tk.END, h.get("display", ""))
        except Exception:
            pass

    def clear_history(self):
        if messagebox.askyesno("Clear History", "Clear action history?"):
            self._history_data = []
            try:
                self.history_list.delete(0, tk.END)
            except Exception:
                pass

    def _on_history_right(self, event):
        sel = self.history_list.curselection()
        menu = tk.Menu(self, tearoff=0)
        menu.add_command(label="Copy", command=self._copy_history_item)
        menu.add_command(label="Re-run", command=lambda: self._history_rerun(None))
        menu.add_command(label="Delete", command=self._delete_history_item)
        try:
            menu.tk_popup(event.x_root, event.y_root)
        finally:
            try:
                menu.grab_release()
            except Exception:
                pass

    def _copy_history_item(self):
        sel = self.history_list.curselection()
        if not sel:
            return
        idx = sel[0]
        txt = self._history_data[idx].get("raw", "")
        try:
            self.clipboard_clear()
            self.clipboard_append(txt)
        except Exception:
            pass

    def _delete_history_item(self):
        sel = self.history_list.curselection()
        if not sel:
            return
        idx = sel[0]
        self._history_data.pop(idx)
        try:
            self.history_list.delete(idx)
        except Exception:
            pass

    # --- Query execution (background thread) ---
    def run_query(self):
        ed = self.editors[self.tab_control.index(self.tab_control.select())]
        try:
            # Use selected SQL if available
            raw = ed.text.get(tk.SEL_FIRST, tk.SEL_LAST).strip()
        except Exception:
            # Otherwise use entire editor content
            raw = ed.get().strip()

        if not raw:
            messagebox.showwarning("No SQL", "Type a query or select one.")
            return

        if not self.active_conn:
            messagebox.showwarning("Not connected", "Connect to DB first.")
            return
        if self.running_thread and self.running_thread.is_alive():
            messagebox.showwarning("Busy", "A query is already running.")
            return

        # UI: show running state
        try:
            self.query_status_var.set("○ Running…")
            self.query_status_label.configure(foreground="#0b5ed7")
            self.progress.start(12)
            self._show_loading_overlay("Running query…")
        except Exception:
            pass

        t = threading.Thread(target=self._query_worker, args=(raw,), daemon=True)
        t.start()
        self.running_thread = t
        self._log_history("Query started (background)")

    # Variants for split run
    def run_query_all(self):
        try:
            ed = self.editors[self.tab_control.index(self.tab_control.select())]
            raw = ed.get().strip()
            if not raw:
                show_toast(self, "No SQL to run", kind="warn")
                return
            t = threading.Thread(target=self._query_worker, args=(raw,), daemon=True)
            t.start()
            self.running_thread = t
            self._log_history("Run All started")
        except Exception as exc:
            show_toast(self, str(exc), kind="error")

    def run_query_with_limit(self, limit=1000):
        try:
            ed = self.editors[self.tab_control.index(self.tab_control.select())]
            raw = ed.get().strip()
            if not raw:
                show_toast(self, "No SQL to run", kind="warn")
                return
            lowered = raw.lower()
            if lowered.startswith("select") and (" limit " not in lowered):
                raw = raw + f"\nLIMIT {int(limit)}"
            t = threading.Thread(target=self._query_worker, args=(raw,), daemon=True)
            t.start()
            self.running_thread = t
            self._log_history(f"Run with LIMIT {limit}")
        except Exception as exc:
            show_toast(self, str(exc), kind="error")

    def _query_worker(self, raw):
        preview = re.sub(r"\s+", " ", raw).strip()
        if len(preview) > 160:
            preview = preview[:157] + "..."
        start = time.time()
        cur = None
        try:
            cur = self.active_conn.cursor()
            try:
                tid = self.active_conn.thread_id()
                self.running_tid = tid
            except Exception:
                self.running_tid = None
            cur.execute(raw)
            elapsed = time.time() - start
            if cur.description:
                rows = cur.fetchall()
                cols = [d[0] for d in cur.description]
                self.after(0, lambda rows=rows, cols=cols, preview=preview, elapsed=elapsed, raw=raw:
                           self._on_select_success(preview, rows, cols, elapsed, raw))
            else:
                affected = cur.rowcount if (cur.rowcount is not None and cur.rowcount >= 0) else 0
                try:
                    self.active_conn.commit()
                except Exception:
                    pass
                self.after(0, lambda preview=preview, affected=affected, elapsed=elapsed, raw=raw:
                           self._on_nonselect_success(preview, affected, elapsed, raw))
        except Exception as exc:
            elapsed = time.time() - start
            self.after(0, lambda exc=exc, preview=preview, elapsed=elapsed, raw=raw:
                       self._on_query_error(preview, str(exc), elapsed, raw))
        finally:
            try:
                if cur:
                    cur.close()
            except Exception:
                pass
            def _clear_running():
                self.running_thread = None
                self.running_tid = None
            try:
                self.after(0, _clear_running)
            except Exception:
                _clear_running()

    def _on_select_success(self, preview, rows, cols, elapsed, raw):
        try:
            self.query_rows = rows
            self.result_tree["columns"] = cols
            for c in cols:
                self.result_tree.heading(c, text=c, command=lambda _c=c: self.sort_column(_c, False))
                self.result_tree.column(c, width=100, anchor="w")
            self.current_page = 1
            self.update_result_grid()
            # brief results flash animation
            self._animate_results_flash()
            ts = datetime.now().strftime("%H:%M:%S")
            status = f"{ts}  {preview}  {len(rows)} row(s) returned  {elapsed:.3f}s"
            self.status_var.set(status)
            self._add_history(preview, status, raw)
            # 🔥 Flash success message
            self.flash_status("Query executed successfully 🎉", 2500)
            # Indicator success (green tick)
            try:
                self.query_status_var.set("✔ Success")
                self.query_status_label.configure(foreground="#198754")
                self.progress.stop()
                self._hide_loading_overlay()
            except Exception:
                pass
        except Exception as exc:
            try:
                messagebox.showerror("Display error", str(exc))
            except Exception:
                pass

    def _on_nonselect_success(self, preview, affected, elapsed, raw):
        try:
            self.query_rows = []
            try:
                self.result_tree.delete(*self.result_tree.get_children())
            except Exception:
                pass
            self.result_tree["columns"] = []
            ts = datetime.now().strftime("%H:%M:%S")
            status = f"{ts}  {preview}  {affected} row(s) affected  {elapsed:.3f}s"
            self.status_var.set(status)
            self._add_history(preview, status, raw)
            # 🔥 Flash affected rows
            self.flash_status(f"{affected} row(s) affected ✅", 2500)
            try:
                self.query_status_var.set("✔ Success")
                self.query_status_label.configure(foreground="#198754")
                self.progress.stop()
                self._hide_loading_overlay()
            except Exception:
                pass
        except Exception as exc:
            try:
                messagebox.showerror("Display error", str(exc))
            except Exception:
                pass

    def _on_query_error(self, preview, err, elapsed, raw):
        ts = datetime.now().strftime("%H:%M:%S")
        status = f"{ts}  ERROR: {err}  {elapsed:.3f}s"
        self.status_var.set(status)
        self._add_history(preview, status, raw)
        try:
            # also append to Messages tab
            self.messages_box.configure(state="normal")
            self.messages_box.insert(tk.END, f"{status}\n")
            self.messages_box.configure(state="disabled")
        except Exception:
            pass
        try:
            messagebox.showerror("Query Error", err)
        except Exception:
            pass
        try:
            self.query_status_var.set("✖ Error")
            self.query_status_label.configure(foreground="#dc3545")
            self.progress.stop()
            self._hide_loading_overlay()
        except Exception:
            pass

    def cancel_query(self):
        if not self.running_tid:
            messagebox.showinfo("Cancel", "No running query detected or cannot determine thread id.")
            return
        if not self.active_conn_name:
            messagebox.showerror("Cancel", "No active connection name available.")
            return
        if messagebox.askyesno("Cancel Query", "Attempt to cancel running query?"):
            cfg = self.connections.get(self.active_conn_name)
            if not cfg:
                messagebox.showerror("Cancel", "Connection config missing.")
                return
            host = cfg.get("host")
            port = int(cfg.get("port", 3306))
            user = cfg.get("user")
            pw = cfg.get("password", "")
            if pw and _HAS_CRYPTO:
                try:
                    pw = decrypt_password(pw)
                except Exception:
                    pass
            try:
                killer = pymysql.connect(host=host, port=port, user=user, password=pw,
                                         autocommit=True, cursorclass=pymysql.cursors.DictCursor)
                kcur = killer.cursor()
                kcur.execute(f"KILL QUERY {self.running_tid}")
                kcur.close()
                killer.close()
                self._log_history(f"Sent KILL QUERY {self.running_tid}")
            except Exception as exc:
                messagebox.showerror("Cancel failed", str(exc))

    # --- EXPLAIN ---
    def run_explain(self):
        if not self.active_conn:
            messagebox.showwarning("Not connected", "Connect to DB first.")
            return
        ed = self.editors[self.tab_control.index(self.tab_control.select())]
        sql = ed.selection_get().strip()
        if not sql:
            messagebox.showwarning("No SQL", "Please select or type a query first.")
            return
        try:
            cur = self.active_conn.cursor()
            cur.execute("EXPLAIN " + sql)
            rows = cur.fetchall()
            cols = [d[0] for d in cur.description]
            cur.close()

            self.result_tree.delete(*self.result_tree.get_children())
            self.result_tree["columns"] = cols

            # ✅ Auto expand columns equally across result area
            total_cols = len(cols)
            for c in cols:
                self.result_tree.heading(c, text=c, command=lambda _c=c: self.sort_column(_c, False))
                self.result_tree.column(
                    c,
                    width=int(self.result_tree.winfo_width() / max(total_cols, 1)),
                    anchor="w",
                    stretch=True
                )

            for i, r in enumerate(rows):
                if isinstance(r, dict):
                    vals = [r.get(c, "") for c in cols]
                else:
                    vals = list(r)
                tag = "oddrow" if i % 2 else "evenrow"
                self.result_tree.insert("", tk.END, values=vals, tags=(tag,))

            self._log_history("Ran EXPLAIN successfully")
        except Exception as exc:
            messagebox.showerror("EXPLAIN failed", str(exc))

    # --- Schema browser refresh ---
    def refresh_schema_browser(self):
        try:
            self.schema_tree.delete(*self.schema_tree.get_children())
        except Exception:
            pass
        if not self.active_conn:
            return
        try:
            cur = self.active_conn.cursor()
            cur.execute("SHOW TABLES")
            rows = cur.fetchall()
            tables = []
            for r in rows:
                if isinstance(r, dict):
                    tables.append(list(r.values())[0])
                elif isinstance(r, (list, tuple)):
                    tables.append(r[0])
            for t in tables:
                parent = self.schema_tree.insert("", tk.END, text=t, open=False)
                try:
                    cur.execute(f"DESCRIBE `{t}`")
                    cols = [r['Field'] for r in cur.fetchall()]
                except Exception:
                    cols = []
                for c in cols:
                    self.schema_tree.insert(parent, tk.END, text=c, values=(t, c))
            cur.close()

            # Populate schema_cache for autocomplete and other features
            self.schema_cache = {}
            for t in tables:
                try:
                    cur2 = self.active_conn.cursor()
                    cur2.execute(f"DESCRIBE `{t}`")
                    self.schema_cache[t] = [r['Field'] for r in cur2.fetchall()]
                    cur2.close()
                except Exception:
                    self.schema_cache[t] = []

            # update autocomplete lists in all editors
            for ed in self.editors:
                try:
                    ed.schema_tables = list(self.schema_cache.keys())
                except Exception:
                    pass
        except Exception as exc:
            try:
                messagebox.showerror("Schema Error", str(exc))
            except Exception:
                pass

    def _on_schema_double(self, event):
        sel = self.schema_tree.selection()
        if not sel:
            return
        item = sel[0]
        parent = self.schema_tree.parent(item)
        if parent == "":
            table_name = self.schema_tree.item(item, "text")
            self.open_query_builder_with_schema(initial_table=table_name)
        else:
            col = self.schema_tree.item(item, "text")
            table = self.schema_tree.item(parent, "text")
            sql = f"SELECT `{col}` FROM `{table}` LIMIT 100;"
            self.add_tab(initial_text=sql)

    # --- History add & management already above (_add_history etc) ---

    # --- result double click ---
    def _on_result_double(self, event):
        region = self.result_tree.identify_region(event.x, event.y)
        if region == "heading":
            col_id = self.result_tree.identify_column(event.x)
            try:
                idx = int(col_id.replace("#","")) - 1
            except Exception:
                idx = None
            cols = list(self.result_tree["columns"])
            if idx is not None and 0 <= idx < len(cols):
                self._autofit_column(cols[idx])
        else:
            item = self.result_tree.identify_row(event.y)
            if not item:
                return
            cols = list(self.result_tree["columns"])
            vals = self.result_tree.item(item).get("values", [])
            lines = []
            for c, v in zip(cols, vals):
                lines.append(f"{c}: {v}")
            messagebox.showinfo("Row Details", "\n".join(lines))

    def _autofit_column(self, col):
        try:
            maxw = len(col)
            for rid in self.result_tree.get_children():
                val = str(self.result_tree.set(rid, col))
                if len(val) > maxw:
                    maxw = len(val)
            width = min(max(COL_MIN_WIDTH, maxw * 8), COL_MAX_WIDTH)
            self.result_tree.column(col, width=width)
        except Exception:
            pass

    # --- copy/cut/export helpers for results ---
    def copy_cell(self):
        sel = self.result_tree.selection()
        if not sel:
            return
        item = sel[0]
        cols = list(self.result_tree["columns"])
        if not cols:
            return
        # copy first column value of selected row (common expectation)
        try:
            val = str(self.result_tree.item(item, "values")[0])
            self.clipboard_clear()
            self.clipboard_append(val)
        except Exception:
            pass

    def copy_row(self):
        sel = self.result_tree.selection()
        if not sel:
            return
        item = sel[0]
        vals = self.result_tree.item(item, "values")
        try:
            self.clipboard_clear()
            self.clipboard_append("\t".join(str(v) for v in vals))
        except Exception:
            pass

    def export_csv(self):
        if not self.query_rows:
            messagebox.showwarning("No Data", "No results to export.")
            return
        path = filedialog.asksaveasfilename(defaultextension=".csv", filetypes=[("CSV Files","*.csv")])
        if not path:
            return
        cols = list(self.result_tree["columns"])
        try:
            with open(path, "w", newline="", encoding="utf-8") as f:
                writer = csv.writer(f)
                writer.writerow(cols)
                for r in self.query_rows:
                    if isinstance(r, dict):
                        writer.writerow([r.get(c,"") for c in cols])
                    else:
                        writer.writerow(list(r))
            messagebox.showinfo("Export", f"Results exported to {path}")
        except Exception as exc:
            messagebox.showerror("Export failed", str(exc))

    def export_excel(self):
        if not _HAS_OPENPYXL:
            messagebox.showerror("Missing", "openpyxl is required for Excel export.")
            return
        if not self.query_rows:
            messagebox.showwarning("No Data", "No results to export.")
            return
        path = filedialog.asksaveasfilename(defaultextension=".xlsx", filetypes=[("Excel Files","*.xlsx")])
        if not path:
            return
        cols = list(self.result_tree["columns"])
        try:
            wb = openpyxl.Workbook()
            ws = wb.active
            ws.append(cols)
            for r in self.query_rows:
                if isinstance(r, dict):
                    ws.append([r.get(c,"") for c in cols])
                else:
                    ws.append(list(r))
            for i,c in enumerate(cols,1):
                col_letter = get_column_letter(i)
                ws.column_dimensions[col_letter].width = min(50, max(len(str(c)),12))
            wb.save(path)
            messagebox.showinfo("Export", f"Results exported to {path}")
        except Exception as exc:
            messagebox.showerror("Export failed", str(exc))

    def copy_selection_as(self, format="csv"):
        try:
            cols = list(self.result_tree["columns"]) or []
            targets = list(self.result_tree.selection()) or list(self.result_tree.get_children(""))
            rows = []
            for iid in targets:
                vals = [self.result_tree.set(iid, c) for c in cols]
                rows.append(vals)
            if not rows:
                return
            if format == "csv":
                out = ",".join(cols) + "\n" + "\n".join(
                    ",".join([str(v).replace("\n"," ") for v in r]) for r in rows
                )
            elif format == "tsv":
                out = "\t".join(cols) + "\n" + "\n".join(
                    "\t".join([str(v).replace("\n"," ") for v in r]) for r in rows
                )
            elif format == "md":
                header = "| " + " | ".join(cols) + " |\n" + "|" + "---|" * len(cols)
                body = "\n".join(["| " + " | ".join(map(str, r)) + " |" for r in rows])
                out = header + "\n" + body
            elif format == "insert":
                table = "results"
                values_sql = ",\n".join(["(" + ", ".join([repr(v) for v in r]) + ")" for r in rows])
                out = f"INSERT INTO `{table}` (" + ", ".join([f"`{c}`" for c in cols]) + ") VALUES\n" + values_sql + ";"
            else:
                out = "\n".join([", ".join(map(str, r)) for r in rows])
            self._clipboard_copy(out)
            show_toast(self, f"Copied as {format.upper()}", kind="success")
        except Exception as exc:
            show_toast(self, str(exc), kind="error")

    # --- results pagination & rendering ---
    def update_result_grid(self):
        try:
            self.result_tree.delete(*self.result_tree.get_children())
        except Exception:
            pass
        rows = self.query_rows
        total = len(rows)
        try:
            self.total_label.config(text=str(total))
        except Exception:
            pass
        if total == 0:
            return
        start = (self.current_page-1)*PAGE_SIZE
        end = start + PAGE_SIZE
        chunk = rows[start:end]
        for i,r in enumerate(chunk):
            if isinstance(r, dict):
                vals = [r.get(c,"") for c in self.result_tree["columns"]]
            else:
                vals = list(r)
            tag = "oddrow" if i%2 else "evenrow"
            self.result_tree.insert("", tk.END, values=vals, tags=(tag,))
        try:
            self.page_entry.delete(0, tk.END)
            self.page_entry.insert(0, str(self.current_page))
        except Exception:
            pass

    def next_page(self):
        if self.current_page*PAGE_SIZE < len(self.query_rows):
            self.current_page += 1
            self.update_result_grid()

    def prev_page(self):
        if self.current_page > 1:
            self.current_page -= 1
            self.update_result_grid()

    def _jump_to_page(self):
        try:
            val = int(self.page_entry.get())
        except Exception:
            return
        if val < 1: return
        if (val-1)*PAGE_SIZE >= len(self.query_rows): return
        self.current_page = val
        self.update_result_grid()

    # --- sort column ---
    def sort_column(self, col, reverse):
        try:
            l = [(self.result_tree.set(k,col), k) for k in self.result_tree.get_children("")]
            try:
                l.sort(key=lambda t: float(t[0]), reverse=reverse)
            except Exception:
                l.sort(key=lambda t: str(t[0]), reverse=reverse)
            for i,(val,k) in enumerate(l):
                self.result_tree.move(k,"",i)
            self.result_tree.heading(col, command=lambda: self.sort_column(col, not reverse))
        except Exception:
            pass

    # --- apply global filter from toolbar search ---
    def apply_filter(self):
        val = self.search_var.get().strip().lower()

        # Case 1: Empty search -> reload full grid
        if not val:
            self.update_result_grid()
            return

        # Case 2: Filtered search -> detach non-matching rows
        for rid in self.result_tree.get_children():
            try:
                row_text = " ".join(str(x) for x in self.result_tree.item(rid).get("values", []))
            except Exception:
                row_text = ""
            if val in row_text.lower():
                try:
                    self.result_tree.reattach(rid, "", "end")
                except Exception:
                    pass
            else:
                try:
                    self.result_tree.detach(rid)
                except Exception:
                    pass

    def _nl_to_sql(self, question: str, schema_hint: str = "") -> str:
        """
        Convert a natural language request into a SQL query using Gemini.
        Restricts output to SELECT statements only.
        """
        try:
            # Build schema hint string from cached schema
            schema_hint = "\n".join(
                f"{t}: {', '.join(cols)}" for t, cols in self.schema_cache.items()
            )

            prompt = f"""
            You are an expert MySQL assistant.

            Task: Convert the following natural language request into a valid MySQL SELECT query.
            Rules:
            - ONLY generate SELECT queries (no INSERT, UPDATE, DELETE, DROP, CREATE).
            - Use ONLY these tables and columns:
            {schema_hint}
            - Do NOT invent table or column names.
            - Output ONLY the SQL (no explanations, no markdown).

            User request: {question}
            """

            response = genai.GenerativeModel("gemini-2.0-flash").generate_content(prompt)
            sql = (response.text or "").strip()

            # Remove any accidental markdown formatting
            sql = sql.replace("```sql", "").replace("```", "").strip()

            # 🚨 Enforce SELECT-only using regex (allowing WITH ... SELECT etc.)
            import re
            if not re.match(r'^\s*(select|with)\b', sql, flags=re.IGNORECASE):
                raise ValueError("AI generated a non-SELECT statement, which is not allowed.")

            return sql

        except Exception as exc:
            messagebox.showerror("AI Error", str(exc))
            return ""

    def ask_nl_to_sql(self):
        if not self.active_conn:
            messagebox.showwarning("Not connected", "Connect to DB first.")
            return

        question = simpledialog.askstring("AI → SQL", "Enter your question (natural language):")
        if not question:
            return

        # Build schema hint for AI
        schema_hint = "\n".join(f"{t}: {', '.join(cols)}" for t, cols in self.schema_cache.items())
        sql = self._nl_to_sql(question, schema_hint=schema_hint)
        if not sql:
            return

        # Clean any markdown fences the model might return
        sql = sql.replace("```sql", "").replace("```", "").strip()

        # Add to a new tab, select all, and run
        self.add_tab(initial_text=sql)
        try:
            ed = self.editors[self.tab_control.index(self.tab_control.select())]
            ed.text.tag_add("sel", "1.0", "end-1c")  # select all so run_query surely picks it up
        except Exception:
            pass
        self._log_history(f"AI generated SQL: {sql}")
        self.run_query()

    # --- AI SQL Cleaner ---
    def _clean_ai_sql(self, sql: str) -> str:
        # Remove markdown fences like ```sql ... ```
        sql = sql.replace("```sql", "").replace("```", "").strip()

        # Optionally restrict AI queries to SELECT only
        if not sql.lower().startswith("select"):
            raise ValueError("AI-generated query is restricted to SELECT statements only.")
        return sql

    # --- Natural Language to Chart (Gemini powered) ---
    def ask_nl_to_chart(self):
        question = simpledialog.askstring("Ask for Chart", "What chart do you want?")
        if not question:
            return

        # Build schema hint
        schema_hint = "\n".join(f"{t}: {cols}" for t, cols in self.schema_cache.items())
        prompt = f"""
        You are a SQL + data visualization expert.
        Convert this request into a valid MySQL query.
        Only output SQL, nothing else.

        Database schema: {schema_hint}
        User request: {question}
        """

        try:
            response = genai.GenerativeModel("gemini-2.0-flash").generate_content(prompt)
            sql = response.text.strip()

            # Run query
            cur = self.active_conn.cursor()
            cur.execute(sql)
            rows = cur.fetchall()
            cur.close()

            if not rows:
                messagebox.showinfo("No Data", "Query returned no results.")
                return

            # Pick columns
            cols = list(rows[0].keys()) if isinstance(rows[0], dict) else []
            if len(cols) < 2:
                messagebox.showerror("Chart Error", "Need at least 2 columns for chart.")
                return

            x_vals = [list(r.values())[0] for r in rows]
            y_vals = [list(r.values())[1] for r in rows]

            # Plot
            try:
                import matplotlib.pyplot as plt  # type: ignore[import-not-found]
            except Exception:
                messagebox.showerror("Chart Error", "matplotlib is not installed.")
                return
            plt.figure(figsize=(8, 5))
            plt.bar(x_vals, y_vals)
            plt.title(question)
            plt.xticks(rotation=45, ha="right")
            plt.tight_layout()
            plt.show()

            self._log_history(f"AI chart query: {sql}")
        except Exception as exc:
            messagebox.showerror("AI Chart Error", str(exc))

    # --- Query builder (simple) & EER show (uses schema) ---
    def open_query_builder_with_schema(self, initial_table=None):
        if not self.active_conn:
            messagebox.showwarning("Not connected","Connect to DB first.")
            return
        try:
            cur = self.active_conn.cursor()
            cur.execute("SHOW TABLES")
            tables = []
            for r in cur.fetchall():
                if isinstance(r, dict):
                    tables.append(list(r.values())[0])
                else:
                    tables.append(r[0])
            cur.close()
            schema = {}
            for t in tables:
                try:
                    cur2 = self.active_conn.cursor()
                    cur2.execute(f"DESCRIBE `{t}`")
                    schema[t] = [r['Field'] for r in cur2.fetchall()]
                    cur2.close()
                except Exception:
                    schema[t] = []
            self._show_interactive_eer(self, schema)
        except Exception as exc:
            messagebox.showerror("Error", str(exc))

    def _get_foreign_keys(self):
        fks = []
        if not self.active_conn:
            return fks
        try:
            cur = self.active_conn.cursor()
            cur.execute("SELECT DATABASE()")
            row = cur.fetchone()
            if isinstance(row, dict):
                dbname = list(row.values())[0]
            elif isinstance(row, (list, tuple)):
                dbname = row[0]
            else:
                dbname = None
            cur.close()
            if not dbname:
                return fks

            cur = self.active_conn.cursor()
            query = """
            SELECT TABLE_NAME, COLUMN_NAME, REFERENCED_TABLE_NAME, REFERENCED_COLUMN_NAME
            FROM information_schema.KEY_COLUMN_USAGE
            WHERE TABLE_SCHEMA = %s
                AND REFERENCED_TABLE_NAME IS NOT NULL
            """
            cur.execute(query, (dbname,))
            rows = cur.fetchall()
            cur.close()
            for r in rows:
                if isinstance(r, dict):
                    fks.append({"table": r.get("TABLE_NAME"), "col": r.get("COLUMN_NAME"),
                                "ref_table": r.get("REFERENCED_TABLE_NAME"), "ref_col": r.get("REFERENCED_COLUMN_NAME")})
                elif isinstance(r, (list, tuple)):
                    fks.append({"table": r[0], "col": r[1], "ref_table": r[2], "ref_col": r[3]})
        except Exception as exc:
            print("Foreign key fetch failed:", exc)
        return fks

    def _show_interactive_eer(self, parent_window, schema, gv_positions=None):
        win = tk.Toplevel(self)
        win.title("Interactive EER Diagram")
        win.geometry("1400x900")

        # --- Controls ------------------------------------------------------------
        ctlbar = ttk.Frame(win); ctlbar.pack(fill=tk.X, pady=(4,4))
        ttk.Label(ctlbar, text="Layout:").pack(side=tk.LEFT, padx=4)
        layout_var = tk.StringVar(value="grid")
        layout_cb = ttk.Combobox(ctlbar, textvariable=layout_var,
                                values=["grid","hierarchical","radial"],
                                width=14, state="readonly")
        layout_cb.pack(side=tk.LEFT)
        ttk.Button(ctlbar, text="Apply Layout",
                   command=lambda: apply_layout(layout_var.get())).pack(side=tk.LEFT, padx=6)

        ttk.Label(ctlbar, text="Quality:").pack(side=tk.LEFT, padx=4)
        quality_var = tk.StringVar(value="High")
        quality_cb = ttk.Combobox(ctlbar, textvariable=quality_var,
                                  values=["Normal","High","Ultra"],
                                  width=10, state="readonly")
        quality_cb.pack(side=tk.LEFT, padx=4)

        ttk.Button(ctlbar, text="Export PNG",
                   command=lambda: export_canvas("png")).pack(side=tk.LEFT, padx=6)
        ttk.Button(ctlbar, text="Export PDF",
                   command=lambda: export_canvas("pdf")).pack(side=tk.LEFT, padx=6)
        ttk.Button(ctlbar, text="Close", command=win.destroy).pack(side=tk.RIGHT, padx=6)

        ttk.Label(ctlbar, text="Zoom:").pack(side=tk.LEFT, padx=4)
        zoom_var = tk.DoubleVar(value=1.0)
        zoom_slider = ttk.Scale(ctlbar, from_=0.4, to=2.4, variable=zoom_var,
                                orient=tk.HORIZONTAL, length=150)
        zoom_slider.pack(side=tk.LEFT, padx=(6,0))

        # --- Canvas + Scrollbars ------------------------------------------------
        canvas_bg = "#1e1e1e" if getattr(self, "dark_mode", False) else "white"
        wrap = ttk.Frame(win); wrap.pack(fill=tk.BOTH, expand=True)

        vbar = ttk.Scrollbar(wrap, orient=tk.VERTICAL)
        hbar = ttk.Scrollbar(wrap, orient=tk.HORIZONTAL)

        canvas = tk.Canvas(
            wrap, bg=canvas_bg,
            xscrollcommand=hbar.set, yscrollcommand=vbar.set
        )
        canvas.grid(row=0, column=0, sticky="nsew")
        vbar.config(command=canvas.yview); vbar.grid(row=0, column=1, sticky="ns")
        hbar.config(command=canvas.xview); hbar.grid(row=1, column=0, sticky="ew")
        wrap.rowconfigure(0, weight=1); wrap.columnconfigure(0, weight=1)

        # Ensure sizes are realized before we compute layouts
        win.update_idletasks()

        # --- Zoom (slider + Ctrl+Wheel) -----------------------------------------
        current_scale = {"s": 1.0}

        def _apply_scale_at(cx, cy, factor):
            canvas.scale("all", cx, cy, factor, factor)
            bbox = canvas.bbox("all")
            if bbox:
                canvas.configure(scrollregion=bbox)

        def set_zoom(val):
            try:
                val = float(val)
            except Exception:
                return
            if val <= 0: 
                return
            if abs(val - current_scale["s"]) < 1e-6:
                return
            factor = val / current_scale["s"]
            current_scale["s"] = val
            # zoom around canvas center
            cx = canvas.canvasx(canvas.winfo_width() // 2)
            cy = canvas.canvasy(canvas.winfo_height() // 2)
            _apply_scale_at(cx, cy, factor)

        zoom_var.trace_add("write", lambda *_: set_zoom(zoom_var.get()))

        # Ctrl + mouse wheel (Win/mac), Button-4/5 (Linux)
        def wheel_zoom(event):
            # Require Ctrl to avoid accidental zoom
            if (event.state & 0x0004) == 0:  # Control mask
                return
            # Wheel delta normalization
            delta = 0
            if hasattr(event, "delta") and event.delta != 0:
                delta = 1 if event.delta > 0 else -1
            elif getattr(event, "num", None) in (4, 5):  # Linux
                delta = 1 if event.num == 4 else -1
            if delta == 0:
                return
            step = 1.1 if delta > 0 else (1/1.1)
            new_scale = max(0.2, min(3.0, current_scale["s"] * step))
            factor = new_scale / current_scale["s"]
            current_scale["s"] = new_scale
            cx, cy = canvas.canvasx(event.x), canvas.canvasy(event.y)
            _apply_scale_at(cx, cy, factor)
            try:
                zoom_var.set(new_scale)  # keep slider in sync
            except Exception:
                pass

        canvas.bind("<Control-MouseWheel>", wheel_zoom)   # Windows/mac
        canvas.bind("<Control-Button-4>", wheel_zoom)     # Linux up
        canvas.bind("<Control-Button-5>", wheel_zoom)     # Linux down
        canvas.focus_set()

        # --- Panning (middle button or Space+drag) ------------------------------
        # Middle-button pan (or two-finger drag on some touchpads)
        def pan_mark(e):
            canvas.scan_mark(e.x, e.y)
        def pan_drag(e):
            canvas.scan_dragto(e.x, e.y, gain=1)
        canvas.bind("<ButtonPress-2>", pan_mark)
        canvas.bind("<B2-Motion>", pan_drag)

        # Space + left-drag pan (hand tool style)
        pan_mode = {"on": False}
        def space_down(e):
            pan_mode["on"] = True
            canvas.configure(cursor="fleur")
        def space_up(e):
            pan_mode["on"] = False
            canvas.configure(cursor="")
        def pan_if_space_mark(e):
            if pan_mode["on"]:
                canvas.scan_mark(e.x, e.y)
        def pan_if_space_drag(e):
            if pan_mode["on"]:
                canvas.scan_dragto(e.x, e.y, gain=1)
        canvas.bind_all("<KeyPress-space>", space_down)
        canvas.bind_all("<KeyRelease-space>", space_up)
        canvas.bind("<ButtonPress-1>", pan_if_space_mark, add="+")
        canvas.bind("<B1-Motion>", pan_if_space_drag, add="+")

        # --- Build nodes/edges ---------------------------------------------------
        tables = list(schema.keys())
        if not tables:
            canvas.create_text(
                20, 40, anchor="nw", text="No tables found",
                font=("Helvetica", 12, "bold"),
                fill=("#ffffff" if getattr(self, "dark_mode", False) else "#333333")
            )
            return

        nodes = {}
        cols_grid = max(1, int((len(tables) ** 0.5) + 0.5))
        index = 0
        for t in tables:
            if gv_positions and t in gv_positions:
                x, y = gv_positions[t]
            else:
                row, col = index // cols_grid, index % cols_grid
                x, y = 80 + col * 320, 80 + row * 220
                index += 1
            nd = TableNode(canvas, t, schema[t], x=x, y=y)
            nodes[t] = nd

        fk_list = self._get_foreign_keys()
        edges = []
        colors = ["#ff5555", "#55aa55", "#5577dd", "#ffaa00", "#bb44cc"]

        for i, fk in enumerate(fk_list):
            t, col, rt, ref_col = fk["table"], fk["col"], fk["ref_table"], fk["ref_col"]
            if t not in nodes or rt not in nodes:
                continue
            na, nb = nodes[t], nodes[rt]
            x1 = na.x + na.width
            y1 = na.y + TableNode.HEADER_H + TableNode.ROW_H / 2
            x2 = nb.x
            y2 = nb.y + TableNode.HEADER_H + TableNode.ROW_H / 2
            color = colors[i % len(colors)]
            lid = canvas.create_line(x1, y1, x2, y2, arrow=tk.LAST, width=2,
                                     smooth=True, splinesteps=36, fill=color,
                                     tags=("fk_line",))
            label = f"{t}.{col} → {rt}.{ref_col}"
            mx, my = (x1 + x2)/2, (y1 + y2)/2
            label_id = canvas.create_text(mx, my-12, text=label,
                                         font=("Helvetica", 8, "italic"),
                                         fill=color, tags=("fk_label",))
            edges.append({"line": lid, "label": label_id, "from": t, "to": rt, "fk": fk})

        # Node drag already works through TableNode bindings. Update lines on drag:
        def _update_edges():
            for e in edges:
                na, nb = nodes[e["from"]], nodes[e["to"]]
                x1 = na.x + na.width
                y1 = na.y + TableNode.HEADER_H + TableNode.ROW_H / 2
                x2 = nb.x
                y2 = nb.y + TableNode.HEADER_H + TableNode.ROW_H / 2
                try:
                    canvas.coords(e["line"], x1, y1, x2, y2)
                    mx, my = (x1 + x2)/2, (y1 + y2)/2
                    canvas.coords(e["label"], mx, my-12)
                except Exception:
                    pass

        # Hook into canvas motion to keep edges updated while dragging nodes
        canvas.bind("<B1-Motion>", lambda e: (_update_edges(),), add="+")

        # Initial scrollregion
        bbox = canvas.bbox("all")
        if bbox:
            canvas.configure(scrollregion=bbox)

        # --- Layouts -------------------------------------------------------------
        def apply_layout(mode="grid"):
            count = len(nodes)
            if count == 0:
                return

            win.update_idletasks()  # make sure we have real sizes
            if mode == "grid":
                cols = max(1, int((count ** 0.5) + 0.5))
                i = 0
                for tname, nd in nodes.items():
                    row, col = i // cols, i % cols
                    target_x, target_y = 80 + col * 320, 80 + row * 220
                    dx, dy = target_x - nd.x, target_y - nd.y
                    nd.move(dx, dy)
                    i += 1
            elif mode == "radial":
                center_x = canvas.canvasx(canvas.winfo_width() // 2)
                center_y = canvas.canvasy(canvas.winfo_height() // 2)
                radius = max(200, min(800, len(nodes) * 40))
                i, total = 0, len(nodes)
                for tname, nd in nodes.items():
                    ang = (2 * math.pi * i) / total
                    target_x = center_x + math.cos(ang) * radius
                    target_y = center_y + math.sin(ang) * radius
                    dx, dy = target_x - nd.x, target_y - nd.y
                    nd.move(dx, dy); i += 1

            _update_edges()
            bbox = canvas.bbox("all")
            if bbox:
                canvas.configure(scrollregion=bbox)

        # --- Export --------------------------------------------------------------
        def export_canvas(fmt="png"):
            q = quality_var.get()
            scale_factor = 4
            if q == "Normal": scale_factor = 2
            elif q == "Ultra": scale_factor = 6
            ext = fmt.lower()
            filetypes = [(f"{ext.upper()} files", f"*.{ext}"), ("All files", "*.*")]
            out_file = filedialog.asksaveasfilename(defaultextension=f".{ext}",
                                                    filetypes=filetypes,
                                                    title=f"Save EER Diagram as {ext.upper()}")
            if not out_file:
                return
            ps_file = out_file + ".ps"
            try:
                # Higher page size for better vector -> raster conversion
                canvas.postscript(file=ps_file, colormode="color",
                                  pagewidth=canvas.winfo_width()*2,
                                  pageheight=canvas.winfo_height()*2)
                if not _HAS_PIL:
                    messagebox.showinfo("Export",
                        f"Pillow not installed.\nPostScript file saved as {ps_file}\nOpen it in Inkscape/Illustrator.")
                    return
                try:
                    from PIL import Image  # type: ignore[import-not-found]
                except Exception:
                    messagebox.showerror("Export Error", "Pillow is not installed.")
                    return
                try:
                    img = Image.open(ps_file)
                    w, h = img.size
                    # Pillow 9+: use Image.Resampling.LANCZOS
                    try:
                        img = img.resize((w * scale_factor, h * scale_factor), Image.Resampling.LANCZOS)
                    except Exception:
                        img = img.resize((w * scale_factor, h * scale_factor), Image.LANCZOS)
                    img.save(out_file)
                    os.remove(ps_file)
                    messagebox.showinfo("Export", f"EER diagram exported to {out_file}")
                except OSError:
                    messagebox.showwarning("Export", f"Ghostscript not found.\nPostScript file saved as {ps_file}")
            except Exception as exc:
                messagebox.showerror("Export Error", str(exc))

    # --- Data visualization (uses matplotlib if available) ---
    def open_data_viz(self):
        if not _HAS_MATPLOTLIB:
            messagebox.showerror("Missing", "matplotlib is required for plotting.")
            return
        if not self.active_conn:
            messagebox.showwarning("Not connected", "Connect to DB first.")
            return
        try:
            self.refresh_schema_browser()
        except Exception:
            pass

        dv = tb.Toplevel(self)   # use ttkbootstrap Toplevel
        dv.title("Data Visualization")
        dv.geometry("1000x700")

        # --- Left control panel ---
        left = ttk.Frame(dv, width=300)
        left.pack(side=tk.LEFT, fill=tk.Y, padx=10, pady=10)

        ttk.Label(left, text="Select Table:").pack(anchor="w", pady=(8, 0))
        table_var = tk.StringVar()
        table_cb = ttk.Combobox(
            left, values=sorted(list(self.schema_cache.keys())),
            textvariable=table_var, state="readonly"
        )
        table_cb.pack(fill=tk.X, pady=4)

        ttk.Label(left, text="X Axis Column:").pack(anchor="w")
        x_var = tk.StringVar()
        x_cb = ttk.Combobox(left, textvariable=x_var, state="readonly")
        x_cb.pack(fill=tk.X, pady=4)

        ttk.Label(left, text="Y Axis Column:").pack(anchor="w")
        y_var = tk.StringVar()
        y_cb = ttk.Combobox(left, textvariable=y_var, state="readonly")
        y_cb.pack(fill=tk.X, pady=4)

        ttk.Label(left, text="Chart Type:").pack(anchor="w", pady=(8, 2))
        chart_type = tk.StringVar(value="bar")
        for t in ["bar", "line", "pie"]:
            ttk.Radiobutton(left, text=t.capitalize(), variable=chart_type, value=t).pack(anchor="w")

        # --- Chart Style Controls ---
        ttk.Label(left, text="Bar Width:").pack(anchor="w", pady=(10, 0))
        bar_width_var = tk.DoubleVar(value=0.6)   # default width
        ttk.Scale(left, from_=0.1, to=2.0, variable=bar_width_var,
                orient=tk.HORIZONTAL, length=180).pack(fill=tk.X, pady=2)

        ttk.Label(left, text="Line Thickness:").pack(anchor="w", pady=(10, 0))
        line_thickness_var = tk.DoubleVar(value=2.0)  # default line width
        ttk.Scale(left, from_=0.5, to=5.0, variable=line_thickness_var,
                orient=tk.HORIZONTAL, length=180).pack(fill=tk.X, pady=2)

        # --- Chart Area ---
        fig = plt.Figure(figsize=(6, 5))
        fig.cursor = None
        canvas_fig = FigureCanvasTkAgg(fig, master=dv)
        canvas_fig.get_tk_widget().pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

        # --- Update column dropdowns when table selected ---
        def update_columns(event=None):
            t = table_var.get()
            if t in self.schema_cache:
                cols = self.schema_cache[t]
                x_cb["values"] = cols
                y_cb["values"] = cols
                if cols:
                    x_var.set(cols[0])
                    if len(cols) > 1:
                        y_var.set(cols[1])

        table_cb.bind("<<ComboboxSelected>>", update_columns)

        # --- Plot function ---
        def draw_plot():
            table = table_var.get()
            xcol = x_var.get()
            ycol = y_var.get()
            ctype = chart_type.get()

            if not table or not xcol or not ycol:
                messagebox.showwarning("Missing Selection", "Please select table and columns.")
                return

            try:
                cur = self.active_conn.cursor()
                cur.execute(f"SELECT `{xcol}`, `{ycol}` FROM `{table}` LIMIT 500")
                rows = cur.fetchall()
                cur.close()

                if not rows:
                    messagebox.showinfo("No Data", "No rows returned.")
                    return

                # Extract data
                xs = [list(r.values())[0] for r in rows]
                try:
                    ys = [float(list(r.values())[1]) for r in rows]  # allow decimals
                except Exception:
                    ys = [0 for _ in rows]

                fig.clf()
                ax = fig.add_subplot(111)

                # disconnect old hover if exists
                if hasattr(fig, "cursor") and fig.cursor:
                    try:
                        fig.cursor.disconnect()
                    except Exception:
                        pass
                    fig.cursor = None

                if ctype == "bar":
                    width = bar_width_var.get()
                    if all(isinstance(x, (int, float)) for x in xs):
                        bars = ax.bar(range(len(xs)), ys, width=width, linewidth=0.8, edgecolor="black")
                        ax.set_xticks(range(len(xs)))
                        ax.set_xticklabels(xs, rotation=45, ha="right")
                    else:
                        bars = ax.bar(xs, ys, width=width, linewidth=0.8, edgecolor="black")

                    if _HAS_MPLCURSORS:
                        mplcursors.cursor(bars, hover=True).connect(
                            "add",
                            lambda sel: sel.annotation.set_text(
                                f"x = {xs[int(sel.index)]}\n"
                                f"y = {format(float(ys[int(sel.index)]), ',')}"
                            )
                        )

                elif ctype == "line":
                    lw = line_thickness_var.get()
                    line, = ax.plot(xs, ys, marker="o", linewidth=lw)
                    if _HAS_MPLCURSORS:
                        mplcursors.cursor(line, hover=True).connect(
                            "add",
                            lambda sel: sel.annotation.set_text(
                                f"x = {xs[int(sel.index)]}\n"
                                f"y = {format(float(ys[int(sel.index)]), ',')}"
                            )
                        )

                elif ctype == "pie":
                    ys_num = [float(v) if isinstance(v, (int, float)) else 0 for v in ys]
                    wedges, texts, autotexts = ax.pie(
                        ys_num,
                        labels=[str(v) for v in xs],
                        autopct="%d%%"
                    )
                    if _HAS_MPLCURSORS:
                        mplcursors.cursor(wedges, hover=True).connect(
                            "add",
                            lambda sel: sel.annotation.set_text(
                                f"x = {xs[int(sel.index)]}\n"
                                f"y = {format(float(ys_num[int(sel.index)]), ',')}"
                            )
                        )

                # Labels & title
                ax.set_xlabel(xcol)
                if ctype != "pie":
                    ax.set_ylabel(ycol)
                ax.set_title(f"{ctype.capitalize()} Chart of {ycol} vs {xcol}")
                fig.tight_layout()
                canvas_fig.draw()

            except Exception as exc:
                messagebox.showerror("Plot Error", str(exc))

        # --- Styled Buttons ---
        tb.Button(left, text="Plot", bootstyle=SUCCESS, command=draw_plot).pack(pady=10, fill=tk.X)
        tb.Button(left, text="Close", bootstyle=DANGER, command=dv.destroy).pack(pady=2, fill=tk.X)

    # --- Server admin (compact) ---
    def open_server_admin(self):
        if not self.active_conn:
            messagebox.showwarning("Not connected","Connect to DB first.")
            return
        win=tk.Toplevel(self); win.title("Server Admin Tools"); win.geometry("800x600")
        nb=ttk.Notebook(win); nb.pack(fill=tk.BOTH, expand=True)
        f1=ttk.Frame(nb); nb.add(f1,text="Users")
        f2=ttk.Frame(nb); nb.add(f2,text="Variables")
        f3=ttk.Frame(nb); nb.add(f3,text="Processlist")
        f4=ttk.Frame(nb); nb.add(f4,text="SQL")
        tv_users=ttk.Treeview(f1,show="headings"); tv_users.pack(fill=tk.BOTH,expand=True)
        try:
            cur=self.active_conn.cursor(); cur.execute("SELECT User,Host FROM mysql.user"); rows=cur.fetchall()
            if rows:
                tv_users["columns"]=list(rows[0].keys())
                for c in rows[0].keys(): tv_users.heading(c,text=c)
                for r in rows: tv_users.insert("",tk.END,values=list(r.values()))
        except Exception: pass
        tv_vars=ttk.Treeview(f2,show="headings"); tv_vars.pack(fill=tk.BOTH,expand=True)
        try:
            cur=self.active_conn.cursor(); cur.execute("SHOW VARIABLES"); rows=cur.fetchall()
            tv_vars["columns"]=["Variable_name","Value"]; tv_vars.heading("Variable_name",text="Variable_name"); tv_vars.heading("Value",text="Value")
            for r in rows: tv_vars.insert("",tk.END,values=list(r.values()))
        except Exception: pass
        tv_proc=ttk.Treeview(f3,show="headings"); tv_proc.pack(fill=tk.BOTH,expand=True)
        try:
            cur=self.active_conn.cursor(); cur.execute("SHOW PROCESSLIST"); rows=cur.fetchall()
            if rows:
                tv_proc["columns"]=list(rows[0].keys())
                for c in rows[0].keys(): tv_proc.heading(c,text=c)
                for r in rows: tv_proc.insert("",tk.END,values=list(r.values()))
        except Exception: pass
        txt_sql=SQLEditor(f4,font=self.editor_font,dark=self.dark_mode); txt_sql.pack(fill=tk.BOTH, expand=True)
        def run_custom():
            raw=txt_sql.get().strip()
            if not raw: return
            try:
                cur=self.active_conn.cursor(); cur.execute(raw)
                rows=cur.fetchall() if cur.description else []
                messagebox.showinfo("SQL Result", f"{len(rows)} rows")
            except Exception as exc:
                messagebox.showerror("Error", str(exc))
        tb.Button(f4, text="Run", bootstyle="primary", command=run_custom).pack(pady=6)

    # --- Workspace Save / Restore ---
    def save_workspace(self):
        ws = {"queries": [ed.get() for ed in self.editors], "active": self.tab_control.index(self.tab_control.select())}
        try:
            with open(WORKSPACE_FILE,"w",encoding="utf-8") as f: json.dump(ws,f)
            messagebox.showinfo("Workspace","Workspace saved")
        except Exception as exc: messagebox.showerror("Error", str(exc))

    def restore_workspace(self):
        if not os.path.exists(WORKSPACE_FILE):
            return
        try:
            with open(WORKSPACE_FILE, "r", encoding="utf-8") as f:
                ws = json.load(f)

            # ✅ Clear all existing tabs safely
            for tab in self.tab_control.tabs():
                self.tab_control.forget(tab)
            self.editors.clear()

            # ✅ Restore queries into tabs
            for q in ws.get("queries", []):
                self.add_tab(initial_text=q)

            # ✅ Select the last active tab if still valid
            idx = ws.get("active", 0)
            if 0 <= idx < len(self.editors):
                self.tab_control.select(idx)

            messagebox.showinfo("Workspace", "Workspace restored")
        except Exception as exc:
            messagebox.showerror("Error", str(exc))

    def on_closing(self):
        if messagebox.askokcancel("Quit","Do you want to quit?"):
            try:
                if self.active_conn: self.active_conn.close()
            except Exception: pass
            self.destroy()


# ---- Password encryption helpers (optional) ----
def encrypt_password(pw):
    if not _HAS_CRYPTO: return pw
    keyfile="secret.key"
    if not os.path.exists(keyfile):
        key=Fernet.generate_key()
        with open(keyfile,"wb") as f: f.write(key)
    else:
        with open(keyfile,"rb") as f: key=f.read()
    f=Fernet(key)
    return f.encrypt(pw.encode()).decode()

def decrypt_password(pw):
    if not _HAS_CRYPTO: return pw
    keyfile="secret.key"
    if not os.path.exists(keyfile): return pw
    with open(keyfile,"rb") as f: key=f.read()
    f=Fernet(key)
    try: return f.decrypt(pw.encode()).decode()
    except Exception: return pw


if __name__ == "__main__":
    # just create your app directly — it already is a tb.Window
    app = HRMSQueryGUI()   # will auto-pick theme based on system (darkly/flatly)
    
    # OR override with a custom theme explicitly:
    # app = HRMSQueryGUI(themename="cyborg")

    app.protocol("WM_DELETE_WINDOW", app.on_closing)
    app.mainloop()
