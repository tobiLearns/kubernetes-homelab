[<p style="text-align:right;">back to main README</p>](./README.md)
## Access to Applications via Cloudflare-Tunnel

To make a locally hosted application securely accessible from the internet, this setup uses a Cloudflare Tunnel. A Cloudflare Tunnel creates an outbound-only connection from a host inside your network to Cloudflare's edge — no open inbound ports or public IP required. The `cloudflared` daemon running on your host establishes the tunnel and proxies traffic to local services. Cloudflare routes incoming requests for your domain through the tunnel to the target service, while your network remains fully shielded behind your firewall.

### Set up Cloudflared
#### Connection Pre-Checks
- Perform [Connectivity pre-checks](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/troubleshoot-tunnels/connectivity-prechecks/):
    - Test DNS with your current resolver:
      ````bash
      dig A us-region1.v2.argotunnel.com
      dig AAAA us-region1.v2.argotunnel.com
      dig A us-region2.v2.argotunnel.com
      dig AAAA us-region2.v2.argotunnel.com
      ````
        - worked also for region1/2, but with lower TTL (Time-To-Live value, indicating the lifetime of the DNS resolvers cache)
    - Test network connectivity:
      ````bash
      nc -uvz -w 3 198.41.192.167 7844
          Connection to 198.41.192.167 7844 port [udp/*] succeeded!
      nc -vz -w 3 198.41.192.167 7844
          nc: connect to 198.41.192.167 port 7844 (tcp) timed out: Operation now in progress
      ````
        - This example output shows a problem, arising due to the firewall configuration of the router:
            - Outbound UDP is allowed, but TCP on port 7844 is blocked or inspected.
              cloudflared will only be able to connect using quic. If you force http2 in your configuration while TCP is blocked, the tunnel will fail.
              To resolve: Either allow TCP on your local network firewall on port 7844 or stop forcing http2 to allow cloudflared to connect over QUIC instead.
              Refer to the Protocol parameter documentation for more information.

#### Install cloudflared
- Add cloudflare gpg key:
  ````bash
  sudo mkdir -p --mode=0755 /usr/share/keyrings
  curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
  ````

- Add this repo to your apt repositories:
  ````bash
  echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
  ````

- install cloudflared:
  ````bash
  sudo apt-get update && sudo apt-get install cloudflared
  ````
- Authenticate cloudflared:
  ````bash
  cloudflared tunnel login
  ````
- This creates a file ~/.cloudflared/cert.pem

#### Create cloudflare tunnel
- Create a cloudflare tunnel:
  ````bash
  cloudflared tunnel create <tunnel-name>
  ````
  This also creates a credentials file in `~/.cloudflared/`.
- Create config file `~/.cloudflared/config.yaml`:
  ````yaml
  url: http://localhost:9090
  tunnel: <tunnel-ID>
  credentials-file: /home/<USER>/.cloudflared/<tunnel-ID>.json
  ````
- Create a CNAME record pointing to `<UUID>.cfargotunnel.com`:
  ````bash
  cloudflared tunnel route dns <tunnel-name> <subdomain>.<domain-at-cloudflare>
  ````
    - Or manually in cloudflare account:
        - Add DNS record CNAME:
            - Under "DNS/Records" add record with type "CNAME"
            - Field "name" represents the later subdomain, under which the tunnel is accessible
            - Field "target" has to be `<tunnel-ID>.cfargotunnel.com`

[<p style="text-align:right;">back to main README</p>](./README.md)
