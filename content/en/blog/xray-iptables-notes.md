---
title: "Xray + iptables installation and configuration notes"
date: 2026-03-12T04:50:00+08:00
summary: A fuller English rewrite that stays close to the original notes, restoring section hierarchy, config templates, and iptables scripts.
description: Practical notes for installing and routing traffic with Xray and iptables.
tags:
  - Xray
  - iptables
  - Linux
  - Proxy
  - Networking
---

> This is a **shareable** general-purpose note. It does not include any real nodes, UUIDs, public keys, passwords, or other private information.
>
> All commands, scripts, and configuration examples are given in code blocks so they can be copied directly into Typora or any other Markdown reader.
>
> **Reminder:** the `iptables`, `systemctl`, `/usr/local/bin/`, and `/etc/systemd/system/` operations shown here all require `sudo`. Before making changes, make sure you still have a recovery path such as SSH, console access, or another way back into the machine.
>
> If you only want the practical path, start from **Section 5**.

## 1. What is this architecture doing?

This note describes a common local traffic steering architecture:

- **iptables** steers locally generated traffic into a chosen entry point.
- **Xray** receives that traffic, identifies the target, and decides which outbound path should be used.

You can think of it like this:

- `iptables` is the **traffic police**.
- `Xray` is the **routing control center**.

`iptables` does not really understand whether traffic is for OpenAI, ChatGPT, or Codex. What it is good at is:

- sending traffic to a port,
- allowing some address ranges,
- rejecting some protocols or ports.

Xray is better at:

- deciding which outbound to use,
- distinguishing different domains or targets,
- providing transparent proxy, SOCKS, and HTTP inbounds.

## 2. How does the traffic flow work?

### 2.1 Flow breakdown

1. **An application sends a request**  
   For example a browser, `curl`, OpenClaw, or another CLI tool.
2. **iptables intercepts locally generated TCP traffic**  
   If the destination is loopback or private network space, it is allowed directly. If it is external traffic that should be proxied, it is redirected to local port `12345`.
3. **Xray's transparent inbound receives that traffic**  
   In this setup that is usually a `dokodemo-door` inbound.
4. **Xray evaluates routing rules**  
   OpenAI / ChatGPT / Codex related traffic goes to node 1; other traffic goes to node 2 or direct; private traffic goes to `direct`.
5. **The selected outbound sends the traffic out**  
   So Xray decides *which path to use*, while iptables decides *which traffic enters Xray first*.

### 2.2 One-sentence summary

```text
iptables sends traffic into Xray, and Xray sends that traffic to the correct outbound path.
```

## 3. Structure diagram

![Xray + iptables architecture](/media/blog/xray-iptables-architecture.jpg)

The diagram above gives a quick visual view of how the components relate to each other.

<p class="figure-caption"><strong>Local traffic steering relationship between Xray and iptables</strong></p>

## 4. How this note is organized

This note follows the same sequence as the source document:

5. **Complete Xray installation and configuration flow**
6. **Xray commands and troubleshooting**
7. **iptables configuration flow**
   - Case 1: forced transparent proxying, no condition check
   - Case 2: Xray health check to avoid a total network outage
8. **Appendices**
   - Appendix A: single-exit configuration template
   - Appendix B: dual-exit configuration template
   - Appendix C: consolidated iptables Bash command area

If you only want the main path, do it in this order:

1. Install Xray.
2. Write and test `config.json`.
3. Confirm that `12345 / 10808 / 10809` are listening.
4. Choose an `iptables` strategy.
   - either forced transparent proxying,
   - or health check with automatic fallback.
5. Then configure `systemd` startup.

## 5. Xray installation and configuration flow

### 5.1 What is Xray doing here?

In this architecture, Xray is responsible for:

- providing a transparent proxy inbound such as `12345`,
- providing local proxy inbounds such as `10808 / 10809`,
- deciding which outbound path traffic should use.

In this note, its main job is:

```text
Receive traffic redirected by iptables, then split it according to routing rules.
```

This section gives a **complete uninterrupted setup flow** first.

### 5.2 Install Xray

