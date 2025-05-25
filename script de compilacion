import gi
gi.require_version("Gtk", "3.0")
from gi.repository import Gtk, GdkPixbuf, GLib, GObject, Pango
import requests
import json
import os
import base64
import tempfile
import threading

CONFIG_FILE = "config.json"
ICON_CACHE_DIR = ".icons_cache"
DEFAULT_ICON_PATH = "default_icon.png"
VERSION_FILE = "version.json"

def ensure_icon_cache_dir():
    if not os.path.exists(ICON_CACHE_DIR):
        os.makedirs(ICON_CACHE_DIR)

def get_temp_file_path():
    temp_dir = tempfile.gettempdir()
    return os.path.join(temp_dir, "json_programs_autosave.list")

def load_app_icon():
    base_dir = os.path.dirname(os.path.abspath(__file__))
    icon_path = os.path.join(base_dir, "assets", "zpm.png")
    if os.path.exists(icon_path):
        try:
            pixbuf = GdkPixbuf.Pixbuf.new_from_file(icon_path)
            return pixbuf
        except Exception as e:
            print(f"No se pudo cargar el icono desde {icon_path}: {e}")
    else:
        print(f"Icono no encontrado en la ruta: {icon_path}")
    return None

class ImageDownloader(threading.Thread):
    def __init__(self, url, callback):
        super().__init__()
        self.url = url
        self.callback = callback

    def run(self):
        ensure_icon_cache_dir()
        try:
            local_name = os.path.basename(self.url)
            if not local_name or '.' not in local_name:
                local_name = f"icon_{abs(hash(self.url))}.png"
            local_path = os.path.join(ICON_CACHE_DIR, local_name)
            if not os.path.exists(local_path):
                r = requests.get(self.url, stream=True, timeout=10)
                r.raise_for_status()
                with open(local_path, 'wb') as f:
                    for chunk in r.iter_content(1024):
                        f.write(chunk)
            GLib.idle_add(self.callback, self.url, local_path)
        except Exception:
            GLib.idle_add(self.callback, self.url, None)

class UploadThread(threading.Thread):
    def __init__(self, api_url, headers, data_to_send, progress_callback, finished_callback):
        super().__init__()
        self.api_url = api_url
        self.headers = headers
        self.data_to_send = data_to_send
        self.progress_callback = progress_callback
        self.finished_callback = finished_callback

    def run(self):
        try:
            self.progress_callback(50)
            r = requests.put(self.api_url, headers=self.headers, json=self.data_to_send)
            if r.status_code in [200, 201]:
                self.progress_callback(100)
                self.finished_callback(True, "Archivo subido/modificado correctamente en GitHub.")
            else:
                self.finished_callback(False, f"Error al subir archivo:\n{r.status_code}\n{r.text}")
        except Exception as e:
            self.finished_callback(False, f"No se pudo subir el archivo:\n{e}")

