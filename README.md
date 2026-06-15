# katisha — proxy

Nginx reverse proxy for the Katisha platform. The only container with public
ports (80 and 443). Routes traffic to all other services on `katisha-net` and
terminates TLS using Let's Encrypt certificates issued by `server-setup`.

---

## How it works

```
internet
    │
    ▼
nginx :80 / :443
    │  katisha-net
    ├── vault.katisha.online       → infisical:8080
    ├── api.katisha.online         → api-gw:8081
    ├── partner-api.katisha.online → api-gw:8081  (same backend; separate host = isolated auth cookie)
    ├── app.katisha.online         → app:3001
    ├── katisha.online          → www:80
    ├── www.katisha.online      → www:80
    ├── rabbitmq.katisha.online → rabbitmq:15672
    └── cdn.katisha.online      → cdn:8333
```

**Plug-n-play upstreams** — nginx uses Docker's internal DNS resolver
(`127.0.0.11`) with `proxy_pass` variables. This means:
- nginx starts successfully even if upstream containers are not running
- upstream containers come online automatically with no nginx restart needed

---

## Repository layout

```
proxy/
├── Dockerfile                  # nginx:alpine base
├── docker-compose.yml          # container, ports, cert mounts
├── nginx.conf                  # global config, resolver, shared headers
├── conf.d/
│   ├── vault.conf              # vault.katisha.online       → infisical:8080
│   ├── api.conf                # api.katisha.online         → api-gw:8081
│   ├── partner-api.conf        # partner-api.katisha.online → api-gw:8081 (operator portal)
│   ├── app.conf                # app.katisha.online         → app:3001
│   ├── www.conf                # katisha.online + www    → www:80
│   ├── rabbitmq.conf           # rabbitmq.katisha.online → rabbitmq:15672
│   └── cdn.conf                # cdn.katisha.online      → cdn:8333
├── actions.env                 # fill and use to set GitHub secrets
├── .github/workflows/
│   └── deploy.yml
└── README.md
```

---

## Adding a new service

1. Create `conf.d/<service>.conf` following the pattern of any existing file
2. Use a `set $upstream http://<container>:<port>;` variable in `proxy_pass`
3. Add the certbot webroot `location` block to the HTTP server block
4. Commit and push — nginx reloads automatically on deploy

---

## TLS certificates

Certificates live at `/etc/letsencrypt/live/<domain>/` on the host and are
mounted read-only into the container. They are issued and renewed by
`server-setup/setup.sh` — nothing in this repo manages certs.

Certbot renewal uses webroot mode — nginx keeps running during renewal.
Every HTTP server block must include:

```nginx
location /.well-known/acme-challenge/ {
    root /var/www/certbot;
}
```

All existing `conf.d/` files already include this.

---

## GitHub Actions secrets

| Secret | Description |
|---|---|
| `SERVER_HOST` | Server IP or hostname |
| `SERVER_USER` | SSH username |
| `SERVER_SSH_KEY` | Private SSH key |

---

## Service ports reference

| Domain | Container | Port |
|---|---|---|
| `vault.katisha.online` | `infisical` | `8080` |
| `api.katisha.online` | `api-gw` | `8081` |
| `partner-api.katisha.online` | `api-gw` | `8081` |
| `app.katisha.online` | `app` | `3001` |
| `katisha.online` / `www.katisha.online` | `www` | `80` |
| `rabbitmq.katisha.online` | `rabbitmq` | `15672` |
| `cdn.katisha.online` | `cdn` | `8333` |
