import json
import os
import re
import shutil
import subprocess
import sys
import webbrowser
import zipfile
from datetime import datetime
from pathlib import Path
import tkinter as tk
from tkinter import ttk, filedialog, messagebox


APP_NAME = "RDR2 Save Manager"
APP_AUTHOR = "Goneberg"
APP_COPYRIGHT = "© Goneberg 2026"
APP_VERSION = "1.4.9"

# User data is kept outside the application folder.
APP_DATA_DIR = Path.home() / "Documents" / "RDR2 Save Manager"
CONFIG_FILE = APP_DATA_DIR / "rdr2_config.json"
HISTORY_FILE = APP_DATA_DIR / "rdr2_history.json"
DEFAULT_DEST = Path.home() / "Documents" / "RDR2_Backups"
DEFAULT_GAME_EXE = Path("C:/Program Files (x86)/Red Dead Redemption 2/RDR2.exe")

NEXUS_URL = "https://www.nexusmods.com/reddeadredemption2/mods/10566"
DISCORD_NAME = "Goneji"

CHAPTER_OPTIONS = [
    "Chapter 1 - Colter",
    "Chapter 2 - Horseshoe Overlook",
    "Chapter 3 - Clemens Point",
    "Chapter 4 - Shady Belle",
    "Chapter 5 - Guarma",
    "Chapter 6 - Beaver Hollow",
    "Epilogue 1 - Pronghorn Ranch",
    "Epilogue 2 - Beecher's Hope",
]
PLACEHOLDER_TEXT = "e.g. Last mission: Pouring Forth Oil, 100% Gold..."

# Neutral, high-contrast palette.
BG = "#101010"
PANEL = "#171717"
PANEL_2 = "#1d1d1d"
FIELD = "#242424"
WHITE = "#f2f2f2"
MUTED = "#a7a7a7"
SILVER = "#d8d8d8"
BORDER = "#4a4a4a"
ACCENT = "#e8e8e8"
DANGER = "#c95b5b"
SUCCESS = "#79c98b"