```bash
# Install Xray (must use sudo, and keep the whole command on one line)
sudo bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

> Do not move `@ install` onto the next line, or the shell may treat it as a separate `install` command.

### 5.3 Check whether installation succeeded

```bash
# Show Xray version
/usr/local/bin/xray version
```

**Expected example output:**

```text
Xray 1.x.x (Xray, Penetrates Everything.)
```

If you get `No such file or directory` or `command not found`, the installation is not complete yet.

### 5.4 Confirm the config file location

```bash
# Check whether the default config file exists
ls -l /usr/local/etc/xray/config.json
```

**Expected example output:**

```text
-rw-r--r-- 1 root root 1234 ... /usr/local/etc/xray/config.json
```

If the file does not exist yet, you can create it manually in the next step.

### 5.5 Choose a configuration type

The templates here are the base skeletons for generating `config.json`. They determine where different traffic types eventually go.

If your requirement is:

- OpenAI / ChatGPT / Codex / GPT related traffic goes through **node 1**
- all other traffic goes **direct**

use:

```text
Appendix A: single-exit configuration template
```

If your requirement is:

- OpenAI / ChatGPT / Codex / GPT related traffic goes through **node 1**
- all other traffic goes through **node 2**

use:

```text
Appendix B: dual-exit configuration template
```

> Practical note: if you do not want to fill every field by hand, you can give “one or more proxy node links + the template you want” to a commercial model to generate a first draft of `config.json`, and then validate it yourself locally. To reduce sensitive-data trace risk, temporary chats or one-off sessions are safer for that step.

### 5.6 Edit the configuration with nano

```bash
# Edit the Xray configuration file with nano
sudo nano /usr/local/etc/xray/config.json
```

Paste the full template you chose, then replace the placeholders with your own values.

#### 5.6.1 Save and exit in nano

```text
Ctrl + O   -> write file
Enter      -> confirm file name
Ctrl + X   -> exit nano
```

If you want to quit without saving:

```text
Ctrl + X
press N
```

#### 5.6.2 Reference docs

```text
https://www.nano-editor.org/docs.php
https://xtls.github.io/config/
```

### 5.7 Test the config syntax

```bash
# Check whether Xray can parse the config correctly
sudo /usr/local/bin/xray run -test -config /usr/local/etc/xray/config.json
```

**Expected example output:**

```text
Xray 1.x.x ...
Configuration OK.
```

If this step fails, do not start the service yet. Go back and fix the config first.

### 5.8 Enable and start Xray

```bash
# Reload systemd configuration
sudo systemctl daemon-reload

# Enable Xray on boot
sudo systemctl enable xray

# Start or restart Xray
sudo systemctl restart xray

# Show current Xray status
sudo systemctl status xray --no-pager -n 50
```

**Expected example output:**

```text
● xray.service - Xray Service
     Loaded: loaded (...)
     Active: active (running)
```

If you see:

```text
Active: failed
```

then the configuration or startup parameters still need troubleshooting.

### 5.9 Check whether the listening ports are correct

```bash
# Check whether transparent proxy, SOCKS, and HTTP inbounds are listening
ss -ltnup | grep -E '12345|10808|10809'
```

**Expected example output:**

```text
tcp   LISTEN 0 4096 0.0.0.0:12345    0.0.0.0:*
tcp   LISTEN 0 4096 127.0.0.1:10808  0.0.0.0:*
tcp   LISTEN 0 4096 127.0.0.1:10809  0.0.0.0:*
udp   UNCONN 0 0    127.0.0.1:10808  0.0.0.0:*
```

At minimum you should see:

- `12345`
- `10808`
- `10809`

Once those appear, the Xray installation and configuration flow is done.

## 6. Xray commands and troubleshooting information

This section is not the main setup path anymore. It is the set of commands you will commonly use after installation.

```bash
# Start Xray
sudo systemctl start xray

# Stop Xray
sudo systemctl stop xray

# Restart Xray
sudo systemctl restart xray

# Try to reload config (if the service supports reload)
sudo systemctl reload xray

# Enable on boot
sudo systemctl enable xray

# Disable on boot
sudo systemctl disable xray

# Check Xray service status
sudo systemctl status xray --no-pager -n 50
```

```bash
# Show the latest 100 lines of Xray logs
journalctl -u xray -n 100 --no-pager

# Follow Xray logs in real time
journalctl -u xray -f
```

```bash
# Check whether key ports are listening
ss -ltnup | grep -E '12345|10808|10809'
```

```bash
# Query egress IP through SOCKS proxy
curl -x socks5h://127.0.0.1:10808 https://api.ipify.org

# Query egress IP through HTTP proxy
curl -x http://127.0.0.1:10809 https://api.ipify.org
```

## 7. iptables configuration flow

### 7.1 What is iptables doing here?

In this architecture, iptables is responsible for:

- intercepting locally generated traffic,
- deciding which destination ranges should be allowed directly,
- sending traffic that should be proxied to Xray's transparent inbound.

In one sentence:

```text
Xray decides where traffic should go, and iptables decides which traffic enters Xray first.
```

This is where transparent proxy rules begin. There are two cases.

## 8. Case 1: forced transparent proxying, with no condition checks

If your requirement is:

- all external TCP traffic must be redirected into Xray,
- you accept that network access may fail if Xray is down,
- you do not need automatic fallback to direct access,

then you can use the following **forced transparent proxy** rule set.

### 8.1 Write the script with nano

Create the script file first:

```bash
sudo nano /usr/local/bin/xray-iptables-apply.sh
```

Paste the following content in full:

```bash
#!/usr/bin/env bash
set -e

