# Lokales LLM-Setup: WSL2 Sandbox & Ollama (Hybrid-Loop-Architektur)

Dieses Dokument dient als persistenten Zustand (State) für LLM-Sessions. Es beschreibt die aktuelle Systemlandschaft, Konfigurationsdatei, optimierte Parameter und den Betrieb lokaler Modelle mit strikter Trennung von Host und Sandbox.

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

[user]
default=seppo007

# Do not allow default interaction with windows .exe files and don't populate path with host system path
[interop]
enabled=false
appendWindowsPath=false

# Do not automatically mount all available partitions of the host system
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
* **Lokale Websuche via SearXNG & Podman (Juli 2026):** Eine datenschutzfreundliche Metasuchmaschine (SearXNG) wurde als automatisierter, rootless Systemd-Dienst (`container-searxng.service`) auf Port `8888` eingerichtet.
* **API-Sperren-Behebung (403-Fix) (Juli 2026):** Die standardmäßige JSON-Blockade von SearXNG wurde über eine globale Einstellungsdatei unter `/etc/searxng/settings.yml` behoben. Das JSON-Format wurde sicher für programmatische Abfragen freigeschaltet, während der Server netzwerktechnisch an den lokalen Sandbox-Host gebunden bleibt (`127.0.0.1`).
* **Nativer Agenten-Host via OpenAI-SDK (Juli 2026):** Im Verzeichnis `~/llm-agent` wurde eine `uv`-Umgebung mit einem Python-Agenten-Host aufgesetzt. Das Modell `mistral-small3.2:latest` agiert nun erfolgreich im Multi-Tool-Modus, bricht seinen Trainings-Cutoff und steuert parallel die lokale Googlesuche sowie Schreiboperationen für den Obsidian-Tresor an. Der Parametric Knowledge Bias wurde durch sauberes Prompting eliminiert.

---

## 6. Dokumentation: Lokale Websuche & Agenten-Setup (Juli 2026)

Dieses Kapitel dokumentiert die exakte Einrichtung der privaten Suchmaschinen-API und des Python-Agenten-Hosts in der WSL2-Sandbox.

### 6.1 SearXNG API-Infrastruktur (Podman & Systemd)

Um die Blockade des JSON-Formats (Fehler `403 Forbidden`) und UID-Rechteprobleme im Rootless-Modus zu umgehen, nutzt das System eine globale Konfigurationsdatei und eine bereinigte Systemd-Struktur.

#### 1. Globale Konfigurationsdatei (`/etc/searxng/settings.yml`)
Deaktiviert den Bot-Schutz für die Sandbox und schaltet das JSON-Format explizit frei.
```yaml
use_default_settings: true

server:
  port: 8080
  bind_address: "0.0.0.0"
  secret_key: "Lp8to5RdH5Jrbmdi95RgRC6n6E9njo6DoSGL6cDi"

search:
  formats:
    - html
    - json
```

#### 2. Systemd-Dienstdatei (`~/.config/systemd/user/container-searxng.service`)
Spiegelt die Konfiguration über ein sicheres Volume-Flag (`:ro`) in den Container, leitet Port `8888` an den Localhost weiter und startet vollautomatisch mit WSL2.
```ini
# container-searxng.service
# autogenerated by Podman 4.9.3
# Wed Jul  1 18:53:02 CEST 2026

[Unit]
Description=Podman container-searxng.service
Documentation=man:podman-generate-systemd(1)
Wants=network-online.target
After=network-online.target
RequiresMountsFor=%t/containers

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=always
TimeoutStopSec=70
ExecStart=/usr/bin/podman run \
        --cidfile=%t/%n.ctr-id \
        --cgroups=no-conmon \
        --rm \
        --sdnotify=conmon \
        --replace \
        -d \
        --name searxng \
        -p 127.0.0.1:8888:8080 \
        -v /etc/searxng/settings.yml:/etc/searxng/settings.yml:ro \
        docker.io/searxng/searxng:latest
ExecStop=/usr/bin/podman stop \
        --ignore -t 10 \
        --cidfile=%t/%n.ctr-id
ExecStopPost=/usr/bin/podman rm \
        -f \
        --ignore -t 10 \
        --cidfile=%t/%n.ctr-id
Type=notify
NotifyAccess=all

[Install]
WantedBy=default.target
```

#### 3. Befehle zur Aktivierung
```bash
systemctl --user daemon-reload
systemctl --user enable --now container-searxng.service
# Funktionstest (erwartet valide JSON-Struktur)
curl -s "http://localhost:8888/search?q=Bundeskanzler+Deutschland&format=json"
```

---

### 6.2 Python Agenten-Host (`~/llm-agent`)

