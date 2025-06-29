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

A simple Dockerized Nginx setup to verify SSL/TLS certificate validity (including OCSP stapling) over HTTP/2. Perfect for testing your certificate chains, private keys, and server configuration.

---

## Table of Contents

- [Prerequisites](#prerequisites)  
- [Directory Structure](#directory-structure)  
- [Configuration](#configuration)  
  - [docker-compose.yml](#docker-composeyml)  
  - [nginx.conf](#nginxconf)  
- [Setup & Usage](#setup--usage)  
- [Testing Certificate Validity](#testing-certificate-validity)  
- [Enabling OCSP Stapling](#enabling-ocsp-stapling)  
- [Troubleshooting](#troubleshooting)  
- [License](#license)  

---

## Prerequisites

- Docker Engine (v20+)  
- Docker Compose (v1.27+)  
- A valid SSL/TLS certificate bundle (`full_chain2.pem`)  
- Corresponding private key (`<your>.key`)  
- Optional: OCSP responder reachable from your host  

---

## Directory Structure

```bash
check-validity-of-cert/
├── certs/
│   └── full_chain2.pem        # Your certificate chain (end-entity + intermediates)
├── private/
│   └── <your>.key             # Your private key
├── nginx.conf                 # Nginx server configuration
├── docker-compose.yml         # Docker Compose service definition
└── README.md                  # This file
````

---

## Configuration

### `docker-compose.yml`

```yaml
version: '3.7'

services:
  nginx:
    image: nginx:latest
    container_name: nginx-ssl
    ports:
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/ssl/certs
      - ./private:/etc/ssl/private
    networks:
      - ssl_network
    restart: unless-stopped

networks:
  ssl_network:
    driver: bridge
```

### `nginx.conf`

```nginx
events {
    worker_connections 1024;
}

http {
    server {
        listen 443 ssl http2;

        # SSL/TLS protocols and ciphers
        ssl_protocols       TLSv1.3;
        ssl_ciphers         ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES128-GCM-SHA256;
        ssl_prefer_server_ciphers off;

        # Certificate files (mount point inside container)
        ssl_certificate     /etc/ssl/certs/full_chain2.pem;
        ssl_certificate_key /etc/ssl/private/<your>.key;

        # OCSP Stapling (optional but recommended)
        ssl_stapling        on;
        ssl_stapling_verify on;
        resolver            1.1.1.1 valid=300s;   # Cloudflare DNS
        resolver_timeout    5s;

        # Your hostname
        server_name         <your.domain>;

        # Simple test page
        location / {
            root   /usr/share/nginx/html;
            index  index.html;
        }
    }
}
```

> **⚠️ Remember:**
>
> * Replace `<your>.key` with the filename of your private key.
> * Replace `<your.domain>` (and in the cert files, if needed) with your actual server name.

---

## Setup & Usage

1. **Clone or download** this repository:

   ```bash
   git clone https://github.com/your-org/check-validity-of-cert.git
   cd check-validity-of-cert
   ```

2. **Place your certificate bundle** in `./certs/full_chain2.pem`.

3. **Place your private key** in `./private/<your>.key`.

4. **Edit** `nginx.conf`:

   * Update `ssl_certificate_key` to match your key filename.
   * Set `server_name` to your domain or IP.

5. **Start the service**:

   ```bash
   docker-compose up -d
   ```

6. **Verify** that the container is running:

   ```bash
   docker-compose ps
   ```

---

## Testing Certificate Validity

### Using `openssl`:

```bash
openssl s_client -connect localhost:443 -status \
  -servername <your.domain>
```

* Look for `OCSP response:` and certificate chain details.
* Ensure there are **no errors** in the handshake.

### Using `curl` (with HTTP/2):

```bash
curl -I --http2 https://localhost --resolve localhost:443:127.0.0.1
```

* Check for `HTTP/2 200` response.
* Verify the certificate details in the output.

---

## Enabling OCSP Stapling

OCSP stapling allows Nginx to fetch and cache the OCSP response from the CA, improving TLS handshake performance and reliability.

1. Ensure `ssl_stapling on; ssl_stapling_verify on;` are set in `nginx.conf`.
2. Configure a DNS resolver for OCSP:

   ```nginx
   resolver 1.1.1.1 valid=300s;
   resolver_timeout 5s;
   ```
3. Reload Nginx:

   ```bash
   docker-compose exec nginx nginx -s reload
   ```

Test stapling specifically:

```bash
openssl s_client -connect localhost:443 -status
```

* You should see a non-`no response sent` OCSP response block.

---

## Troubleshooting

* **Port conflicts**: Make sure nothing else is binding to host port 443.
* **Invalid cert/key paths**: Check volume mounts and file permissions.
* **OCSP timeouts**: Adjust `resolver_timeout` or verify network access.
* **Cipher issues**: Modify `ssl_ciphers` if your platform requires different suites.

---
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