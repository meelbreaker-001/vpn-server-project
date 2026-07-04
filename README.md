# VPN Server Project (Python + WireGuard)

A Python-based VPN server management system built on top of **WireGuard**.
Python is used as the control/orchestration layer (peer management, REST API,
monitoring); WireGuard handles the actual cryptographic tunneling, since VPN
crypto should never be hand-rolled.

## Architecture

```
Client  <---encrypted tunnel (WireGuard)--->  VPN Server
                                                  |
                                          Python control layer
                                          (peer mgmt, REST API,
                                           monitoring, NAT setup)
```

- **Tunneling**: WireGuard (`wg`, `wg-quick`)
- **Encryption**: Curve25519 (key exchange), ChaCha20-Poly1305 (cipher)
- **NAT/Routing**: iptables/nftables masquerading + IP forwarding
- **Control plane**: Python (FastAPI REST API for peer management)

## Project Structure

```
vpn-server-project/
├── vpnserver/
│   ├── main.py              # entrypoint / CLI
│   ├── config.py            # env-based settings
│   ├── core/
│   │   ├── wireguard.py     # wraps wg / wg-quick commands
│   │   ├── keys.py          # keypair generation
│   │   └── network.py       # iptables / NAT / IP forwarding
│   ├── peers/
│   │   ├── manager.py       # add/remove/list peers
│   │   └── models.py        # Peer data model
│   ├── api/
│   │   ├── app.py           # FastAPI app
│   │   └── routes.py        # REST endpoints
│   ├── monitoring/
│   │   └── stats.py         # traffic stats, connected peers
│   └── utils/
│       ├── logger.py
│       └── qrcode_gen.py
├── scripts/
│   ├── install.sh           # server bootstrap script
│   └── systemd/vpnserver.service
├── tests/
├── docs/architecture.md
└── .github/workflows/       # CI + CD pipelines
```

## Setup (local dev)

```bash
git clone <your-repo-url>
cd vpn-server-project
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # fill in your values
```

## Server Bootstrap (production)

```bash
sudo bash scripts/install.sh
sudo systemctl enable --now vpnserver
```

## Running the API

```bash
uvicorn vpnserver.api.app:app --host 0.0.0.0 --port 8000
```

Then add a client:

```bash
curl -X POST http://localhost:8000/peers -d '{"name": "laptop"}'
```

This returns a WireGuard client config + QR code you can scan in the
WireGuard mobile app.

## GitHub Deployment

1. Push this repo to GitHub.
2. Add repo secrets: `SSH_HOST`, `SSH_USER`, `SSH_KEY`.
3. CI (`ci.yml`) runs lint + tests on every push/PR.
4. CD (`deploy.yml`) SSHes into your VPS on merge to `main`, pulls latest
   code, and restarts the `vpnserver` systemd service.

## Security Notes

- Server private keys never leave the server.
- `.env`, `*.conf`, and key files are git-ignored — never commit secrets.
- API should sit behind auth (token/JWT) before exposing publicly.
- Firewall should only allow the WireGuard UDP port + API port (ideally
  restricted to your own IP or behind a VPN-only admin panel).

## License

MIT — see LICENSE.
