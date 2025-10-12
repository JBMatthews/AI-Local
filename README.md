# Home AI Server (LAN-Only) — Ollama + Open WebUI

> **Build your own “mini ChatGPT” at home**—private, fast, and free.  
> Because sometimes you want AI to analyze your notes without mailing them to a mysterious server farm with a nicer view than your office.

This repo shows you how to turn a spare PC into a **local AI server** that anyone in your household can use from a browser. We’ll run:

- **Ollama** — the local model server (LLMs + embeddings)  
- **Open WebUI** — the friendly web interface with accounts, history, and optional RAG

Everything is **LAN-only** (no port forwarding), so your prompts and documents **never leave your house**.

---

## Why run AI locally?

1. **Privacy & Data Ownership**  
   Your prompts and files never leave your machine. No logs, no telemetry, no third-party storage.

2. **Speed & Offline Access**  
   No internet latency or API rate limits. Works even when your internet doesn’t.

3. **Zero Ongoing Cost**  
   After setup, every token is free. Great for heavy use and experiments.

4. **Full Control & Customization**  
   Choose your models, tune behavior, integrate with anything on your network.

5. **Reliability & Independence**  
   No vendor outages, no key expirations, no region restrictions.

6. **Real Learning**  
   Understand quantization, embeddings, and how inference servers actually work.

7. **Security & Compliance**  
   Keep sensitive data behind your own firewall. Easier to meet internal rules.

8. **Experimentation Freedom**  
   Try new open models, benchmark them, wire them into your tools or apps.

---

## What you’ll build

- A **Windows 10 Pro** PC (the “AI box”) running Docker containers for:
  - `ollama/ollama:latest` on port **11434**
  - `open-webui` on port **3000**
- Accessed from any device on your **home network** (Mac/PC/phone) via browser.

> Tested with: Dell Optiplex 7020 Tower, i7-4770 @ 3.4GHz, **32GB RAM**, 240GB SSD + 1TB HDD, Windows 10 Pro.  
> No discrete GPU required (we’ll use **quantized 7B/8B models** for good CPU performance).

---

## At a glance

```text
[ Mac / iPad / PC ]  ──(http://AI-BOX:3000)──>  [ Open WebUI ]
                                              ↘
                                               └─(http://AI-BOX:11434)─> [ Ollama (LLMs + embeddings) ]
```

- **LAN-only**: No router port forwarding.  
- **Auth**: First user in Open WebUI becomes admin; add household accounts there.

---

## Prerequisites

- A spare **Windows 10 Pro** PC (on Ethernet/Wi-Fi)
- Admin access to your home router (Xfinity example below)
- **Docker Desktop** for Windows (WSL2 enabled)
- Your **home network set as “Private”** in Windows

---

## Lesson 1 — Network Basics

### Step 1: Give the AI box a fixed LAN IP (DHCP reservation)

**Why:** Your household needs a stable address like `http://10.0.0.6:3000`.

1. On the Windows PC, find the MAC address:  
   Open **PowerShell** → `ipconfig /all` → copy the **Physical Address** for your active adapter.
2. In your router’s admin (Xfinity users):
   - Go to **Xfinity app** → **Advanced Settings** → **LAN / Reserved IP** (often under *Port Forwarding* → *See connected devices* → *Reserve IP*).  
   - Reserve an IP for the PC, e.g., **10.0.0.6**.
3. Reboot the PC or disable/enable the adapter and confirm the new IP:
   - `ping 10.0.0.6` from your Mac/another device.

> **Do not** create any **port forwarding** rules. We are staying LAN-only.

### Step 2: Prevent sleep

On the Windows PC: **Control Panel → Power Options → High performance**  
- Put the computer to sleep: **Never**  
- Turn off hard disk: **Never** (or long interval)

### Step 3: Windows Firewall (open only for your Private LAN)

Run **PowerShell as Administrator** and add these two inbound rules:

```powershell
netsh advfirewall firewall add rule name="Ollama 11434 (Private)" dir=in action=allow protocol=TCP localport=11434 profile=Private
netsh advfirewall firewall add rule name="Open WebUI 3000 (Private)" dir=in action=allow protocol=TCP localport=3000 profile=Private
```

Confirm:
```powershell
netsh advfirewall firewall show rule name=all | findstr 11434
netsh advfirewall firewall show rule name=all | findstr 3000
```

