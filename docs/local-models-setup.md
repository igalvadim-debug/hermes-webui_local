# Hermes WebUI mit lokalen Modellen betreiben (start.py + llama-server)

Diese Anleitung ergänzt [`docs/docs/models.md`](docs/models.md) (Bestandsliste aller
Modell-Dateien auf der Maschine) und zeigt, wie Hermes WebUI mit den lokalen
Modellen läuft, die über [`docs/start.py`](start.py) gestartet werden.

Kurzversion:

1. `python docs\start.py [Modellnummer]` startet den lokalen OpenAI-kompatiblen
   Server `llama-server` auf **http://127.0.0.1:8484** (Port 8484 ist fest in
   `start.py` gesetzt; das Modell wird per Nummer gewählt, Kontext per `--ctx N`).
2. In Hermes WebUI (Onboarding-Wizard oder **Settings → Providers**) einen
   Self-hosted / Custom-OpenAI-kompatiblen Provider anlegen mit Base URL
   `http://127.0.0.1:8484/v1` (API-Key kann leer bleiben).
3. **Test connection** drücken – die Modelle kommen aus `/v1/models` des laufenden
   llama-servers. Details zum Wizard: [`docs/onboarding.md`](onboarding.md).

## Welche Modelle start.py anbietet

`start.py` (Windows, Windows Terminal mit Tabs) kennt aktuell diese Modelle –
sie stammen alle aus der Liste in [`docs/docs/models.md`](docs/models.md):

| Nr. | GGUF-Datei | Typ / Hinweise |
|---|---|---|
| 1 | `Qwen2.5-Coder-7B-Instruct-Q4_K_M.gguf` | Coding, GPU (`-ngl 99`) |
| 2 | `Qwopus3.5-4B-Coder-MTP-Q8_0.gguf` | Coding (MTP), GPU |
| 3 | `gemma-4-E4B-it-heretic-Q4_K_M.gguf` | Allgemein, GPU |
| 4 | `Qwen3.8-2B-BF16.gguf` | Klein/schnell, 32K+ Kontext |
| 5 | `Gemma-4-E2B-Uncensored-HauhauCS-Aggressive-Q6_K_P.gguf` | Vision (mit `mmproj-*-f16.gguf`) |
| 6 | `Qwopus-MoE-35B-A3B-APEX-I-Mini.gguf` | MoE 35B → `--fit on`, gepatchtes Chat-Template |
| 7 | `Qwen3.6-35B-A3B-abliterated.IQ4_XS.gguf` | MoE 35B → `--fit on` |

Zusätzlich fährt `start.py` einen Embedding-Server (embeddinggemma-2) auf Port
**8512** (`/v1/embeddings`) sowie drei MCP-Dienste hoch:

| Dienst | Adresse | Zweck in Hermes WebUI |
|---|---|---|
| llama-server (Chat) | `http://127.0.0.1:8484/v1` | LLM-Provider (OpenAI-kompatibel) |
| embeddinggemma | `http://127.0.0.1:8512/v1` | Embeddings (LanceDB-Index) |
| Filesystem MCP | `http://127.0.0.1:8001/mcp` | optionale MCP-Anbindung |
| RepoMapper MCP | `http://127.0.0.1:8002/mcp` | optionale MCP-Anbindung |
| LanceDB MCP | `http://127.0.0.1:8003/mcp` | optionale MCP-Anbindung |

Das als „Modell“ eingetragene Ergebnis von `llama-server --alias …` bzw. der
Dateiname erscheint danach im Modell-Dropdown von WebUI. Die bereits
downloadeten Modelle sind zusätzlich in [`docs/docs/local_models.json`](docs/local_models.json)
als Register erfasst (install.py lädt nichts, was dort schon auf der Platte liegt).

## Provider in Hermes WebUI eintragen

### Variante A: Onboarding-Wizard (Erststart)

