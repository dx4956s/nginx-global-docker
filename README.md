# nginx-global-docker
# Fix : docker

Nginx reverse proxy with automatic SSL via Let's Encrypt, deployed with GitHub Actions.

Built on [jonasal/nginx-certbot](https://github.com/JonasAlfredsson/docker-nginx-certbot).

## How it works

- Nginx serves as a reverse proxy for multiple apps
- Certbot automatically issues and renews SSL certificates
- Each app gets its own `.conf.template` file
- GitHub Actions deploys on every push to `main`

## Adding an app

1. Create `nginx/your-app.conf.template`:

```nginx
server {
    listen 80;
    server_name ${YOURAPPDOMAIN};

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name ${YOURAPPDOMAIN};

    ssl_certificate     /etc/letsencrypt/live/${YOURAPPDOMAIN}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/${YOURAPPDOMAIN}/privkey.pem;
    ssl_dhparam         /etc/letsencrypt/dhparams/dhparam.pem;

    location / {
        proxy_pass         http://host.docker.internal:${YOURAPPPORT};
        proxy_http_version 1.1;
        proxy_set_header   Host       $host;
        proxy_set_header   X-Real-IP  $remote_addr;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}
```

2. Add to `docker-compose.yml` environment:
```yaml
YOURAPPDOMAIN: ${YOURAPPDOMAIN}
YOURAPPPORT: ${YOURAPPPORT}
```

3. Add to GitHub Actions secrets and the `.env` block in `deploy.yml`

## GitHub Actions secrets

| Secret | Description |
|---|---|
| `SSH_HOST` | Server IP or hostname |
| `SSH_USER` | SSH username |
| `SSH_KEY` | Private SSH key for server access |
| `CERTBOT_EMAIL` | Email for Let's Encrypt notifications |
| `RESUMEFORGEDOMAIN` | Domain for Resume Forge |
| `RESUMEFORGEPORT` | Port Resume Forge runs on |

## Server setup

### SSH access
Ensure your server's `~/.ssh/authorized_keys` contains the public key matching `SSH_KEY`.

### GitHub deploy key (for cloning private repos)
```bash
ssh-keygen -t ed25519 -f ~/.ssh/deploy_key_nginx_proxy -N ""
cat ~/.ssh/deploy_key_nginx_proxy.pub  # add this to GitHub repo → Settings → Deploy keys
```

Add to `~/.ssh/config`:
```
Host github-nginxglobaldocker
  HostName github.com
  IdentityFile ~/.ssh/deploy_key_nginx_proxy
```

### Rootless Docker (if needed)
If Docker is running in rootless mode, allow binding to ports 80/443:
```bash
echo 'net.ipv4.ip_unprivileged_port_start=80' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Upstream proxy

Apps can run anywhere on the host — Docker containers with published ports or plain local processes. `host.docker.internal` resolves to the host machine's IP, so any service listening on the host is reachable.
