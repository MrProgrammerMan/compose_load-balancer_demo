# Docker Compose Load Balancer Demo

A minimal demo that shows how a **load balancer** distributes HTTP requests across multiple back-end web servers, using only **Docker Compose** and **Nginx**. Designed as exam-prep material for Operating Systems / Distributed Systems topics.

---

## Architecture

```
Client (browser / curl)
        │
        ▼  port 8080
┌───────────────────┐
│  balancer (Nginx) │   ← reverse proxy / load balancer
└─────────┬─────────┘
          │  random selection
    ┌─────┴─────┐
    ▼           ▼
┌───────┐   ┌───────┐
│ web1  │   │ web2  │   ← back-end web servers
│(Nginx)│   │(Nginx)│
└───────┘   └───────┘
```

Three Docker containers are started by Docker Compose:

| Service    | Role                   | Exposed port |
|------------|------------------------|--------------|
| `web1`     | Back-end web server #1 | internal only |
| `web2`     | Back-end web server #2 | internal only |
| `balancer` | Nginx load balancer    | **8080 → 80** |

---

## File Structure

```
.
├── compose.yaml          # Docker Compose service definitions
├── nginx/
│   ├── nginx.conf        # Main Nginx config for the balancer
│   └── conf.d/
│       └── default.conf  # Upstream pool + virtual server block
├── web1/
│   └── index.html        # Page served by back-end server 1
└── web2/
    └── index.html        # Page served by back-end server 2
```

---

## How It Works

### 1. Docker Compose (`compose.yaml`)

```yaml
services:
  web1:
    image: nginx:latest
    volumes:
      - ./web1:/usr/share/nginx/html   # serve web1/index.html

  web2:
    image: nginx:latest
    volumes:
      - ./web2:/usr/share/nginx/html   # serve web2/index.html

  balancer:
    image: nginx:latest
    volumes:
      - ./nginx:/etc/nginx             # custom Nginx config
    ports:
      - "8080:80"                      # expose balancer to the host
    depends_on:
      - web1
      - web2
```

- **`web1` / `web2`** are plain Nginx containers whose document roots are replaced with the local `web1/` and `web2/` directories.  
- **`balancer`** uses a custom Nginx config mounted from `./nginx/`. Docker Compose guarantees it starts only after both web servers are ready (`depends_on`).
- Docker's internal DNS lets the balancer reach `web1` and `web2` **by service name** — no IP addresses needed.

### 2. Load-Balancer Config (`nginx/conf.d/default.conf`)

```nginx
upstream backend {
    random;           # load-balancing algorithm: pick a server at random
    server web1;      # resolved by Docker's internal DNS
    server web2;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;   # forward every request to the pool
    }
}
```

- The `upstream` block defines a **pool** of back-end servers named `backend`.
- The **`random`** directive is the load-balancing algorithm — each request is forwarded to a randomly selected server in the pool.
- The `server` block listens on port 80 (mapped to 8080 on the host) and proxies all requests to the pool.

### 3. Main Nginx Config (`nginx/nginx.conf`)

```nginx
events {}
http {
    include /etc/nginx/conf.d/*.conf;   # load default.conf
}
```

Minimal config that simply includes all `.conf` files in `conf.d/`.

---

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) (with the Compose plugin, or standalone `docker-compose`)

---

## Running the Demo

```bash
# Start all containers in the background
docker compose up -d

# Send several requests — you should see responses from web1 and web2 alternating
curl http://localhost:8080
curl http://localhost:8080
curl http://localhost:8080

# Stop and remove all containers
docker compose down
```

Each `curl` response will contain either `Hello from WEB1` or `Hello from WEB2`, demonstrating that the load balancer is distributing requests.

---

## Key Concepts (OS / Networking Exam Notes)

| Concept | How it appears in this demo |
|---------|-----------------------------|
| **Process isolation** | Each service runs in its own container (isolated process namespace, filesystem, network stack) |
| **Inter-process communication (IPC)** | Containers communicate over a virtual Docker bridge network using TCP/IP |
| **Load balancing** | The balancer spreads client requests across multiple server processes to prevent any single process from being overwhelmed |
| **Reverse proxy** | The balancer sits in front of the back-ends; clients only ever talk to the balancer |
| **Service discovery** | Docker's embedded DNS resolves `web1` / `web2` to container IPs automatically |
| **Port mapping** | Host port 8080 is forwarded to container port 80 — a form of network address translation (NAT) |
| **`depends_on`** | Expresses a startup dependency — the OS / container runtime must schedule `balancer` after `web1` and `web2` |

---

## Experimenting Further

- **Change the algorithm** — replace `random` with `least_conn` (least connections) or remove the directive entirely to use round-robin.
- **Add more servers** — duplicate a `web` service in `compose.yaml` and add it to the `upstream` block.
- **Observe traffic** — add `access_log /var/log/nginx/access.log;` to `nginx.conf` and run `docker compose logs -f balancer`.
