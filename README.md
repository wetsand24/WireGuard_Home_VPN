# WireGuard Home VPN with WGDashboard

Your own VPN at home. Your phone connects to your house over WireGuard and works like it's on your home Wi-Fi:
file shares, Remote Desktop, SSH, cameras and the apps you host yourself. The router opens one UDP port and
nothing else. WGDashboard is a web page for WireGuard, so after a short install in the terminal you do the rest
in the browser.

Every command and file from the video, in order. The addresses below are the ones from the video:

| | Address | What it is |
|---|---|---|
| WireGuard box | `10.37.0.45` | the Debian 13 machine running WireGuard and WGDashboard |
| VPN network | `10.9.0.1/24` | a small private network just for your VPN devices |
| WireGuard port | UDP `51821` | the port the router forwards to the box |
| `YOUR_PUBLIC_IP` | | your home's public address (step 1 shows how to find it) |

Replace them with yours.

▶️ Video: https://youtu.be/1uTLrhAv2cs

## What you need
- A **Debian 13** machine on your home network. A Proxmox container works: in the video it's an unprivileged
  container, and WireGuard uses the Proxmox host's kernel module.
  Debian 12 or Ubuntu 24.04? They need extra steps first, see [Debian 12 and Ubuntu 24.04](#debian-12-and-ubuntu-2404-extra-steps).
- Access to your router's port forwarding page.
- A public IP that isn't behind CGNAT (step 1 checks this).
- The free WireGuard app on your phone.

Tested in the video with WGDashboard v4.3.4 on Debian 13.

## Step 1. Check your router can be reached
Some internet providers put your router behind one of their own routers. That's called CGNAT. If yours does,
nothing from outside can reach your router, and this setup can't work.

1. Open your router's status page and find the internet (WAN) address. Write it down.
2. Open a "what's my IP" website, for example https://ifconfig.me.

You should see the same address in both. Then you're good.

You're behind CGNAT if:
- the router's WAN address is anywhere from `100.64.x.x` to `100.127.x.x`, or
- the two addresses don't match.

Then port forwarding can't work. For web pages, use a Cloudflare Tunnel instead:
https://youtu.be/-m-GpkTv3Lo. Or ask your provider for a public IP.

## Before you start: log in with SSH
From your Windows PC: Start menu → type `Terminal` → open it, then:
```
ssh root@10.37.0.45
```
The first time it asks if you trust the machine: type `yes`. Then the root password (nothing shows while you
type, that's on purpose). When the line ends with `#`, you're logged in.

**Can't connect, or "Permission denied"?** SSH is often switched off on a fresh install. Do this once on the
machine itself, logged in as root. On a Proxmox container that's the Proxmox web page → click the container →
**Console**.
```bash
apt install -y openssh-server
systemctl enable --now ssh
echo "PermitRootLogin yes" > /etc/ssh/sshd_config.d/10-root-login.conf
systemctl restart ssh
```
The third line is needed because Debian doesn't let root log in over SSH with a password by default. If you'd
rather not allow that, log in as your normal user instead (`ssh yourname@10.37.0.45`) and type `su -` to become
root.

## Step 2. Install WireGuard and WGDashboard
Refresh the package list. Without it the install can fail on a fresh box:
```bash
apt update
```
Install what the dashboard needs:
```bash
apt install -y sudo git iptables wireguard-tools net-tools
```
`-y` installs everything without asking you to confirm. `wireguard-tools` gives you the WireGuard commands, the
installer needs `sudo`, `iptables` is for the two lines in step 4, and `git` downloads the dashboard.

Download WGDashboard and go into its folder (`cd` means change folder):
```bash
git clone https://github.com/WGDashboard/WGDashboard.git
cd WGDashboard/src
```
Make the installer runnable, then run it:
```bash
chmod +x wgd.sh
./wgd.sh install
```
It asks which download mirror to use. Press `Enter` for the default. It takes about two minutes.
You should see:
```
WGDashboard installed successfully!
```

**IP forwarding.** This lets the box pass your phone's traffic on to the rest of your home network. It's off
after the install. Put it in its own small file and apply it:
```bash
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-wireguard.conf
sysctl -p /etc/sysctl.d/99-wireguard.conf
```
You should see:
```
net.ipv4.ip_forward = 1
```
(Debian 13 has no `/etc/sysctl.conf`, which is why the line goes into `/etc/sysctl.d/`.)

**Start it by itself after a reboot or a power cut.** WGDashboard comes with a service file. This line fills in
the dashboard's folder and copies the file to where Linux keeps services. Run it from inside `WGDashboard/src`
(also in [service.txt](service.txt)):
```bash
sed "s|<absolute_path_of_wgdashboard_src>|$PWD|g" wg-dashboard.service > /etc/systemd/system/wg-dashboard.service
```
Make Linux read the new file, then start the dashboard now and on every boot:
```bash
systemctl daemon-reload
systemctl enable --now wg-dashboard
systemctl is-active wg-dashboard
```
You should see:
```
active
```
The dashboard is now running on port `10086`. In my test, after a reboot of the container the dashboard and
`wg0` came back by themselves.

## Debian 12 and Ubuntu 24.04 extra steps
WGDashboard v4.3.4 checks two things during `./wgd.sh install`: Python 3.12 or newer, and Node.js 20.19+ or
22.12+. Debian 13 ships Python 3.13 and Node.js 20.19, so it passes both. Debian 12 and Ubuntu 24.04 don't, and
the plain install stops.

### Debian 12
Debian 12 has Python 3.11.2 and Node.js 18.20.4, so it fails twice:
```
WGDashboard required Python 3.12 or above. Halting install now.
```
and, once Python is fixed:
```
Found Node.js v18.20.4, but the frontend requires Node.js 20.19+ or 22.12+. Halting install now.
```
These 12 commands were tested end to end on a fresh Debian 12 Proxmox container, as root. They install Python
3.12 with [uv](https://github.com/astral-sh/uv) and Node.js 22 from NodeSource:
```bash
apt update
apt install -y sudo git iptables wireguard-tools net-tools curl
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env
uv python install 3.12
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs
git clone https://github.com/WGDashboard/WGDashboard.git
cd WGDashboard/src
chmod +x wgd.sh
./wgd.sh install
./wgd.sh start
```
Press `Enter` at the mirror question. You should see `Found Python 3.12. Will be using [python3.12]`, then
`WGDashboard installed successfully!` and `WGDashboard w/ Gunicorn started successfully`. The whole list took
about 2.5 minutes.

The service file from Step 2 was also tested on Debian 12: `systemctl enable --now wg-dashboard` gave `active`,
and it was still `active` after a reboot.

Things to know:
- Run `source $HOME/.local/bin/env` in the same window before `./wgd.sh install` (or log out and back in).
  Otherwise `python3.12` isn't found and the install stops with the Python message again.
- The dashboard's Python lives in `~/.local/share/uv`. Running `uv python uninstall 3.12` or deleting that folder
  breaks the dashboard. A reboot doesn't.
- NodeSource is a third-party package source. It stays added and updates Node.js with your normal `apt upgrade`.
- When the installer stops, it exits with code 143. That's how it halts, not a crash.

### Ubuntu 24.04
Ubuntu 24.04 has Python 3.12.3, which is fine, but Node.js 18.19.1, so the plain install from Step 2 stops with:
```
Found Node.js v18.19.1, but the frontend requires Node.js 20.19+ or 22.12+. Halting install now.
```
Tested on a fresh Ubuntu 24.04 Proxmox container, as root, right after that failure, still inside `WGDashboard/src`:
```bash
apt install -y curl
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs
./wgd.sh install
```
You should see `WGDashboard installed successfully!`. `./wgd.sh start` then started the dashboard.

What was **not** tested on Ubuntu 24.04: the service file and a reboot. Newer Ubuntu versions weren't tested at all.

## Step 3. Open the dashboard
In your browser:
```
http://10.37.0.45:10086
```
The first login is `admin` / `admin`. Straight away it asks you to create your own account: pick a username and
a real password, at least 8 characters. This page controls who gets into your house.

Next it offers two-factor login (MFA). Scan it with an authenticator app, or click **I don't need MFA** to skip
it for now.

## Step 4. Create the WireGuard network
This is the step where most setups go wrong.

Click **+ Configuration** and fill in:

| Field | Value | What it is |
|---|---|---|
| Name | `wg0` | |
| Listen Port | `51820` is the default | The port the router will forward. The video uses `51821` because my own VPN at home already uses `51820`. Any free UDP port works. |
| IP Address/CIDR | `10.9.0.1/24` | A small private network just for your VPN devices. It can't be the same as your home network. |

The keys are filled in for you.

**The two lines.** Without them, the phone says "connected" but nothing loads: the box lets the phone in, but
doesn't pass its traffic on to the house.

Open **Optional Settings**. There are two boxes, PostUp and PostDown. PostUp runs when WireGuard starts,
PostDown when it stops.

PostUp (also in [postup.txt](postup.txt)):
```
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; iptables -A FORWARD -i wg0 -j ACCEPT
```
PostDown (also in [postdown.txt](postdown.txt)):
```
iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; iptables -D FORWARD -i wg0 -j ACCEPT
```

What they do:

| Part | What it does |
|---|---|
| `iptables -A FORWARD -i wg0 -j ACCEPT` | lets the phone's traffic through this box and into your home network |
| `iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE` | puts the box's own address on that traffic, so your home devices know where to send the answer |
| PostDown (`-D` instead of `-A`) | removes both rules again when WireGuard stops |

`eth0` must be your network card's name. Check it on the box:
```bash
ip -br a
```
It's the first column. In a Proxmox container it shows as `eth0@if13` (the number differs); the name to use is
still `eth0`.

Save, then switch the configuration on. Its status turns green.

## Step 5. Open the port on your router
Every router looks different. Look for the setting called **port forwarding**. On OpenWrt it's
**Network → Firewall → Port Forwards**.

Add a rule:

| Field | Value |
|---|---|
| Protocol | **UDP** |
| External port | `51821` |
| Internal IP | your WireGuard box, `10.37.0.45` |
| Internal port | `51821` |

Save and apply.

Only forward the WireGuard port. **Never forward 10086**, the dashboard's port. The dashboard stays inside your
house.

## Step 6. Add your phone
**First, one setting.** In the dashboard, go to **Settings**, the peers settings tab. **Peer Remote Endpoint** is
the address your phone will connect to. It's filled in with the box's home address (`10.37.0.45`), which only
works inside the house. Change it to your public address from step 1:
```
YOUR_PUBLIC_IP
```
If your home address changes now and then, use a free [DuckDNS](https://www.duckdns.org) name instead.

**Add the peer.** Open `wg0` and click **+ Peer**. Name it after the device, so you know which key is which
(the video uses `phone`). Open **Advanced Options**:

| Field | Default | What it means |
|---|---|---|
| Endpoint Allowed IPs | `0.0.0.0/0` | all of the phone's IPv4 traffic goes through your home while the tunnel is on |
| DNS | `1.1.1.1` | the server your phone asks for websites. If you run Pi-hole (or want your router's DNS), put its home address here, and ads are blocked on mobile data too |

Click **Add**. On the new peer, click the three dots, then the QR code.

On your phone: open the WireGuard app → **+** → **Scan from QR code**, point it at the screen, and give the
tunnel a name.

Treat that QR code like a password. It's the key to your house.

## Step 7. Test it
1. Turn **Wi-Fi off** on your phone. Testing on your home Wi-Fi proves nothing, because you're already inside.
2. Switch the tunnel on in the WireGuard app.
3. In the dashboard, the phone's peer turns green and shows a recent handshake. A handshake means the phone and
   your box just exchanged keys.
4. Open something that only works at home, for example `http://10.37.0.45:10086` in the phone's browser.
   It opens over mobile data.
5. A "what's my IP" page on the phone now shows your home address, because all the phone's traffic goes through
   the house.
6. Switch the tunnel off and the home page won't load anymore.

## Troubleshooting
First tell the two cases apart:
- **No handshake** (the peer never turns green): the phone isn't reaching the box. Check the router port and the
  endpoint.
- **Handshake, but nothing loads**: the phone reaches the box, but the box doesn't pass the traffic on. Check IP
  forwarding and the two lines.

Things to check:
- **IP forwarding is on.** `sysctl net.ipv4.ip_forward` has to print `net.ipv4.ip_forward = 1`. If not, run the
  two IP forwarding lines from Step 2 again.
- **The two lines are in place.** PostUp and PostDown under Optional Settings, with your network card's name
  instead of `eth0` (`ip -br a`).
- **The router forwards UDP, not TCP**, on the right port, to the right box.
- **Peer Remote Endpoint is your public address** (or DuckDNS name), not the box's LAN IP. With the LAN IP the
  tunnel works at home and fails on mobile data.
- **CGNAT.** If your router's WAN address is `100.64.x.x` to `100.127.x.x`, or doesn't match what's-my-IP,
  port forwarding can't work. See step 1.
- **You tested on Wi-Fi.** Turn Wi-Fi off and test on mobile data.

## Good to know
- WireGuard doesn't answer anyone without a valid key, so to a scanner the open port looks like nothing is there.
- Make a separate peer for every device. If you lose your phone, delete its peer, and that key stops working right
  away.
- Keep WGDashboard updated. It got a security fix in 2026, so check for updates now and then.
- Don't put your only way in on the same machine as everything else. When the server running my Pi-hole lost its
  network, my WireGuard was on it too, so I couldn't get in to fix it.

## Files
| File | What it is |
|---|---|
| [postup.txt](postup.txt) | the PostUp line for step 4 |
| [postdown.txt](postdown.txt) | the PostDown line for step 4 |
| [service.txt](service.txt) | the service file line for step 2 |

## Links
- WGDashboard: https://github.com/WGDashboard/WGDashboard
- WireGuard: https://www.wireguard.com/