class RDR2SaveManager:
    def __init__(self, root):
        self.root = root
        self.busy = False
        self.root.title(f"{APP_NAME} — Version {APP_VERSION}")
        self.root.geometry("1020x820")
        self.root.minsize(900, 720)
        self.root.configure(bg=BG)
        self.root.protocol("WM_DELETE_WINDOW", self.on_close)

        self.source_dir, self.dest_dir, self.game_exe = self.load_config()
        self.history = self.load_history()

        self.setup_style()
        self.setup_ui()

        if not self.source_dir.exists() or not self.has_valid_save_files(self.source_dir):
            self.auto_find_saves(silent=True)

        self.refresh_backup_list()
        self.set_status("Ready", "normal")

    # ---------- Configuration ----------

    @staticmethod
    def default_save_candidates():
        home = Path.home()
        documents = home / "Documents"
        one_drive = home / "OneDrive" / "Documents"
        candidates = [
            documents / "Rockstar Games" / "Red Dead Redemption 2" / "Profiles",
            one_drive / "Rockstar Games" / "Red Dead Redemption 2" / "Profiles",
            documents / "Rockstar Games" / "Red Dead Redemption 2",
            one_drive / "Rockstar Games" / "Red Dead Redemption 2",
        ]
        # De-duplicate while preserving order.
        return list(dict.fromkeys(candidates))

    @staticmethod
    def has_valid_save_files(folder):
        if not folder or not Path(folder).exists():
            return False
        base = Path(folder)
        try:
            if base.is_file():
                return base.name.lower().startswith("srdr")
            for root, _, files in os.walk(base):
                if any(name.lower().startswith("srdr") for name in files):
                    return True
        except OSError:
            return False
        return False

    @staticmethod
    def count_save_files(folder):
        count = 0
        base = Path(folder)
        try:
            for root, _, files in os.walk(base):
                count += sum(name.lower().startswith("srdr") for name in files)
        except OSError:
            return 0
        return count

    def load_config(self):
        source = self.default_save_candidates()[0]
        dest = DEFAULT_DEST
        game = DEFAULT_GAME_EXE
        try:
            if CONFIG_FILE.exists():
                with CONFIG_FILE.open("r", encoding="utf-8") as f:
                    data = json.load(f)
                configured_source = data.get("source_dir")
                configured_dest = data.get("backup_dir")
                configured_game = data.get("game_exe")
                if configured_source and Path(configured_source).exists():
                    source = Path(configured_source)
                if configured_dest and Path(configured_dest).exists():
                    dest = Path(configured_dest)
                if configured_game and Path(configured_game).exists():
                    game = Path(configured_game)
        except (OSError, ValueError, TypeError):
            pass
        return source, dest, game

    def save_config(self):
        try:
            APP_DATA_DIR.mkdir(parents=True, exist_ok=True)
            with CONFIG_FILE.open("w", encoding="utf-8") as f:
                json.dump({
                    "source_dir": str(self.source_dir),
                    "backup_dir": str(self.dest_dir),
                    "game_exe": str(self.game_exe),
                }, f, indent=4)
        except OSError as exc:
            messagebox.showerror("Configuration Error", f"Could not save settings:\n\n{exc}")

    def load_history(self):
        try:
            if HISTORY_FILE.exists():
                with HISTORY_FILE.open("r", encoding="utf-8") as f:
                    data = json.load(f)
                return data if isinstance(data, dict) else {}
        except (OSError, ValueError, TypeError):
            pass
        return {}

    def save_history(self):
        try:
            APP_DATA_DIR.mkdir(parents=True, exist_ok=True)
            with HISTORY_FILE.open("w", encoding="utf-8") as f:
                json.dump(self.history, f, indent=4)
        except OSError as exc:
            messagebox.showerror("History Error", f"Could not save backup history:\n\n{exc}")

    def reset_settings(self):
        if self.busy:
            return
        if not messagebox.askyesno(
            "Reset Settings",
            "Reset save location, backup location and game path to defaults?\n\n"
            "Your existing backup ZIP files and backup history will NOT be deleted."
        ):
            return
        self.source_dir = self.default_save_candidates()[0]
        self.dest_dir = DEFAULT_DEST
        self.game_exe = DEFAULT_GAME_EXE
        self.save_config()
        self.update_path_labels()
        self.auto_find_saves(silent=True)
        self.refresh_backup_list()
        self.set_status("Settings reset to defaults.", "success")

    # ---------- Style / UI ----------

    def setup_style(self):
        style = ttk.Style(self.root)
        style.theme_use("clam")
        style.configure(".", background=BG, foreground=WHITE, font=("Segoe UI", 9))
        style.configure(
            "TButton", background=PANEL_2, foreground=WHITE,
            bordercolor=BORDER, lightcolor=BORDER, darkcolor=BORDER,
            padding=(12, 7), font=("Segoe UI", 9, "bold")
        )
        style.map(
            "TButton",
            background=[("active", "#303030"), ("disabled", "#191919")],
            foreground=[("disabled", "#666666")]
        )
        style.configure(
            "Accent.TButton", background="#eeeeee", foreground="#111111",
            bordercolor="#eeeeee", padding=(13, 8),
            font=("Segoe UI", 9, "bold")
        )
        style.map("Accent.TButton", background=[("active", "#ffffff")])
        style.configure(
            "Danger.TButton", background="#3a2020", foreground="#f0caca",
            bordercolor="#5c3030", padding=(11, 7)
        )
        style.map("Danger.TButton", background=[("active", "#512828")])
        style.configure(
            "Treeview", background="#151515", foreground="#e2e2e2",
            fieldbackground="#151515", rowheight=30,
            bordercolor=BORDER, lightcolor=BORDER, darkcolor=BORDER
        )
        style.map(
            "Treeview",
            background=[("selected", "#dddddd")],
            foreground=[("selected", "#111111")]
        )
        style.configure(
            "Treeview.Heading", background="#252525", foreground="#eeeeee",
            font=("Segoe UI", 9, "bold"), relief="flat", padding=(7, 6)
        )
        style.configure(
            "TCombobox", fieldbackground=FIELD, background=FIELD,
            foreground=WHITE, arrowcolor=WHITE
        )
        style.configure("TEntry", fieldbackground=FIELD, foreground=WHITE)

    def setup_ui(self):
        # Header
        header = tk.Frame(
            self.root, bg="#0c0c0c", height=84,
            highlightbackground=BORDER, highlightthickness=1
        )
        header.pack(fill="x", side="top")
        header.pack_propagate(False)

        brand = tk.Frame(header, bg="#0c0c0c")
        brand.pack(side="left", padx=24, fill="y")

        tk.Label(
            brand, text="RDR2 SAVE MANAGER", bg="#0c0c0c", fg=WHITE,
            font=("Segoe UI", 19, "bold")
        ).pack(anchor="w", pady=(13, 0))
        tk.Label(
            brand, text=f"Version {APP_VERSION}  •  {APP_AUTHOR}",
            bg="#0c0c0c", fg=MUTED, font=("Segoe UI", 9)
        ).pack(anchor="w")

        header_buttons = tk.Frame(header, bg="#0c0c0c")
        header_buttons.pack(side="right", padx=20, pady=20)
        ttk.Button(header_buttons, text="INFO", command=self.show_about).pack(side="left", padx=4)
        ttk.Button(header_buttons, text="SETTINGS", command=self.show_settings).pack(side="left", padx=4)

        # Main area
        outer = tk.Frame(
            self.root, bg=BG, highlightbackground=BORDER,
            highlightcolor=ACCENT, highlightthickness=1
        )
        outer.pack(fill="both", expand=True, padx=14, pady=12)

        content = tk.Frame(outer, bg=BG)
        content.pack(fill="both", expand=True, padx=12, pady=12)

        # Status / overview
        overview = tk.Frame(content, bg=PANEL_2, highlightbackground=BORDER, highlightthickness=1)
        overview.pack(fill="x", pady=(0, 10))

        self.lbl_stat_count = self.make_stat(overview, "BACKUPS", "0", 0)
        self.lbl_stat_size = self.make_stat(overview, "STORAGE", "0 MB", 1)
        self.lbl_stat_latest = self.make_stat(overview, "LAST BACKUP", "None", 2)
        self.lbl_stat_save = self.make_stat(overview, "SAVE FILES", "—", 3)

        # Paths
        paths = tk.LabelFrame(
            content, text=" SAVE & BACKUP LOCATIONS ",
            bg=PANEL, fg=SILVER, bd=1, relief="solid",
            highlightbackground=BORDER, highlightcolor=BORDER,
            padx=12, pady=8, font=("Segoe UI", 9, "bold")
        )
        paths.pack(fill="x", pady=(0, 10))

        self.lbl_source_path = self.path_row(paths, "Save location", 0, self.change_source_folder)
        self.lbl_dest_path = self.path_row(paths, "Backup location", 1, self.change_dest_folder)

        actions_row = tk.Frame(paths, bg=PANEL)
        actions_row.grid(row=2, column=0, columnspan=3, sticky="ew", pady=(8, 0))
        ttk.Button(actions_row, text="AUTO-DETECT SAVES", style="Accent.TButton",
                   command=lambda: self.auto_find_saves(silent=False)).pack(side="left")
        ttk.Button(actions_row, text="RESET SETTINGS",
                   command=self.reset_settings).pack(side="left", padx=7)
        ttk.Button(actions_row, text="OPEN BACKUP FOLDER",
                   command=self.open_dest_folder).pack(side="left")
        paths.columnconfigure(1, weight=1)

        # Create backup
        create = tk.LabelFrame(
            content, text=" CREATE BACKUP ",
            bg=PANEL, fg=SILVER, bd=1, relief="solid",
            highlightbackground=BORDER, padx=12, pady=8,
            font=("Segoe UI", 9, "bold")
        )
        create.pack(fill="x", pady=(0, 10))

        tk.Label(create, text="Chapter", bg=PANEL, fg=MUTED).grid(row=0, column=0, sticky="w")
        self.combo_chapter = ttk.Combobox(
            create, values=CHAPTER_OPTIONS, state="readonly", width=28
        )
        self.combo_chapter.current(1)
        self.combo_chapter.grid(row=1, column=0, sticky="w", padx=(0, 14), pady=(3, 0))

        tk.Label(create, text="Note / last mission", bg=PANEL, fg=MUTED).grid(
            row=0, column=1, sticky="w"
        )
        self.entry_note = tk.Entry(
            create, bg=FIELD, fg=MUTED, insertbackground=WHITE,
            relief="flat", font=("Segoe UI", 9)
        )
        self.entry_note.insert(0, PLACEHOLDER_TEXT)
        self.entry_note.bind("<FocusIn>", self.on_entry_click)
        self.entry_note.bind("<FocusOut>", self.on_entry_focus_out)
        self.entry_note.grid(row=1, column=1, sticky="ew", ipady=5, pady=(3, 0))

        ttk.Button(create, text="CREATE BACKUP", style="Accent.TButton",
                   command=self.create_backup).grid(
            row=1, column=2, padx=(12, 0), pady=(3, 0)
        )
        create.columnconfigure(1, weight=1)

        # Backup list
        list_frame = tk.LabelFrame(
            content, text=" SAVED BACKUPS ",
            bg=PANEL, fg=SILVER, bd=1, relief="solid",
            highlightbackground=BORDER, padx=10, pady=8,
            font=("Segoe UI", 9, "bold")
        )
        list_frame.pack(fill="both", expand=True)

        toolbar = tk.Frame(list_frame, bg=PANEL)
        toolbar.pack(fill="x", pady=(0, 8))

        tk.Label(toolbar, text="Search", bg=PANEL, fg=MUTED).pack(side="left")
        self.entry_search = tk.Entry(
            toolbar, bg=FIELD, fg=WHITE, insertbackground=WHITE,
            relief="flat", width=30
        )
        self.entry_search.pack(side="left", fill="x", expand=True, padx=8, ipady=4)
        self.entry_search.bind("<KeyRelease>", lambda _e: self.refresh_backup_list())

        ttk.Button(toolbar, text="REFRESH", command=self.refresh_backup_list).pack(side="right")

        columns = ("date", "chapter", "note", "size", "filename")
        self.tree = ttk.Treeview(list_frame, columns=columns, show="headings", selectmode="browse")
        headings = [
            ("date", "Date & Time", 145),
            ("chapter", "Chapter", 180),
            ("note", "Note / Mission", 240),
            ("size", "Size", 80),
            ("filename", "Backup File", 210),
        ]
        for col, heading, width in headings:
            self.tree.heading(col, text=heading)
            self.tree.column(col, width=width, anchor="w")
        self.tree.column("date", anchor="center")
        self.tree.column("size", anchor="center")

        scrollbar = ttk.Scrollbar(list_frame, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)
        self.tree.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")
        self.tree.bind("<Double-1>", lambda _e: self.restore_backup())

        # Bottom action bar
        action_bar = tk.Frame(content, bg=BG)
        action_bar.pack(fill="x", pady=(10, 0))
        ttk.Button(action_bar, text="RESTORE SELECTED", style="Accent.TButton",
                   command=self.restore_backup).pack(side="left")
        ttk.Button(action_bar, text="DELETE BACKUP", style="Danger.TButton",
                   command=self.delete_backup).pack(side="left", padx=7)
        ttk.Button(action_bar, text="LAUNCH RDR2",
                   command=self.launch_game).pack(side="left")

        self.status_var = tk.StringVar(value="Ready")
        status = tk.Frame(
            self.root, bg="#0b0b0b", height=30,
            highlightbackground=BORDER, highlightthickness=1
        )
        status.pack(fill="x", side="bottom")
        tk.Label(
            status, textvariable=self.status_var, bg="#0b0b0b",
            fg=MUTED, font=("Segoe UI", 8)
        ).pack(side="left", padx=14, pady=6)
        tk.Label(
            status, text=f"Data: {APP_DATA_DIR}", bg="#0b0b0b",
            fg="#777777", font=("Segoe UI", 8)
        ).pack(side="right", padx=14, pady=6)

    def make_stat(self, parent, label, value, column):
        frame = tk.Frame(parent, bg=PANEL_2)
        frame.grid(row=0, column=column, sticky="ew", padx=14, pady=9)
        parent.columnconfigure(column, weight=1)
        tk.Label(frame, text=label, bg=PANEL_2, fg="#888888",
                 font=("Segoe UI", 8, "bold")).pack(anchor="w")
        label_widget = tk.Label(frame, text=value, bg=PANEL_2, fg=WHITE,
                                font=("Segoe UI", 11, "bold"))
        label_widget.pack(anchor="w", pady=(2, 0))
        return label_widget

    def path_row(self, parent, label, row, command):
        tk.Label(parent, text=label, bg=PANEL, fg=MUTED,
                 font=("Segoe UI", 9, "bold")).grid(
            row=row, column=0, sticky="w", pady=4
        )
        value = tk.Label(parent, text="", bg=PANEL, fg=SILVER,
                         font=("Consolas", 8), anchor="w")
        value.grid(row=row, column=1, sticky="ew", padx=12, pady=4)
        ttk.Button(parent, text="CHANGE", command=command).grid(
            row=row, column=2, sticky="e", pady=2
        )
        return value

    def update_path_labels(self):
        self.lbl_source_path.config(text=str(self.source_dir))
        self.lbl_dest_path.config(text=str(self.dest_dir))

    def set_status(self, message, kind="normal"):
        self.status_var.set(message)
        color = {"success": SUCCESS, "error": "#e08a8a", "normal": MUTED}.get(kind, MUTED)
        for child in self.root.winfo_children():
            if isinstance(child, tk.Frame) and child.cget("bg") == "#0b0b0b":
                for item in child.winfo_children():
                    if isinstance(item, tk.Label) and item.cget("textvariable") == str(self.status_var):
                        item.configure(fg=color)

    # ---------- Save discovery ----------

    def auto_find_saves(self, silent=False):
        candidates = self.default_save_candidates()
        found = None

        for base in candidates:
            if not base.exists():
                continue
            try:
                if base.name.lower() == "profiles":
                    profiles = [
                        p for p in base.iterdir()
                        if p.is_dir() and self.has_valid_save_files(p)
                    ]
                    if profiles:
                        profiles.sort(key=lambda p: p.stat().st_mtime, reverse=True)
                        found = profiles[0]
                        break
                elif self.has_valid_save_files(base):
                    for root, _, files in os.walk(base):
                        if any(name.lower().startswith("srdr") for name in files):
                            found = Path(root)
                            break
                    if found:
                        break
            except OSError:
                continue

        if found:
            self.source_dir = found
            self.save_config()
            self.update_path_labels()
            count = self.count_save_files(found)
            self.lbl_stat_save.config(text=str(count))
            self.set_status(f"Save location found • {count} save file(s)", "success")
            if not silent:
                messagebox.showinfo(
                    "Save Location Found",
                    f"RDR2 save files were found here:\n\n{found}\n\n"
                    f"Detected save files: {count}"
                )
            return found

        self.lbl_stat_save.config(text="0")
        self.set_status("Save folder not found — use Change to select it manually.", "error")
        if not silent:
            messagebox.showwarning(
                "Save Location Not Found",
                "No valid RDR2 save folder was detected in the standard Documents locations.\n\n"
                "Use CHANGE beside Save location to select the folder containing your SRDR save files."
            )
        return None

    # ---------- Settings / info ----------

    def show_settings(self):
        dialog = tk.Toplevel(self.root)
        dialog.title("RDR2 Save Manager — Settings")
        dialog.geometry("560x330")
        dialog.resizable(False, False)
        dialog.configure(bg=PANEL)
        dialog.transient(self.root)
        dialog.grab_set()

        tk.Label(dialog, text="Settings", bg=PANEL, fg=WHITE,
                 font=("Segoe UI", 16, "bold")).pack(anchor="w", padx=22, pady=(18, 4))
        tk.Label(
            dialog,
            text="Your settings are stored in Documents\\RDR2 Save Manager.",
            bg=PANEL, fg=MUTED, font=("Segoe UI", 9)
        ).pack(anchor="w", padx=22, pady=(0, 15))

        box = tk.Frame(dialog, bg=PANEL_2, highlightbackground=BORDER, highlightthickness=1)
        box.pack(fill="x", padx=22, pady=4)

        tk.Label(box, text="Config file", bg=PANEL_2, fg=MUTED).grid(
            row=0, column=0, sticky="w", padx=12, pady=9
        )
        tk.Label(box, text=str(CONFIG_FILE), bg=PANEL_2, fg=SILVER,
                 font=("Consolas", 8)).grid(row=1, column=0, sticky="w", padx=12, pady=(0, 9))

        ttk.Button(dialog, text="OPEN DATA FOLDER",
                   command=lambda: self.open_path(APP_DATA_DIR)).pack(side="left", padx=22, pady=20)
        ttk.Button(dialog, text="RESET SETTINGS",
                   command=lambda: [self.reset_settings(), dialog.destroy()]).pack(
            side="left", padx=4, pady=20
        )
        ttk.Button(dialog, text="CLOSE", command=dialog.destroy).pack(
            side="right", padx=22, pady=20
        )

    def show_about(self):
        dialog = tk.Toplevel(self.root)
        dialog.title(f"About {APP_NAME}")
        dialog.geometry("520x390")
        dialog.resizable(False, False)
        dialog.configure(bg=PANEL)
        dialog.transient(self.root)
        dialog.grab_set()

        tk.Label(dialog, text=APP_NAME, bg=PANEL, fg=WHITE,
                 font=("Segoe UI", 18, "bold")).pack(pady=(24, 3))
        tk.Label(dialog, text=f"Version {APP_VERSION}  •  Developed by {APP_AUTHOR}",
                 bg=PANEL, fg=MUTED, font=("Segoe UI", 9)).pack()

        info = tk.Frame(dialog, bg=PANEL_2, highlightbackground=BORDER, highlightthickness=1)
        info.pack(fill="x", padx=30, pady=22)

        tk.Label(
            info,
            text="A simple backup and restore utility for Red Dead Redemption 2 save files.",
            bg=PANEL_2, fg=SILVER, wraplength=430, justify="left",
            font=("Segoe UI", 9)
        ).pack(anchor="w", padx=16, pady=(15, 8))

        tk.Label(
            info, text=f"Discord: {DISCORD_NAME}",
            bg=PANEL_2, fg=WHITE, font=("Segoe UI", 10, "bold")
        ).pack(anchor="w", padx=16, pady=5)

        tk.Label(
            info,
            text="Use the buttons below for the project page and Discord contact.",
            bg=PANEL_2, fg=MUTED, font=("Segoe UI", 8)
        ).pack(anchor="w", padx=16, pady=(0, 14))

        ttk.Button(
            dialog, text="OPEN NEXUS MOD PAGE", style="Accent.TButton",
            command=lambda: webbrowser.open(NEXUS_URL)
        ).pack(fill="x", padx=70, pady=5)

        ttk.Button(
            dialog, text="COPY DISCORD NAME",
            command=lambda: self.copy_to_clipboard(DISCORD_NAME)
        ).pack(fill="x", padx=70, pady=5)

        tk.Label(
            dialog,
            text="Unofficial utility. Not affiliated with Rockstar Games.",
            bg=PANEL, fg="#777777", font=("Segoe UI", 8)
        ).pack(pady=15)

    def copy_to_clipboard(self, value):
        self.root.clipboard_clear()
        self.root.clipboard_append(value)
        self.root.update()
        self.set_status(f"Copied: {value}", "success")

    # ---------- Folder/game actions ----------

    def change_source_folder(self):
        selected = filedialog.askdirectory(title="Select RDR2 Save Directory")
        if not selected:
            return
        selected_path = Path(selected)
        if not self.has_valid_save_files(selected_path):
            if not messagebox.askyesno(
                "No Save Files Detected",
                "No SRDR save files were detected in this folder.\n\nUse it anyway?"
            ):
                return
        self.source_dir = selected_path
        self.save_config()
        self.update_path_labels()
        self.lbl_stat_save.config(text=str(self.count_save_files(self.source_dir)))
        self.set_status("Save location updated.", "success")

    def change_dest_folder(self):
        selected = filedialog.askdirectory(title="Select Backup Destination")
        if selected:
            self.dest_dir = Path(selected)
            self.save_config()
            self.update_path_labels()
            self.refresh_backup_list()
            self.set_status("Backup location updated.", "success")

    def open_path(self, path):
        try:
            Path(path).mkdir(parents=True, exist_ok=True)
            os.startfile(str(path))
        except OSError as exc:
            messagebox.showerror("Open Folder Failed", str(exc))

    def open_dest_folder(self):
        self.open_path(self.dest_dir)

    def launch_game(self):
        if self.busy:
            return
        if self.game_exe.exists():
            try:
                os.startfile(str(self.game_exe))
                self.set_status("RDR2 launch requested.", "success")
            except OSError as exc:
                messagebox.showerror("Launch Failed", str(exc))
            return

        selected = filedialog.askopenfilename(
            title="Locate RDR2.exe",
            filetypes=[("RDR2 executable", "RDR2.exe"), ("Executable Files", "*.exe")]
        )
        if selected:
            self.game_exe = Path(selected)
            self.save_config()
            try:
                os.startfile(str(self.game_exe))
                self.set_status("RDR2 path saved and launch requested.", "success")
            except OSError as exc:
                messagebox.showerror("Launch Failed", str(exc))

    # ---------- Busy state ----------

    def set_busy(self, busy):
        self.busy = busy
        state = "disabled" if busy else "normal"
        for widget in self.root.winfo_children():
            self._set_buttons_recursive(widget, state)
        self.root.update_idletasks()

    def _set_buttons_recursive(self, widget, state):
        if isinstance(widget, (ttk.Button, tk.Button)):
            try:
                widget.configure(state=state)
            except tk.TclError:
                pass
        for child in widget.winfo_children():
            self._set_buttons_recursive(child, state)

    # ---------- Backup ----------

    def create_backup(self):
        if self.busy:
            return
        if not self.source_dir.exists() or not self.has_valid_save_files(self.source_dir):
            messagebox.showerror(
                "Save Folder Not Found",
                "The configured save directory does not contain valid RDR2 save files.\n\n"
                "Use AUTO-DETECT SAVES or CHANGE to select the correct folder."
            )
            return

        self.set_busy(True)
        try:
            self.dest_dir.mkdir(parents=True, exist_ok=True)
            chapter = self.combo_chapter.get()
            raw_note = self.entry_note.get().strip()
            note = "" if raw_note == PLACEHOLDER_TEXT else raw_note
            combined = f"{chapter} - {note}" if note else chapter
            safe_name = re.sub(r"[^a-zA-Z0-9_-]", "_", combined)

            now = datetime.now()
            timestamp = now.strftime("%Y-%m-%d %H:%M:%S")
            filename = f"RDR2_Backup_{now.strftime('%Y%m%d_%H%M%S')}_{safe_name[:25]}.zip"
            zip_path = self.dest_dir / filename

            file_count = 0
            with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as archive:
                for root, _, files in os.walk(self.source_dir):
                    for file_name in files:
                        file_path = Path(root) / file_name
                        archive.write(file_path, file_path.relative_to(self.source_dir))
                        file_count += 1

            with zipfile.ZipFile(zip_path, "r") as archive:
                bad_file = archive.testzip()
                if bad_file is not None:
                    raise zipfile.BadZipFile(f"Archive verification failed at: {bad_file}")

            size_mb = zip_path.stat().st_size / (1024 * 1024)
            self.history[filename] = {
                "date": timestamp,
                "chapter": chapter,
                "note": note or "None",
                "size": f"{size_mb:.2f} MB",
                "files": file_count,
                "verified": True,
            }
            self.save_history()
            self.entry_note.delete(0, tk.END)
            self.entry_note.insert(0, PLACEHOLDER_TEXT)
            self.entry_note.config(fg=MUTED)
            self.refresh_backup_list()

            self.set_status(
                f"Backup verified • {file_count} file(s) • {size_mb:.2f} MB", "success"
            )
            messagebox.showinfo(
                "Backup Created",
                f"Backup created and verified successfully.\n\n"
                f"Files: {file_count}\nSize: {size_mb:.2f} MB\n\n{zip_path}"
            )
        except (OSError, zipfile.BadZipFile, ValueError) as exc:
            messagebox.showerror("Backup Failed", str(exc))
            self.set_status("Backup failed.", "error")
        finally:
            self.set_busy(False)

    def refresh_backup_list(self):
        if not hasattr(self, "tree"):
            return
        for item in self.tree.get_children():
            self.tree.delete(item)
        if not self.dest_dir.exists():
            self.update_stats(0, 0, "None")
            return

        query = self.entry_search.get().lower().strip()
        try:
            zip_files = sorted(
                self.dest_dir.glob("RDR2_Backup_*.zip"),
                key=lambda item: item.stat().st_mtime,
                reverse=True
            )
        except OSError:
            zip_files = []

        total_bytes = 0
        latest = "None"

        for index, zip_file in enumerate(zip_files):
            try:
                stat = zip_file.stat()
            except OSError:
                continue

            filename = zip_file.name
            meta = self.history.get(filename, {})
            date = meta.get(
                "date", datetime.fromtimestamp(stat.st_mtime).strftime("%Y-%m-%d %H:%M:%S")
            )
            chapter = meta.get("chapter", "Unknown Chapter")
            note = meta.get("note", "None")
            size = meta.get("size", f"{stat.st_size / (1024 * 1024):.2f} MB")
            total_bytes += stat.st_size
            if index == 0:
                latest = date

            if query and not any(
                query in str(value).lower()
                for value in (date, chapter, note, filename)
            ):
                continue

            self.tree.insert(
                "", "end", values=(date, chapter, note, size, filename)
            )

        self.update_stats(len(zip_files), total_bytes, latest)
        self.lbl_stat_save.config(text=str(self.count_save_files(self.source_dir)))

    def update_stats(self, count, total_bytes, latest):
        self.lbl_stat_count.config(text=str(count))
        self.lbl_stat_size.config(text=f"{total_bytes / (1024 * 1024):.1f} MB")
        self.lbl_stat_latest.config(text=latest)

    # ---------- Restore/delete ----------

    @staticmethod
    def _safe_zip_members(archive):
        for name in archive.namelist():
            path = Path(name)
            if path.is_absolute() or ".." in path.parts:
                raise ValueError("The selected backup contains an unsafe file path.")

    def _make_restore_safety_backup(self):
        if not self.source_dir.exists() or not self.has_valid_save_files(self.source_dir):
            return None

        safety_dir = self.dest_dir / "_RestoreSafety"
        safety_dir.mkdir(parents=True, exist_ok=True)
        safety_zip = safety_dir / (
            f"BeforeRestore_{datetime.now().strftime('%Y%m%d_%H%M%S')}.zip"
        )
        with zipfile.ZipFile(safety_zip, "w", zipfile.ZIP_DEFLATED) as archive:
            for root, _, files in os.walk(self.source_dir):
                for file_name in files:
                    file_path = Path(root) / file_name
                    archive.write(file_path, file_path.relative_to(self.source_dir))
        return safety_zip

    def get_selected_backup(self):
        selection = self.tree.selection()
        if not selection:
            messagebox.showwarning("No Backup Selected", "Select a backup first.")
            return None
        filename = self.tree.item(selection[0])["values"][4]
        zip_path = self.dest_dir / filename
        if not zip_path.exists():
            messagebox.showerror("Missing Backup", "The selected backup file no longer exists.")
            return None
        return filename, zip_path

    def restore_backup(self):
        if self.busy:
            return
        selected = self.get_selected_backup()
        if not selected:
            return
        filename, zip_path = selected

        try:
            with zipfile.ZipFile(zip_path, "r") as archive:
                self._safe_zip_members(archive)
                members = archive.namelist()
                file_count = sum(1 for name in members if not name.endswith("/"))
                bad_file = archive.testzip()
                if bad_file:
                    raise zipfile.BadZipFile(f"Backup verification failed at: {bad_file}")
        except (OSError, zipfile.BadZipFile, ValueError) as exc:
            messagebox.showerror("Invalid Backup", str(exc))
            return

        if not messagebox.askyesno(
            "Confirm Restore",
            f"Restore this verified backup?\n\n{filename}\n\n"
            f"Files in backup: {file_count}\n\n"
            "A safety copy of the current save will be created first when possible.\n"
            "The current save folder will then be replaced by the backup."
        ):
            return

        self.set_busy(True)
        try:
            safety_zip = self._make_restore_safety_backup()

            with zipfile.ZipFile(zip_path, "r") as archive:
                if self.source_dir.exists():
                    shutil.rmtree(self.source_dir)
                self.source_dir.mkdir(parents=True, exist_ok=True)
                archive.extractall(self.source_dir)

            self.save_config()
            self.lbl_stat_save.config(text=str(self.count_save_files(self.source_dir)))
            self.set_status(f"Restore complete • {file_count} file(s)", "success")

            extra = f"\nSafety backup: {safety_zip}" if safety_zip else ""
            messagebox.showinfo(
                "Restore Complete",
                f"The selected save was restored successfully.\n\n"
                f"Files restored: {file_count}{extra}"
            )
        except (OSError, zipfile.BadZipFile, ValueError) as exc:
            messagebox.showerror("Restore Failed", str(exc))
            self.set_status("Restore failed.", "error")
        finally:
            self.set_busy(False)

    def delete_backup(self):
        if self.busy:
            return
        selected = self.get_selected_backup()
        if not selected:
            return
        filename, zip_path = selected
        if not messagebox.askyesno(
            "Delete Backup",
            f"Delete this backup permanently?\n\n{filename}"
        ):
            return
        try:
            zip_path.unlink(missing_ok=True)
            self.history.pop(filename, None)
            self.save_history()
            self.refresh_backup_list()
            self.set_status("Backup deleted.", "success")
        except OSError as exc:
            messagebox.showerror("Delete Failed", str(exc))

    # ---------- Helpers ----------

    def on_entry_click(self, _event):
        if self.entry_note.get() == PLACEHOLDER_TEXT:
            self.entry_note.delete(0, tk.END)
            self.entry_note.config(fg=WHITE)

    def on_entry_focus_out(self, _event):
        if not self.entry_note.get().strip():
            self.entry_note.insert(0, PLACEHOLDER_TEXT)
            self.entry_note.config(fg=MUTED)

    def on_close(self):
        if self.busy:
            messagebox.showwarning("Operation In Progress", "Please wait for the current operation to finish.")
            return
        self.root.destroy()


def main():
    root = tk.Tk()
    RDR2SaveManager(root)
    root.mainloop()


if __name__ == "__main__":
    main()