class ProgramCard(Gtk.Box):
    __gsignals__ = {
        'changed': (GObject.SignalFlags.RUN_FIRST, None, ()),
        'deleted': (GObject.SignalFlags.RUN_FIRST, None, ())
    }
    def __init__(self, data):
        super().__init__(orientation=Gtk.Orientation.HORIZONTAL, spacing=10)
        self.data = data
        self.set_border_width(10)
        self._pixbuf = None

        self.image = Gtk.Image()
        self.image.set_pixel_size(64)
        self.pack_start(self.image, False, False, 0)

        self.name_label = Gtk.Label(label="", use_markup=True, xalign=0)
        self.version_label = Gtk.Label(label="", xalign=0)
        self.description_label = Gtk.Label(label="", xalign=0, wrap=True)
        vbox = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=5)
        vbox.pack_start(self.name_label, False, False, 0)
        vbox.pack_start(self.version_label, False, False, 0)
        vbox.pack_start(self.description_label, False, False, 0)
        self.pack_start(vbox, True, True, 0)

        btn_box = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=5)
        self.pack_start(btn_box, False, False, 0)

        self.edit_button = Gtk.Button(label="Editar")
        self.delete_button = Gtk.Button(label="Eliminar")
        btn_box.pack_start(self.edit_button, False, False, 0)
        btn_box.pack_start(self.delete_button, False, False, 0)

        self.edit_button.connect("clicked", self.on_edit_clicked)
        self.delete_button.connect("clicked", self.on_delete_clicked)

        self._icon_url = None
        self.update_display()

    def update_display(self):
        self.name_label.set_markup(f"<b>{self.data.get('nombre', '')}</b>")
        self.version_label.set_text(f"Versión: {self.data.get('version', '')}")
        desc = self.data.get('descripcion', '')
        desc_short = (desc[:100] + '...') if len(desc) > 100 else desc
        self.description_label.set_text(desc_short)
        icon_url = self.data.get("icono", "")

        if self._pixbuf and icon_url == self._icon_url:
            self.image.set_from_pixbuf(self._pixbuf)
            return

        if icon_url.startswith("http"):
            self.load_image_from_url(icon_url)
        else:
            self.load_image_from_local(icon_url)

    def load_image_from_url(self, url):
        self.image.set_from_icon_name("system-run", Gtk.IconSize.DIALOG)
        if url == self._icon_url:
            return
        self._icon_url = url
        def callback(u, local_path):
            if local_path:
                try:
                    pixbuf = GdkPixbuf.Pixbuf.new_from_file_at_scale(local_path, 64, 64, True)
                    self._pixbuf = pixbuf
                    self.image.set_from_pixbuf(pixbuf)
                except:
                    self._pixbuf = None
                    self.image.set_from_icon_name("image-missing", Gtk.IconSize.DIALOG)
            else:
                self._pixbuf = None
                self.image.set_from_icon_name("image-missing", Gtk.IconSize.DIALOG)
        downloader = ImageDownloader(url, callback)
        downloader.start()

    def load_image_from_local(self, path):
        if path and os.path.exists(path):
            try:
                pixbuf = GdkPixbuf.Pixbuf.new_from_file_at_scale(path, 64, 64, True)
                self._pixbuf = pixbuf
                self.image.set_from_pixbuf(pixbuf)
                return
            except:
                pass
        if os.path.exists(DEFAULT_ICON_PATH):
            try:
                pixbuf = GdkPixbuf.Pixbuf.new_from_file_at_scale(DEFAULT_ICON_PATH, 64, 64, True)
                self._pixbuf = pixbuf
                self.image.set_from_pixbuf(pixbuf)
                return
            except:
                pass
        self._pixbuf = None
        self.image.set_from_icon_name("image-missing", Gtk.IconSize.DIALOG)

    def on_edit_clicked(self, button):
        dlg = EditDialog(self.data, self.get_toplevel())
        response = dlg.run()
        if response == Gtk.ResponseType.OK:
            new_data = dlg.get_data()
            self.data = new_data
            self.update_display()
            self.emit('changed')
        dlg.destroy()

    def on_delete_clicked(self, button):
        md = Gtk.MessageDialog(parent=self.get_toplevel(), flags=0,
                               message_type=Gtk.MessageType.QUESTION,
                               buttons=Gtk.ButtonsType.YES_NO,
                               text=f"¿Eliminar programa '{self.data.get('nombre', '')}'?")
        response = md.run()
        md.destroy()
        if response == Gtk.ResponseType.YES:
            self.emit('deleted')

    def get_data(self):
        return self.data

class LinksBox(Gtk.Box):
    def __init__(self):
        super().__init__(orientation=Gtk.Orientation.VERTICAL, spacing=5)
        self.link_rows = []
        self.set_halign(Gtk.Align.FILL)

        self.links_container = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=5)
        self.pack_start(self.links_container, True, True, 0)

        self.add_link_button = Gtk.Button(label="Añadir enlace")
        self.add_link_button.connect("clicked", self.on_add_link_clicked)
        self.pack_start(self.add_link_button, False, False, 0)

    def on_add_link_clicked(self, button):
        dialog = Gtk.MessageDialog(
            transient_for=self.get_toplevel(),
            flags=0,
            message_type=Gtk.MessageType.QUESTION,
            buttons=Gtk.ButtonsType.OK_CANCEL,
            text="Añadir nuevo enlace de descarga o comando"
        )
        dialog.format_secondary_text("Introduce la URL o comando a añadir:")

        entry = Gtk.Entry()
        entry.set_activates_default(True)
        dialog.get_content_area().pack_end(entry, False, False, 0)

        dialog.set_default_size(500, 100)

        dialog.set_default_response(Gtk.ResponseType.OK)
        dialog.show_all()
        response = dialog.run()
        text = entry.get_text().strip()
        dialog.destroy()
        if response == Gtk.ResponseType.OK and text:
            self.add_link_row(url_text=text)

    def add_link_row(self, button=None, url_text=""):
        row = Gtk.Box(spacing=6)
        label = Gtk.Label(label=url_text, xalign=0)
        label.set_ellipsize(Pango.EllipsizeMode.END)
        label.set_hexpand(True)
        label.set_max_width_chars(80)

        delete_btn = Gtk.Button(label="Eliminar")
        delete_btn.set_size_request(80, -1)
        delete_btn.connect("clicked", lambda btn: self.remove_link_row(row))

        row.pack_start(label, True, True, 0)
        row.pack_start(delete_btn, False, False, 0)

        self.links_container.pack_start(row, False, False, 0)
        self.link_rows.append((row, label))
        self.links_container.show_all()

    def remove_link_row(self, row):
        for i, (w, label) in enumerate(self.link_rows):
            if w == row:
                self.links_container.remove(w)
                self.link_rows.pop(i)
                break

    def clear(self):
        for w, _ in self.link_rows:
            self.links_container.remove(w)
        self.link_rows.clear()

    def set_links(self, links):
        self.clear()
        for l in links:
            if l.strip():
                self.add_link_row(url_text=l.strip())

    def get_links(self):
        return [label.get_text() for _, label in self.link_rows]

