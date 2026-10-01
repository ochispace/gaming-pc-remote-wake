Wake and shut down your gaming PC from anywhere (Raspberry Pi + Tailscale)

This setup lets you wake up and cleanly shut down your gaming PC from anywhere (even over mobile data/hotspot) through a simple webpage on your phone. No port forwarding, no open port on the internet, works even behind CGNAT (common on mobile routers).

How it works: A Raspberry Pi runs permanently at home on the same network as your PC. It is reachable from anywhere via Tailscale (a free private VPN), with no port forwarding needed. A small webpage on the Pi lets you send a Wake-on-LAN signal to the PC (wake it up), or has the Pi log into the PC over SSH with a tightly restricted account that is only allowed to run the shutdown command (shut it down).

What you need
A Raspberry Pi (even an old/cheap Zero W works) running Raspberry Pi OS, on the same home network as your PC
A Windows gaming PC, connected to the router via Ethernet cable (Wi-Fi generally doesn't work reliably for Wake-on-LAN)
A free Tailscale account
Wake-on-LAN needs to be enabled later in the BIOS and in the Windows network adapter (can't be automated, see below)
Step 1: Find your PC's IP address and MAC address

On the gaming PC, open Command Prompt (cmd):

ipconfig /all

Note down the IPv4 address of your Ethernet adapter (e.g. 10.0.0.47) and the Physical Address / MAC address (e.g. D8-5E-D3-AE-8A-EB — replace the dashes with colons when entering it later: D8:5E:D3:AE:8A:EB).

Step 2: Run the Pi setup

Copy the code below into a file called setup-pi.sh on the Pi (e.g. with nano setup-pi.sh, paste, save) and run it:

bash setup-pi.sh

The script installs Tailscale and wakeonlan, logs you into Tailscale (a link to log in will appear), interactively asks for your PC's IP, MAC address, and a password for the web interface, and sets up a background service that starts automatically on every Pi reboot.

bash
#!/bin/bash
# ============================================================
#  Gaming PC Remote Control - Raspberry Pi Setup
# ============================================================
# Run with:  bash setup-pi.sh
# (don't run with sudo - the script asks for sudo itself when needed)

set -e

echo "=================================================="
echo "  Gaming PC Remote Control - Pi Setup"
echo "=================================================="
echo

# ---------- 1. Install packages ----------
echo "[1/6] Installing required packages (wakeonlan, curl) ..."
sudo apt-get update -qq
sudo apt-get install -y -qq wakeonlan curl

# ---------- 2. Install Tailscale ----------
if ! command -v tailscale &> /dev/null; then
    echo "[2/6] Installing Tailscale ..."
    curl -fsSL https://tailscale.com/install.sh | sh
else
    echo "[2/6] Tailscale is already installed."
fi

if ! sudo tailscale status &> /dev/null; then
    echo
    echo ">>> Please log in now using the link shown (Google/Microsoft/Email) <<<"
    sudo tailscale up
fi

MY_TS_IP=$(tailscale ip -4)
MY_LOCAL_IP=$(ip route get 1.1.1.1 2>/dev/null | awk '{print $7; exit}')

# ---------- 3. Generate SSH key ----------
echo "[3/6] Generating SSH key for the shutdown feature ..."
if [ ! -f ~/.ssh/pc_key ]; then
    ssh-keygen -t ed25519 -f ~/.ssh/pc_key -N "" -q
fi
PUB_KEY=$(cat ~/.ssh/pc_key.pub)

# ---------- 4. Interactive prompts ----------
echo
echo "[4/6] A few details about your gaming PC:"
read -p "  IPv4 address of the gaming PC on your home network (e.g. 10.0.0.47): " PC_IP
read -p "  MAC address of the gaming PC (format AA:BB:CC:DD:EE:FF): " MAC_ADDRESS
read -p "  Windows username for the shutdown feature [shutdownbot]: " PC_USER
PC_USER=${PC_USER:-shutdownbot}

while true; do
    read -s -p "  Set a password for the web interface: " WEBPASS
    echo
    read -s -p "  Repeat password: " WEBPASS2
    echo
    if [ "$WEBPASS" == "$WEBPASS2" ] && [ -n "$WEBPASS" ]; then
        break
    fi
    echo "  Passwords don't match or are empty, try again."
done

# ---------- 5. Write the server script ----------
echo "[5/6] Writing web server script ..."
cat > ~/wol_server.py << 'PYEOF'
from http.server import HTTPServer, BaseHTTPRequestHandler
import subprocess
import hmac
import time

MAC_ADDRESS = "__MAC__"
PC_USER = "__PCUSER__"
PC_IP = "__PCIP__"
PASSWORD = "__PASSWORD__"

HTML_PAGE = """
<!DOCTYPE html>
<html>
<head>
<title>Gaming PC</title>
<meta name="viewport" content="width=device-width, initial-scale=1">
<style>
body { font-family: sans-serif; background: #1a1a1a; color: white; text-align: center; padding-top: 60px; }
input { font-size: 22px; padding: 14px; border-radius: 10px; border: none; width: 220px; text-align: center; }
button { font-size: 24px; padding: 28px 0; width: 260px; color: white; border: none; border-radius: 15px; margin: 15px; }
#on { background: #4CAF50; }
#off { background: #c0392b; }
#status { margin-top: 20px; font-size: 18px; }
#statusrow { margin-top: 30px; font-size: 20px; display: flex; align-items: center; justify-content: center; gap: 10px; }
.dot { width: 16px; height: 16px; border-radius: 50%; background: #777; display: inline-block; transition: background 0.3s; }
.dot.on { background: #4CAF50; }
.dot.off { background: #555; }
</style>
</head>
<body>
<h1>Gaming PC</h1>
<div id="statusrow"><span class="dot" id="dot"></span><span id="statustext">Checking...</span></div>
<input type="password" id="pw" placeholder="Password" style="margin-top:30px"><br>
<button id="on" onclick="send('/wake')">Wake Up</button><br>
<button id="off" onclick="if(confirm('Really shut down the PC?')) send('/shutdown')">Shut Down</button>
<div id="msg"></div>
<script>
function send(path) {
  var pw = document.getElementById('pw').value;
  if (!pw) { document.getElementById('msg').innerText = "Please enter a password."; return; }
  document.getElementById('msg').innerText = "Sending...";
  fetch(path, {headers: {'X-Pw': pw}}).then(r => r.text()).then(t => {
    document.getElementById('msg').innerText = t;
    document.getElementById('pw').value = "";
    setTimeout(checkStatus, 3000);
  });
}
function checkStatus() {
  fetch('/status').then(r => r.text()).then(t => {
    var dot = document.getElementById('dot');
    var txt = document.getElementById('statustext');
    if (t === "online") {
      dot.className = "dot on";
      txt.innerText = "PC is on";
    } else {
      dot.className = "dot off";
      txt.innerText = "PC is off";
    }
  });
}
checkStatus();
setInterval(checkStatus, 5000);
</script>
</body>
</html>
"""

class Handler(BaseHTTPRequestHandler):
    def reply(self, text, ctype="text/plain", code=200):
        self.send_response(code)
        self.send_header("Content-type", ctype + "; charset=utf-8")
        self.end_headers()
        self.wfile.write(text.encode())

    def pw_ok(self):
        given = self.headers.get("X-Pw", "")
        ok = hmac.compare_digest(given.encode(), PASSWORD.encode())
        if not ok:
            time.sleep(2)
        return ok

    def do_GET(self):
        if self.path == "/status":
            r = subprocess.run(["ping", "-c", "1", "-W", "1", PC_IP],
                                stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
            self.reply("online" if r.returncode == 0 else "offline")
        elif self.path == "/wake":
            if not self.pw_ok():
                self.reply("Wrong password.", code=403)
                return
            subprocess.run(["wakeonlan", MAC_ADDRESS])
            self.reply("Signal sent!")
        elif self.path == "/shutdown":
            if not self.pw_ok():
                self.reply("Wrong password.", code=403)
                return
            r = subprocess.run(["ssh", "-i", "/home/" + "__WHOAMI__" + "/.ssh/pc_key",
                                "-o", "StrictHostKeyChecking=accept-new",
                                "-o", "ConnectTimeout=5",
                                PC_USER + "@" + PC_IP, "shutdown /s /t 0"])
            self.reply("PC is shutting down." if r.returncode == 0 else "Error shutting down.")
        else:
            self.reply(HTML_PAGE, "text/html")

server = HTTPServer(("0.0.0.0", 8080), Handler)
print("Server running on port 8080")
server.serve_forever()
PYEOF

WHOAMI=$(whoami)
sed -i "s/__MAC__/$MAC_ADDRESS/" ~/wol_server.py
sed -i "s/__PCUSER__/$PC_USER/" ~/wol_server.py
sed -i "s/__PCIP__/$PC_IP/" ~/wol_server.py
sed -i "s/__PASSWORD__/$WEBPASS/" ~/wol_server.py
sed -i "s/__WHOAMI__/$WHOAMI/" ~/wol_server.py

# ---------- 6. Set up the systemd service ----------
echo "[6/6] Setting up a service to keep the server running in the background ..."
sudo bash -c "cat > /etc/systemd/system/wolserver.service" << SVCEOF
[Unit]
Description=WOL Webserver
After=network.target

[Service]
ExecStart=/usr/bin/python3 $HOME/wol_server.py
WorkingDirectory=$HOME
Restart=always
User=$WHOAMI

[Install]
WantedBy=multi-user.target
SVCEOF

sudo systemctl daemon-reload
sudo systemctl enable wolserver --quiet
sudo systemctl restart wolserver

echo
echo "=================================================="
echo "  Done! Web interface: http://$MY_TS_IP:8080"
echo "=================================================="
echo
echo "NEXT STEP - on the gaming PC (Windows):"
echo "  1. Run setup-windows.ps1 as Administrator"
echo "  2. Enter these values when asked:"
echo
echo "     This Pi's local IP (for the Windows firewall):"
echo "       $MY_LOCAL_IP"
echo
echo "     This Pi's public key:"
echo "       $PUB_KEY"
echo
echo "Still to do: install the Tailscale app on your phone, log into the"
echo "same account, and enable Wake-on-LAN in the BIOS + Windows adapter."
echo "=================================================="
Step 3: Run the Windows setup

On the gaming PC: copy the code below into a file called setup-windows.ps1. Right-click the file -> "Run with PowerShell (as Administrator)". If Windows shows a SmartScreen warning ("Windows protected your PC"), click "More info" then "Run anyway" (normal for self-written/unsigned scripts).

The script will ask for the Pi's local IP and the public key that setup-pi.sh printed at the end.

powershell
# ============================================================
#  Gaming PC Remote Control - Windows Setup
# ============================================================
# IMPORTANT: Run as Administrator!
#   Right-click the file -> "Run with PowerShell (as Administrator)"
#   or from an Administrator PowerShell: .\setup-windows.ps1
#
# Asks for the needed values interactively if not passed as parameters:
#   .\setup-windows.ps1 -PiLocalIP "10.0.0.56" -PublicKey "ssh-ed25519 AAAA..."

param(
    [string]$PiLocalIP,
    [string]$PublicKey
)

# ---------- 0. Admin check ----------
$isAdmin = ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltinRole]::Administrator)
if (-not $isAdmin) {
    Write-Host "This script must run as Administrator. Please restart it (right-click -> Run as Administrator)." -ForegroundColor Red
    Read-Host "Press Enter to exit"
    exit 1
}

Write-Host "=================================================="
Write-Host "  Gaming PC Remote Control - Windows Setup"
Write-Host "=================================================="
Write-Host ""

if (-not $PiLocalIP) {
    $PiLocalIP = Read-Host "Local IPv4 address of the Raspberry Pi (e.g. 10.0.0.56)"
}
if (-not $PublicKey) {
    $PublicKey = Read-Host "Pi's public SSH key (the whole line, starts with ssh-ed25519)"
}
$PublicKey = $PublicKey.Trim()

$ACCOUNT = "shutdownbot"

# ---------- 1. Install OpenSSH Server ----------
Write-Host "[1/9] Checking OpenSSH Server ..."
$cap = Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'
if ($cap.State -ne "Installed") {
    Write-Host "      Installing OpenSSH Server ..."
    Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0 | Out-Null
} else {
    Write-Host "      already installed."
}

# ---------- 2. Create the account ----------
Write-Host "[2/9] Setting up restricted account '$ACCOUNT' ..."
$existingUser = Get-LocalUser -Name $ACCOUNT -ErrorAction SilentlyContinue

# Generate a random, strong password - it's never needed manually,
# since the account only ever logs in via SSH key.
Add-Type -AssemblyName System.Web -ErrorAction SilentlyContinue
function New-RandomPassword {
    $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'
    -join ((1..24) | ForEach-Object { $chars[(Get-Random -Maximum $chars.Length)] })
}
$plainPw = New-RandomPassword
$securePw = ConvertTo-SecureString $plainPw -AsPlainText -Force

if (-not $existingUser) {
    New-LocalUser -Name $ACCOUNT -Password $securePw -Description "Restricted to shutting down via SSH" -PasswordNeverExpires | Out-Null
} else {
    Set-LocalUser -Name $ACCOUNT -Password $securePw
}

# Add to the normal Users group (required, otherwise login is denied)
try {
    Add-LocalGroupMember -SID "S-1-5-32-545" -Member $ACCOUNT -ErrorAction Stop
} catch {
    # already a member - fine
}

# ---------- 3. Force profile creation ----------
Write-Host "[3/9] Creating user profile ..."
$cred = New-Object System.Management.Automation.PSCredential($ACCOUNT, $securePw)
try {
    $p = Start-Process cmd -Credential $cred -ArgumentList '/c exit' -WorkingDirectory "C:\" -WindowStyle Hidden -PassThru -ErrorAction Stop
    Start-Sleep -Seconds 2
} catch {
    Write-Host "      Note: profile creation reported: $($_.Exception.Message)" -ForegroundColor Yellow
}

# Find the real profile folder (e.g. shutdownbot.PCNAME)
$profileDir = Get-ChildItem "C:\Users" -Directory | Where-Object { $_.Name -like "$ACCOUNT*" } | Sort-Object { $_.Name.Length } -Descending | Select-Object -First 1
if (-not $profileDir) {
    Write-Host "      Profile folder not found, creating it manually." -ForegroundColor Yellow
    New-Item -ItemType Directory -Force "C:\Users\$ACCOUNT" | Out-Null
    $profileDir = Get-Item "C:\Users\$ACCOUNT"
}
$sshDir = Join-Path $profileDir.FullName ".ssh"
Write-Host "      Profile folder: $($profileDir.FullName)"

# ---------- 4. Install the SSH key ----------
Write-Host "[4/9] Installing SSH key (restricted to the 'shutdown' command, only from $PiLocalIP) ..."
New-Item -ItemType Directory -Force $sshDir | Out-Null
$authLine = 'restrict,from="' + $PiLocalIP + '",command="shutdown /s /t 0" ' + $PublicKey
Set-Content -Path (Join-Path $sshDir "authorized_keys") -Value $authLine -Encoding ascii

# ---------- 5. Fix permissions on the .ssh folder ----------
Write-Host "[5/9] Setting file permissions ..."
icacls $sshDir /inheritance:r | Out-Null
icacls $sshDir /grant "*S-1-5-18:(OI)(CI)F" "*S-1-5-32-544:(OI)(CI)F" "${ACCOUNT}:(OI)(CI)R" | Out-Null
icacls $sshDir /setowner "*S-1-5-32-544" /T | Out-Null

# ---------- 6. Configure sshd_config ----------
Write-Host "[6/9] Configuring the SSH server (key login only, this account only) ..."
$cfg = "C:\ProgramData\ssh\sshd_config"
$cfgContent = Get-Content $cfg -Raw
if ($cfgContent -notmatch "AllowUsers $ACCOUNT") {
    $newLines = @(
        "PasswordAuthentication no",
        "PubkeyAuthentication yes",
        "AuthenticationMethods publickey",
        "AllowUsers $ACCOUNT",
        ""
    )
    $merged = $newLines + (Get-Content $cfg)
    Set-Content -Path $cfg -Value $merged -Encoding ascii
}
& "C:\Windows\System32\OpenSSH\sshd.exe" -t
if ($LASTEXITCODE -ne 0) {
    Write-Host "      ERROR: sshd_config is invalid!" -ForegroundColor Red
    exit 1
}

# ---------- 7. Start the service ----------
Write-Host "[7/9] Starting the SSH service ..."
Set-Service -Name sshd -StartupType Automatic
Restart-Service sshd

# ---------- 8. Firewall rules ----------
Write-Host "[8/9] Setting firewall rules (limited to the Pi: $PiLocalIP) ..."
if (-not (Get-NetFirewallRule -Name "SSH-Pi-Only" -ErrorAction SilentlyContinue)) {
    New-NetFirewallRule -Name "SSH-Pi-Only" -DisplayName "SSH from Raspberry Pi only" -Direction Inbound -Protocol TCP -LocalPort 22 -RemoteAddress $PiLocalIP -Action Allow -Profile Any | Out-Null
}
if (-not (Get-NetFirewallRule -Name "Ping-Pi-Only" -ErrorAction SilentlyContinue)) {
    New-NetFirewallRule -Name "Ping-Pi-Only" -DisplayName "ICMP ping from Raspberry Pi only" -Direction Inbound -Protocol ICMPv4 -IcmpType 8 -RemoteAddress $PiLocalIP -Action Allow -Profile Any | Out-Null
}

# ---------- 9. Grant shutdown privileges ----------
Write-Host "[9/9] Granting shutdown rights to '$ACCOUNT' ..."
$sid = (Get-LocalUser $ACCOUNT).SID.Value
$tmpCfg = "$env:TEMP\secpol_vdd.cfg"
secedit /export /cfg $tmpCfg | Out-Null
$content = Get-Content $tmpCfg
foreach ($right in @("SeShutdownPrivilege", "SeRemoteShutdownPrivilege")) {
    $found = $false
    for ($i = 0; $i -lt $content.Count; $i++) {
        if ($content[$i] -match "^$right\s*=") {
            if ($content[$i] -notmatch [regex]::Escape($sid)) {
                $content[$i] = $content[$i].TrimEnd() + ",*$sid"
            }
            $found = $true
        }
    }
    if (-not $found) {
        $idx = ($content | Select-String "\[Privilege Rights\]").LineNumber
        $content = $content[0..($idx - 1)] + "$right = *$sid" + $content[$idx..($content.Count - 1)]
    }
}
Set-Content -Path $tmpCfg -Value $content -Encoding Unicode
secedit /configure /db "$env:TEMP\secedit_vdd.sdb" /cfg $tmpCfg /areas USER_RIGHTS | Out-Null
Remove-Item $tmpCfg, "$env:TEMP\secedit_vdd.sdb" -ErrorAction SilentlyContinue

# ---------- Summary ----------
$mac = (Get-NetAdapter | Where-Object { $_.Status -eq "Up" -and $_.Virtual -eq $false } | Select-Object -First 1).MacAddress
$ip = (Get-NetIPAddress -AddressFamily IPv4 | Where-Object { $_.InterfaceAlias -notmatch "Loopback" -and $_.IPAddress -notmatch "^169\." } | Select-Object -First 1).IPAddress

Write-Host ""
Write-Host "=================================================="
Write-Host "  Done!" -ForegroundColor Green
Write-Host "=================================================="
Write-Host ""
Write-Host "You'll need these on the Pi (if the Pi script hasn't run yet):"
Write-Host "  This PC's IPv4 address : $ip"
Write-Host "  This PC's MAC address  : $mac"
Write-Host ""
Write-Host "Still to do (can't be automated):"
Write-Host "  - Enable Wake-on-LAN in the BIOS"
Write-Host "  - Enable Wake-on-LAN on the network adapter in Windows"
Write-Host "    (Device Manager -> Network adapters -> Properties -> Power Management)"
Write-Host "  - Install the Tailscale app on your phone and log in"
Write-Host "=================================================="
Step 4: Manual steps (can't be automated)
Enable Wake-on-LAN in the BIOS - the name varies by motherboard (e.g. "Power On By PCI-E/PCIE", "Resume by LAN"), usually under "Power Management".
Enable Wake-on-LAN in Windows - Device Manager -> Network adapters -> Properties -> Power Management -> check "Allow this device to wake the computer". Also check the "Advanced" tab for "Wake on Magic Packet" and enable it.
Install the Tailscale app on your phone and log in with the same account the Pi is connected to.
Done - open the web interface in your phone's browser: http://<Pi's Tailscale IP>:8080
Security, briefly
No port is opened on your router. Everything is reachable only through the private Tailscale network, tied to your own account.
The Windows account used for shutting down is not an administrator, and the SSH key is technically restricted to running only the shutdown command - no interactive access possible.
SSH on the PC is additionally firewalled to only accept connections from the Pi's local IP, blocked for every other device on the network.
The web interface itself is password-protected.
Using this guide with Claude

If you get stuck on a step (firewall errors, SSH connection failing, the monitor not waking up automatically when streaming remotely, etc.), you can get help directly from Claude:

Open a new chat with Claude.
Copy this whole guide (or upload it as a file) and say something like: "I want to set this up on my machine, walk me through it step by step and wait for my confirmation/screenshots after each step before continuing."
Substitute in your own values (IP addresses, MAC address, usernames) - never share real passwords or credentials with Claude or anywhere else beyond what this setup needs.
Claude can help you debug error messages, read screenshots of terminal output, and adapt the scripts to your hardware if needed (different Pi models, different Windows language, etc.).

Tested with: Raspberry Pi Zero W, Raspberry Pi OS (Legacy, 32-bit), Windows 10/11, Steam Remote Play. Other configurations may need small adjustments.
