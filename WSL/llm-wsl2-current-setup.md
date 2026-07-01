# Lokales LLM-Setup: WSL2 Sandbox & Ollama (Hybrid-Loop-Architektur)

Dieses Dokument dient als persistenten Zustand (State) für LLM-Sessions. Es beschreibt die aktuelle Systemlandschaft, Konfigurationsdateien, optimierte Parameter und den Betrieb lokaler Modelle mit strikter Trennung von Host und Sandbox.

---

## 1. Systemlandschaft & Hardware-Spezifikationen

* **Host-Modell:** Alienware 18 Area-51 Laptop (Modell-ID: AA18250)
* **Betriebssystem:** Windows 11 Home x64
* **CPU:** Intel Core Ultra 9 275HX
* **GPU:** NVIDIA GeForce RTX 5090 Mobile (24 GB VRAM)
* **RAM:** 64 GB DDR5-SDRAM (2x Kingston Fury)
* **Speicher (SSDs):**
  * System-Drive: 1x PCB01 SED SK hynix 2TB (WSL2-Basis)
  * Model-Drive: 1x Samsung SSD 980 PRO 2TB (Eingebunden als Laufwerk `I:`)

---

## 2. Software-Architektur & Sandbox Environment

Das System nutzt eine strikt isolierte WSL2-Umgebung als Sandbox für den Modellbetrieb, um eine Pfad-Verschmutzung durch das Host-System zu verhindern. Um den bekannten `mmap`-Fehler von VirtioFS bei großen Dateien zu umgehen, ohne auf die Performance für Agenten zu verzichten, wird ein virtuelles Linux-Loop-Device verwendet.

* **WSL2 Version:** 2.7.10.0 (oder neuer)
* **Linux-Distribution:** Ubuntu-24.04 LTS
* **Laufzeitumgebungen (WSL2):**
  * `fnm` (Fast Node Manager) -> Verwaltet Node.js LTS
  * `uv` (Fast Python Package Installer) -> Verwaltet Python LTS
* **LLM-Backend:** Ollama (aktuell v0.31.1)

---

## 3. Konfigurationsdateien & Einstellungen

### 3.1 `.wslconfig` (Windows Host-Pfad: `%USERPROFILE%\.wslconfig`)
Optimiert für aggressive Speicherfreigabe an den Windows-Host und hohe I/O-Performance via VirtioFS für zukünftige Agenten-Strukturen.
```ini
[wsl2]
memory=42949672960   # 40 GB RAM für WSL2 reserved
swap=10737418240     # 10 GB Swap-Datei
virtiofs=true        # Schnelles Filesystem-Sharing (wichtig für Agenten-I/O)

[experimental]
autoMemoryReclaim=dropCache  # Automatische Rückgabe von ungenutztem Cache-RAM an Windows
```

### 3.2 `/etc/wsl.conf` (Innerhalb Ubuntu)
Strikte Sandbox-Konfiguration. Keine Interoperabilität mit Windows-Binaries, kein automatischer Mount aller Partitionen.
```ini
[boot]
systemd=true

[interop]
enabled=false
appendWindowsPath=false

[automount]
enabled=false

[gpu]
enabled=true
```

### 3.3 `/etc/fstab` (Innerhalb Ubuntu)
Dedizierter Mount des Windows-Laufwerks `I:`. Das virtuelle Loop-Image wurde hier bewusst entfernt und in einen Systemd-Boot-Dienst ausgelagert, um Race-Conditions beim WSL2-Start zu verhindern.
```text
I: /mnt/i drvfs noatime,defaults,uid=1000,gid=1000,nodev,nosuid 0 0
```

### 3.4 `/etc/systemd/system/mount-llm-storage.service` (Neu: Automount-Dienst)
Verhindert Boot-Fehler, indem das 100-GB-ext4-Image erst geladen wird, wenn das Windows-Dateisystem stabil bereitsteht.
```ini
[Unit]
Description=Mount virtuelles LLM Speicher-Image
After=local-fs.target
Before=ollama.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=/usr/bin/sleep 2
ExecStart=/usr/bin/mount -o loop /mnt/i/WSL/llm_storage.img /usr/share/ollama/.ollama/models

[Install]
WantedBy=multi-user.target
```