# Create XRAY chain, ignore if it already exists
sudo iptables -t nat -N XRAY 2>/dev/null || true

# Reject UDP 443 to reduce QUIC bypass around the TCP path (deduplicate before add)
while sudo iptables -C OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable 2>/dev/null; do
  sudo iptables -D OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable
done
sudo iptables -A OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable

# Ensure only one OUTPUT -> XRAY jump exists (deduplicate before add)
while sudo iptables -t nat -C OUTPUT -p tcp -j XRAY 2>/dev/null; do
  sudo iptables -t nat -D OUTPUT -p tcp -j XRAY
done
sudo iptables -t nat -A OUTPUT -p tcp -j XRAY

# Flush XRAY chain first so repeated runs do not stack rules
sudo iptables -t nat -F XRAY

# Loopback should not be proxied
sudo iptables -t nat -A XRAY -d 127.0.0.0/8 -j RETURN

# 10.0.0.0/8 private range should not be proxied
sudo iptables -t nat -A XRAY -d 10.0.0.0/8 -j RETURN

# 172.16.0.0/12 private range should not be proxied
sudo iptables -t nat -A XRAY -d 172.16.0.0/12 -j RETURN

# 192.168.0.0/16 private range should not be proxied
sudo iptables -t nat -A XRAY -d 192.168.0.0/16 -j RETURN

# Multicast should not be proxied
sudo iptables -t nat -A XRAY -d 224.0.0.0/4 -j RETURN

# Reserved addresses should not be proxied
sudo iptables -t nat -A XRAY -d 240.0.0.0/4 -j RETURN

# Skip a specific UID to avoid transparently proxying some system accounts
sudo iptables -t nat -A XRAY -m owner --uid-owner 65534 -j RETURN

# Redirect all remaining TCP traffic to Xray transparent inbound 12345
sudo iptables -t nat -A XRAY -p tcp -j REDIRECT --to-ports 12345
```

### 8.2 Make the script executable and run it

```bash
# Make it executable
sudo chmod +x /usr/local/bin/xray-iptables-apply.sh

# Run the script
sudo /usr/local/bin/xray-iptables-apply.sh
```

### 8.3 What you should see after applying the rules

```bash
# Export the current iptables rules
sudo iptables-save
```

**Expected example output:**

```text
*filter
-A OUTPUT -p udp -m udp --dport 443 -j REJECT --reject-with icmp-port-unreachable
COMMIT
*nat
:XRAY - [0:0]
-A OUTPUT -p tcp -j XRAY
-A XRAY -d 127.0.0.0/8 -j RETURN
...
-A XRAY -p tcp -j REDIRECT --to-ports 12345
COMMIT
```

Pay attention to whether these entries exist:

- `-A OUTPUT -p tcp -j XRAY`
- `-A XRAY -p tcp -j REDIRECT --to-ports 12345`
- `-A OUTPUT -p udp --dport 443 -j REJECT ...`

### 8.4 How to verify that transparent proxying is actually working

```bash
# 1) Confirm Xray ports are still listening
ss -ltnup | grep -E '12345|10808|10809'

# 2) Check current egress IP via SOCKS
curl -x socks5h://127.0.0.1:10808 https://api.ipify.org

# 3) Access a site that you explicitly expect to be proxied
curl -I https://api.openai.com --connect-timeout 10

# 4) Observe Xray logs
journalctl -u xray -n 100 --no-pager
```

If you find that:

- the rules are present in `iptables-save`,
- but external access is still broken,

then check in this order:

1. whether `xray.service` is really running,
2. whether port `12345` is still listening,
3. whether routing rules are sending traffic into a bad outbound,
4. whether the host itself can reach the network directly.

### 8.5 Teardown script

```bash
sudo nano /usr/local/bin/xray-iptables-teardown.sh
```

Put the following content into it:

```bash
#!/usr/bin/env bash
set -e

# Remove UDP 443 reject rule
sudo iptables -D OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable 2>/dev/null || true

# Remove OUTPUT -> XRAY jump
sudo iptables -t nat -D OUTPUT -p tcp -j XRAY || true

# Flush rules in XRAY chain
sudo iptables -t nat -F XRAY || true

