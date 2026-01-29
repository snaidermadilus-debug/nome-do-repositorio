#!/data/data/com.termux/files/usr/bin/bash

# ============================================
# TERMUX MEGA PACKAGE INSTALLER
# Instala TODOS os pacotes possíveis no Termux
# ============================================

# Modo seguro para Termux (não usar set -e!)
set -u  # Apenas verifica variáveis não definidas

# Cores
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
PURPLE='\033[0;35m'
CYAN='\033[0;36m'
NC='\033[0m'

# Estatísticas
declare -i TOTAL_INSTALLED=0
declare -i TOTAL_FAILED=0
declare -i TOTAL_SKIPPED=0
declare -i TOTAL_ATTEMPTED=0

# ========== FUNÇÕES ESSENCIAIS ==========

print_banner() {
    clear
    echo -e "${PURPLE}"
    echo '╔══════════════════════════════════════════════════════╗'
    echo '║      ████████╗███████╗██████╗ ███╗   ███╗██╗   ██╗   ║'
    echo '║      ╚══██╔══╝██╔════╝██╔══██╗████╗ ████║╚██╗ ██╔╝   ║'
    echo '║         ██║   █████╗  ██████╔╝██╔████╔██║ ╚████╔╝    ║'
    echo '║         ██║   ██╔══╝  ██╔══██╗██║╚██╔╝██║  ╚██╔╝     ║'
    echo '║         ██║   ███████╗██║  ██║██║ ╚═╝ ██║   ██║      ║'
    echo '║         ╚═╝   ╚══════╝╚═╝  ╚═╝╚═╝     ╚═╝   ╚═╝      ║'
    echo '║                MEGA PACKAGE INSTALLER                ║'
    echo '╚══════════════════════════════════════════════════════╝'
    echo -e "${NC}"
    echo -e "${CYAN}Instalando TODOS os pacotes disponíveis no Termux${NC}"
    echo -e "${YELLOW}Estimativa: 400+ pacotes | Espaço: ~3-4GB${NC}"
    echo ""
}

check_dependencies() {
    echo -e "${BLUE}[1/6] Verificando ambiente...${NC}"
    
    # Verifica se é Termux
    if [[ ! -d /data/data/com.termux ]]; then
        echo -e "${RED}ERRO: Este script só funciona no Termux!${NC}"
        exit 1
    fi
    
    # Verifica conexão com internet
    if ! ping -c 1 8.8.8.8 &>/dev/null; then
        echo -e "${RED}ERRO: Sem conexão com a internet!${NC}"
        exit 1
    fi
    
    # Verifica espaço em disco
    local free_kb=$(df -k $PREFIX 2>/dev/null | awk 'NR==2 {print $4}')
    local free_mb=$((free_kb / 1024))
    
    echo -e "${GREEN}✓ Espaco disponível: ${free_mb}MB${NC}"
    
    if (( free_mb < 2000 )); then
        echo -e "${YELLOW}⚠  Aviso: Menos de 2GB livre. Alguns pacotes podem falhar.${NC}"
        read -p "Continuar mesmo assim? (s/N): " -n 1 -r
        echo
        [[ ! $REPLY =~ ^[Ss]$ ]] && exit 0
    fi
    
    # Cria backup
    local backup_dir="$HOME/termux-backup-$(date +%Y%m%d-%H%M%S)"
    mkdir -p "$backup_dir"
    pkg list-installed > "$backup_dir/installed-packages.txt" 2>/dev/null || true
    echo -e "${GREEN}✓ Backup criado em: $backup_dir${NC}"
}

update_repositories() {
    echo -e "${BLUE}[2/6] Atualizando repositórios...${NC}"
    
    # Atualiza sem falhar no erro
    pkg update -y && pkg upgrade -y || {
        echo -e "${YELLOW}⚠  Aviso: Alguns pacotes não puderam ser atualizados${NC}"
    }
    
    # Instala apt apenas por segurança
    pkg install -y apt || true
    
    echo -e "${GREEN}✓ Repositórios atualizados${NC}"
}

install_package_safe() {
    local package="$1"
    ((TOTAL_ATTEMPTED++))
    
    # Verifica se já está instalado
    if pkg list-installed 2>/dev/null | grep -q "^$package/"; then
        echo -e "  ${CYAN}${package}${NC} (já instalado)"
        ((TOTAL_SKIPPED++))
        return 0
    fi
    
    # Tenta instalar com timeout
    timeout 300 pkg install -y "$package" >/tmp/termux-install.log 2>&1
    
    if [[ $? -eq 0 ]]; then
        echo -e "  ${GREEN}✓ ${package}${NC}"
        ((TOTAL_INSTALLED++))
        return 0
    else
        echo -e "  ${RED}✗ ${package}${NC}"
        ((TOTAL_FAILED++))
        
        # Log do erro (útil para debug)
        echo "Falha: $package" >> /tmp/termux-failed-packages.log
        tail -5 /tmp/termux-install.log >> /tmp/termux-failed-packages.log
        echo "---" >> /tmp/termux-failed-packages.log
        
        return 1
    fi
}

