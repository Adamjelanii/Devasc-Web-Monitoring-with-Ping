# Devasc-Web-Monitoring-with-Ping
Skrip Python sederhana untuk memantau kesehatan website dan server dari terminal Anda. Alat ini menyediakan dashboard real-time yang menampilkan status HTTP (UP/DOWN), waktu respons aplikasi, packet loss, latensi jaringan (ping), dan TTL. Ideal untuk diagnostik jaringan cepat.

#Program yang digunakan (Website dapat diganti sesuai kebutuhan)
  import requests
import time
import os
import urllib3
import subprocess  # Ditambahkan untuk menjalankan ping
import re            # Ditambahkan untuk membaca output ping
from urllib.parse import urlparse  # Ditambahkan untuk mengambil hostname dari URL

# -----------------------------------------------------------
# MENONAKTIFKAN SSL WARNING
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
# -----------------------------------------------------------

# DAFTAR WEBSITE YANG AKAN DIPANTAU
WEBSITE_LIST = [
    "https://www.google.com",
    "https://www.cisco.com",
    "https://www.youtube.com",
    "https://developer.cisco.com",
    "https://pcr.ac.id/",
    "https://www.unilak.ac.id/",
    "https://www.uin-suska.ac.id/",
    "https://uir.ac.id/",
    "https://unri.ac.id/"
]

HEADERS = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.110 Safari/537.36'
}

def clear_screen():
    """Membersihkan layar terminal."""
    os.system('cls' if os.name == 'nt' else 'clear')

def check_http_status(url):
    """
    Mengecek status HTTP (L7) dan waktu respons aplikasi.
    """
    status_msg = ""
    latency_msg = ""
    try:
        start_time = time.time()
        response = requests.get(url, headers=HEADERS, timeout=7, verify=False)
        latency = (time.time() - start_time) * 1000
        latency_msg = f"{latency:.0f} ms"

        if response.status_code >= 200 and response.status_code < 300:
            status_msg = "✅ UP"
        else:
            status_msg = f"⚠️ ERROR ({response.status_code})"
    except requests.exceptions.RequestException:
        status_msg = "❌ DOWN"
        latency_msg = "N/A"
    
    return status_msg, latency_msg

def check_server_ping(hostname):
    """
    Mengecek server (L3) menggunakan ping untuk Packet Loss, Latency, dan TTL.
    Mengirim 3 paket ping.
    """
    loss_str = "N/A"
    ping_latency_str = "N/A"
    ttl_str = "N/A"

    # Perintah ping: -c 3 (kirim 3 paket), -W 2 (timeout 2 detik)
    command = ['ping', '-c', '3', '-W', '2', hostname]
    
    try:
        # Menjalankan perintah ping
        output = subprocess.run(command, capture_output=True, text=True, timeout=10).stdout
        
        # 1. Mencari Packet Loss
        loss_match = re.search(r"(\d+)% packet loss", output)
        if loss_match:
            loss_str = f"{loss_match.group(1)}%"
        
        # 2. Mencari TTL (mengambil dari baris balasan pertama)
        ttl_match = re.search(r"ttl=(\d+)", output)
        if ttl_match:
            ttl_str = ttl_match.group(1)
            
        # 3. Mencari Latency Rata-rata (avg)
        # Format: rtt min/avg/max/mdev = 13.931/14.854/15.420/0.648 ms
        rtt_match = re.search(r"rtt min/avg/max/mdev = [\d.]+/([\d.]+)/", output)
        if rtt_match:
            avg_latency = float(rtt_match.group(1))
            ping_latency_str = f"{avg_latency:.0f} ms"

        # Jika 100% loss, rtt_match akan gagal
        if "100% packet loss" in output:
            loss_str = "100%"
            ping_latency_str = "N/A"
            ttl_str = "N/A"

    except subprocess.TimeoutExpired:
        loss_str = "100%"
    except Exception:
        loss_str = "ERR" # Error (misal: unknown host)

    return loss_str, ping_latency_str, ttl_str

def main():
    """Fungsi utama untuk menjalankan dashboard loop."""
    try:
        while True:
            clear_screen()
            print("--- Website & Server Health Monitor (Real-Time) ---")
            print(f"Me-refresh setiap 5 detik... (Tekan Ctrl+C untuk berhenti)\n")
            
            # Header tabel baru yang lebih lebar
            print(f"{'Hostname':<25} | {'HTTP Status':<12} | {'HTTP Time':<10} | {'Ping Loss':<9} | {'Ping Time':<10} | {'TTL':<5}")
            print("-" * 88)
            
            for url in WEBSITE_LIST:
                # 1. Dapatkan hostname (misal: www.google.com) dari URL
                try:
                    hostname = urlparse(url).hostname
                except:
                    hostname = "Invalid URL"
                    continue

                # 2. Cek Status HTTP (Aplikasi)
                http_status, http_latency = check_http_status(url)
                
                # 3. Cek Status Ping (Server)
                loss, ping_latency, ttl = check_server_ping(hostname)
                
                # 4. Cetak semua data
                print(f"{hostname:<25} | {http_status:<12} | {http_latency:<10} | {loss:<9} | {ping_latency:<10} | {ttl:<5}")
            
            time.sleep(5)
            
    except KeyboardInterrupt:
        print("\nMonitor dihentikan oleh pengguna.")

if __name__ == "__main__":
    main()