# Delete custom XRAY chain
sudo iptables -t nat -X XRAY || true
```

After saving, make it executable:

```bash
sudo chmod +x /usr/local/bin/xray-iptables-teardown.sh
```

Run it when you want to restore direct access:

```bash
sudo /usr/local/bin/xray-iptables-teardown.sh
```

### 8.6 What you should see after teardown

```bash
# Export current iptables rules again
sudo iptables-save
```

**Expected example output:**

```text
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
COMMIT
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
COMMIT
```

That means the `XRAY` chain and its related jump rules should be gone.

### 8.7 Make it start automatically on boot (for the forced mode)

If you choose the forced transparent proxy mode, running the script once manually is not enough, because the iptables rules usually will not survive a reboot.

A direct way to handle this is:

- let the Xray service start on boot,
- create an extra `systemd` service that runs `xray-iptables-apply.sh` after boot.

#### 8.7.1 Create a systemd service

```bash
sudo nano /etc/systemd/system/xray-iptables-apply.service
```

```ini
[Unit]
Description=Apply Xray iptables redirect rules
After=network-online.target xray.service
Wants=network-online.target
Requires=xray.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/xray-iptables-apply.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

#### 8.7.2 Enable automatic startup

```bash
# Reload systemd configuration
sudo systemctl daemon-reload

# Ensure Xray itself starts on boot
sudo systemctl enable --now xray

# Enable the iptables rule service
sudo systemctl enable --now xray-iptables-apply.service
```

#### 8.7.3 Check service status

```bash
sudo systemctl status xray --no-pager
sudo systemctl status xray-iptables-apply.service --no-pager
```

If everything is correct, you should see:

```text
xray.service: active (running)
xray-iptables-apply.service: active (exited)
```

`active (exited)` is normal here because the service only runs a script once and does not stay resident.

#### 8.7.4 If you want to disable it later

```bash
sudo systemctl disable --now xray-iptables-apply.service
```

Then run the teardown script to go back to direct mode:

```bash
sudo /usr/local/bin/xray-iptables-teardown.sh
```

## 9. Case 2: check whether Xray is healthy and avoid a total network outage when it is not

If your requirement is:

- traffic should be transparently proxied while Xray is healthy,
- but you do not want the whole host to lose outbound TCP when Xray fails,
- and you want scripts to switch iptables state automatically,

then use:

```text
A health-check script plus two iptables states
```

That means:

- Xray healthy -> apply proxy rules
- Xray unhealthy -> remove proxy rules and return to direct mode

### 9.1 Write the health-check script with nano

```bash
sudo nano /usr/local/bin/xray-iptables-healthcheck.sh
```

Paste the following content:

```bash
#!/usr/bin/env bash
set -e

XRAY_CHAIN="XRAY"
XRAY_PORT="12345"

xray_is_healthy() {
  # First check whether the xray service is active
  if systemctl is-active --quiet xray; then

    # Then check whether port 12345 is really listening
    if ss -ltn | awk '{print $4}' | grep -qE '(^|:)12345$'; then

      # If both checks pass, treat Xray as healthy
      return 0
    fi
  fi

  # Otherwise treat it as unhealthy
  return 1
}

apply_proxy_rules() {
  # Create XRAY chain, ignore if it already exists
  sudo iptables -t nat -N "$XRAY_CHAIN" 2>/dev/null || true

  # Flush XRAY first so old rules do not stack up
  sudo iptables -t nat -F "$XRAY_CHAIN"

  # Ensure UDP 443 reject rule exists
  sudo iptables -D OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable 2>/dev/null || true
  sudo iptables -A OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable

  # Ensure OUTPUT has a jump into XRAY chain
  while sudo iptables -t nat -C OUTPUT -p tcp -j "$XRAY_CHAIN" 2>/dev/null; do
    sudo iptables -t nat -D OUTPUT -p tcp -j "$XRAY_CHAIN"
  done
  sudo iptables -t nat -A OUTPUT -p tcp -j "$XRAY_CHAIN"

  # Loopback direct
  sudo iptables -t nat -A "$XRAY_CHAIN" -d 127.0.0.0/8 -j RETURN

  # 10.0.0.0/8 direct
  sudo iptables -t nat -A "$XRAY_CHAIN" -d 10.0.0.0/8 -j RETURN

  # 172.16.0.0/12 direct
  sudo iptables -t nat -A "$XRAY_CHAIN" -d 172.16.0.0/12 -j RETURN

  # 192.168.0.0/16 direct
  sudo iptables -t nat -A "$XRAY_CHAIN" -d 192.168.0.0/16 -j RETURN

  # Multicast direct
  sudo iptables -t nat -A "$XRAY_CHAIN" -d 224.0.0.0/4 -j RETURN

  # Reserved ranges direct
  sudo iptables -t nat -A "$XRAY_CHAIN" -d 240.0.0.0/4 -j RETURN

  # Skip a specific UID
  sudo iptables -t nat -A "$XRAY_CHAIN" -m owner --uid-owner 65534 -j RETURN

  # Redirect remaining TCP traffic to Xray transparent inbound
  sudo iptables -t nat -A "$XRAY_CHAIN" -p tcp -j REDIRECT --to-ports "$XRAY_PORT"
}

apply_direct_rules() {
  # Remove OUTPUT -> XRAY jump so TCP returns to direct mode
  while sudo iptables -t nat -C OUTPUT -p tcp -j "$XRAY_CHAIN" 2>/dev/null; do
    sudo iptables -t nat -D OUTPUT -p tcp -j "$XRAY_CHAIN"
  done

  # Flush XRAY chain
  sudo iptables -t nat -F "$XRAY_CHAIN" 2>/dev/null || true

  # When Xray is unhealthy, remove UDP 443 reject rules and restore direct mode
  while sudo iptables -C OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable 2>/dev/null; do
    sudo iptables -D OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable
  done
}

if xray_is_healthy; then
  echo "Xray healthy -> apply proxy rules"
  apply_proxy_rules
else
  echo "Xray unhealthy -> switch to direct mode"
  apply_direct_rules
fi
```

