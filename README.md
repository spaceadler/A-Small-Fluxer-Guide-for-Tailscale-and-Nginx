# Fluxer Self-Hosted Guide

So, you want to self-host Fluxer, but you already run Nginx Proxy Manager (NPM) and don't want Caddy hijacking your ports.

Now by default, Fluxer’s architecture heavily relies on Caddy as a "Router Brain" to strip API paths and serve frontend files. Putting a double reverse proxy in front of a Single Page Application (SPA) usually breaks the API routing.

This guide strips out the Docker "black box" anonymous volumes, maps everything locally, and teaches Nginx exactly how to bypass Caddy to feed data directly into the Fluxer API without triggering `404` or `502` HTML errors.

## 0. Some Pre-work

Follow the Fluxer documentation up until Step 2, setting up Docker, and downloading the necessary files.
The codeblocks that follow this (ESPECIALLY the ones that have comments) are what we will *change* the default configuration with.
You're still supposed to curl the files in the first place!

With the environment ready, let's begin.

## 1. Sourcing a Certificate (No Cloudflare)

Since Caddy will still handle the internal frontend, it strictly requires valid HTTPS certificates to load anything securely without screaming out a "NetworkError". If you don't use Cloudflare, you can handle this natively through NPM:

1. Go to your Nginx Proxy Manager dashboard.
2. Go to **SSL Certificates** -> **Add SSL Certificate** -> **Let's Encrypt**.
3. Generate a wildcard or specific domain cert for your domain (e.g., `chat.example.com`).

(Or get Tailscale to give out a certificate for your machine that you can use here. Warning, Tailscale hands out only one (1) certificate per machine. Although you can spin up more Tailscale containers, add them to your network, and farm out extra 4 certificates.)

## 2. Generate Your Secrets

Fluxer requires a massive amount of cryptographic keys. Instead of typing them manually, run this quick bash script in your terminal to generate the secure strings you need for your `.env` file.

```bash
for key in POSTGRES_PASSWORD MEILI_MASTER_KEY FLUXER_S3_SECRET_KEY \
  FLUXER_SUDO_MODE_SECRET FLUXER_CONNECTION_INITIATION_SECRET \
  FLUXER_GATEWAY_RPC_AUTH_TOKEN FLUXER_MEDIA_PROXY_SECRET_KEY \
  FLUXER_ADMIN_SECRET_KEY_BASE FLUXER_ADMIN_OAUTH_CLIENT_SECRET \
  LIVEKIT_API_SECRET; do
  sed -i "s|^$key=.*|$key=$(openssl rand -hex 32)|" .env
done

sed -i "s|^FLUXER_MEDIA_PROXY_UPLOAD_RELAY_SECRET_BASE64=.*|FLUXER_MEDIA_PROXY_UPLOAD_RELAY_SECRET_BASE64=$(openssl rand -base64 32)|" .env

VAPID=$(docker run --rm node:24-alpine npx --yes web-push generate-vapid-keys --json)
pub=$(printf '%s' "$VAPID" | grep -o '"publicKey":"[^"]*"' | cut -d'"' -f4)
priv=$(printf '%s' "$VAPID" | grep -o '"privateKey":"[^"]*"' | cut -d'"' -f4)
sed -i "s|^FLUXER_VAPID_PUBLIC_KEY=.*|FLUXER_VAPID_PUBLIC_KEY=$pub|" .env
sed -i "s|^FLUXER_VAPID_PRIVATE_KEY=.*|FLUXER_VAPID_PRIVATE_KEY=$priv|" .env

```

## 3. The `.env` Configuration

This environment file tells the frontend that the outside world communicates on standard port `443`, but the internal Caddy server is isolated on `8443` (This is a magic number, change to whatever you want if this port is already taken. I will stick to 8443 for the rest of this guide, feel free to replace every instance of this if you do this on your own.)

```env
# Core Routing
FLUXER_DOMAIN=chat.example.com  # change to your own preferred reverse-proxied domain.
FLUXER_PUBLIC_SCHEME=https      # https to allow voice and video
FLUXER_PUBLIC_PORT=443          # 443 to be passed as HTTPS and eventually land in Nginx, instead of Caddy.
FLUXER_CADDY_SITE_ADDRESS=:8443 # this is the port where you want Nginx to reach the service. Should be the same as in the compose file.

# (Paste all the generated secrets from the bash script below here)
# ...

```

## 4. LiveKit Networking (`livekit.yaml`)

If you are running this over a VPN (like Tailscale) or avoiding external IPs, ensure LiveKit binds to your specific host IP so UDP WebRTC packets don't get lost in Docker's internal networks.

```yaml
port: 7880
log_level: info

rtc:
  tcp_port: 7881
  udp_port: 7882
  use_external_ip: false
  node_ip: 100.124.254.50

turn:
  enabled: false

webhook:
  api_key: fluxer
  urls:
    - http://YOUR_SERVER_IP:8442/webhooks/livekit

```

