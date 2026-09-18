# Network proxy configuration (essential reading for mainland China users)

> Docker Hub, GitHub, and other resources are restricted for mainland China
> access; a proxy must be configured to pull images and updates normally.

---

## Contents

- [Proxy approach comparison](#proxy-approach-comparison)
- [Approach 1: Privoxy + SOCKS5 bridge (recommended)](#approach-1-privoxy--socks5-bridge-recommended)
- [Approach 2: Direct HTTP proxy configuration](#approach-2-direct-http-proxy-configuration)
- [Verify the proxy is working](#verify-the-proxy-is-working)
- [Common issues](#common-issues)

---

## Proxy approach comparison

Because the Docker daemon's HTTP client **only supports HTTP/HTTPS proxies,
not SOCKS5**, if you only have a SOCKS5 proxy (such as an OpenWrt side router,
SS/SSR/V2Ray client), you need an additional bridge.

| Approach | Pros | Cons | Recommended |
|----------|------|------|-------------|
| Privoxy + SOCKS5 bridge | Stable, works with any SOCKS5 proxy | One extra service to manage | ✅ |
| Direct HTTP proxy | Simple | Must have an HTTP proxy already | |
| No proxy | — | Cannot pull from Docker Hub | ❌ |

---

## Approach 1: Privoxy + SOCKS5 bridge (recommended)

### Install Privoxy on the host

```bash
apt install -y privoxy
```

### Configure Privoxy to forward to the SOCKS5 proxy

Edit the Privoxy configuration:

```bash
nano /etc/privoxy/config
```

Add at the end of the file:

```text
forward-socks5t / 127.0.0.1:7890 .
listen-address 127.0.0.1:8118
```

- `forward-socks5t / 127.0.0.1:7890`: forward all traffic to the SOCKS5 proxy
  running on the same machine at port 7890
- `listen-address 127.0.0.1:8118`: Privoxy listens on port 8118 as an HTTP proxy

### Start Privoxy

```bash
systemctl enable privoxy
systemctl start privoxy
```

### Configure Docker to use Privoxy

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/proxy.conf << 'EOF'
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:8118"
Environment="HTTPS_PROXY=http://127.0.0.1:8118"
Environment="NO_PROXY=localhost,127.0.0.1,192.168.0.0/16,10.0.0.0/8"
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

## Approach 2: Direct HTTP proxy configuration

If you already have an HTTP proxy (not SOCKS5):

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/proxy.conf << 'EOF'
[Service]
Environment="HTTP_PROXY=http://your-proxy:port"
Environment="HTTPS_PROXY=http://your-proxy:port"
Environment="NO_PROXY=localhost,127.0.0.1,192.168.0.0/16,10.0.0.0/8"
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

## Verify the proxy is working

```bash
# Test pulling an image
docker pull hello-world

# Test from within the Docker daemon
docker info | grep -i proxy

# Should show:
#  HTTP Proxy: http://127.0.0.1:8118
#  HTTPS Proxy: http://127.0.0.1:8118
```

---

## Common issues

### Privoxy is not forwarding traffic

→ Check that the SOCKS5 proxy is running on the same machine:
```bash
curl -x socks5h://127.0.0.1:7890 https://httpbin.org/ip
# Should return the proxy's IP
```

### Docker still cannot pull images

→ Check that the systemd proxy configuration was applied:
```bash
sudo systemctl show docker --property=Environment
# Should show HTTP_PROXY and HTTPS_PROXY
```

### apt / pip / npm are slow but Docker works

→ Configure the proxy for each tool separately:
```bash
# apt: /etc/apt/apt.conf.d/proxy.conf
# pip: ~/.pip/pip.conf or ~/.config/pip/pip.conf
# npm: ~/.npmrc
```

### Speed is still slow

→ The SOCKS5 proxy itself may be slow. Test the SOCKS5 proxy speed directly:
```bash
curl -x socks5h://127.0.0.1:7890 -o /dev/null -sw '%{speed_download}\n' \
  https://docker.com
```
