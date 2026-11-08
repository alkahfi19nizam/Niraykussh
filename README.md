# Niraykussh

Tentu! Berikut adalah **ringkasan lengkap** dari Tahap 1 hingga Tahap 3. Saya susun secara berurutan agar Anda bisa mempelajarinya dari logika paling dasar (koneksi) hingga menjadi aplikasi GUI utuh.

---

# 📚 Ringkasan Lengkap: Membangun Aplikasi SSH Tunnel (Python)

## 🎯 Gambaran Besar
Kita membangun aplikasi yang meniru DarkTunnel/NetMod dengan alur:
```
[PC Anda] → [HTTP Proxy] → [Server SSH] → [Internet]
```
- **Tahap 1**: Menembus proxy dengan payload HTTP + membuat koneksi SSH.
- **Tahap 2**: Mengubah koneksi SSH menjadi Proxy Lokal (SOCKS5).
- **Tahap 3**: Membungkus semuanya ke dalam GUI Windows.

---

## 📦 Persiapan: Library yang Dibutuhkan
Buka CMD dan jalankan:
```bash
pip install paramiko customtkinter
```
- `paramiko` → untuk koneksi SSH.
- `customtkinter` → untuk tampilan GUI modern.

Library lain (`socket`, `threading`, `struct`, `select`, `time`) sudah bawaan Python.

---

## 🥇 Tahap 1: Logika Inti (Menembus Proxy + SSH)
**Tujuan:** Membuat koneksi SSH melewati HTTP Proxy dengan mengirim *payload* khusus.

### Konsep Kunci:
1. **Socket Mentah** dibuat ke alamat Proxy, bukan ke server SSH.
2. **Payload HTTP** dikirim ke Proxy. Isinya adalah trik `Upgrade: websocket` agar proxy mau membuka jalur.
3. Jika proxy merespons `101 Switching Protocols` atau `200 Connection Established`, artinya jalur sudah terbuka.
4. **Socket tersebut diserahkan ke Paramiko** (`sock=sock`) agar SSH berjalan di atas jalur proxy tersebut.

### Skrip Inti (Ringkas):
```python
import socket, paramiko

def create_ssh_tunnel(host, port, user, pwd, proxy_host, proxy_port, payload):
    # 1. Buka socket ke Proxy
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect((proxy_host, proxy_port))
    
    # 2. Kirim Payload (ubah [crlf] jadi \r\n, [host] jadi host SSH)
    formatted = payload.replace("[crlf]", "\r\n").replace("[host]", host)
    sock.sendall(formatted.encode())

    # 3. Baca respons proxy
    response = b""
    while b"\r\n\r\n" not in response:
        response += sock.recv(4096)
    
    if b"101" not in response and b"200" not in response:
        print("[-] Proxy menolak.")
        return None

    # 4. Sambungkan SSH ke socket yang sudah tembus proxy
    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    ssh.connect(host, port=port, username=user, password=pwd, sock=sock)
    print("[+] SSH Terhubung!")
    return ssh
```

> 💡 **Poin Belajar:** `sock=sock` adalah kunci utama. Kita "menipu" Paramiko agar mengira sedang menyambung ke server SSH langsung, padahal jalurnya lewat proxy.

---

## 🥈 Tahap 2: Membuat Proxy Lokal (SOCKS5)
**Tujuan:** Menyediakan "gerbang" lokal di `127.0.0.1:1080` agar browser bisa menggunakan koneksi SSH tadi.

### Konsep Kunci:
1. **Server Socket** dibuat mendengarkan port `1080`.
2. Setiap kali browser terhubung, kita **buka channel baru lewat SSH** (`open_channel("direct-tcpip", ...)`).
3. Data dari browser diteruskan ke channel SSH, dan sebaliknya (dua arah).
4. Setiap koneksi dijalankan di **thread terpisah** agar bisa melayani banyak tab browser sekaligus.

### Skrip Inti (Ringkas):
```python
import socket, threading, struct, select

def handle_client(client_sock, ssh_transport):
    # 1. Handshake SOCKS5
    client_sock.recv(2)  # versi + jumlah metode
    client_sock.recv(255) # metode auth
    client_sock.sendall(b"\x05\x00")  # OK, tanpa auth
    
    # 2. Baca permintaan (alamat tujuan)
    ver, cmd, _, atyp = client_sock.recv(4)
    if atyp == 1:   # IPv4
        addr = socket.inet_ntoa(client_sock.recv(4))
    elif atyp == 3: # Domain
        length = client_sock.recv(1)[0]
        addr = client_sock.recv(length).decode()
    port = struct.unpack('>H', client_sock.recv(2))[0]
    
    # 3. Buka channel lewat SSH
    channel = ssh_transport.open_channel(
        "direct-tcpip", (addr, port), client_sock.getpeername()
    )
    client_sock.sendall(b"\x05\x00\x00\x01" + socket.inet_aton("0.0.0.0") + b"\x00\x00")
    
    # 4. Forwarding data dua arah
    while True:
        r, _, _ = select.select([client_sock, channel], [], [], 0.1)
        if client_sock in r:
            data = client_sock.recv(4096)
            if not data: break
            channel.sendall(data)
        if channel in r:
            data = channel.recv(4096)
            if not data: break
            client_sock.sendall(data)
    client_sock.close()

def start_socks_server(ssh, port=1080):
    transport = ssh.get_transport()
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.bind(("127.0.0.1", port))
    server.listen(100)
    while True:
        c, _ = server.accept()
        threading.Thread(target=handle_client, args=(c, transport), daemon=True).start()
```

