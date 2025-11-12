# Check SSL Certificate Validity with Nginx & Docker Compose
```
 _______  _______  ______    _______      __   __  _______  ___      ___   ______   ___   _______  __   __ 
|       ||       ||    _ |  |       |    |  | |  ||   _   ||   |    |   | |      | |   | |       ||  | |  |
|       ||    ___||   | ||  |_     _|    |  |_|  ||  |_|  ||   |    |   | |  _    ||   | |_     _||  |_|  |
|       ||   |___ |   |_||_   |   |      |       ||       ||   |    |   | | | |   ||   |   |   |  |       |
|      _||    ___||    __  |  |   |      |       ||       ||   |___ |   | | |_|   ||   |   |   |  |_     _|
|     |_ |   |___ |   |  | |  |   |       |     | |   _   ||       ||   | |       ||   |   |   |    |   |  
|_______||_______||___|  |_|  |___|        |___|  |__| |__||_______||___| |______| |___|   |___|    |___|  
```

Here’s your improved **`README.md`** with a clean and clickable **Table of Contents** — ideal for GitHub, GitLab, or documentation use.
It’s structured, production-ready, and still general enough to reuse as a template for any Nginx-based project.



# 🧩 NGINX Docker Template

This repository provides a **ready-to-use Nginx Docker setup** designed for flexibility and production-like behavior.  
It includes support for:
- **SSL/TLS (HTTPS)** with mounted certificates  
- **Non-SNI fallback** (for embedded or legacy clients)  
- **Access and error logs** stored on the host  
- **Modular configuration** (`nginx.conf` + site configs in `conf.d/`)  
- **Reusability as a template** for future Nginx deployments



## 📑 Table of Contents

1. [Directory Structure](#-directory-structure)
2. [Quick Start](#-quick-start)
3. [Configuration Details](#-configuration-details)
   - [docker-compose.yml](#-docker-composeyml)
   - [nginx.conf](#-nginxconf)
   - [Example Site Config (`tms.conf`)](#-example-confdtmsconf)
4. [Verifying SSL and SNI Behavior](#-verifying-ssl-and-sni-behavior)
5. [Maintenance Commands](#-maintenance-commands)
6. [Use as Template](#-use-as-template)
7. [License](#-license)



## 📁 Directory Structure

```

nginx-ssl/
├── docker-compose.yml        # Compose file to run Nginx container
├── nginx.conf                 # Main global Nginx configuration
├── conf.d/                    # Folder for per-site configs
│   └── tms.conf               # Example reverse proxy config
├── certs/                     # Public certificates (fullchain.pem)
├── private/                   # Private key (privkey.pem)
└── logs/                      # Mounted logs (access/error)

````

You can duplicate this repository and only modify:
- `conf.d/*.conf` → For each domain or reverse proxy target  
- `certs/` & `private/` → With your own SSL certs  
- `ports` → In `docker-compose.yml` if needed (e.g. `808:80`, `4043:443`)



## 🚀 Quick Start

1. **Clone this repository**

```bash
   git clone https://github.com/<yourusername>/nginx-template.git
   cd nginx-template
````

2. **Add your SSL certificates**

   ```
   certs/fullchain.pem
   private/privkey.pem
   ```

3. **Adjust site config**

   Edit `conf.d/example.conf` or create a new one for your domain.

4. **Run Nginx**

   ```bash
   docker compose up -d
   ```

5. **Check logs**

   ```bash
   docker compose logs -f nginx
   # or directly:
   tail -f logs/*.log
   ```



## ⚙️ Configuration Details

### 🔸 docker-compose.yml

```yaml
version: '3.7'

services:
  nginx:
    image: nginx:latest
    container_name: nginx-ssl
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/ssl/certs:ro
      - ./private:/etc/ssl/private:ro
      - ./logs:/var/log/nginx
    networks:
      - ssl_network

networks:
  ssl_network:
    driver: bridge
```

* Uses the official **nginx:latest** image
* Exposes HTTP/HTTPS ports
* Mounts configs, certs, and logs
* Restarts automatically unless stopped manually



### 🔸 nginx.conf

```nginx
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile        on;
    keepalive_timeout  65;
    include /etc/nginx/conf.d/*.conf;
}
```

This is the **global configuration**.
You can add more directives or performance tweaks here, but site configs stay under `conf.d/`.



### 🔸 Example `conf.d/tms.conf`

This example supports **both SNI and non-SNI** clients and proxies traffic to an internal HTTPS backend.

```nginx
map $scheme $hsts_header {
    https   "max-age=63072000; preload";
}

# Default (non-SNI) SSL server
server {
    listen 443 default_server ssl;
    listen [::]:443 default_server ssl;

    ssl_certificate     /etc/ssl/certs/fullchain.pem;
    ssl_certificate_key /etc/ssl/private/privkey.pem;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    return 301 https://tms.stage.eniac-tech.com$request_uri;
}

# Main HTTPS Virtual Host
server {
    listen 80;
    listen [::]:80;
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name tms.stage.eniac-tech.com;
    http2 off;

    ssl_certificate     /etc/ssl/certs/fullchain.pem;
    ssl_certificate_key /etc/ssl/private/privkey.pem;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    access_log /var/log/nginx/tms_access.log;
    error_log  /var/log/nginx/tms_error.log;

    location / {
        proxy_pass https://192.168.33.141:443;
        proxy_ssl_server_name off;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```



## 🔍 Verifying SSL and SNI Behavior

### ✅ Test with SNI

```bash
openssl s_client -connect tms.stage.eniac-tech.com:443 -servername tms.stage.eniac-tech.com
```

### ✅ Test without SNI

```bash
openssl s_client -connect tms.stage.eniac-tech.com:443
```

If both return a valid certificate, your **non-SNI fallback** is active and functional.



## 🧹 Maintenance Commands

Reload configuration:

```bash
docker exec -it nginx-ssl nginx -t
docker exec -it nginx-ssl nginx -s reload
```

Stop and remove:

```bash
docker compose down
```

Rebuild:

```bash
docker compose up -d --force-recreate
```


# acknowledgment
## Contributors

APA 🖖🏻

```
  aaaaaaaaaaaaa  ppppp   ppppppppp     aaaaaaaaaaaaa   
  a::::::::::::a p::::ppp:::::::::p    a::::::::::::a  
  aaaaaaaaa:::::ap:::::::::::::::::p   aaaaaaaaa:::::a 
           a::::app::::::ppppp::::::p           a::::a 
    aaaaaaa:::::a p:::::p     p:::::p    aaaaaaa:::::a 
  aa::::::::::::a p:::::p     p:::::p  aa::::::::::::a 
 a::::aaaa::::::a p:::::p     p:::::p a::::aaaa::::::a 
a::::a    a:::::a p:::::p    p::::::pa::::a    a:::::a 
a::::a    a:::::a p:::::ppppp:::::::pa::::a    a:::::a 
a:::::aaaa::::::a p::::::::::::::::p a:::::aaaa::::::a 
 a::::::::::aa:::ap::::::::::::::pp   a::::::::::aa:::a
  aaaaaaaaaa  aaaap::::::pppppppp      aaaaaaaaaa  aaaa
                  p:::::p                              
                  p:::::p                              
                 p:::::::p                             
                 p:::::::p                             
                 p:::::::p                             
                 ppppppppp                             
```