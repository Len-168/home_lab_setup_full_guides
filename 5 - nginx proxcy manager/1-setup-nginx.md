# Install Nginx Proxy Manager with Docker Compose

Based on the official documentation: [nginxproxymanager.com](https://nginxproxymanager.com/).

Nginx Proxy Manager (NPM) is a web interface for Nginx. It lets you forward domains to your apps and get free SSL certificates from Let's Encrypt, without editing Nginx config files by hand.

## Before you start

- Docker and Docker Compose are installed (see [4 - install docker](../4%20-%20install%20docker/docker.md)).
- Ports **80**, **443** and **81** are not used by anything else on the server.

## Step 1: Create a folder for Nginx Proxy Manager

This folder will hold the Compose file and all of NPM's data:

```bash
mkdir -p ~/nginx-proxy-manager
cd ~/nginx-proxy-manager
```

## Step 2: Create the `docker-compose.yml` file

Use vim to create the file:

```bash
vim docker-compose.yml
```

Press `i` to switch to insert mode, then paste in the following:

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:2.16.0'
    restart: unless-stopped
    ports:
      # Format is <host-port>:<container-port>
      - '80:80'   # Public HTTP port
      - '443:443' # Public HTTPS port
      - '81:81'   # Admin web port
      # Add any other stream ports you want to expose, for example:
      # - '21:21' # FTP
    environment:
      TZ: "Australia/Brisbane"
      # Postgres connection settings
      DB_POSTGRES_HOST: 'db'
      DB_POSTGRES_PORT: '5432'
      DB_POSTGRES_USER: 'npm'
      DB_POSTGRES_PASSWORD: 'npmpass'
      DB_POSTGRES_NAME: 'npm'
      # Uncomment this if IPv6 is not enabled on your host
      # DISABLE_IPV6: 'true'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    depends_on:
      - db

  db:
    image: postgres:17
    restart: unless-stopped
    environment:
      POSTGRES_USER: 'npm'
      POSTGRES_PASSWORD: 'npmpass'
      POSTGRES_DB: 'npm'
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
```

> **Important:** Change `npmpass` to a strong password. It must be the same in both places: `DB_POSTGRES_PASSWORD` (under `app`) and `POSTGRES_PASSWORD` (under `db`). Also set `TZ` to your own [time zone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

Save and exit vim:

1. Press `Esc` to leave insert mode.
2. Type `:wq` and press `Enter` to save the file and exit.

> **Tip:** If the pasted text gets extra indentation, type `:set paste` before pressing `i`, then paste again.

## Step 3: Start the containers

```bash
docker compose up -d
```

The `-d` flag runs the containers in the background. Check that both are running:

```bash
docker compose ps
```

Both `app` and `db` should show a status of `Up`. If something is wrong, check the logs:

```bash
docker compose logs -f app
```

Press `Ctrl + C` to stop following the logs.

## Step 4: Open the admin panel

In your browser, go to:

```
http://<your-server-ip>:81
```

On first visit you will be asked to create your admin account. Use a strong password.

## Useful commands

Run these from the `~/nginx-proxy-manager` folder:

| Task | Command |
| --- | --- |
| Stop the containers | `docker compose down` |
| Restart the containers | `docker compose restart` |
| View logs | `docker compose logs -f` |
| Update to a new version | Change the image tag in `docker-compose.yml`, then run `docker compose pull && docker compose up -d` |

## More information

Full documentation: [https://nginxproxymanager.com/](https://nginxproxymanager.com/)