## 5. The `docker-compose.yml` (Local Volumes & API Bypass)

We are moving away from anonymous Docker volumes to localized `./` folders so your data survives tear-downs. Crucially, we are exposing the `api` service to port `8442` so Nginx can reach it directly. `8442` is another magic number, change it to whatever you like.

```yaml
services:
  caddy:
    image: caddy:2.10-alpine
    restart: unless-stopped
    ports:
      - "8443:8443"
    environment:
      FLUXER_CADDY_SITE_ADDRESS: ${FLUXER_CADDY_SITE_ADDRESS:?set FLUXER_CADDY_SITE_ADDRESS in .env}
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro   # dont forget to
      - ./caddy-data:/data                    # add the dot "."
      - ./caddy-config:/config                # before the slash "/"
    depends_on: [api, gateway, media-proxy, static-proxy, admin]

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: fluxer
      POSTGRES_USER: fluxer
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - ./postgres-data:/postgres # also here

  valkey:
    image: valkey/valkey:8.1-alpine
    restart: unless-stopped
    command: ["valkey-server", "--save", "", "--appendonly", "no"]

  nats:
    image: nats:2.14-alpine
    restart: unless-stopped
    command: ["-js", "-sd", "/data", "-m", "8222"]
    volumes:
      - ./nats-data:/nats-data # dont forget here

  meilisearch:
    image: getmeili/meilisearch:v1.12
    restart: unless-stopped
    environment:
      MEILI_ENV: production
      MEILI_NO_ANALYTICS: "true"
      MEILI_MASTER_KEY: ${MEILI_MASTER_KEY}
    volumes:
      - ./meilisearch-data:/meili_data # here too

  seaweedfs:
    image: chrislusf/seaweedfs:4.34
    restart: unless-stopped
    command: ["server", "-s3", "-dir=/data"]
    volumes:
      - ./seaweedfs-data:/seaweedfs-data # almost there

  livekit:
    image: livekit/livekit-server:v1.12.0
    restart: unless-stopped
    network_mode: "host"  # <--- THIS IS THE MAGIC FIX
    command: ["--config", "/etc/livekit.yaml"]
    environment:
      LIVEKIT_KEYS: "${LIVEKIT_API_KEY}:${LIVEKIT_API_SECRET}"
    volumes:
      - ./livekit.yaml:/etc/livekit.yaml:ro # almooost there!
    #ports: <------- we comment these out because we gave livekit host network access to be able to read WebRTC calls.
    #  - "${FLUXER_LIVEKIT_TCP_PORT:-7881}:7881"
    #  - "${FLUXER_LIVEKIT_UDP_PORT:-7882}:7882/udp" 
  api:
    image: ${FLUXER_REGISTRY:-ghcr.io/${FLUXER_REGISTRY_OWNER:-fluxerapp}}/fluxer-api:${FLUXER_IMAGE_TAG:-v1}
    environment:
      # Pass through the env variables
      FLUXER_API_PORT: "8080" 
    ports:
      - "8442:8080" # THE CRITICAL BYPASS: Exposes the API directly to NPM!
                    # port 8080 was already taken so i created this to reroute it to 8442, we will use this port in nginx shortly.
```

## 6. The Nginx Proxy Manager (Advanced Routing)

If you blindly proxy Nginx to Caddy, Caddy will fail to route data requests (`/.well-known` and `/api/`) and will return `index.html` web pages to your browser, causing JavaScript crashes.

To fix this, we use path-based routing via raw Nginx configurations to send data straight to the API container while stripping necessary prefixes.

1. In NPM, edit your Proxy Host for `chat.example.com`.
2. **Details Tab:** Set the Forward IP to your Docker Host IP, and the Port to `8443`, http mode (yes, http mode). Select the certificate Force SSL.
3. **Custom Locations Tab:** Leave this completely blank.
4. **Advanced (cog on the top right):** Paste the following configuration, replacing `YOUR_SERVER_IP` with your actual host machine's IP (e.g., `192.168.1.50`) and port if need be.

```nginx
location /api/ {
    rewrite ^/api/(.*)$ /$1 break;
    proxy_pass http://YOUR_SERVER_IP:8442;
    
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}

location /.well-known/ {
    proxy_pass http://YOUR_SERVER_IP:8442;
    
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}

location /livekit/ {
    rewrite ^/livekit/(.*)$ /$1 break;
    proxy_pass http://100.124.254.50:7880;
    
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}

```

Start your stack (`docker compose up -d`). **Note:** The API container takes over 3 minutes to run each time, especially the first time. Once it is fully booted, your Nginx proxy will perfectly orchestrate traffic between the Caddy frontend and the bypassed API backend.


## 7. Troubleshooting

1. Verify that UFW is active and that your tailscale0 interface has a clean, green light to accept UDP and TCP traffic on ports 7880, 7881, and 7882