### 3.5 `ollama.service` (Systemd-Konfiguration in Ubuntu)
Konfiguriert für maximale VRAM-Auslastung der RTX 5090 (24 GB) unter Berücksichtigung dynamischer Host-Prozesse und optimierter Ressourcen-Freigabe nach Inaktivität.
```ini
[Unit]
Requires=mount-llm-storage.service
After=mount-llm-storage.service

[Service]
Environment="OLLAMA_HOST=127.0.0.1"
Environment="OLLAMA_MAX_LOADED_MODELS=1"
Environment="OLLAMA_MODELS=/usr/share/ollama/.ollama/models"
Environment="OLLAMA_NO_CLOUD=1"
Environment="OLLAMA_GPU_OVERHEAD=2147483648" # 2 GB VRAM für WSL2/WSLg-GUI (z.B. Obsidian) reserviert
Environment="OLLAMA_KEEP_ALIVE=10m"          # Schnelle VRAM-Freigabe (10 Min) nach Agenten-Aktivität
Environment="OLLAMA_NUM_PARALLEL=1"
Environment="OLLAMA_FLASH_ATTENTION=1"
```

---

## 4. Modell-Inventar & Performance-Status

Durch das native `ext4`-Dateisystem innerhalb der virtuellen Festplatte (`llm_storage.img`) läuft der `mmap`-Befehl des Linux-Kernels fehlerfrei. Die Modelle laden mit maximaler NVMe-Geschwindigkeit direkt in den VRAM. Es findet kein CPU/RAM-Offloading statt.

Physischer Speicherort auf dem Host: `I:\WSL\llm_storage.img` (Größe: 100 GB, ext4-formatiert)

Aktuell installierte Modelle:
* `codestral:22b` (~12 GB, Q4_K_M Standard)
* `codestral:22b-v0.1-q5_K_S` (~15 GB)
* `mistral-small:24b` (~14 GB)
* `mistral-small3.2:latest` (~15 GB)

---

## 5. Getroffene Optimierungen & Historie
* **VirtioFS mmap-Fix:** Anstatt VirtioFS zu deaktivieren (was Agenten ausbremsen würde), wurde der Modell-Speicher in ein virtuelles Linux-Loop-Device ausgelagert. Das WSL2-System-Image (`ext4.vhdx`) bleibt dadurch schlank, während die Modelle physisch auf Host-Laufwerk `I:` verbleiben. Innerhalb des Images freigegebener Speicher steht sofort für neue Modelle bereit.
* **OLLAMA_GPU_OVERHEAD Kalibrierung:** Reduziert von 4 GB auf 2 GB. Da Ollama freien VRAM dynamisch via CUDA ermittelt, blockiert ein zu hoher statischer Overhead wertvollen Speicher. 2 GB reichen als Puffer für WSL2-GUI-Apps (z.B. Obsidian via WSLg) vollständig aus.
* **Inaktivitäts-Timeout (Keep-Alive):** Reduziert von 1h auf 10m. Sichert eine zügige Freigabe des VRAMs für Windows-Host-Anwendungen (Browsing, Gaming), sobald Agenten-Loops oder Chats pausieren.
* **Dynamische Kontext-Steuerung:** Da Ollama standardmäßig nur kleine Kontexte (2k/4k) lädt und keine globale Umgebungsvariable für den Kontext existiert, wird das Limit flexibel und anwendungsspezifisch gesteuert. Über die aufrufende Anwendung (z. B. Python via `uv` oder Web-UIs) wird der Parameter `num_ctx` zur Laufzeit übergeben (erfolgreich getestet mit 16.384 Token bei ~18,6 GB VRAM-Gesamtauslastung).