class AddProgramDialog(Gtk.Dialog):
    def __init__(self, parent=None):
        super().__init__(title="Añadir nuevo programa", transient_for=parent, flags=0)
        self.set_default_size(700, 520)
        self.add_buttons(Gtk.STOCK_CANCEL, Gtk.ResponseType.CANCEL,
                         Gtk.STOCK_OK, Gtk.ResponseType.OK)

        box = self.get_content_area()
        grid = Gtk.Grid(column_spacing=10, row_spacing=10, margin=10)
        box.add(grid)

        self.nombre_entry = Gtk.Entry()
        self.nombre_entry.set_hexpand(True)

        self.version_entry = Gtk.Entry()
        self.version_entry.set_hexpand(True)

        self.categoria_combo = Gtk.ComboBoxText()
        categorias = ["Utilidades", "Desarrollo", "Multimedia", "Internet", "Juegos", "Otros"]
        for cat in categorias:
            self.categoria_combo.append_text(cat)
        self.categoria_combo.set_active(0)
        self.categoria_combo.set_hexpand(True)

        self.arquitectura_combo = Gtk.ComboBoxText()
        self.arquitectura_combo.append_text("amd64")
        self.arquitectura_combo.append_text("i386")
        self.arquitectura_combo.set_active(0)
        self.arquitectura_combo.set_hexpand(True)

        self.descripcion_buffer = Gtk.TextBuffer()
        self.descripcion_view = Gtk.TextView(buffer=self.descripcion_buffer)
        self.descripcion_view.set_size_request(-1, 150)
        self.descripcion_view.set_hexpand(True)
        self.descripcion_view.set_vexpand(True)

        self.icono_entry = Gtk.Entry()
        self.icono_entry.set_hexpand(True)

        self.links_box = LinksBox()
        self.links_box.set_hexpand(True)

        grid.attach(Gtk.Label(label="Nombre:"), 0, 0, 1, 1)
        grid.attach(self.nombre_entry, 1, 0, 1, 1)
        grid.attach(Gtk.Label(label="Versión:"), 0, 1, 1, 1)
        grid.attach(self.version_entry, 1, 1, 1, 1)
        grid.attach(Gtk.Label(label="Arquitectura:"), 0, 2, 1, 1)
        grid.attach(self.arquitectura_combo, 1, 2, 1, 1)
        grid.attach(Gtk.Label(label="Descripción:"), 0, 3, 1, 1)
        grid.attach(self.descripcion_view, 1, 3, 1, 1)
        grid.attach(Gtk.Label(label="Icono (URL o ruta):"), 0, 4, 1, 1)
        grid.attach(self.icono_entry, 1, 4, 1, 1)
        grid.attach(Gtk.Label(label="Categoría:"), 0, 5, 1, 1)
        grid.attach(self.categoria_combo, 1, 5, 1, 1)
        grid.attach(Gtk.Label(label="Enlaces de descarga:"), 0, 6, 1, 1)
        grid.attach(self.links_box, 1, 6, 1, 1)

        self.show_all()

    def get_data(self):
        text_start = self.descripcion_buffer.get_start_iter()
        text_end = self.descripcion_buffer.get_end_iter()
        descripcion = self.descripcion_buffer.get_text(text_start, text_end, True)

        return {
            "nombre": self.nombre_entry.get_text().strip(),
            "version": self.version_entry.get_text().strip(),
            "arquitectura": self.arquitectura_combo.get_active_text(),
            "descripcion": descripcion.strip(),
            "icono": self.icono_entry.get_text().strip(),
            "categoria": self.categoria_combo.get_active_text(),
            "enlace_descarga": self.links_box.get_links()
        }

