# Tailscale

Current configuration through Unraid tailscale plugin

Jellyfin is being funneled through Tailscale to allow access to the server from outside the home network.

Commands to run on the server to get Tailscale working:

```bash
tailscale funnel --bg 8096

tailscale status
```

will return something like this:

```
# Funnel on:
#     - https://tower.risk-lenok.ts.net

https://tower.risk-lenok.ts.net (Funnel on)
|-- / proxy http://127.0.0.1:8096

```

# Update!

This is now handled through the Unraid Tailscale plugin directly. Jellyfin itself is now funneled without exposing any other applications on Unraid. It also auto starts, and keeps its own persistant URL.