### 9.2 Make the health-check script executable and test it once manually

```bash
# Make it executable
sudo chmod +x /usr/local/bin/xray-iptables-healthcheck.sh

# Run it once manually to confirm there is no syntax or permission issue
sudo /usr/local/bin/xray-iptables-healthcheck.sh
```

### 9.3 Run the health-check script periodically with systemd

After this is set up, systemd will run the health-check script once per minute:

- if Xray is healthy, proxy rules stay applied,
- if Xray is unhealthy, rules are switched back toward direct mode.

#### 9.3.1 Create the systemd service

```bash
sudo nano /etc/systemd/system/xray-iptables-healthcheck.service
```

```ini
[Unit]
Description=Xray iptables healthcheck
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/xray-iptables-healthcheck.sh
```

#### 9.3.2 Create the systemd timer

```bash
sudo nano /etc/systemd/system/xray-iptables-healthcheck.timer
```

```ini
[Unit]
Description=Run Xray iptables healthcheck every minute

[Timer]
OnBootSec=30s
OnUnitActiveSec=60s
Unit=xray-iptables-healthcheck.service

[Install]
WantedBy=timers.target
```

#### 9.3.3 Enable the timer

```bash
# Reload systemd configuration
sudo systemctl daemon-reload

# Enable timer on boot
sudo systemctl enable xray-iptables-healthcheck.timer

# Start timer
sudo systemctl start xray-iptables-healthcheck.timer

# Check whether the timer is active
sudo systemctl list-timers --all | grep xray
```

**Expected example output:**

```text
... xray-iptables-healthcheck.timer loaded active waiting ...
```

### 9.4 Useful health-check commands (run these after enabling the timer)

**1) Check whether xray is running normally**

```bash
systemctl is-active xray
```

Expected when healthy:

```text
active
```

**2) Check whether the timer is waiting properly**

```bash
systemctl is-active xray-iptables-healthcheck.timer
```

Expected when healthy:

```text
active
```

**3) Check whether the latest healthcheck run succeeded**

```bash
systemctl status xray-iptables-healthcheck.service --no-pager -n 20
```

A healthy run should include something like:

```text
code=exited, status=0/SUCCESS
```

**4) Check whether the transparent proxy jump exists when xray is healthy**

```bash
sudo iptables -t nat -S OUTPUT | grep -c -- "-A OUTPUT -p tcp -j XRAY"
```

Expected when healthy:

```text
1
```

**5) Check the number of UDP 443 reject rules and make sure it is not duplicated**

```bash
sudo iptables -S OUTPUT | grep -c -- "--dport 443 -j REJECT --reject-with icmp-port-unreachable"
```

Expected when healthy:

```text
1
```

**6) Check whether the XRAY chain contains the transparent redirect**

```bash
sudo iptables -t nat -S XRAY | grep -c -- "--to-ports 12345"
```

Expected when healthy:

```text
1
```

**7) Listening port check**

```bash
ss -ltnup | grep -E '12345|10808|10809'
```

Normal output should include listeners for `:12345`, `:10808`, and `:10809`.

These results usually indicate a problem:

