# Deployment

## Tailscale

We use Tailscale funnel to create a public tunnel:  
```bash
sudo tailscale set --operator=<mac_username>
tailscale funnel --bg 3000
```
This will a public url which does not rotate, something like: `https://dures-mac-mini.tail6b69fd.ts.net/`. Use this url for UAT purposes.