Make sure your network profile is **Private** (Settings → Network & Internet → your network → *Private*).

---

## Lesson 2 — Install & Launch (Docker)

### Step 1: Install Docker Desktop (Windows)

- Download: https://www.docker.com/products/docker-desktop/  
- Keep defaults; ensure **“Use WSL 2 based engine”** is checked.  
- Start Docker and wait for **“Docker Engine is running.”**

### Step 2: Create the project folder and compose file

Create `C:\ai\docker-compose.yml` with:

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    restart: unless-stopped
    ports:
      - "11434:11434"
    environment:
      - OLLAMA_HOST=0.0.0.0
    volumes:
      - ollama:/root/.ollama

  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    depends_on:
      - ollama
    restart: unless-stopped
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - WEBUI_NAME=Home AI
    volumes:
      - openwebui:/app/backend/data

volumes:
  ollama:
  openwebui:
```

> Tip: SSD storage improves first-token latency and model load times.

### Step 3: Start the stack

Open **PowerShell as Administrator**:

```powershell
cd C:\ai
docker compose pull
docker compose up -d
docker ps
```

You should see both containers **Up**.

---

## Lesson 3 — First Use (from your Mac or any device)

1. Open **http://10.0.0.6:3000**  
   - Create the first account → this is your **admin** in Open WebUI.
2. In **Settings → Models**, choose **Ollama** as the provider.
3. Pull a starter model (good CPU-friendly picks):
   - `phi3:mini` *(snappy, tiny)*
   - `mistral:7b-instruct` *(balanced)*
   - `llama3.1:8b-instruct` *(a bit larger; still fine on 32 GB with quantization)*
   - For embeddings (RAG): `nomic-embed-text`
4. Start chatting. First response may be slower while the model warms up.

### Quick API tests

From your Mac:

```bash
# Can you reach the PC?
ping 10.0.0.6

# Is Ollama alive?
curl http://10.0.0.6:11434/api/tags

# Open the web UI
open http://10.0.0.6:3000
```

---

## Model notes (CPU-only)

- Prefer **quantized** models (GGUF / q4_K_M or q5_0) for memory and speed.  
- 7B/8B class models run well on an i7-4770 + 32 GB RAM.  
- Keep active models on the SSD; archive extras on HDD if needed.

---

## Security checklist

- ✅ **LAN-only**: No router port forwarding.  
- ✅ **Open WebUI auth**: Create accounts for household members.  
- ✅ **Windows network = Private**: Firewall rules apply only on your home LAN.  
- ✅ Optional remote access: Use **Tailscale/WireGuard** (do **not** expose ports to the internet).

---

## Troubleshooting

- **Can’t reach `http://10.0.0.6:3000`**  
  - Verify Docker is running: `docker ps` on the PC.  
  - Confirm firewall rules (Private profile).  
  - Ensure your Mac is on the same SSID/LAN (not guest network).

- **Models won’t download in Open WebUI**  
  - Open WebUI → Settings → Models → Provider is **Ollama**.  
  - Or pull via CLI on the PC:  
    ```powershell
    docker exec -it $(docker ps -qf "name=ollama") ollama pull mistral:7b-instruct
    ```

- **Slow generations**  
  - Try a smaller quant (`q4_K_M`) or a smaller model (`phi3:mini`).  
  - Close other heavy apps on the PC; keep models on SSD.

---

## Add-ons (Optional)

- **RAG in Open WebUI**: Settings → **Knowledge** → create a collection → upload PDFs → ask questions.  
- **AnythingLLM**: Multi-workspace RAG UI (can be added to `docker-compose.yml`).  
- **whisper.cpp**: Local speech-to-text server for voice notes.

---

## Maintenance

Update to latest images:

```powershell
cd C:\ai
docker compose pull
docker compose up -d
```

Back up by copying the volumes (`ollama`, `openwebui`) or by switching to host-path mounts and backing up those folders.

---

## License

Use, fork, and adapt freely. If this saves you a few API dollars, buy yourself a coffee and enjoy your sovereign AI. ☕️

---

### Author Notes

- Built for a home LAN with **Windows AI box + Mac clients**, but works cross-platform.  
- If your router UI differs from Xfinity’s, any DHCP reservation feature works the same: bind the PC’s **MAC address** to a **fixed IP** (e.g., `10.0.0.6`).
