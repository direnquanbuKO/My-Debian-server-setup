# My Debian Server Setup Guide

A personal reference guide for setting up and hardening a Debian x86_64 cloud server. Covers system configuration, SSH security, firewall rules, time synchronization, memory management, kernel tuning, network optimization, serving websites with Nginx and so on.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [System Upgrade & Package Installation](#system-upgrade--package-installation)
- [SSH Security Hardening](#ssh-security-hardening)
- [Firewall Configuration](#firewall-configuration)
- [NTP Time Zone Synchronization](#ntp-time-zone-synchronization)
- [Configure ZRAM](#configure-zram)
- [Configure SWAP](#configure-swap)
- [Linux Kernel Configuration](#linux-kernel-configuration)
- [Network Optimization](#network-optimization)
- [SSD Trim Optimization](#ssd-trim-optimization)
- [Shell Customization](#shell-customization)
- [Nginx Install & Configuration](#nginx-install--configuration)
- [Additional Recommended Tools](#additional-recommended-tools)

---

## Prerequisites

- A fresh Debian x86_64 cloud server (Debian 12 Bookworm or Debian 13 Trixie)
- Root or sudo access
- An SSH key pair generated on your local machine
- Basic familiarity with the Linux command line

---

## System Upgrade & Package Installation

Update and upgrade the system, then install a set of practical CLI tools:

```bash
apt update && apt upgrade -y
apt install info curl lsof htop man bind9-dnsutils unattended-upgrades ca-certificates unzip watchdog tmux wget tre-command btop command-not-found pipx net-tools iputils-ping sudo gnupg cron -y && apt update && update-command-not-found
```

Download and install additional packages from GitHub releases:

> ⚠️ **Note:** The version numbers below may be outdated. Check each project's GitHub releases page for the latest version before downloading.

```bash
wget https://github.com/lsd-rs/lsd/releases/download/v1.2.0/lsd_1.2.0_amd64.deb \
     https://github.com/muesli/duf/releases/download/v0.9.1/duf_0.9.1_linux_amd64.deb \
     https://github.com/sharkdp/bat/releases/download/v0.26.1/bat_0.26.1_amd64.deb \
     https://github.com/sharkdp/fd/releases/download/v10.4.2/fd_10.4.2_amd64.deb

dpkg -i lsd_1.2.0_amd64.deb duf_0.9.1_linux_amd64.deb bat_0.26.1_amd64.deb fd_10.4.2_amd64.deb && apt install -f
rm lsd_1.2.0_amd64.deb duf_0.9.1_linux_amd64.deb bat_0.26.1_amd64.deb fd_10.4.2_amd64.deb
```

Install shell utilities and configure aliases:

```bash
mkdir -m 700 ~/.gnupg
modprobe softdog
echo "softdog" | tee /etc/modules-load.d/softdog.conf
pipx install tldr
curl -sSfL https://raw.githubusercontent.com/ajeetdsouza/zoxide/main/install.sh | sh
echo 'eval "$(zoxide init bash)"' >> ~/.profile
echo "alias ls='lsd'" >> ~/.bashrc
pipx ensurepath && source ~/.bashrc && source ~/.profile
tldr -u
dpkg-reconfigure unattended-upgrades
nano /etc/watchdog.conf
```

> **Watchdog configuration tips:** Key parameters to set in `/etc/watchdog.conf` include `watchdog-device = /dev/watchdog` (the hardware/software watchdog device), `interval = 10` (heartbeat interval in seconds), and `max-load-1 = 24` (reboot if 1-minute load average exceeds this value).
>
> **Optional packages:** `apt-transport-https axel vim inxi`
>
> **For compilation:** `apt install autoconf automake libtool pkg-config cmake build-essential git`

---

## SSH Security Hardening

### 1. Configure Public Key Authentication

Add your public key to the server:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "YOUR_PUBLIC_KEY_HERE" | tee ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys
```

### 2. Edit `sshd_config`

Create a dedicated drop-in configuration file:

```bash
tee /etc/ssh/sshd_config.d/99-custom.conf << EOF
LoginGraceTime 30
StrictModes yes
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding no
ClientAliveInterval 60
ClientAliveCountMax 3
UseDNS no
EOF
```

| Parameter | Value | Effect |
| --- | --- | --- |
| `LoginGraceTime` | `30` | Reduce login grace time for automatic disconnection if no password/key is entered for more than 30 seconds |
| `StrictModes` | `yes` | Enable strict mode to check home directory permissions |
| `PubkeyAuthentication` | `yes` | Enable SSH public key login |
| `PasswordAuthentication` | `no` | Disable password authentication (use keys only) |
| `PermitEmptyPasswords` | `no` | Disable empty passwords |
| `KbdInteractiveAuthentication` | `no` | Disable interactive keyboard authentication (to prevent bypassing password restrictions via PAM) |
| `UsePAM` | `yes` | Debian requires yes by default; otherwise, it may affect some session functions |
| `X11Forwarding` | `no` | Disable X11 Forwarding as it's an unused auth method |
| `ClientAliveInterval` | `60` | For maintaining SSH session connection. The server sends a heartbeat packet every 60 seconds |
| `ClientAliveCountMax` | `3` | Disconnect only if the client does not respond after 3 attempts |
| `UseDNS` | `no` | Disable DNS reverse lookup |

> **Optional:** Customize the port sshd listens on (not strictly necessary):
>
> ```text
> Port 65535
> ```

### 3. Validate and Restart SSH

```bash
sshd -t
systemctl restart ssh
```

### 4. Install and Configure fail2ban

```bash
apt install fail2ban
```

**For Debian 12:**

```bash
tee /etc/fail2ban/jail.d/sshd.conf << EOF
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1
banaction = nftables
banaction_allports = nftables[type=allports]

[sshd]
backend = systemd
maxretry = 3
bantime = 1h
mode = aggressive
EOF
```

**For Debian 13:**

```bash
tee /etc/fail2ban/jail.d/sshd.conf << EOF
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1

[sshd]
mode = aggressive
maxretry = 3
bantime = 1h
EOF
```

Restart and verify:

```bash
systemctl restart fail2ban
fail2ban-client status sshd
```

---

## Firewall Configuration

### nftables (strongly recommended)

#### 1. Install nftables

```bash
apt install nftables
```

#### 2. Configure nftables

```bash
tee /etc/nftables.conf << EOF
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
        chain input {
                type filter hook input priority filter; policy drop;
                iif lo accept
                ct state invalid drop
                ct state { established, related } accept
                ip protocol icmp accept
                ip6 nexthdr icmpv6 accept
                tcp dport { 22, 80, 443 } accept
                udp dport 443 accept
        }
        chain forward {
                type filter hook forward priority filter; policy drop;
        }
        chain output {
                type filter hook output priority filter; policy accept;
        }
}
EOF
```

#### 3. Apply changes and verify

```bash
nft -c -f /etc/nftables.conf       # Validate syntax
systemctl enable --now nftables    # Apply and enable on boot
nft list ruleset                   # Verify the live ruleset
```

### Firewalld

> ⚠️ **Note:** Firewalld may conflict with cloud-init on some providers.

#### 1. Install firewalld

```bash
apt install firewalld
```

#### 2. Add Firewall Rules

```bash
firewall-cmd --add-service=http --permanent
firewall-cmd --add-service=https --permanent
```

> **Optional — HTTP/3 support:**
>
> ```bash
> firewall-cmd --add-service=http3 --permanent
> ```

#### 3. Apply and Verify

```bash
firewall-cmd --reload
firewall-cmd --list-all
```

---

## NTP Time Zone Synchronization

### 1. Install Chrony

Replace `systemd-timesyncd` with Chrony for better performance:

```bash
systemctl mask systemd-timesyncd && apt purge systemd-timesyncd && apt install chrony
```

### 2. Add Cloudflare NTP with NTS

```bash
echo "server time.cloudflare.com iburst nts prefer" >> /etc/chrony/conf.d/cloudflare-ntp.conf
```

### 3. Apply and Verify

```bash
systemctl restart chrony && chronyc sources -v && chronyc authdata
```

---

## Configure ZRAM

### 1. Install and Configure zramtool

```bash
apt install zram-tools
nano /etc/default/zramswap
```

```bash
# Percentage of RAM to use for ZRAM (e.g. 50% of 1GB = ~512MB compressed)
PERCENT=50

# Compression algorithm: lz4 is fastest; zstd offers better compression
ALGO=lz4

# Priority: higher value = preferred over disk swap
PRIORITY=100
```

### 2. Apply and Verify

```bash
systemctl restart zramswap && zramctl
```

---

## Configure SWAP

### 1. Create and Enable Swapfile

```bash
fallocate -l 1G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw,pri=-2 0 0' | tee -a /etc/fstab
```

### 2. Verify

```bash
free -h
swapon --show
cat /proc/swaps
```

### 3. Tune Kernel Swap Parameters

**For Debian 12 and older:**

```bash
echo "vm.swappiness=70
vm.vfs_cache_pressure=150" >> /etc/sysctl.conf
sysctl -p
```

**For Debian 13 and newer:**

```bash
echo "vm.swappiness=70
vm.vfs_cache_pressure=150" >> /etc/sysctl.d/swap-configuration.conf
sysctl -p /etc/sysctl.d/swap-configuration.conf
```

| Parameter | Value | Effect |
| --- | --- | --- |
| `vm.swappiness` | `70` | Only swaps when ~30% of RAM is used, it shouldn't be too high. The default value is 60 |
| `vm.vfs_cache_pressure` | `150` | Reduces cache pressure — good for low-RAM servers, but not too high. The default value is 100 |

More infomation can be seen from [Gluster Docs](https://docs.gluster.org/en/main/Administrator-Guide/Linux-Kernel-Tuning) and [Official Documentation](https://www.kernel.org/doc/Documentation/sysctl/vm.txt)

---

## Linux Kernel Configuration

### 1. Switch to the Cloud Kernel

```bash
apt install -y linux-image-cloud-amd64
```

### 2. Reboot

```bash
reboot
```

### 3. Remove the Old Kernel

List installed kernel packages:

```bash
dpkg -l | grep linux-image
```

Remove the old metapackage and its specific version (replace version numbers with those from the output above):

```bash
apt purge -y linux-image-amd64 linux-image-6.12.63+deb13-amd64
update-grub
apt autoremove --purge -y
```

---

## Network Optimization

Enables BBR TCP congestion control and FQ packet scheduling — especially beneficial for proxy workloads.

**For Debian 12 and older:**

```bash
echo "net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
```

**For Debian 13 and newer:**

```bash
echo "net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.d/bbr-optimization.conf
sysctl -p /etc/sysctl.d/bbr-optimization.conf
```

---

## SSD Trim Optimization

### 1. Check Trim Support

Confirm the `DISC-MAX` column is non-zero:

```bash
lsblk -D
```

### 2. Enable Scheduled Trim

```bash
systemctl enable fstrim.timer
```

### 3. Run Trim Manually

```bash
fstrim -v /
```

---

## Shell Customization

### Install Starship

#### Prerequisites

- A Nerd Font installed and enabled in your terminal (for example, try the FiraCode Nerd Font).

```bash
curl -sS https://starship.rs/install.sh | sh
```

#### Set up your shell to use Starship

Add the following to the end of `~/.bashrc`:

```bash
echo 'eval "$(starship init bash)"' >> ~/.bashrc
```

#### Configure Starship

Personally I use the Catppuccin Powerline Preset for the theme:

```bash
starship preset catppuccin-powerline -o ~/.config/starship.toml
```

Then set `palette` to `catppuccin_latte` to change the flavour, and add hostname behind the user. In `~/.config/starship.toml`:

```text
$username\
$hostname\
[](bg:peach fg:red)\
```

```text
[username]
...

[hostname]
style = "bg:red fg:crust"
format = '[ $hostname ]($style)'

[directory]
...
```

---

## Nginx Install & Configuration

### 1. Install Nginx from the official APT repository (More information from the [Nginx official website](https://nginx.org/en/linux_packages.html#Debian))

Install the prerequisites:

```bash
apt install curl gnupg2 ca-certificates lsb-release debian-archive-keyring
```

Import an official nginx signing key so apt can verify the package's authenticity. Fetch the key:

```bash
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
    | tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
```

Verify that the downloaded file contains the proper key:

```bash
gpg --dry-run --quiet --no-keyring --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg
```

The output should contain the full fingerprint `573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62` as follows:

```text
pub   rsa2048 2011-08-19 [SC] [expires: 2027-05-24]
      573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62
uid                      nginx signing key <signing-key@nginx.com>
```

Note that the output can contain other keys used to sign the packages.

To set up the apt repository for stable nginx packages, run the following command:

```bash
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
https://nginx.org/packages/debian `lsb_release -cs` nginx" \
    | tee /etc/apt/sources.list.d/nginx.list
```

Set up repository pinning to prefer our packages over distribution-provided ones:

```bash
echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
    | tee /etc/apt/preferences.d/99nginx
```

To install nginx, run the following commands:

```bash
apt update
apt install nginx
```

### 2. Add config files

Backup the default config file:

```bash
mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.bak
```

#### Create a custom config file for your own website

The Nginx folder includes a `nginx.conf` file for default configuration. Place your site config files in the `conf.d` folder and backup the `default.conf` file. Here is an example for most static sites:

```nginx
server_tokens off;
tcp_nopush on;
tcp_nodelay on;

gzip on;
gzip_min_length 1024;
gzip_comp_level 6;
gzip_vary on;
gzip_proxied any;
gzip_types
    application/javascript application/json
    image/svg+xml image/x-icon
    text/css text/javascript text/plain;

add_header_inherit merge;

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    return 444;
}

server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 308 https://$host$request_uri;
}

server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    ssl_reject_handshake on;
}

server {
    listen 443 ssl;
    listen 443 quic reuseport;
    listen [::]:443 ssl;
    listen [::]:443 quic reuseport;
    http2 on;
    server_name example.com www.example.com;

    root /var/www/html;
    index index.html;

    ssl_certificate /etc/nginx/certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/example.com/private.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers off;
    ssl_ecdh_curve X25519MLKEM768:X25519:prime256v1:secp384r1;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;
    ssl_session_timeout 1d;

    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/nginx/certs/example.com/cert.pem;
    resolver 127.0.0.1;
    resolver_timeout 5s;

    quic_retry on;

    add_header Alt-Svc 'h3=":443"; ma=86400' always;
    add_header Content-Security-Policy "default-src 'none'; connect-src 'self'; font-src 'self' data:; img-src 'self' data:; media-src 'self'; script-src 'self' 'wasm-unsafe-eval'; script-src-elem 'self'; style-src 'self' 'unsafe-inline'" always;
    add_header Cross-Origin-Opener-Policy "noopener-allow-popups" always; # noopener-allow-popups, same-origin or same-origin-allow-popups
    add_header Cross-Origin-Resource-Policy "same-origin" always; # same-origin or same-site
    add_header Referrer-Policy "strict-origin-when-cross-origin" always; # no-referrer, same-origin, strict-origin or strict-origin-when-cross-origin
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always; # DENY or SAMEORIGIN
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    location ~* \.(css|js|woff2?|svg|png|jpg|jpeg|gif|ico|webp|avif|mp3)$ {
        expires max;
        add_header Cache-Control "public, immutable";
        access_log off;
        try_files $uri =404;
    }

    location ~ /\. {
        deny all;
    }

    location ~ \.php$ {
        deny all;
    }

    location / {
        try_files $uri $uri/ =404;
    }

    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;

    location = /50x.html {
        root /usr/share/nginx/html;
    }
}

```

---

## Additional Recommended Tools

### 1. [NextTrace](https://github.com/nxtrace/Ntrace-core), an open source visual route tracking CLI tool

#### Automated Install

- Debian / Ubuntu
  - Recommended: install from the official `nexttrace-debs` APT repository
    - Supports: `amd64`, `i386`, `arm64`, `armel`, `armhf`, `loong64`, `mipsel`, `mips64el`, `ppc64el`, `riscv64`, `s390x`
    - Add the repository and install the default package:

      ```shell
      sudo install -d -m 0755 /etc/apt/keyrings
      curl -fsSL -o /tmp/nexttrace-archive-keyring.gpg https://github.com/nxtrace/nexttrace-debs/releases/latest/download/nexttrace-archive-keyring.gpg
      sudo install -m 0644 /tmp/nexttrace-archive-keyring.gpg /etc/apt/keyrings/nexttrace.gpg
      rm -f /tmp/nexttrace-archive-keyring.gpg
      printf '%s\n' 'Types: deb' 'URIs: https://github.com/nxtrace/nexttrace-debs/releases/latest/download/' 'Suites: ./' 'Signed-By: /etc/apt/keyrings/nexttrace.gpg' | sudo tee /etc/apt/sources.list.d/nexttrace.sources >/dev/null
      sudo apt update
      sudo apt install nexttrace
      ```

    - Optionally install additional flavors:

      ```shell
      sudo apt install nexttrace-tiny
      sudo apt install ntr
      ```

    - Packages can be installed side by side. Commands: `nexttrace`, `nexttrace-tiny`, `ntr`

- Linux / macOS / BSD
  - One-click installation script (Full, default)

    ```shell
    curl -sL https://nxtrace.org/nt | bash
    ```

  - One-click installation script (Tiny)

    ```shell
    curl -sL https://nxtrace.org/nt | bash -s -- --flavor tiny
    ```

  - One-click installation script (NTR)

    ```shell
    curl -sL https://nxtrace.org/nt | bash -s -- --flavor ntr
    ```

  - Installed command names: Full `nexttrace`, Tiny `nexttrace-tiny`, NTR `ntr`

Please note:

- The `nexttrace-debs` APT repository is maintained by nxtrace and wcbing.
- Other package sources above are maintained by open-source enthusiasts. Availability and timely updates are not guaranteed. If you encounter problems, please contact the repository maintainer to solve them, or use the binary packages provided by the official build of this project.

### 2. [dnscrypt-proxy](https://github.com/DNSCrypt/dnscrypt-proxy) - A flexible DNS proxy, with support for encrypted DNS protocols

> 📄 *Instructions adapted from the [official dnscrypt-proxy wiki](https://github.com/DNSCrypt/dnscrypt-proxy/wiki/Installation-linux).*

#### Step 1: download and extract dnscrypt-proxy to `/etc` directory

```shell
wget https://github.com/DNSCrypt/dnscrypt-proxy/releases/download/2.1.15/dnscrypt-proxy-linux_x86_64-2.1.15.tar.gz
tar xzf dnscrypt-proxy-linux_x86_64-2.1.15.tar.gz -C /etc/ && mv /etc/linux-x86_64 /etc/dnscrypt-proxy
rm dnscrypt-proxy-linux_x86_64-2.1.15.tar.gz && cd /etc/dnscrypt-proxy
```

You're going to install a new command and important configuration files on your system. If you're new to Linux, for good hygiene, you can quickly check that the directory you extracted cannot be modified by other system users. Here's one way to do it. Type:

```sh
sudo -u \#1 touch .
```

(don't forget the final `.`, and there's a space before)

If the command asks for a password, enter your password. If the command immediately returns without printing an error, it's recommended to check the permissions of this directory and all its parent directories ([tutorial on Linux permissions](https://www.howtogeek.com/437958/how-to-use-the-chmod-command-on-linux/#:~:text=Linux%20file%20permissions%20can%20be,to%20add%20or%20remove%20permissions.)).

#### Step 2: check what else is possibly already listening to port 53

If you already have a local DNS cache, it has to be eventually replaced with dnscrypt-proxy. Both can be used simultaneously, but this is outside of the scope of this guide (or, at least, of this Wiki page).

Type the following command:

```sh
ss -lp 'sport = :domain'
```

This may output something similar to:

```text
tcp    LISTEN     0      128    127.0.0.1:domain                *:*                     users:(("unbound",pid=28146,fd=6))
tcp    LISTEN     0      128    127.0.0.1:domain                *:*                     users:(("unbound",pid=28146,fd=4))
```

Uninstall the corresponding package (in the above example: `unbound`), with a distribution-specific command such as `apt-get remove` or `pacman -R`, then check again with `ss -lp 'sport = :domain'`: there shouldn't be anything listening to the `domain` port any more.

You may also see the port being served by `systemd-resolved`. That one cannot be uninstalled, but can be disabled with the following commands:

```sh
systemctl stop systemd-resolved
systemctl disable systemd-resolved
```

Check that nothing is listening to port 53 any more:

```sh
ss -lp 'sport = :domain'
```

Looks fine? Let's move to the next step.

#### Step 3: Tweak the configuration file

Create a configuration file based on the example one:

```sh
cp example-dnscrypt-proxy.toml dnscrypt-proxy.toml
```

You must still be in the `dnscrypt-proxy` directory at this point.

The `dnscrypt-proxy.toml` file has plenty of options you can tweak. Tweak them if you like. But tweak them one by one, so that if you ever screw up, you will know what exact change made this happen.

The message `bare keys cannot contain '\n'` typically means that there is a syntax error in the configuration file.

Type `./dnscrypt-proxy` to start the server, and `Control`-`C` to stop it. Test, tweak, stop, test, tweak, stop until you are satisfied.

Are you satisfied? Good, let's jump to step 4!

#### Step 4: change the system DNS settings

Does your system have a directory called `/etc/resolvconf` (not the `resolv.conf` file)? If this is the case, remove it:

```sh
apt-get remove resolvconf
```

Now, make a backup of the `/etc/resolv.conf` file:

```sh
cp /etc/resolv.conf /etc/resolv.conf.backup
```

Then delete the `/etc/resolv.conf` file (important, since this can be a dangling link instead of an actual file):

```sh
rm -f /etc/resolv.conf
```

And create a new `/etc/resolv.conf` file with the following content:

```text
nameserver 127.0.0.1
options edns0
```

> **Optional:** Add `trust-ad` to options if your DNS provider does support DNSSEC validation  
> Add `rotate` for distributing the query load if you set multiple nameservers

Let's check that everything works by sending a first query using dnscrypt-proxy:

```sh
./dnscrypt-proxy -resolve example.com
```

Looks like it was successfully able to resolve `example.com`? Sweet! Try a few more things: web browsing, file downloads, use your system normally and see if you can still connect without any DNS-related issues.

If anything ever goes wrong and you want to revert everything:

- If you uninstalled `resolvconf`, reinstall it with `apt-get install resolvconf`
- Restore the `/etc/resolv.conf` backup: `cp /etc/resolv.conf.backup /etc/resolv.conf`
- If you really can't resolve anything any more, even after rebooting, put this in `/etc/resolv.conf`: `nameserver 1.0.0.1`

#### Step 5: install the proxy as a system service

Hit `Control` and `C` in the `dnscrypt-proxy` terminal window to stop the proxy.

Now, register this as a system service (still with `root` privileges, and while being in the directory containing the configuration files):

```sh
./dnscrypt-proxy -service install
```

If it doesn't spit out any errors, this is great! Your Linux distribution is compatible with the built-in installer.

This assumes that the executable and the configuration file are in the same directory. If you didn't follow these recommendations, you're on your own modifying the unit files.

Now that it's installed, it can be started:

```sh
./dnscrypt-proxy -service start
```

Done!

#### Troubleshooting

Does it look like it started properly? If not, try to find out why. Here are some hints:

- `dnscrypt-proxy.toml: no such file or directory`: copy the example configuration file as `dnscrypt-proxy.toml` as documented above.
- `not found ELF - not found - Syntax error: ")" unexpected` or something similar: you didn't download the correct file for your operating system and CPU.
- `listen udp 127.0.0.1:53: bind: permission denied`: you are not using a root shell (see step 1). Use `sudo -s` to get one. Or `su` if `sudo` doesn't exist on your system.
- `listen udp 127.0.0.1:53: bind: address already in use`: something is already listening to the DNS port. Maybe something else, maybe a previous instance of dnscrypt-proxy that you didn't stop before starting a new one. Go back to step 2 and try again.
- `dnscrypt-proxy.socket: TCP_NODELAY failed: Protocol not available`: Those warnings are expected when using systemd socket activation and can be safely ignored. They happen because systemd tries to apply TCP only options for UDP socket. This shouldn't affect functionality.
- `dnscrypt-proxy.socket: TCP_DEFER_ACCEPT failed: Protocol not available`: ditto.
- `systemctl failed`: you jumped the gun and didn't follow the instructions above.
- `<something about IPv6 not being available>`: edit `dnscrypt-proxy.toml` and remove `, [::1]:53` from `listen_addresses`.

No errors? Amazing!

If it does spit out errors, consult your Linux distribution's documentation for specific instructions.

`Failed to start DNSCrypt client proxy: "systemctl" failed: exit status 5` means that you tried to `start` the service without `install`ing it first.

Want to stop the service?

```sh
./dnscrypt-proxy -service stop
```

Want to restart the currently running service after a configuration file change?

```sh
./dnscrypt-proxy -service restart
```

Want to uninstall the service?

```sh
./dnscrypt-proxy -service uninstall
```

Want to check that DNS resolution works?

```sh
./dnscrypt-proxy -resolve example.com
```

Want to completely delete that thing?

Delete the directory. Done.