Der Agenten-Host steuert den Hybrid-Loop. Er verwaltet die Werkzeuge, fängt die Funktionsaufrufe des Modells ab und speist Ergebnisse ein.

#### 1. Projekt-Initialisierung (via `uv`)
```bash
mkdir -p ~/llm-agent && cd ~/llm-agent
uv init
uv add httpx openai
```



#### 2. Der Multi-Tool-Quellcode (`agent.py`)
```python
import os
import json
import httpx
from openai import OpenAI

# 1. Parameter & Pfade definieren
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")
SEARXNG_URL = "http://localhost:8888/search"
MODEL_NAME = "mistral-small3.2:latest"

OBSIDIAN_VAULT_DIR = "/home/seppo007/obsidian-vault/KI-Generiert"
os.makedirs(OBSIDIAN_VAULT_DIR, exist_ok=True)

# 2. Tool 1: Lokale Websuche (SearXNG API)
def web_search(query: str) -> str:
    try:
        response = httpx.get(SEARXNG_URL, params={"q": query, "format": "json"}, timeout=5.0)
        data = response.json()
        results = data.get("results", [])[:3]
        if not results:
            return "Keine Ergebnisse über Google gefunden."
            
        summary = []
        for r in results:
            summary.append(f"Titel: {r.get('title')}\nURL: {r.get('url')}\nInhalt: {r.get('content')}\n---")
        return "\n".join(summary)
    except Exception as e:
        return f"Fehler bei der Websuche: {str(e)}"

# 3. Tool 2: Datei in Obsidian schreiben
def write_obsidian_note(filename: str, content: str) -> str:
    try:
        if not filename.endswith(".md"):
            filename += ".md"
        safe_filename = os.path.basename(filename)
        full_path = os.path.join(OBSIDIAN_VAULT_DIR, safe_filename)
        
        with open(full_path, "w", encoding="utf-8") as f:
            f.write(content)
        return f"Erfolgreich! Die Notiz wurde unter '{full_path}' gespeichert."
    except Exception as e:
        return f"Fehler beim Schreiben der Notiz: {str(e)}"

# 4. Tool-Deklaration für OpenAI-Schema
tools = [
    {
        "type": "function",
        "function": {
            "name": "web_search",
            "description": "Durchsucht das Internet via Google nach aktuellen Informationen.",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "Der Suchbegriff für Google"}
                },
                "required": ["query"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "write_obsidian_note",
            "description": "Erstellt eine neue strukturierte Markdown-Notiz im Obsidian-Vault.",
            "parameters": {
                "type": "object",
                "properties": {
                    "filename": {"type": "string", "description": "Der Name der Notiz (z.B. bundeskanzler.md)"},
                    "content": {"type": "string", "description": "Der vollständige Inhalt im Markdown-Format"}
                },
                "required": ["filename", "content"]
            }
        }
    }
]

def run_agent(prompt: str):
    print(f"\n[User]: {prompt}")
    
    messages = [
        {
            "role": "system",
            "content": "Du bist ein autonomer KI-Analyst im Jahr 2026. Du kannst das Internet durchsuchen UND Notizen direkt in Obsidian anlegen. Nutze deine Werkzeuge intelligent. Suchergebnisse haben oberste Priorität vor deinem Trainingswissen."
        },
        {"role": "user", "content": prompt}
    ]
    
    response = client.chat.completions.create(
        model=MODEL_NAME, messages=messages, tools=tools, tool_choice="auto"
    )
    response_message = response.choices[0].message
    
    if response_message.tool_calls:
        messages.append(response_message)
        
        for tool_call in response_message.tool_calls:
            function_name = tool_call.function.name
            args = json.loads(tool_call.function.arguments)
            
            if function_name == "web_search":
                print(f"\n[Agent entscheidet]: Suche im Web nach: '{args['query']}'...")
                result = web_search(args["query"])
            elif function_name == "write_obsidian_note":
                print(f"\n[Agent entscheidet]: Schreibe Obsidian-Notiz '{args['filename']}'...")
                result = write_obsidian_note(args["filename"], args["content"])
                
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "name": function_name,
                "content": result
            })
        
        final_response = client.chat.completions.create(
            model=MODEL_NAME, messages=messages
        )
        print(f"\n[Agent Antwort]: {final_response.choices[0].message.content}")
    else:
        print(f"\n[Agent Antwort (Keine Tools)]: {response_message.content}")

# Ausführung des Agenten-Tasks
run_agent("Recherchiere wer aktuell Bundeskanzler von Deutschland ist und erstelle mir dazu eine strukturierte Informationsseite als Obsidian-Notiz mit dem Namen bundeskanzler.md")
```
