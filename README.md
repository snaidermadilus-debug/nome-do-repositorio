📦 DEPENDÊNCIA (SÓ UMA) 

pip install requests --no-cache-dir
____________________ 

▶️ COMO USAR 

python criarimg.py
____________________ 

nano criarimg.py
________
pkg update -y
pkg upgrade -y
pkg install python termux-api -y
termux-setup-storage









#!/usr/bin/env python3
import requests
import os
import time
import re
import subprocess
import random
import threading

# ===== CONFIG =====
SAVE_DIR = "/sdcard/Pictures/IA"
HEADERS = {
    "User-Agent": "Mozilla/5.0 (Linux; Android)"
}

BANNER = r"""
████████╗██████╗ ██╗ █████╗ ██████╗     ██╗███╗   ███╗ █████╗  ██████╗ ███████╗███╗   ███╗
╚══██╔══╝██╔══██╗██║██╔══██╗██╔══██╗    ██║████╗ ████║██╔══██╗██╔════╝ ██╔════╝████╗ ████║
   ██║   ██████╔╝██║███████║██████╔╝    ██║██╔████╔██║███████║██║  ███╗█████╗  ██╔████╔██║
   ██║   ██╔══██╗██║██╔══██║██╔══██╗    ██║██║╚██╔╝██║██╔══██║██║   ██║██╔══╝  ██║╚██╔╝██║
   ██║   ██║  ██║██║██║  ██║██║  ██║    ██║██║ ╚═╝ ██║██║  ██║╚██████╔╝███████╗██║ ╚═╝ ██║
   ╚═╝   ╚═╝  ╚═╝╚═╝╚═╝  ╚═╝╚═╝  ╚═╝    ╚═╝╚═╝     ╚═╝╚═╝  ╚═╝ ╚═════╝ ╚══════╝╚═╝     ╚═╝
"""

stop_counter = False

# ===== FUNÇÕES =====
def limpar_nome(txt):
    return re.sub(r"[^\w]+", "_", txt.lower()).strip("_")

def contador_infinito():
    i = 1
    while not stop_counter:
        print(f"\r{i} ", end="", flush=True)
        time.sleep(0.4)
        i = i + 1 if i < 10 else 1

def buscar_ddg(query):
    try:
        r = requests.post("https://duckduckgo.com/", data={"q": query}, headers=HEADERS, timeout=10)
        m = re.search(r"vqd=([\d-]+)", r.text)
        if not m:
            return None
        vqd = m.group(1)
        url = f"https://duckduckgo.com/i.js?o=json&q={query}&vqd={vqd}&p=1"
        return requests.get(url, headers=HEADERS, timeout=10).json()["results"][0]["image"]
    except:
        return None

def buscar_bing(query):
    try:
        html = requests.get(
            f"https://www.bing.com/images/search?q={query}",
            headers=HEADERS,
            timeout=10
        ).text
        imgs = re.findall(r"murl&quot;:&quot;(.*?)&quot;", html)
        return random.choice(imgs) if imgs else None
    except:
        return None

# ===== LOOP =====
while True:
    os.system("clear")
    print(BANNER)
    print("\ncriar próxima imagem:")
    texto = input(">> ").strip()

    if texto.lower() == "sair":
        break
    if not texto:
        continue

    stop_counter = False
    t = threading.Thread(target=contador_infinito)
    t.start()

    os.makedirs(SAVE_DIR, exist_ok=True)
    nome = limpar_nome(texto)
    caminho = f"{SAVE_DIR}/{nome}_{int(time.time())}.jpg"

    img_url = buscar_ddg(texto) or buscar_bing(texto)
    if img_url:
        try:
            img = requests.get(img_url, headers=HEADERS, timeout=20)
            with open(caminho, "wb") as f:
                f.write(img.content)
        except:
            stop_counter = True
            t.join()
            continue

    subprocess.run(["termux-media-scan", caminho], check=False)

    # ===== GALERIA ABRE → CONTADOR PARA =====
    stop_counter = True
    t.join()

    subprocess.run([
        "am", "start",
        "-a", "android.intent.action.VIEW",
        "-d", f"file://{caminho}",
        "-t", "image/*",
        "com.sec.android.gallery3d"
    ], check=False)