- `systemctl is-active xray` returns `inactive`
- the healthcheck service exits with `status=1/FAILURE`
- the jump / UDP / redirect counts are not what you expect, for example duplicates greater than `1`

## 10. Other useful troubleshooting commands

```bash
# Check current Xray status
sudo systemctl status xray --no-pager -n 50

# Show recent Xray logs
journalctl -u xray -n 100 --no-pager

# Follow Xray logs in real time
journalctl -u xray -f

# Check key listening ports
ss -ltnup | grep -E '12345|10808|10809'

# Export current iptables rules
sudo iptables-save

# Show nft backend rules
sudo nft list ruleset

# Query egress IP via SOCKS
curl -x socks5h://127.0.0.1:10808 https://api.ipify.org

# Query egress IP via HTTP
curl -x http://127.0.0.1:10809 https://api.ipify.org
```

## 11. Final checklist

### 11.1 Is the Xray service itself healthy?

```bash
sudo systemctl status xray --no-pager
ss -ltnup | grep -E '12345|10808|10809'
```

You should at least see:

- `xray.service` as `active (running)`
- `12345 / 10808 / 10809` listening

### 11.2 Does the proxy egress match expectations?

```bash
curl -x socks5h://127.0.0.1:10808 https://api.ipify.org
curl -x http://127.0.0.1:10809 https://api.ipify.org
```

If the IP is wrong here, check:

- whether node parameters were filled incorrectly,
- whether routing sends traffic to the wrong outbound,
- whether the node itself is usable.

### 11.3 Do the iptables rules match the mode you chose?

```bash
sudo iptables-save
```

You need to confirm which mode you are in:

- **forced transparent proxy mode**: rules should stay present,
- **health-check fallback mode**: rules should change with Xray health.

### 11.4 Does startup behavior match expectations?

If you use the forced mode, check:

```bash
sudo systemctl status xray-iptables-apply.service --no-pager
```

If you use the health-check mode, check:

```bash
sudo systemctl status xray-iptables-healthcheck.timer --no-pager
sudo systemctl list-timers --all | grep xray
```

### 11.5 Where should you look first when something breaks?

Check in this order first:

1. `sudo systemctl status xray --no-pager`
2. `journalctl -u xray -n 100 --no-pager`
3. `ss -ltnup | grep -E '12345|10808|10809'`
4. `sudo iptables-save`
5. `curl -x socks5h://127.0.0.1:10808 https://api.ipify.org`

## 12. Related file locations

The files directly related to this shareable note in the workspace are:

```text
tmp/xray-config/xray-single-exit-template.json
tmp/xray-config/xray-dual-exit-template.json
tmp/xray-config/Xray-iptables-notes.md
```

As absolute paths:

```text
/home/claw1/.openclaw/workspace/tmp/xray-config/xray-single-exit-template.json
/home/claw1/.openclaw/workspace/tmp/xray-config/xray-dual-exit-template.json
/home/claw1/.openclaw/workspace/tmp/xray-config/Xray-iptables-notes.md
```

## Appendix A: single-exit configuration template

```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "redir-in",
      "listen": "0.0.0.0",
      "port": 12345,
      "protocol": "dokodemo-door",
      "settings": {
        "network": "tcp",
        "followRedirect": true
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "streamSettings": {
        "sockopt": {
          "tproxy": "redirect"
        }
      }
    },
    {
      "tag": "socks-in",
      "listen": "127.0.0.1",
      "port": 10808,
      "protocol": "socks",
      "settings": {
        "auth": "noauth",
        "udp": true
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls", "quic"]
      }
    },
    {
      "tag": "http-in",
      "listen": "127.0.0.1",
      "port": 10809,
      "protocol": "http",
      "settings": {}
    }
  ],
  "outbounds": [
    {
      "tag": "node1-openai",
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "YOUR_NODE1_SERVER",
            "port": 443,
            "users": [
              {
                "id": "YOUR_UUID",
                "encryption": "none",
                "flow": "xtls-rprx-vision"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "serverName": "YOUR_SERVER_NAME",
          "fingerprint": "random",
          "publicKey": "YOUR_PUBLIC_KEY",
          "shortId": "",
          "spiderX": "/"
        }
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    },
    {
      "tag": "block",
      "protocol": "blackhole",
      "settings": {}
    }
  ],
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "type": "field",
        "ip": [
          "geoip:private",
          "192.168.2.0/24"
        ],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "domain": [
          "domain:openai.com",
          "domain:chatgpt.com",
          "domain:oaistatic.com",
          "domain:oaiusercontent.com",
          "domain:openaiapi-site.azureedge.net",
          "domain:auth.openai.com",
          "domain:platform.openai.com",
          "domain:api.openai.com",
          "domain:cdn.openai.com",
          "domain:chat.openai.com",
          "domain:ab.chatgpt.com",
          "domain:files.oaiusercontent.com",
          "domain:chatgpt.livekit.cloud",
          "domain:featuregates.org",
          "regexp:(^|\\.)codex\\.",
          "regexp:(^|\\.)gpt\\.",
          "regexp:(^|\\.)openai\\.",
          "regexp:(^|\\.)chatgpt\\.",
          "keyword:openai",
          "keyword:chatgpt",
          "keyword:codex",
          "keyword:gpt"
        ],
        "outboundTag": "node1-openai"
      },
      {
        "type": "field",
        "network": "tcp,udp",
        "outboundTag": "direct"
      }
    ]
  }
}
```