class EditDialog(AddProgramDialog):
    def __init__(self, data, parent=None):
        super().__init__(parent)
        self.set_title(f"Editar programa: {data.get('nombre', '')}")
        self.nombre_entry.set_text(data.get("nombre", ""))
        self.version_entry.set_text(data.get("version", ""))
        arquitectura = data.get("arquitectura", "amd64")
        if arquitectura in ["amd64", "i386"]:
            self.arquitectura_combo.set_active(["amd64", "i386"].index(arquitectura))
        else:
            self.arquitectura_combo.set_active(0)
        self.descripcion_buffer.set_text(data.get("descripcion", ""))
        self.icono_entry.set_text(data.get("icono", ""))
        categoria = data.get("categoria", "")
        model = self.categoria_combo.get_model()
        found_index = -1
        for i, row in enumerate(model):
            if row[0] == categoria:
                found_index = i
                break
        self.categoria_combo.set_active(found_index if found_index >= 0 else 0)
        enlaces = data.get("enlace_descarga", [])
        if isinstance(enlaces, str):
            enlaces = [enlaces] if enlaces else []
        elif isinstance(enlaces, list):
            if len(enlaces) > 0 and isinstance(enlaces[0], dict):
                enlaces = [e.get("url", "") for e in enlaces if "url" in e]
        self.links_box.set_links(enlaces)

