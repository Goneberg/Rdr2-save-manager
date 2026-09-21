import json
import os
import re
import shutil
import subprocess
import sys
import zipfile
from datetime import datetime
from pathlib import Path
import tkinter as tk
from tkinter import ttk, filedialog, messagebox


APP_NAME = "RDR2 Save Manager"
APP_AUTHOR = "Goneberg"
APP_COPYRIGHT = "© Goneberg 2026"
APP_VERSION = "1.4"

DEFAULT_DEST = Path.home() / "Documents" / "RDR2_Backups"
DEFAULT_GAME_EXE = Path("C:/Program Files (x86)/Red Dead Redemption 2/RDR2.exe")

# Store settings beside the script, or beside the EXE when packaged.
APP_DIR = Path(sys.executable).resolve().parent if getattr(sys, "frozen", False) else Path(__file__).resolve().parent
CONFIG_FILE = APP_DIR / "rdr2_config.json"
HISTORY_FILE = APP_DIR / "rdr2_history.json"

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

PLACEHOLDER_TEXT = "e.g., Last mission: Pouring Forth Oil, 100% Gold..."


class RDR2SaveManager:
    def __init__(self, root):
        self.root = root
        self.root.title(f"{APP_NAME} — Version {APP_VERSION}")
        self.root.geometry("920x860")
        self.root.minsize(860, 760)
        self.root.configure(bg="#121212")

        self.source_dir, self.dest_dir, self.game_exe = self.load_config()
        self.history = self.load_history()

        self.setup_style()
        self.setup_ui()

        if not self.source_dir.exists():
            self.auto_find_saves(silent=True)

        self.refresh_backup_list()

    # ---------- Configuration ----------

    def load_config(self):
        source = Path.home() / "Documents" / "Rockstar Games" / "Red Dead Redemption 2" / "Profiles"
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
            CONFIG_FILE.parent.mkdir(parents=True, exist_ok=True)
            with CONFIG_FILE.open("w", encoding="utf-8") as f:
                json.dump(
                    {
                        "source_dir": str(self.source_dir),
                        "backup_dir": str(self.dest_dir),
                        "game_exe": str(self.game_exe),
                    },
                    f,
                    indent=4,
                )
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
            HISTORY_FILE.parent.mkdir(parents=True, exist_ok=True)
            with HISTORY_FILE.open("w", encoding="utf-8") as f:
                json.dump(self.history, f, indent=4)
        except OSError as exc:
            messagebox.showerror("History Error", f"Could not save backup history:\n\n{exc}")

    # ---------- UI ----------

    def setup_style(self):
        style = ttk.Style(self.root)
        style.theme_use("clam")

        style.configure(
            ".",
            background="#121212",
            foreground="#f0f0f0",
            font=("Segoe UI", 9),
        )
        style.configure(
            "Card.TLabelframe",
            background="#1a1818",
            bordercolor="#3a2222",
            relief="solid",
        )
        style.configure(
            "Card.TLabelframe.Label",
            background="#1a1818",
            foreground="#ff5252",
            font=("Segoe UI", 10, "bold"),
        )
        style.configure(
            "Red.TButton",
            background="#8b0000",
            foreground="#ffffff",
            font=("Segoe UI", 9, "bold"),
            borderwidth=0,
            padding=(10, 6),
        )
        style.map("Red.TButton", background=[("active", "#d32f2f")])

        style.configure(
            "Dark.TButton",
            background="#2a2424",
            foreground="#f0f0f0",
            borderwidth=1,
            bordercolor="#4a3333",
            padding=(9, 6),
        )
        style.map("Dark.TButton", background=[("active", "#3d3232")])

        style.configure(
            "TCombobox",
            fieldbackground="#262323",
            background="#332a2a",
            foreground="#ffffff",
            arrowcolor="#ff5252",
        )
        style.configure("TEntry", fieldbackground="#262323", foreground="#ffffff")
        style.configure(
            "Treeview",
            background="#161414",
            foreground="#e0e0e0",
            fieldbackground="#161414",
            rowheight=28,
        )
        style.configure(
            "Treeview.Heading",
            background="#2b1a1a",
            foreground="#ff5252",
            font=("Segoe UI", 9, "bold"),
            relief="flat",
        )
        style.map(
            "Treeview",
            background=[("selected", "#d32f2f")],
            foreground=[("selected", "#ffffff")],
        )

    def setup_ui(self):
        # Simple header. No images, banners, sounds, or external assets.
        header = tk.Frame(self.root, bg="#8b0000", height=76)
        header.pack(fill="x", side="top")
        header.pack_propagate(False)

        title_box = tk.Frame(header, bg="#8b0000")
        title_box.pack(side="left", padx=22)

        tk.Label(
            title_box,
            text=APP_NAME,
            bg="#8b0000",
            fg="#ffffff",
            font=("Segoe UI", 18, "bold"),
        ).pack(anchor="e", pady=(10, 0))

        tk.Label(
            title_box,
            text=f"Public  •  {APP_COPYRIGHT}",
            bg="#8b0000",
            fg="#ffcdd2",
            font=("Segoe UI", 8),
        ).pack(anchor="e", pady=(0, 8))

        container = tk.Frame(self.root, bg="#121212")
        container.pack(fill="both", expand=True, padx=15, pady=10)

        stats = tk.Frame(
            container,
            bg="#1e1818",
            highlightbackground="#3d2222",
            highlightthickness=1,
        )
        stats.pack(fill="x", pady=(0, 8), ipady=5)

        self.lbl_stat_count = tk.Label(
            stats, text="Total Backups: 0", bg="#1e1818",
            fg="#ff8a80", font=("Segoe UI", 9, "bold")
        )
        self.lbl_stat_count.pack(side="left", padx=20)

        self.lbl_stat_size = tk.Label(
            stats, text="Storage Used: 0 MB", bg="#1e1818",
            fg="#ff8a80", font=("Segoe UI", 9, "bold")
        )
        self.lbl_stat_size.pack(side="left", padx=20)

        self.lbl_stat_latest = tk.Label(
            stats, text="Last Backup: None", bg="#1e1818",
            fg="#e0e0e0", font=("Segoe UI", 9)
        )
        self.lbl_stat_latest.pack(side="right", padx=20)

        # Paths
        path_frame = ttk.LabelFrame(
            container, text=" Settings & Paths ", style="Card.TLabelframe", padding=10
        )
        path_frame.pack(fill="x", pady=(0, 8))

        row1 = tk.Frame(path_frame, bg="#1a1818")
        row1.pack(fill="x", pady=2)
        tk.Label(
            row1, text="RDR2 Save Location:", bg="#1a1818",
            fg="#e0e0e0", font=("Segoe UI", 9, "bold")
        ).grid(row=0, column=0, sticky="w")

        self.lbl_source_path = tk.Label(
            row1, text=str(self.source_dir), font=("Consolas", 8),
            bg="#1a1818", fg="#ffb74d", anchor="w"
        )
        self.lbl_source_path.grid(row=0, column=1, sticky="ew", padx=10)

        ttk.Button(
            row1, text="Auto-Find Saves", style="Red.TButton",
            command=lambda: self.auto_find_saves(silent=False)
        ).grid(row=0, column=2, sticky="e", padx=(0, 4))

        ttk.Button(
            row1, text="Browse Folder...", style="Dark.TButton",
            command=self.change_source_folder
        ).grid(row=0, column=3, sticky="e")

        row2 = tk.Frame(path_frame, bg="#1a1818")
        row2.pack(fill="x", pady=(6, 2))
        tk.Label(
            row2, text="Backup Folder:", bg="#1a1818",
            fg="#e0e0e0", font=("Segoe UI", 9, "bold")
        ).grid(row=0, column=0, sticky="w")

        self.lbl_dest_path = tk.Label(
            row2, text=str(self.dest_dir), font=("Consolas", 8),
            bg="#1a1818", fg="#4fc3f7", anchor="w"
        )
        self.lbl_dest_path.grid(row=0, column=1, sticky="ew", padx=10)

        ttk.Button(
            row2, text="Change Backup Folder", style="Dark.TButton",
            command=self.change_dest_folder
        ).grid(row=0, column=2, sticky="e")

        ttk.Button(
            row2, text="Open Folder", style="Dark.TButton",
            command=self.open_dest_folder
        ).grid(row=0, column=3, sticky="e", padx=(4, 0))

        row1.columnconfigure(1, weight=1)
        row2.columnconfigure(1, weight=1)

        # Create backup
        backup_frame = ttk.LabelFrame(
            container, text=" Create New Backup ", style="Card.TLabelframe", padding=10
        )
        backup_frame.pack(fill="x", pady=5)

        grid = tk.Frame(backup_frame, bg="#1a1818")
        grid.pack(fill="x")

        tk.Label(
            grid, text="Select Chapter:", bg="#1a1818", fg="#e0e0e0"
        ).grid(row=0, column=0, sticky="w", pady=2)

        self.combo_chapter = ttk.Combobox(
            grid, values=CHAPTER_OPTIONS, state="readonly", width=30
        )
        self.combo_chapter.current(1)
        self.combo_chapter.grid(row=1, column=0, sticky="w", padx=(0, 15), pady=(0, 5))

        tk.Label(
            grid, text="Custom Note / Last Mission Done:",
            bg="#1a1818", fg="#e0e0e0"
        ).grid(row=0, column=1, sticky="w", pady=2)

        self.entry_note = tk.Entry(
            grid, bg="#262323", fg="#777777", insertbackground="white",
            relief="flat", font=("Segoe UI", 9)
        )
        self.entry_note.insert(0, PLACEHOLDER_TEXT)
        self.entry_note.bind("<FocusIn>", self.on_entry_click)
        self.entry_note.bind("<FocusOut>", self.on_entry_focus_out)
        self.entry_note.grid(row=1, column=1, sticky="ew", ipady=3, pady=(0, 5))
        grid.columnconfigure(1, weight=1)

        ttk.Button(
            backup_frame, text="CREATE BACKUP", style="Red.TButton",
            command=self.create_backup
        ).pack(anchor="e", pady=(8, 0))

        # Backup list
        list_frame = ttk.LabelFrame(
            container, text=" Saved Backups ", style="Card.TLabelframe", padding=10
        )
        list_frame.pack(fill="both", expand=True, pady=8)

        search_frame = tk.Frame(list_frame, bg="#1a1818")
        search_frame.pack(fill="x", pady=(0, 8))

        tk.Label(
            search_frame, text="Filter:", bg="#1a1818",
            fg="#ff8a80", font=("Segoe UI", 9, "bold")
        ).pack(side="left", padx=(0, 5))

        self.entry_search = tk.Entry(
            search_frame, bg="#262323", fg="#ffffff",
            insertbackground="white", relief="flat"
        )
        self.entry_search.pack(side="left", fill="x", expand=True, ipady=2)
        self.entry_search.bind("<KeyRelease>", lambda _event: self.refresh_backup_list())

        columns = ("date", "chapter", "note", "size", "filename")
        self.tree = ttk.Treeview(
            list_frame, columns=columns, show="headings", selectmode="browse"
        )
        for col, heading in [
            ("date", "Date & Time"),
            ("chapter", "Chapter"),
            ("note", "Note / Mission"),
            ("size", "Size"),
            ("filename", "Zip File"),
        ]:
            self.tree.heading(col, text=heading)

        self.tree.column("date", width=140, anchor="center")
        self.tree.column("chapter", width=180, anchor="w")
        self.tree.column("note", width=220, anchor="w")
        self.tree.column("size", width=70, anchor="center")
        self.tree.column("filename", width=160, anchor="w")

        scrollbar = ttk.Scrollbar(list_frame, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)
        self.tree.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")

        # Actions
        actions = tk.Frame(container, bg="#121212")
        actions.pack(fill="x", pady=5)

        ttk.Button(
            actions, text="Restore Selected", style="Red.TButton",
            command=self.restore_backup
        ).pack(side="left", padx=(0, 5))

        ttk.Button(
            actions, text="Delete Backup", style="Dark.TButton",
            command=self.delete_backup
        ).pack(side="left", padx=(0, 5))

        ttk.Button(
            actions, text="Launch RDR2", style="Dark.TButton",
            command=self.launch_game
        ).pack(side="left")

        ttk.Button(
            actions, text="Refresh List", style="Dark.TButton",
            command=self.refresh_backup_list
        ).pack(side="right")

        # Footer
        footer = tk.Frame(self.root, bg="#0a0a0a", height=34)
        footer.pack(fill="x", side="bottom")

        tk.Label(
            footer,
            text=f"{APP_NAME}  •  {APP_COPYRIGHT}  •  {APP_AUTHOR}",
            font=("Segoe UI", 8, "bold"),
            bg="#0a0a0a",
            fg="#888888",
        ).pack(side="left", padx=15, pady=7)

        tk.Button(
            footer, text="About", font=("Segoe UI", 8, "bold"),
            bg="#1a1a1a", fg="#ff5252", bd=0, padx=8,
            command=self.show_about
        ).pack(side="right", padx=10, pady=4)

    # ---------- Save discovery ----------

    def auto_find_saves(self, silent=False):
        home = Path.home()
        appdata = Path(os.environ.get("APPDATA", ""))

        candidates = [
            home / "Documents" / "Rockstar Games" / "Red Dead Redemption 2" / "Profiles",
            home / "OneDrive" / "Documents" / "Rockstar Games" / "Red Dead Redemption 2" / "Profiles",
            appdata / ".1911" / "Red Dead Redemption 2",
            appdata / "Goldberg SocialClub Emulator" / "Saves" / "Red Dead Redemption 2",
            home / "Documents" / "Rockstar Games" / "Red Dead Redemption 2",
        ]

        found = None
        for base in candidates:
            if not base.exists():
                continue

            try:
                if base.name.lower() == "profiles":
                    for profile in base.iterdir():
                        if profile.is_dir() and any(
                            f.is_file() and f.name.lower().startswith("srdr")
                            for f in profile.iterdir()
                        ):
                            found = profile
                            break
                else:
                    for root, _, files in os.walk(base):
                        if any(name.lower().startswith("srdr") for name in files):
                            found = Path(root)
                            break
            except OSError:
                continue

            if found:
                break

        if found:
            self.source_dir = found
            self.lbl_source_path.config(text=str(found))
            self.save_config()
            if not silent:
                messagebox.showinfo(
                    "Save Location Found",
                    f"RDR2 saves were located at:\n\n{found}"
                )
        elif not silent:
            messagebox.showwarning(
                "Not Found",
                "Could not automatically detect RDR2 save files.\n\n"
                "Please browse to the save folder manually."
            )

    # ---------- Folder/game actions ----------

    def change_source_folder(self):
        selected = filedialog.askdirectory(title="Select RDR2 Save Directory")
        if selected:
            self.source_dir = Path(selected)
            self.lbl_source_path.config(text=str(self.source_dir))
            self.save_config()

    def change_dest_folder(self):
        selected = filedialog.askdirectory(title="Select Backup Destination")
        if selected:
            self.dest_dir = Path(selected)
            self.lbl_dest_path.config(text=str(self.dest_dir))
            self.save_config()
            self.refresh_backup_list()

    def open_dest_folder(self):
        try:
            self.dest_dir.mkdir(parents=True, exist_ok=True)
            os.startfile(str(self.dest_dir))
        except OSError as exc:
            messagebox.showerror("Open Folder Failed", str(exc))

    def launch_game(self):
        if self.game_exe.exists():
            try:
                os.startfile(str(self.game_exe))
            except OSError as exc:
                messagebox.showerror("Launch Failed", str(exc))
            return

        selected = filedialog.askopenfilename(
            title="Locate RDR2.exe",
            filetypes=[("RDR2 executable", "RDR2.exe"), ("Executable Files", "*.exe")],
        )
        if selected:
            self.game_exe = Path(selected)
            self.save_config()
            try:
                os.startfile(str(self.game_exe))
            except OSError as exc:
                messagebox.showerror("Launch Failed", str(exc))

    # ---------- Backup ----------

    def create_backup(self):
        if not self.source_dir.exists():
            messagebox.showerror(
                "Save Folder Not Found",
                f"The configured save directory does not exist:\n\n{self.source_dir}"
            )
            return

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

            size_mb = zip_path.stat().st_size / (1024 * 1024)
            self.history[filename] = {
                "date": timestamp,
                "chapter": chapter,
                "note": note or "None",
                "size": f"{size_mb:.2f} MB",
                "files": file_count,
            }
            self.save_history()

            self.entry_note.delete(0, tk.END)
            self.entry_note.insert(0, PLACEHOLDER_TEXT)
            self.entry_note.config(fg="#777777")
            self.refresh_backup_list()

            messagebox.showinfo("Backup Created", "Backup created successfully.")
        except (OSError, zipfile.BadZipFile, ValueError) as exc:
            messagebox.showerror("Backup Failed", str(exc))

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
                reverse=True,
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
                "date",
                datetime.fromtimestamp(stat.st_mtime).strftime("%Y-%m-%d %H:%M:%S"),
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

    def update_stats(self, count, total_bytes, latest):
        self.lbl_stat_count.config(text=f"Total Backups: {count}")
        self.lbl_stat_size.config(
            text=f"Storage Used: {total_bytes / (1024 * 1024):.1f} MB"
        )
        self.lbl_stat_latest.config(text=f"Last Backup: {latest}")

    # ---------- Restore/delete ----------

    @staticmethod
    def _safe_zip_members(archive):
        for name in archive.namelist():
            path = Path(name)
            if path.is_absolute() or ".." in path.parts:
                raise ValueError("The selected backup contains an unsafe file path.")

    def _make_restore_safety_backup(self):
        if not self.source_dir.exists():
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

    def restore_backup(self):
        selection = self.tree.selection()
        if not selection:
            messagebox.showwarning("No Backup Selected", "Select a backup to restore.")
            return

        filename = self.tree.item(selection[0])["values"][4]
        zip_path = self.dest_dir / filename
        if not zip_path.exists():
            messagebox.showerror("Missing Backup", "The selected backup file no longer exists.")
            return

        if not messagebox.askyesno(
            "Confirm Restore",
            f"Restore the save from:\n\n{filename}\n\n"
            "A safety copy of the current save will be created first."
        ):
            return

        try:
            with zipfile.ZipFile(zip_path, "r") as archive:
                self._safe_zip_members(archive)
                self._make_restore_safety_backup()

                if self.source_dir.exists():
                    shutil.rmtree(self.source_dir)
                self.source_dir.mkdir(parents=True, exist_ok=True)
                archive.extractall(self.source_dir)

            messagebox.showinfo("Restore Complete", "The selected save was restored successfully.")
        except (OSError, zipfile.BadZipFile, ValueError) as exc:
            messagebox.showerror("Restore Failed", str(exc))

    def delete_backup(self):
        selection = self.tree.selection()
        if not selection:
            messagebox.showwarning("No Backup Selected", "Select a backup to delete.")
            return

        filename = self.tree.item(selection[0])["values"][4]
        if not messagebox.askyesno("Delete Backup", f"Delete this backup?\n\n{filename}"):
            return

        try:
            (self.dest_dir / filename).unlink(missing_ok=True)
            self.history.pop(filename, None)
            self.save_history()
            self.refresh_backup_list()
        except OSError as exc:
            messagebox.showerror("Delete Failed", str(exc))

    # ---------- Small UI helpers ----------

    def on_entry_click(self, _event):
        if self.entry_note.get() == PLACEHOLDER_TEXT:
            self.entry_note.delete(0, tk.END)
            self.entry_note.config(fg="#ffffff")

    def on_entry_focus_out(self, _event):
        if not self.entry_note.get().strip():
            self.entry_note.insert(0, PLACEHOLDER_TEXT)
            self.entry_note.config(fg="#777777")

    def show_about(self):
        messagebox.showinfo(
            f"About {APP_NAME}",
            f"{APP_NAME}\n\n"
            f"Developed by {APP_AUTHOR}\n"
            f"{APP_COPYRIGHT}. All rights reserved.\n\n"
            "Unofficial utility. Not affiliated with Rockstar Games.",
        )


def main():
    root = tk.Tk()
    RDR2SaveManager(root)
    root.mainloop()


if __name__ == "__main__":
    main()