## Appendix B: dual-exit configuration template

```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "redir-in",
      "listen": "0.0.0.0",
      "port": 12345,
      "protocol": "dokodemo-door",
      "settings": {
        "network": "tcp",
        "followRedirect": true
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "streamSettings": {
        "sockopt": {
          "tproxy": "redirect"
        }
      }
    },
    {
      "tag": "socks-in",
      "listen": "127.0.0.1",
      "port": 10808,
      "protocol": "socks",
      "settings": {
        "auth": "noauth",
        "udp": true
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls", "quic"]
      }
    },
    {
      "tag": "http-in",
      "listen": "127.0.0.1",
      "port": 10809,
      "protocol": "http",
      "settings": {}
    }
  ],
  "outbounds": [
    {
      "tag": "node1-openai",
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "YOUR_NODE1_SERVER",
            "port": 443,
            "users": [
              {
                "id": "YOUR_UUID",
                "encryption": "none",
                "flow": "xtls-rprx-vision"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "serverName": "YOUR_SERVER_NAME",
          "fingerprint": "random",
          "publicKey": "YOUR_PUBLIC_KEY",
          "shortId": "",
          "spiderX": "/"
        }
      }
    },
    {
      "tag": "node2-default",
      "protocol": "trojan",
      "settings": {
        "servers": [
          {
            "address": "YOUR_NODE2_SERVER",
            "port": 443,
            "password": "YOUR_PASSWORD"
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "tls",
        "tlsSettings": {
          "serverName": "YOUR_NODE2_SERVER_NAME",
          "allowInsecure": true,
          "alpn": ["http/1.1"]
        }
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    },
    {
      "tag": "block",
      "protocol": "blackhole",
      "settings": {}
    }
  ],
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "type": "field",
        "ip": [
          "geoip:private",
          "192.168.2.0/24"
        ],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "domain": [
          "domain:openai.com",
          "domain:chatgpt.com",
          "domain:oaistatic.com",
          "domain:oaiusercontent.com",
          "domain:openaiapi-site.azureedge.net",
          "domain:auth.openai.com",
          "domain:platform.openai.com",
          "domain:api.openai.com",
          "domain:cdn.openai.com",
          "domain:chat.openai.com",
          "domain:ab.chatgpt.com",
          "domain:files.oaiusercontent.com",
          "domain:chatgpt.livekit.cloud",
          "domain:featuregates.org",
          "regexp:(^|\\.)codex\\.",
          "regexp:(^|\\.)gpt\\.",
          "regexp:(^|\\.)openai\\.",
          "regexp:(^|\\.)chatgpt\\.",
          "keyword:openai",
          "keyword:chatgpt",
          "keyword:codex",
          "keyword:gpt"
        ],
        "outboundTag": "node1-openai"
      },
      {
        "type": "field",
        "network": "tcp,udp",
        "outboundTag": "node2-default"
      }
    ]
  }
}
```

## Appendix C: consolidated iptables Bash command area

> This appendix intentionally duplicates Sections 8 and 9 so the commands can be copied from one place.

### C.1 Forced transparent proxy: apply script

```bash
#!/usr/bin/env bash
set -e

iptables -t nat -N XRAY 2>/dev/null || true

# Deduplicate UDP 443 (QUIC) reject first, then add it
while iptables -C OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable 2>/dev/null; do
  iptables -D OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable
done
iptables -A OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable

# Deduplicate OUTPUT -> XRAY jump first, then add it
while iptables -t nat -C OUTPUT -p tcp -j XRAY 2>/dev/null; do
  iptables -t nat -D OUTPUT -p tcp -j XRAY
done
iptables -t nat -A OUTPUT -p tcp -j XRAY

# Rebuild XRAY chain
iptables -t nat -F XRAY
iptables -t nat -A XRAY -d 127.0.0.0/8 -j RETURN
iptables -t nat -A XRAY -d 10.0.0.0/8 -j RETURN
iptables -t nat -A XRAY -d 172.16.0.0/12 -j RETURN
iptables -t nat -A XRAY -d 192.168.0.0/16 -j RETURN
iptables -t nat -A XRAY -d 224.0.0.0/4 -j RETURN
iptables -t nat -A XRAY -d 240.0.0.0/4 -j RETURN
iptables -t nat -A XRAY -m owner --uid-owner 65534 -j RETURN
iptables -t nat -A XRAY -p tcp -j REDIRECT --to-ports 12345
```