class SettingsDialog(Gtk.Dialog):
    def __init__(self, config, parent=None):
        super().__init__(title="Ajustes", transient_for=parent, flags=0)
        self.set_default_size(600, 360)
        self.add_buttons(Gtk.STOCK_CANCEL, Gtk.ResponseType.CANCEL,
                         Gtk.STOCK_SAVE, Gtk.ResponseType.OK)

        self.config = config.copy()

        box = self.get_content_area()
        grid = Gtk.Grid(column_spacing=10, row_spacing=10, margin=10)
        box.add(grid)

        self.username_entry = Gtk.Entry(text=self.config.get("username", ""))
        self.token_entry = Gtk.Entry(text=self.config.get("token", ""))
        self.token_entry.set_visibility(False)
        self.repo_combo = Gtk.ComboBoxText()
        self.repo_combo.set_sensitive(False)
        self.path_combo = Gtk.ComboBoxText()
        self.path_combo.set_sensitive(False)
        self.info_label = Gtk.Label(label="")

        grid.attach(Gtk.Label(label="Usuario GitHub:"), 0, 0, 1, 1)
        grid.attach(self.username_entry, 1, 0, 1, 1)
        grid.attach(Gtk.Label(label="Token GitHub:"), 0, 1, 1, 1)
        grid.attach(self.token_entry, 1, 1, 1, 1)
        grid.attach(Gtk.Label(label="Repositorio:"), 0, 2, 1, 1)
        grid.attach(self.repo_combo, 1, 2, 1, 1)
        grid.attach(Gtk.Label(label="Archivo JSON:"), 0, 3, 1, 1)
        grid.attach(self.path_combo, 1, 3, 1, 1)
        grid.attach(self.info_label, 0, 4, 2, 1)

        self.username_entry.connect("changed", self.on_username_or_token_changed)
        self.token_entry.connect("changed", self.on_username_or_token_changed)
        self.repo_combo.connect("changed", self.on_repo_changed)

        if self.username_entry.get_text() and self.token_entry.get_text():
            self.load_repos()

        self.show_all()

    def on_username_or_token_changed(self, widget):
        self.repo_combo.remove_all()
        self.repo_combo.set_sensitive(False)
        self.path_combo.remove_all()
        self.path_combo.set_sensitive(False)
        if self.username_entry.get_text() and self.token_entry.get_text():
            self.load_repos()

    def load_repos(self):
        self.info_label.set_text("Cargando repositorios...")
        def worker():
            try:
                headers = {"Authorization": f"token {self.token_entry.get_text()}"}
                url = f"https://api.github.com/users/{self.username_entry.get_text()}/repos?per_page=100"
                r = requests.get(url, headers=headers)
                r.raise_for_status()
                repos = r.json()
                GLib.idle_add(self.fill_repos, repos)
            except Exception as e:
                GLib.idle_add(self.info_label.set_text, f"Error al cargar repositorios: {e}")
        threading.Thread(target=worker, daemon=True).start()

    def fill_repos(self, repos):
        self.repo_combo.remove_all()
        for repo in repos:
            self.repo_combo.append_text(repo["full_name"])
        self.repo_combo.set_sensitive(True)
        self.info_label.set_text(f"{len(repos)} repositorios cargados.")

        current_repo = self.config.get("repo")
        if current_repo:
            model = self.repo_combo.get_model()
            for i, row in enumerate(model):
                if row[0] == current_repo:
                    self.repo_combo.set_active(i)
                    break
        else:
            self.repo_combo.set_active(0)

    def on_repo_changed(self, combo):
        repo = combo.get_active_text()
        if not repo:
            self.path_combo.remove_all()
            self.path_combo.set_sensitive(False)
            return
        self.load_repo_files(repo)

    def load_repo_files(self, repo_full_name):
        self.info_label.set_text("Cargando archivos JSON del repo...")
        def worker():
            try:
                headers = {"Authorization": f"token {self.token_entry.get_text()}"}
                url = f"https://api.github.com/repos/{repo_full_name}/contents?ref=main"
                r = requests.get(url, headers=headers)
                r.raise_for_status()
                files = r.json()
                json_files = [f["name"] for f in files if f["name"].endswith((".json", ".list")) and f["type"] == "file"]
                GLib.idle_add(self.fill_files, json_files)
            except Exception as e:
                GLib.idle_add(self.error_files, e)
        threading.Thread(target=worker, daemon=True).start()

    def fill_files(self, json_files):
        self.path_combo.remove_all()
        for f in json_files:
            self.path_combo.append_text(f)
        self.path_combo.set_sensitive(True)
        self.info_label.set_text(f"{len(json_files)} archivos JSON encontrados.")

        current_path = self.config.get("path")
        if current_path:
            model = self.path_combo.get_model()
            for i, row in enumerate(model):
                if row[0] == current_path:
                    self.path_combo.set_active(i)
                    break
        else:
            self.path_combo.set_active(0)

    def error_files(self, e):
        self.info_label.set_text(f"Error al cargar archivos JSON: {e}")
        self.path_combo.remove_all()
        self.path_combo.set_sensitive(False)

    def get_config(self):
        repo = self.repo_combo.get_active_text() if self.repo_combo.get_sensitive() else ""
        path = self.path_combo.get_active_text() if self.path_combo.get_sensitive() else ""
        url_json = f"https://raw.githubusercontent.com/{repo}/main/{path}" if repo and path else ""
        return {
            "username": self.username_entry.get_text().strip(),
            "token": self.token_entry.get_text().strip(),
            "repo": repo,
            "path": path,
            "url_json": url_json
        }

