# ColdSpot — make your Mac disappear 🫥

*Your Mac goes quiet. The network just sees a server somewhere else doing the
talking.*

Route **all** of a Mac's traffic — every app, even ones that ignore proxy
settings — through a paired iPhone and out to a small **exit server you own**, so
the outside world meets your server's address instead of your Mac's. A
from-scratch look at how a VPN-like data path is actually built.

> **The code is in a private repo.** This page is the readable overview — if you
> want to experiment with it, [request access below](#-request-access).

## What it is

A virtual interface captures **all** of a Mac's traffic at Layer 3; `tun2socks`
turns those packets into a SOCKS stream; a reverse tunnel to a paired iPhone
carries each connection onward to a **self-hosted exit server you own**
(authenticated SOCKS5 over TLS) that re-originates it to the internet. Capturing
at Layer 3 means even apps that ignore proxy settings get caught; the iPhone is a
dumb relay, so the Mac↔exit conversation is end-to-end.

Built as **developer/educational material** — a working system to stand up and
learn from, end to end, not a product.

## How it works

Both directions in one figure — **blue solid = forward** (app → internet),
**amber dotted = return** (internet → app):

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0f172a','primaryTextColor':'#e5e7eb','primaryBorderColor':'#475569','lineColor':'#94a3b8','fontSize':'13px','clusterBkg':'#0b1220','clusterBorder':'#334155'}}}%%
flowchart BT
    subgraph MAC["Mac"]
        Safari["Safari<br/>proxy-aware"]
        CLI["git / CLI / daemons<br/>ignore the proxy"]
        utun["utun123<br/>L3 capture"]
        t2s["tun2socks<br/>packets ⇄ stream"]
        proxy["proxy.py<br/>SOCKS5 :1080 + 30 slots :9999"]
    end
    subgraph PHONE["iPhone"]
        app["iPhone app<br/>30 relay slots"]
    end
    subgraph EXIT["exit server (yours)"]
        ex["exit.py<br/>SOCKS5 over TLS"]
    end
    NET["internet<br/>news.com …"]

    %% forward (app → internet)
    Safari -->|"SOCKS5 (loopback)"| proxy
    CLI -->|"bypass → raw IP packets"| utun
    utun --> t2s
    t2s -->|"SOCKS5"| proxy
    proxy -->|"out en0, not utun — no loop · slot"| app
    app -->|"relay over uplink"| ex
    ex -->|"re-originate"| NET

    %% return (internet → app)
    NET -.->|"response"| ex
    ex -.->|"back over uplink"| app
    app -.->|"in via en0"| proxy
    proxy -.->|"direct"| Safari
    proxy -.->|"re-packetize"| t2s
    t2s -.-> utun
    utun -.->|"deliver"| CLI

    linkStyle 0,1,2,3,4,5,6 stroke:#38bdf8,stroke-width:2px,color:#7dd3fc
    linkStyle 7,8,9,10,11,12,13 stroke:#f59e0b,stroke-width:2px,color:#fbbf24
```

The iPhone is a **dumb pipe**: the Mac tells it only to dial the exit, then runs
TLS + authenticated SOCKS5 to the exit *through* that pipe, end-to-end — so the
connection to your server is encrypted and all config stays on the Mac.

## What it looks like

A ❄️ menu-bar toggle on the Mac, and a small relay app on the iPhone:

<p>
  <img src="docs/menubar-menu.png" alt="ColdSpot menu-bar toggle (Mac)" width="280">
  &nbsp;&nbsp;
  <img src="docs/iphone-app.png" alt="ColdSpot relay app (iPhone)" width="220">
</p>

Setup is basically one command — it builds a free Oracle Always-Free exit server
for you (Terraform), installs it over SSH, and wires up the Mac. The iPhone app is
built once from Xcode.

## Concepts it demonstrates

- Layer-3 vs Layer-5 interception (and why lower = unavoidable)
- routing-table internals — longest-prefix match, non-destructive default override
- reverse-tunnel design (inbound slots a NAT'd device holds open)
- SOCKS5 (CONNECT + auth) and TLS with certificate pinning
- self-hosted cloud infrastructure as code (Terraform on Oracle Always-Free)
- automated, host-key-safe SSH config exchange
- fail-safe, self-healing background services (launchd)

## 🔑 Request access

The full source lives in a **private** repo. To try it:

👉 **[Open a "Request access" issue](../../issues/new?template=request-access.yml)**
with your **GitHub username**.

Once approved, you're added as a **read-only collaborator** on the private repo —
you can clone and run it, just not change it. Requests are reviewed manually, so
I'll see who's asking. 🙂