### C.2 Forced transparent proxy: teardown script

```bash
#!/usr/bin/env bash
set -e
sudo iptables -D OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable 2>/dev/null || true
sudo iptables -t nat -D OUTPUT -p tcp -j XRAY || true
sudo iptables -t nat -F XRAY || true
sudo iptables -t nat -X XRAY || true
```

### C.3 Forced mode: common commands

```bash
sudo nano /usr/local/bin/xray-iptables-apply.sh
sudo chmod +x /usr/local/bin/xray-iptables-apply.sh
sudo /usr/local/bin/xray-iptables-apply.sh
sudo nano /usr/local/bin/xray-iptables-teardown.sh
sudo chmod +x /usr/local/bin/xray-iptables-teardown.sh
sudo /usr/local/bin/xray-iptables-teardown.sh
sudo iptables-save
```

### C.4 Health-check mode: script

```bash
#!/usr/bin/env bash
set -e

XRAY_CHAIN="XRAY"
XRAY_PORT="12345"

xray_is_healthy() {
  systemctl is-active --quiet xray && ss -ltn | awk '{print $4}' | grep -qE '(^|:)12345$'
}

apply_proxy_rules() {
  iptables -t nat -N "$XRAY_CHAIN" 2>/dev/null || true
  iptables -t nat -F "$XRAY_CHAIN"

  while iptables -C OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable 2>/dev/null; do
    iptables -D OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable
  done
  iptables -A OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable

  while iptables -t nat -C OUTPUT -p tcp -j "$XRAY_CHAIN" 2>/dev/null; do
    iptables -t nat -D OUTPUT -p tcp -j "$XRAY_CHAIN"
  done
  iptables -t nat -A OUTPUT -p tcp -j "$XRAY_CHAIN"

  iptables -t nat -A "$XRAY_CHAIN" -d 127.0.0.0/8 -j RETURN
  iptables -t nat -A "$XRAY_CHAIN" -d 10.0.0.0/8 -j RETURN
  iptables -t nat -A "$XRAY_CHAIN" -d 172.16.0.0/12 -j RETURN
  iptables -t nat -A "$XRAY_CHAIN" -d 192.168.0.0/16 -j RETURN
  iptables -t nat -A "$XRAY_CHAIN" -d 224.0.0.0/4 -j RETURN
  iptables -t nat -A "$XRAY_CHAIN" -d 240.0.0.0/4 -j RETURN
  iptables -t nat -A "$XRAY_CHAIN" -m owner --uid-owner 65534 -j RETURN
  iptables -t nat -A "$XRAY_CHAIN" -p tcp -j REDIRECT --to-ports "$XRAY_PORT"
}

apply_direct_rules() {
  while iptables -t nat -C OUTPUT -p tcp -j "$XRAY_CHAIN" 2>/dev/null; do
    iptables -t nat -D OUTPUT -p tcp -j "$XRAY_CHAIN"
  done
  iptables -t nat -F "$XRAY_CHAIN" 2>/dev/null || true

  while iptables -C OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable 2>/dev/null; do
    iptables -D OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable
  done
}

if xray_is_healthy; then
  echo "Xray healthy -> apply proxy rules"
  apply_proxy_rules
else
  echo "Xray unhealthy -> switch to direct mode"
  apply_direct_rules
fi
```

### C.5 Health-check mode: systemd files and enable commands

```bash
sudo nano /usr/local/bin/xray-iptables-healthcheck.sh
sudo chmod +x /usr/local/bin/xray-iptables-healthcheck.sh
sudo /usr/local/bin/xray-iptables-healthcheck.sh
sudo nano /etc/systemd/system/xray-iptables-healthcheck.service
sudo nano /etc/systemd/system/xray-iptables-healthcheck.timer
sudo systemctl daemon-reload
sudo systemctl enable xray-iptables-healthcheck.timer
sudo systemctl start xray-iptables-healthcheck.timer
sudo systemctl list-timers --all | grep xray
```