class AboutDialog(Gtk.Dialog):
    def __init__(self, parent=None):
        super().__init__(title="Acerca de", transient_for=parent, flags=0)
        self.set_default_size(320, 220)
        self.add_buttons(Gtk.STOCK_CLOSE, Gtk.ResponseType.CLOSE)

        icon_path = None
        try:
            with open(VERSION_FILE, "r", encoding="utf-8") as f:
                vdata = json.load(f)
                icono_val = vdata.get("icono", None)
                if icono_val:
                    rel_path = icono_val.lstrip("/")
                    if os.path.exists(rel_path):
                        icon_path = rel_path
        except Exception as e:
            print("Error cargando version.json:", e)

        if icon_path:
            try:
                pixbuf = GdkPixbuf.Pixbuf.new_from_file(icon_path)
                self.set_icon(pixbuf)
            except Exception as e:
                print(f"No se pudo establecer icono de ventana: {e}")

        box = self.get_content_area()
        box.set_spacing(10)
        box.set_border_width(10)

        app_name = "ZORN PACKAGE UPLOADER MANAGER"
        version = "desconocida"
        author = ""
        description = "ZORN PACKAGE UPLOADER MANAGER Administrador de paquetes Zpm"

        try:
            with open(VERSION_FILE, "r", encoding="utf-8") as f:
                vdata = json.load(f)
                app_name = vdata.get("App", app_name)
                version = vdata.get("version", version)
                author = vdata.get("author", author)
                description = vdata.get("description", description)
        except Exception:
            pass

        hbox = Gtk.Box(orientation=Gtk.Orientation.HORIZONTAL, spacing=10)
        box.pack_start(hbox, False, False, 0)

        if icon_path:
            try:
                pixbuf_small = GdkPixbuf.Pixbuf.new_from_file_at_scale(icon_path, 64, 64, True)
                image = Gtk.Image.new_from_pixbuf(pixbuf_small)
            except Exception as e:
                print(f"Error cargando imagen del icono: {e}")
                image = Gtk.Image()
        else:
            image = Gtk.Image()
        hbox.pack_start(image, False, False, 0)

        info_text = (
            f"<big><b>{GLib.markup_escape_text(app_name)}</b></big>\n"
            f"<b>Versión:</b> {GLib.markup_escape_text(version)}\n"
        )
        if author:
            info_text += f"<b>Autor:</b> {GLib.markup_escape_text(author)}\n"
        info_text += f"\n{GLib.markup_escape_text(description)}"

        label = Gtk.Label()
        label.set_markup(info_text)
        label.set_justify(Gtk.Justification.LEFT)
        label.set_line_wrap(True)
        label.set_hexpand(True)
        label.set_size_request(280, -1)
        hbox.pack_start(label, True, True, 0)

        self.show_all()