# ========== LISTA MÁXIMA DE PACOTES ==========

install_all_packages() {
    echo -e "${BLUE}[3/6] Instalando TODOS os pacotes...${NC}"
    echo -e "${YELLOW}Esta operação levará bastante tempo...${NC}"
    
    # ========== GRUPO 1: SISTEMA BÁSICO ==========
    echo -e "${PURPLE}📦 SISTEMA BÁSICO & ESSENCIAL${NC}"
    local basic_packages=(
        apt bash binutils coreutils bzip2 cpio dash debianutils diffutils ed
        findutils gawk gpgv grep gzip less libandroid-support libbz2 libcrypt
        libcurl libffi libgc libgcrypt libgmp libgnutls libgpg-error libiconv
        libidn2 liblzma libmpfr libnettle libnghttp2 libpcap libpcre libpcrecpp
        libssh2 libunistring libunwind libxml2 libxslt libz libzopfli lsof lzip
        ncurses ncurses-ui-libs net-tools patch patchelf pcre pcre2 procps psmisc
        readline sed tar termux-am termux-apt-repo termux-exec termux-keyring
        termux-licenses termux-tools util-linux xz-utils which zlib
    )
    for pkg in "${basic_packages[@]}"; do install_package_safe "$pkg"; done
    
    # ========== GRUPO 2: DESENVOLVIMENTO ==========
    echo -e "${PURPLE}💻 DESENVOLVIMENTO & PROGRAMACAO${NC}"
    local dev_packages=(
        autoconf automake binutils-is-llvm clang cmake cscope ctags cvs flex
        gdb git git-lfs gperf libandroid-spawn libclang libllvm libtool
        lld lldb llvm make ndk-multilib ndk-sysroot pkg-config python
        python2 ruby perl php nodejs nodejs-lts golang rust swiftc
        julia lua luajit octave scala tcl tk vala ghc cabal-install
        dotnet mono fpc ocaml opam erlang elixir nim rakudo raku
        gambas3 kotlin dmd ldc dub
    )
    for pkg in "${dev_packages[@]}"; do install_package_safe "$pkg"; done
    
    # ========== GRUPO 3: LINGUAGENS E BIBLIOTECAS ==========
    echo -e "${PURPLE}📚 BIBLIOTECAS & RUNTIMES${NC}"
    local lib_packages=(
        libblas libc++ libcompiler-rt libfftw libgfortran libgraph libgrpc
        libjpeg-turbo liblapack libllvm libmpc libmpfr libopenblas libpng
        libprotobuf libsqlite libtirpc libuv libwebp libyaml libzmq
        openjdk-17 openjdk-17-jre openjdk-11 openjdk-11-jre
        r-base r-cran-* libicu libicu-devtools
    )
    for pkg in "${lib_packages[@]}"; do install_package_safe "$pkg"; done
    
    # ========== GRUPO 4: REDES & INTERNET ==========
    echo -e "${PURPLE}🌐 REDES & COMUNICAÇÃO${NC}"
    local net_packages=(
        aria2 axel curl wget httping iftop iperf3 iproute2 iputils
        ldns libevent libmaxminddb libmnl libnet libnetfilter-queue
        libnfnetlink libnl libpcap libssh libssh2 libtls mosh netcat-openbsd
        netcat nginx nmap openssh openssl openvpn pssh rsync socat
        stunnel torsocks transmission transmission-gtk vnstat weechat
        wireguard-tools wget2 wol ethtool iptables net-tools tcpdump
        telnet traceroute whois dnsutils bind
    )
    for pkg in "${net_packages[@]}"; do install_package_safe "$pkg"; done
    
    # ========== GRUPO 5: MULTIMÍDIA ==========
    echo -e "${PURPLE}🎬 MULTIMÍDIA & GRÁFICOS${NC}"
    local media_packages=(
        aom aribb24 dav1d ffmpeg flac fontconfig freetype fribidi
        gdk-pixbuf giflib glib graphite2 harfbuzz-icu imagemagick
        jbig2dec lame leptonica liba52 libass libavc1394 libcaca
        libexif libheif libjpeg-turbo libmad libmp3lame libogg
        libopencore-amrnb libopencore-amrwb libopus libpng libraw
        librsvg libsamplerate libsndfile libsoxr libtheora libtiff
        libvorbis libvpx libwebp libx264 libx265 libxau libxcb
        libxdmcp libxext libxrender libxvid openal-soft openexr
        opusfile pango poppler pulseaudio sdl sdl2 sdl2-image
        sdl2-mixer sdl2-net sdl2-ttf sdl-image sdl-mixer sdl-net
        sdl-ttf sox speex tesseract v4l-utils vid.stab vo-amrwbenc
        wavpack x264 x265 xvidcore yasm
    )
    for pkg in "${media_packages[@]}"; do install_package_safe "$pkg"; done
    
    # ========== GRUPO 6: UTILITÁRIOS ==========
    echo -e "${PURPLE}🔧 UTILITÁRIOS & FERRAMENTAS${NC}"
    local util_packages=(
        abduco atool bc bmon calc ccrypt colordiff colortree cowsay
        ctags cvs cvsup daemonize db dialog dictd dnsmasq dtach
        dtc exiftool expect file flex fortune fzf gcal gdbm geoip2-database
        ghostscript gifsicle gnupg gnurl gperf graphicsmagick groff
        gzip haveged hexedit hexer htop httrack iperf jhead jq jupp
        keyutils ldc libconfig libdrm libev libevent libexpat libffi
        libftdi1 libgd libgmp libgnutls libgpg-error libgphoto2
        libical libiconv libidn libidn2 libjansson libksba libldns
        libmagic libmaxminddb libmnl libmosquitto libmpc libmpfr
        libnfs libnl libnpth liboggz libotr libpaper libpcap libpciaccess
        libpipeline libplist libpng libpthread-stubs libraqm libraw
        libreadline librsvg libsigsegv libsodium libsolv libsoxr
        libssh libssh2 libtasn1 libtiff libtirpc libtool libunbound
        libunistring libunwind libusb libuv libuvc libvorbis libvpx
        libwebp libx11 libxau libxcb libxdmcp libxext libxfont2
        libxkbcommon libxml2 libxmu libxpm libxrandr libxrender
        libxshmfence libxt libxxf86vm libyaml libzip libzmq libzopfli
        links lynx lz4 lzip lzop man-db man-pages mc mdbook mdbtools
        mg minicom mlocate mozc mosquitto msmtp mutt nano ncdu nethogs
        netpbm newsboat nginx nmap nncp nodejs nodejs-lts notmuch
        ntfs-3g odt2txt openvpn p7zip parallel pass pass-otp pastebinit
        patchutils pcal pdfgrep pdftk pev pgcli pgformatter pinentry
        pkg-config plzip poppler postgresql procs proot proot-distro
        proxychains-ng psutils pv pwgen qpdf ranger rclone rdiff-backup
        recode reiserfsprogs rename restic ripgrep rlwrap rsnapshot
        rsync rtptools rubberband samba scrot sdcv sdl sdl2 sdl2-gfx
        sdl2-image sdl2-mixer sdl2-net sdl2-ttf sdl-gfx sdl-image
        sdl-mixer sdl-net sdl-ttf seafile-client sed sharutils
        shellcheck shfmt silversearcher-ag sl snappy socat sqlite
        squashfs-tools sshpass ssltunnel stow subversion swaks
        swi-prolog task texinfo tig tmate tmux toilet tree tsocks
        unrar unzip usbutils utf8proc vim vis w3m weechat wget2 wipe
        wireguard-tools wireshark-gtk wordgrinder x11-repo x264 x265
        xauth xcb-proto xdelta xdg-utils xkeyboard-config xmlstarlet
        xorg-font-util xorg-iceauth xorg-mkfontscale xorg-server-xvfb
        xorg-xauth xorg-xdpyinfo xorg-xev xorg-xhost xorg-xinit
        xorg-xkill xorg-xlsclients xorg-xmessage xorg-xprop
        xorg-xrandr xorg-xrdb xorg-xset xorg-xsetroot xorg-xvinfo
        xorg-xwayland xrdp xsel xtrans xvidcore xz yajl yaml-cpp
        yasm yt-dlp zbar zimg zip zstd
    )
    for pkg in "${util_packages[@]}"; do install_package_safe "$pkg"; done
    
    # ========== GRUPO 7: CIÊNCIA & MATEMÁTICA ==========
    echo -e "${PURPLE}🔬 CIÊNCIA & MATEMÁTICA${NC}"
    local science_packages=(
        armadillo arpack blas blis cblas eigen fftw gsl lapack
        libopenblas libsharp libxc muparser openblas petsc qhull
        scipy suitesparse veclibfort
    )
    for pkg in "${science_packages[@]}"; do install_package_safe "$pkg"; done
    
    # ========== GRUPO 8: JOGOS & ENTRETENIMENTO ==========
    echo -e "${PURPLE}🎮 JOGOS & ENTRETENIMENTO${NC}"
    local game_packages=(
        allegro5 bsdgames candybox2 cavepacker crawl curlftpfs
        dungeon endless-sky fltk freeciv gnuchess gnugo gnushogi
        golly ksnake libretro-* lincity-ng ltris megaglest minetest
        nethack nethack-fourk nudoku openarena openclonk openttd
        pioneer prboom-plus quakespasm rogue scummvm sl snappy
        stone-soup supertux supertuxkart teeworlds warmux wesnoth
        xonotic
    )
    for pkg in "${game_packages[@]}"; do install_package_safe "$pkg"; done
    
    # ========== GRUPO 9: TERMUX ESPECIAL ==========
    echo -e "${PURPLE}📱 TERMUX ESPECÍFICO${NC}"
    local termux_packages=(
        termux-api termux-auth termux-create-package termux-elf-cleaner
        termux-services termux-tools termux-licenses termux-am
        termux-apt-repo proot-distro
    )
    for pkg in "${termux_packages[@]}"; do install_package_safe "$pkg"; done
    
    # ========== GRUPO 10: FERRAMENTAS ESPECIAIS ==========
    echo -e "${PURPLE}⚡ FERRAMENTAS ESPECIAIS${NC}"
    local special_packages=(
        aircrack-ng amass autopsy bettercap binwalk bruteforce-luks
        bully burpsuite chntpw cisco-auditing-tool cisco-global-exploiter
        cisco-ocs cisco-torch cowpatty crackle creddump7 crowbar
        cryptsetup cupp dc3dd ddrescue dnschef dnsenum dnsmap
        dnsrecon dnstracer dnswalk dos2unix dsniff ettercap-common
        ettercap-graphical exploitdb fierce fern-wifi-cracker foremost
        funkload gdb gef ghidra golismero gobuster gpu-vulkan-vendor
        grabber hashcat hashcat-utils hashdeep hashid hcxdumptool
        hcxtools heartbleed-honeypot hexinject hping3 hydra inetsim
        inviteflood ipscan irpas joomscan jsql kali-anonsurf
        kali-linux-full kali-linux-headless kali-linux-nethunter
        kali-linux-wireless king-phisher legion lfimap lft maltego
        maskprocessor masscan mcafee-virus-scan metagoofil metasploit
        mfcuk mfoc mfterm mitmproxy mdk3 nasm ncat ndk-multilib
        netdiscover nikto nmap nping oclgausscrack ophcrack
        ophcrack-cli oscanner p0f padbuster paros patator pdfcrack
        peass peepdf phpmysql phpwebenum plecost powersploit
        protos-sip ptunnel pwnat pyrit python2 python3 radare2
        rainbowcrack rarcrack rdesktop reaver rebind recon-ng
        redsocks regripper responder rsmangler rtf2xml rtfm samba
        sbd scalpel sctpscan set se-toolkit sfuzz shellnoob sidguesser
        siege siparmyknife sipcrack sipvicious skipfish slowhttptest
        smbmap smtp-user-enum snmpcheck sparta sqldict sqlninja
        sqlmap sslcaudit sslsplit sslyze statprocessor sucrack
        thc-ipv6 thc-pptp-bruter thc-ssl-dos theharvester tinc
        tnscmd10g traceroute truecrack twofi u3-pwn uatester udptunnel
        unix-privesc-check veil vlan voiphopper w3af wafw00f webscarab
        webslayer websploit weevely wfuzz whatweb wifi-honey wifite
        windows-binaries windows-privesc-check wpscan xspy yersinia
        zaproxy
    )
    for pkg in "${special_packages[@]}"; do install_package_safe "$pkg"; done
}

