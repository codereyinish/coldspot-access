# ColdSpot — Architecture

A **system-wide transparent proxy** that routes *all* of a Mac's traffic — even
apps with no proxy support — out through an **exit server you own**. A virtual
interface captures traffic at the **IP layer (Layer 3)**; `tun2socks` converts
those packets into a SOCKS stream; a **reverse tunnel** to a paired iPhone carries
each connection onward to a self-hosted exit (authenticated SOCKS5 over TLS) that
re-originates it to the internet. Capturing at Layer 3 means even apps that ignore
proxy settings get caught; the iPhone is a dumb relay, so the Mac↔exit
conversation is end-to-end.

---

## 1. The problem

```
GOAL: route a Mac's internet traffic out through an iPhone's cellular link,
      system-wide, including apps that know nothing about proxies.

├── Apps that support proxies (Safari)        → easy: point them at a SOCKS proxy
└── Apps that DON'T (git, CLIs, OS daemons)    → ignore proxy settings → they LEAK
        └── must be captured WITHOUT cooperation   ← the hard part
```

The two halves of the solution:

- **The tunnel** — get traffic from the Mac to the iPhone and out to the exit.
- **The capture** — force *every* app's traffic into that tunnel, even uncooperative ones.

---

## 2. High-level data flow

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

Two entry paths converge at `proxy.py` — Safari via SOCKS5, everything else via
L3 capture (`utun123 → tun2socks`) — then leave over **en0, never utun** (that's
what stops the loop) through a slot to the phone and out to the exit. On the way
back they **diverge again**: `proxy.py` writes straight to Safari's socket
(**direct**), while git's bytes are **re-packetized** into utun. The
`proxy.py ↔ exit` leg (SOCKS5 auth + TLS) is **end-to-end** — the iPhone only
shuttles bytes, never seeing the real destination.

---

## 3. Future work / improvements

- **Idle-slot reaper** — reclaim slots held by persistent idle connections
  (WebSockets, keepalives, DoH), but only under pressure (watermark-gated) to
  avoid needless reconnect churn.
- **UDP / QUIC** — the slots are TCP-only (SOCKS5 CONNECT), so UDP falls back to
  TCP; real support needs SOCKS5 UDP-ASSOCIATE or a TUN-level UDP path.
- **Adaptive pool sizing** — the pool is fixed at 30; it could shrink when idle
  and grow under sustained load.

See [ROADMAP.md](ROADMAP.md) for the fuller security roadmap (UDP path, the
WireGuard-deferred analysis, per-device credentials).