class JsonEditor(Gtk.Window):
    def __init__(self):
        super().__init__(title="ZORN PACKAGE UPLOADER MANAGER")

        pixbuf = load_app_icon()
        if pixbuf:
            self.set_icon(pixbuf)
        else:
            print("No se pudo cargar el icono para la ventana principal.")

        self.set_default_size(900, 760)
        self.set_position(Gtk.WindowPosition.CENTER)
        self.connect("destroy", Gtk.main_quit)

        self.config = self.load_config()
        self.json_data = []
        self.file_path = get_temp_file_path()

        main_vbox = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=6)
        main_vbox.set_margin_top(6)
        main_vbox.set_margin_bottom(6)
        main_vbox.set_margin_start(6)
        main_vbox.set_margin_end(6)
        self.add(main_vbox)

        search_box = Gtk.Box(spacing=6)
        main_vbox.pack_start(search_box, False, False, 0)
        search_label = Gtk.Label(label="Buscar:")
        search_box.pack_start(search_label, False, False, 0)
        self.search_entry = Gtk.Entry()
        search_box.pack_start(self.search_entry, True, True, 0)
        self.count_label = Gtk.Label(label="0 programas")
        search_box.pack_start(self.count_label, False, False, 0)

        self.load_button = Gtk.Button(label="Cargar JSON desde GitHub")
        main_vbox.pack_start(self.load_button, False, False, 0)

        self.scrolled = Gtk.ScrolledWindow()
        self.scrolled.set_policy(Gtk.PolicyType.AUTOMATIC, Gtk.PolicyType.AUTOMATIC)
        main_vbox.pack_start(self.scrolled, True, True, 0)

        self.cards_box = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=10)
        self.scrolled.add(self.cards_box)

        self.progress_bar = Gtk.ProgressBar()
        self.progress_bar.set_show_text(True)
        self.progress_bar.set_visible(False)
        main_vbox.pack_start(self.progress_bar, False, False, 0)

        btn_box = Gtk.Box(spacing=10)
        main_vbox.pack_start(btn_box, False, False, 0)
        self.add_button = Gtk.Button(label="Añadir programa")
        self.upload_button = Gtk.Button(label="Subir a GitHub")
        self.settings_button = Gtk.Button(label="Ajustes")
        self.about_button = Gtk.Button(label="Acerca de")
        btn_box.pack_start(self.add_button, False, False, 0)
        btn_box.pack_start(self.upload_button, False, False, 0)
        btn_box.pack_start(self.settings_button, False, False, 0)
        btn_box.pack_start(self.about_button, False, False, 0)

        self.load_button.connect("clicked", self.load_from_github)
        self.add_button.connect("clicked", self.add_new_program)
        self.upload_button.connect("clicked", self.upload_to_github)
        self.settings_button.connect("clicked", self.open_settings_dialog)
        self.about_button.connect("clicked", self.open_about_dialog)
        self.search_entry.connect("changed", self.filter_cards)

        if self.config.get("url_json"):
            self.load_from_github()

        self.show_all()

    def load_config(self):
        if os.path.exists(CONFIG_FILE):
            try:
                with open(CONFIG_FILE, "r", encoding="utf-8") as f:
                    return json.load(f)
            except Exception:
                return {}
        return {}

    def save_config(self, config):
        try:
            with open(CONFIG_FILE, "w", encoding="utf-8") as f:
                json.dump(config, f, indent=2)
        except Exception as e:
            dialog = Gtk.MessageDialog(parent=self, flags=0, message_type=Gtk.MessageType.ERROR,
                                       buttons=Gtk.ButtonsType.OK, text=f"No se pudo guardar la configuración:\n{e}")
            dialog.run()
            dialog.destroy()

    def clear_cards(self):
        for child in self.cards_box.get_children():
            self.cards_box.remove(child)

    def populate_cards(self):
        self.clear_cards()
        if not self.json_data:
            self.update_count_label()
            return
        for program in self.json_data:
            card = ProgramCard(program)
            card.connect('changed', self.auto_save)
            card.connect('deleted', lambda w: self.on_card_deleted(w))
            self.cards_box.pack_start(card, False, False, 0)
        self.cards_box.show_all()
        self.update_count_label()

    def on_card_deleted(self, card):
        self.cards_box.remove(card)
        self.auto_save()

    def load_from_github(self, button=None):
        url = self.config.get("url_json", "")
        if not url:
            md = Gtk.MessageDialog(parent=self, flags=0, message_type=Gtk.MessageType.WARNING,
                                   buttons=Gtk.ButtonsType.OK, text="Configura repo y archivo JSON en Ajustes primero.")
            md.run()
            md.destroy()
            return

        def worker():
            try:
                r = requests.get(url)
                r.raise_for_status()
                data_raw = r.json()
                json_data = []
                for item in data_raw:
                    if isinstance(item, dict):
                        enlaces = item.get("enlace_descarga", [])
                        if isinstance(enlaces, str):
                            enlaces = [enlaces] if enlaces else []
                        elif isinstance(enlaces, list):
                            if len(enlaces) > 0 and isinstance(enlaces[0], dict):
                                enlaces = [e.get("url", "") for e in enlaces if "url" in e]
                        else:
                            enlaces = []
                        item["enlace_descarga"] = enlaces
                        json_data.append(item)
                GLib.idle_add(self._load_complete, json_data)
            except Exception as e:
                GLib.idle_add(self._load_failed, str(e))

        threading.Thread(target=worker, daemon=True).start()

    def _load_complete(self, json_data):
        self.json_data = json_data
        self.populate_cards()
        self.auto_save()
        md = Gtk.MessageDialog(parent=self, flags=0, message_type=Gtk.MessageType.INFO,
                               buttons=Gtk.ButtonsType.OK, text="Datos descargados y cargados correctamente.")
        md.run()
        md.destroy()

    def _load_failed(self, error_msg):
        md = Gtk.MessageDialog(parent=self, flags=0, message_type=Gtk.MessageType.ERROR,
                               buttons=Gtk.ButtonsType.OK, text=f"No se pudo descargar JSON desde GitHub:\n{error_msg}")
        md.run()
        md.destroy()

    def add_new_program(self, button):
        dlg = AddProgramDialog(self)
        response = dlg.run()
        if response == Gtk.ResponseType.OK:
            new_prog = dlg.get_data()
            self.json_data.append(new_prog)
            self.populate_cards()
            self.auto_save()
        dlg.destroy()

    def get_all_data(self):
        data = []
        for card in self.cards_box.get_children():
            if isinstance(card, ProgramCard):
                d = card.get_data()
                if d.get("nombre", "").strip():
                    data.append(d)
        return data

    def auto_save(self, *args):
        data = self.get_all_data()
        try:
            with open(self.file_path, "w", encoding="utf-8") as f:
                json.dump(data, f, indent=2, ensure_ascii=False)
            self.json_data = data
        except Exception as e:
            print(f"Error al guardar: {e}")

    def filter_cards(self, entry):
        text = entry.get_text().lower().strip()
        visible_count = 0
        for card in self.cards_box.get_children():
            if isinstance(card, ProgramCard):
                name = card.data.get("nombre", "").lower()
                version = card.data.get("version", "").lower()
                desc = card.data.get("descripcion", "").lower()
                should_show = text in name or text in version or text in desc
                card.set_visible(should_show)
                if should_show:
                    visible_count += 1
        self.update_count_label(visible_count)

    def update_count_label(self, visible_count=None):
        total = len([c for c in self.cards_box.get_children() if isinstance(c, ProgramCard)])
        if visible_count is None:
            visible_count = total
        self.count_label.set_text(f"Mostrando {visible_count} de {total} programas")

    def open_settings_dialog(self, button):
        dlg = SettingsDialog(self.config, self)
        response = dlg.run()
        if response == Gtk.ResponseType.OK:
            self.config = dlg.get_config()
            self.save_config(self.config)
            md = Gtk.MessageDialog(parent=self, flags=0, message_type=Gtk.MessageType.INFO,
                                   buttons=Gtk.ButtonsType.OK, text="Configuración guardada.")
            md.run()
            md.destroy()
        dlg.destroy()

    def open_about_dialog(self, button):
        dlg = AboutDialog(self)
        dlg.run()
        dlg.destroy()

    def upload_to_github(self, button):
        data = self.get_all_data()
        if not data:
            md = Gtk.MessageDialog(parent=self, flags=0, message_type=Gtk.MessageType.WARNING,
                                   buttons=Gtk.ButtonsType.OK, text="No hay programas para subir.")
            md.run()
            md.destroy()
            return
        token = self.config.get("token", "")
        repo = self.config.get("repo", "")
        path = self.config.get("path", "")
        if not token or not repo or not path:
            md = Gtk.MessageDialog(parent=self, flags=0, message_type=Gtk.MessageType.WARNING,
                                   buttons=Gtk.ButtonsType.OK, text="Configura token, repo y path en Ajustes antes de subir.")
            md.run()
            md.destroy()
            return
        try:
            content = json.dumps(data, indent=2, ensure_ascii=False)
        except Exception as e:
            md = Gtk.MessageDialog(parent=self, flags=0, message_type=Gtk.MessageType.ERROR,
                                   buttons=Gtk.ButtonsType.OK, text=f"No se pudo preparar el contenido JSON:\n{e}")
            md.run()
            md.destroy()
            return
        api_url = f"https://api.github.com/repos/{repo}/contents/{path}"
        headers = {"Authorization": f"token {token}"}
        sha = None

        def check_sha_and_upload():
            nonlocal sha
            try:
                r = requests.get(api_url, headers=headers)
                if r.status_code == 200:
                    sha = r.json().get("sha")
                elif r.status_code == 404:
                    sha = None
                else:
                    GLib.idle_add(show_error_dialog, f"Error al consultar archivo en GitHub:\n{r.status_code}\n{r.text}")
                    return
            except Exception as e:
                GLib.idle_add(show_error_dialog, f"No se pudo obtener info archivo en GitHub:\n{e}")
                return

            data_to_send = {
                "message": "Actualización desde editor JSON",
                "content": base64.b64encode(content.encode("utf-8")).decode("utf-8"),
                "branch": "main"
            }
            if sha:
                data_to_send["sha"] = sha

            def progress_cb(value):
                GLib.idle_add(self.progress_bar.set_fraction, value/100.0)

            def finished_cb(success, message):
                def dialog_handler():
                    self.progress_bar.set_visible(False)
                    if success:
                        md = Gtk.MessageDialog(parent=self, flags=0, message_type=Gtk.MessageType.INFO,
                                               buttons=Gtk.ButtonsType.OK, text=message)
                    else:
                        md = Gtk.MessageDialog(parent=self, flags=0, message_type=Gtk.MessageType.ERROR,
                                               buttons=Gtk.ButtonsType.OK, text=message)
                    md.run()
                    md.destroy()
                GLib.idle_add(dialog_handler)

            GLib.idle_add(self.progress_bar.set_visible, True)
            GLib.idle_add(self.progress_bar.set_fraction, 0)
            upload_thread = UploadThread(api_url, headers, data_to_send, progress_cb, finished_cb)
            upload_thread.start()

        def show_error_dialog(text):
            md = Gtk.MessageDialog(parent=self, flags=0, message_type=Gtk.MessageType.ERROR,
                                   buttons=Gtk.ButtonsType.OK, text=text)
            md.run()
            md.destroy()

        threading.Thread(target=check_sha_and_upload, daemon=True).start()

if __name__ == "__main__":
    ensure_icon_cache_dir()
    win = JsonEditor()
    win.show_all()
    Gtk.main()
