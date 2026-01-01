#!/usr/bin/env python3
# -*- coding: utf-8 -*-

"""
Zero-burocracia image-downloader para Android (Termux)
Autor: SKYNETchat – domínio público
"""

import os
import re
import time
import random
import hashlib
import requests
import socket
from datetime import datetime

# ===== CONFIG =====
SAVE_DIR = "/sdcard/Pictures/IA"
HISTORY_FILE = "history.txt"
TIMEOUT = 8
RETRY = 3

os.makedirs(SAVE_DIR, exist_ok=True)

sess = requests.Session()
sess.headers["User-Agent"] = "Mozilla/5.0 (Linux; Android 13)"

# ===== UTILS =====

def slug(txt):
    """Nome de arquivo seguro"""
    return re.sub(r"\W+", "_", txt.lower()).strip("_")[:50]

def log_history(texto):
    with open(HISTORY_FILE, "a", encoding="utf-8") as f:
        f.write(f"{datetime.now():%d-%m-%Y %H:%M:%S} | {texto}\n")

def short_hash(data: bytes) -> str:
    return hashlib.sha1(data).hexdigest()[:6]

# ===== BUSCA =====

_vqd_cache = {}  # cache com TTL de 5 minutos

def get_vqd(query):
    now = time.time()

    if query in _vqd_cache and _vqd_cache[query]["ttl"] > now:
        return _vqd_cache[query]["vqd"]

    try:
        r = sess.post(
            "https://duckduckgo.com/",
            data={"q": query},
            timeout=TIMEOUT
        )
        vqd = re.search(r"vqd=([\d-]+)", r.text).group(1)
        _vqd_cache[query] = {
            "vqd": vqd,
            "ttl": now + 300
        }
        return vqd
    except:
        return None

def ddg_url(query):
    for _ in range(RETRY):
        vqd = get_vqd(query)
        if not vqd:
            continue
        try:
            r = sess.get(
                f"https://duckduckgo.com/i.js?o=json&q={query}&vqd={vqd}",
                timeout=TIMEOUT
            ).json()
            return r["results"][0]["image"]
        except:
            continue
    return None

def bing_url(query):
    for _ in range(RETRY):
        try:
            html = sess.get(
                f"https://www.bing.com/images/search?q={query}",
                timeout=TIMEOUT
            ).text
            urls = re.findall(r'murl&quot;:&quot;(.*?)&quot;', html)
            return random.choice(urls) if urls else None
        except:
            continue
    return None

def pick_url(query):
    return ddg_url(query) or bing_url(query)

# ===== DOWNLOAD =====

def download_image(url, dest):
    for _ in range(RETRY):
        try:
            r = sess.get(url, timeout=TIMEOUT, stream=True)
            r.raise_for_status()
            with open(dest, "wb") as f:
                for chunk in r.iter_content(1024 * 64):
                    f.write(chunk)
            return True
        except:
            continue
    return False

def open_in_gallery(path):
    os.system(
        f'am start -a android.intent.action.VIEW '
        f'-d "file://{path}" -t "image/*" '
        f'>/dev/null 2>&1'
    )

# ===== CLI =====

def menu():
    print("\n=== SKYNETchat V1 ===")
    print("1 - Uma imagem")
    print("2 - Várias imagens")
    print("3 - Histórico")
    print("0 - Sair")

def main():
    while True:
        menu()
        opt = input(">> ").strip()

        if opt == "0":
            break

        if opt == "3":
            if os.path.exists(HISTORY_FILE):
                os.system(f"less {HISTORY_FILE}")
            else:
                print("Sem histórico.")
            continue

        if opt not in {"1", "2"}:
            continue

        query = input("Descrição: ").strip()
        if not query:
            continue

        log_history(query)

        total = 1
        if opt == "2":
            try:
                total = max(1, int(input("Quantas imagens? ") or 1))
            except:
                total = 1

        for i in range(1, total + 1):
            print(f"Buscando {i}/{total}...", end=" ", flush=True)

            url = pick_url(query)
            if not url:
                print("falhou")
                continue

            filename = (
                f"{slug(query)}_"
                f"{int(time.time()*1000)}_"
                f"{short_hash(url.encode())}.jpg"
            )
            path = os.path.join(SAVE_DIR, filename)

            if download_image(url, path):
                open_in_gallery(path)
                print("ok")
            else:
                print("falhou")

if __name__ == "__main__":
    socket.setdefaulttimeout(TIMEOUT)
    main()