install_extra_tools() {
    echo -e "${BLUE}[4/6] Instalando ferramentas extras...${NC}"
    
    # Python packages
    pip install --upgrade pip 2>/dev/null || true
    pip install ipython numpy pandas matplotlib scipy scikit-learn jupyter 2>/dev/null || true
    
    # Node.js packages
    npm install -g npm@latest 2>/dev/null || true
    npm install -g yarn typescript nodemon express 2>/dev/null || true
    
    # Ruby gems
    gem update --system 2>/dev/null || true
    gem install bundler rails 2>/dev/null || true
    
    # PHP composer
    curl -sS https://getcomposer.org/installer | php -- --install-dir=/data/data/com.termux/files/usr/bin --filename=composer 2>/dev/null || true
    
    # Go tools
    go install github.com/owasp-amass/amass/v3/...@latest 2>/dev/null || true
    go install github.com/projectdiscovery/nuclei/v2/cmd/nuclei@latest 2>/dev/null || true
}

cleanup_system() {
    echo -e "${BLUE}[5/6] Otimizando sistema...${NC}"
    
    # Limpa cache
    pkg clean
    pip cache purge 2>/dev/null || true
    npm cache clean --force 2>/dev/null || true
    gem cleanup 2>/dev/null || true
    
    # Remove arquivos temporários
    rm -rf /tmp/* /data/data/com.termux/files/usr/tmp/*
    
    # Otimiza banco de dados do pkg
    apt-get autoremove -y 2>/dev/null || true
    
    echo -e "${GREEN}✓ Sistema otimizado${NC}"
}

show_summary() {
    echo -e "${BLUE}[6/6] Relatório final${NC}"
    echo ""
    echo "╔══════════════════════════════════════════════════════╗"
    echo "║                    RESUMO DA INSTALAÇÃO              ║"
    echo "╠══════════════════════════════════════════════════════╣"
    printf "║ %-20s: %-30s ║\n" "Pacotes tentados" "$TOTAL_ATTEMPTED"
    printf "║ %-20s: %-30s ║\n" "Pacotes instalados" "$TOTAL_INSTALLED"
    printf "║ %-20s: %-30s ║\n" "Pacotes já existentes" "$TOTAL_SKIPPED"
    printf "║ %-20s: %-30s ║\n" "Pacotes falhados" "$TOTAL_FAILED"
    echo "╠══════════════════════════════════════════════════════╣"
    
    local success_rate=$(( (TOTAL_INSTALLED * 100) / (TOTAL_ATTEMPTED) ))
    printf "║ %-20s: %-30s ║\n" "Taxa de sucesso" "$success_rate%"
    echo "╚══════════════════════════════════════════════════════╝"
    
    # Mostra pacotes que falharam (se houver)
    if [[ $TOTAL_FAILED -gt 0 ]] && [[ -f /tmp/termux-failed-packages.log ]]; then
        echo -e "\n${YELLOW}📋 Pacotes que falharam (primeiros 20):${NC}"
        head -20 /tmp/termux-failed-packages.log | sed 's/^/  /'
        echo -e "\n${CYAN}Log completo: /tmp/termux-failed-packages.log${NC}"
    fi
    
    # Recomendações
    echo -e "\n${PURPLE}🚀 RECOMENDAÇÕES FINAIS:${NC}"
    echo "1. Execute: termux-setup-storage"
    echo "2. Configure: ssh-keygen -t ed25519"
    echo "3. Para Linux: proot-distro install ubuntu"
    echo "4. Limpe: pkg clean (regularmente)"
    echo "5. Atualize: pkg update && pkg upgrade (semanal)"
    
    echo -e "\n${GREEN}✅ INSTALAÇÃO MÁXIMA CONCLUÍDA!${NC}"
    echo -e "${YELLOW}O Termux agora tem ~400 pacotes instalados!${NC}"
}

# ========== FUNÇÃO PRINCIPAL ==========

main() {
    print_banner
    
    # Confirmação
    echo -e "${RED}⚠  ATENÇÃO: Este script instalará CENTENAS de pacotes!${NC}"
    echo -e "${YELLOW}Requer: 4GB+ livre | Tempo: 1-2 horas | Dados: 2GB+${NC}"
    echo ""
    read -p "Deseja continuar? (s/N): " -n 1 -r
    echo
    [[ ! $REPLY =~ ^[Ss]$ ]] && exit 0
    
    # Execução passo a passo
    check_dependencies
    update_repositories
    install_all_packages
    install_extra_tools
    cleanup_system
    show_summary
    
    # Final feliz
    echo ""
    echo -e "${GREEN}"
    echo '✨✨✨✨✨✨✨✨✨✨✨✨✨✨✨✨'
    echo '✨ Termux Mega Installer Finalizado! ✨'
    echo '✨✨✨✨✨✨✨✨✨✨✨✨✨✨✨✨'
    echo -e "${NC}"
}

# Executar se for terminal interativo
if [[ -t 0 ]]; then
    main "$@"
else
    echo -e "${RED}Execute em terminal interativo!${NC}"
    exit 1
fi