1. WebUI öffnen (`http://localhost:8787`).
2. Provider-Gruppe **„Open / self-hosted“** wählen
   (Ollama, LM Studio oder *custom OpenAI-compatible*).
3. Base URL: `http://127.0.0.1:8484/v1`
4. API-Key-Feld leer lassen (lokaler Server ist keyless).
5. **Test connection** → Modellliste wird gefüllt → Modell wählen, speichern.

### Variante B: config.yaml direkt

Die aktiven Pfade zeigt der Wizard unter „Config paths“
(Standard: `%LOCALAPPDATA%\hermes\config.yaml` unter Windows, `~/.hermes/config.yaml`
unter Linux/WSL). Beispiel für einen Custom-Provider:

```yaml
providers:
  local-llama:
    base_url: http://127.0.0.1:8484/v1
    api_key: ""            # keyless – Feld kann fehlen/leer sein
model: qwen2.5-coder-7b-instruct-q4_k_m.gguf   # Name aus /v1/models
```

Danach WebUI neu laden; der Eintrag gilt als **Configured**, erreichbar ist er erst,
wenn `start.py` läuft (das prüft **Test connection**, nicht das Label).

## Start-Reihenfolge und Ports

1. Zuerst `python docs\start.py 1` (oder andere Nummer; optional `--ctx 262144`
   bzw. Umgebungsvariable `LLAMA_CTX`) – wartet selbst bis Port 8484 offen ist.
2. Dann Hermes WebUI starten (`bootstrap.py` / `start.sh` / Docker).
3. Ältere Prozesse auf den Ports 8001–8003, 8484, 8512 beendet `start.py`
   automatisch (`kill_old_processes`), es muss also nichts manuell abgeschossen werden.

Prüfen, ob der Chat-Server erreichbar ist:

```powershell
curl http://127.0.0.1:8484/v1/models
```

## Docker / WSL: `localhost` beachten

Hermes WebUI läuft offiziell unter Linux/macOS/WSL2 (siehe
[`docs/onboarding.md`](onboarding.md)). Der llama-server aus `start.py` läuft auf
dem Windows-Host:

| WebUI-Umgebung | Base URL für llama-server |
|---|---|
| Native Windows / WSL2 mit Host-Netzwerk | `http://127.0.0.1:8484/v1` |
| WSL2 (klassisches NAT) | `http://<Windows-Host-IP>:8484/v1` (z. B. Nameserver-IP aus `/etc/resolv.conf`) |
| Docker Desktop | `http://host.docker.internal:8484/v1` |
| Linux Docker Engine | `http://api.local:8484/v1` + `extra_hosts: ["api.local:host-gateway"]` im Compose-File |

Von einem Container aus bedeutet `localhost` den Container selbst – nie den
Windows-Host, auf dem `start.py` den llama-server startet.

## Probleme

- **Kein Modell in der Liste:** llama-server läuft nicht oder Base URL ohne `/v1`.
  Der Wizard probe `<base-url>/models`; scheitert das, bleibt der Schritt gesperrt.
- **Vision-Modell (Nr. 5):** lädt nur mit passendem `mmproj-…-f16.gguf`; fehlt er,
  bricht `check_prerequisites` mit einer Fehlermeldung ab.
- **MoE-Modelle (Nr. 6/7):** bewusst ohne `-ngl 99`, sondern `--fit on`
  (CPU/GPU-Mischung bei 8 GB VRAM). Chat funktioniert trotzdem über dieselbe Base URL.
- **AIAgent not available / Verbindungsprobleme:** siehe
  [`docs/troubleshooting.md`](troubleshooting.md).
- **Neue Modelle aufnehmen:** Pfad in `MODELS` in `docs/start.py` ergänzen und den
  Dateipfad zusätzlich in [`docs/docs/models.md`](docs/models.md) bzw.
  [`docs/docs/local_models.json`](docs/local_models.json) registrieren.