> 💡 **Poin Belajar:** `open_channel("direct-tcpip", ...)` adalah cara Paramiko membuat "terowongan" untuk setiap permintaan web dari browser Anda.

---

## 🥉 Tahap 3: Membungkus ke GUI (CustomTkinter)
**Tujuan:** Membuat tampilan seperti DarkTunnel (form input + tombol + log berwarna).

### Konsep Kunci:
1. **Class `TunnelApp`** mewarisi `ctk.CTk` (jendela utama).
2. **Threading Wajib**: Koneksi SSH & Server SOCKS dijalankan di thread terpisah, karena GUI harus tetap responsif (jika tidak, jendela akan macet/not responding).
3. **Method `log()`** dipakai untuk menulis ke kotak log dengan aman (mengunci textbox saat menulis).
4. **Tombol Connect/Disconnect** berfungsi ganda dengan mengubah `self.is_running`.

### Skrip Inti (Poin Penting):
```python
import customtkinter as ctk

class TunnelApp(ctk.CTk):
    def __init__(self):
        super().__init__()
        self.is_running = False
        self.ssh_client = None
        
        # Widget input, tombol, dan logbox...
        self.btn_connect = ctk.CTkButton(self, text="CONNECT", command=self.toggle)
    
    def log(self, msg):
        self.log_box.configure(state="normal")
        self.log_box.insert("end", msg + "\n")
        self.log_box.see("end")
        self.log_box.configure(state="disabled")
    
    def toggle(self):
        if not self.is_running:
            self.is_running = True
            # Jalankan logika di thread
            threading.Thread(target=self.start_tunnel, daemon=True).start()
        else:
            self.is_running = False
            if self.ssh_client:
                self.ssh_client.close()
    
    def start_tunnel(self):
        # (Logika Tahap 1: konek proxy + SSH)
        # (Logika Tahap 2: jalankan start_socks_server di thread lain)
        ...
```

> 💡 **Poin Belajar:** `daemon=True` pada thread berarti thread akan otomatis mati saat aplikasi ditutup, mencegah aplikasi "nyangkut" di background Task Manager.

---

## 🗂️ Struktur File Akhir (Rekomendasi)
Sebaiknya pisahkan kode agar mudah dipelajari dan dikembangkan:

```
📁 MyTunnel/
├── 📄 net_core.py       ← Tahap 1 & 2 (logika SSH + SOCKS)
├── 📄 gui_app.py        ← Tahap 3 (GUI CustomTkinter)
└── 📄 requirements.txt  ← paramiko, customtkinter
```

**Contoh `net_core.py` (fungsi yang bisa dipanggil GUI):**
```python
def start_all(target, proxy, payload, log_callback):
    ssh = create_ssh_tunnel(...)
    if ssh:
        log_callback("[+] Connected!")
        start_socks_server(ssh, log_callback=log_callback)
```

**Contoh di `gui_app.py`:**
```python
from net_core import start_all
start_all(target, proxy, payload, log_callback=self.log)
```

---

## 🧪 Panduan Pengujian (Step-by-Step)
| Tahap | Cara Uji | Tanda Berhasil |
|-------|----------|----------------|
| **1** | Jalankan `core_tunnel.py` di CMD | Muncul `[++] SSH TERHUBUNG!` |
| **2** | Jalankan `tahap2_tunnel.py`, atur browser ke SOCKS5 `127.0.0.1:1080` | Buka `whatismyip.com`, IP berubah |
| **3** | Jalankan `NetTunnel_GUI.py`, klik CONNECT | Log hijau muncul, browser jalan |

### ⚠️ Troubleshooting Umum
- **"Connection refused"** → Proxy mati/gratisan sudah tutup. Cari proxy HTTP aktif.
- **"Authentication failed"** → Username/password SSH salah atau sudah expired.
- **Timeout saat handshake** → Payload tidak cocok dengan server proxy. Coba variasi payload lain.
- **GUI freeze** → Pastikan semua koneksi berjalan di `threading.Thread(...)`.

---

## 🚀 Pengembangan Lanjutan (Ide Belajar)
1. **Simpan Konfigurasi**: Gunakan `json` untuk menyimpan Target/Proxy/Payload terakhir.
2. **Auto-Reconnect**: Tambahkan loop `while is_running:` untuk mencoba ulang jika putus.
3. **UDP Forwarding**: Untuk game online (butuh modifikasi SOCKS5 untuk UDP ASSOCIATE).
4. **Packaging**: `pyinstaller --noconsole --onefile gui_app.py` → hasilkan `.exe`.
5. **Multi-Server**: Buat daftar server di `config.json`, lalu GUI menampilkan list pilihan.

---

## ✅ Kesimpulan Pembelajaran
Dari ketiga tahap ini, Anda telah mempelajari **5 konsep inti jaringan**:
1. **Raw Socket** → mengontrol koneksi TCP secara langsung.
2. **HTTP Payload Injection** → trik menembus firewall/proxy.
3. **SSH Tunneling** via Paramiko → mengalihkan koneksi lewat server aman.
4. **SOCKS5 Proxy** → membuat gateway lokal untuk aplikasi lain.
5. **Threading & GUI** → menggabungkan semuanya menjadi aplikasi nyata.

> 📌 **Catatan Etika**: Teknik payload injection ini hanya boleh digunakan pada jaringan **milik sendiri** atau yang **Anda punya izinnya**. Jangan gunakan untuk melewati kebijakan jaringan institusi/perusahaan tanpa izin.

Silakan pelajari skrip-skrip di atas secara berurutan. Jika ada bagian yang ingin diperdalam (misal: cara kerja handshake SOCKS5 baris per baris), tanyakan saja ya! 🚀
