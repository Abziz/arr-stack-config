# arr-stack

A ready-to-run Docker Compose setup for a home media server:

| Service | What it does | Default URL |
|---|---|---|
| Prowlarr | Manages torrent indexers for Sonarr/Radarr | http://localhost:9696 |
| Sonarr | Finds and organizes TV shows | http://localhost:8989 |
| Radarr | Finds and organizes movies | http://localhost:7878 |
| Bazarr | Downloads subtitles | http://localhost:6767 |
| FlareSolverr | Gets Prowlarr past Cloudflare protection | http://localhost:8191 |
| qBittorrent | Downloads torrents | http://localhost:8080 |
| Jellyfin | Streams your media | http://localhost:8096 |

Replace `localhost` with the machine's IP when connecting from another device.

## 1. Requirements

- Linux machine with [Docker Engine](https://docs.docker.com/engine/install/) and the Docker Compose plugin
- Check: `docker compose version`

## 2. Get the files

```bash
git clone https://github.com/Abziz/arr-stack-config.git arr-stack
cd arr-stack
cp .env.example .env
```

## 3. Edit `.env`

Every setting has a default, so the stack runs even without a `.env` file. Open `.env` and change what you need:

| Variable | Default | What to set |
|---|---|---|
| `PUID` / `PGID` | `1000` | Run `id` and use your `uid` and `gid` |
| `TZ` | `Etc/UTC` | Your timezone, e.g. `Asia/Jerusalem`, `Europe/London` |
| `CONFIG_ROOT` | `./configs` | Where app settings/databases are kept. The default (inside this folder) is fine |
| `MEDIA_ROOT` | `./media` | Your media drive/folder, e.g. `/mnt/media-ssd/media` |
| `*_PORT` | app defaults (see table above) | Only change if a port is already in use on this machine |

## 4. Create the media folders

Use the same path you set as `MEDIA_ROOT`:

```bash
MEDIA_ROOT=./media   # <- your path, e.g. /mnt/media-ssd/media
mkdir -p "$MEDIA_ROOT"/{movies,tv-shows,music,books,downloads}
sudo chown -R "$(id -u)":"$(id -g)" "$MEDIA_ROOT"
```

The layout should look like this. Keeping `downloads` on the same drive as the library lets Sonarr/Radarr move files instantly (hardlinks) instead of copying them.

```
MEDIA_ROOT/
├── downloads/   # qBittorrent saves here
├── movies/      # Radarr library
├── tv-shows/    # Sonarr library
├── music/
└── books/
```

## 5. Start the stack

```bash
docker compose up -d
docker compose ps
```

All containers should show `running`.

## 6. First-time setup

Do these in order. Inside the stack, containers talk to each other by **service name** (e.g. `qbittorrent`, `sonarr`), not `localhost`.

### qBittorrent (http://localhost:8080)

1. Get the temporary password: `docker logs arr-suite-qbittorrent 2>&1 | grep -i password`
2. Log in as `admin` with that password.
3. **Tools → Options → Web UI**: set your own username and password.
4. **Tools → Options → Downloads**: set *Default Save Path* to `/media/downloads`.

### Sonarr (http://localhost:8989) and Radarr (http://localhost:7878)

For each one:

1. On first open, create a login (Authentication: *Forms*).
2. **Settings → Media Management → Root Folders**: add
   - Sonarr: `/media/tv-shows`
   - Radarr: `/media/movies`
3. **Settings → Download Clients → + → qBittorrent**:
   - Host: `qbittorrent`, Port: `8080`
   - Username / password: the ones you set in qBittorrent
   - Category: `tv-sonarr` (Sonarr) or `radarr` (Radarr)
   - Click **Test**, then **Save**.
4. **Settings → General**: copy the **API Key** (needed for Prowlarr and Bazarr).

### Prowlarr (http://localhost:9696)

1. Create a login on first open.
2. **Settings → Indexers → + → FlareSolverr**: Host `http://flaresolverr:8191`, Tag `flaresolverr`. Save.
3. **Settings → Apps → +**:
   - Sonarr: Prowlarr Server `http://prowlarr:9696`, Sonarr Server `http://sonarr:8989`, API key from Sonarr.
   - Radarr: Prowlarr Server `http://prowlarr:9696`, Radarr Server `http://radarr:7878`, API key from Radarr.
4. **Indexers → Add Indexer**: add the sites you use. For sites behind Cloudflare, add the `flaresolverr` tag.
   Prowlarr pushes indexers to Sonarr/Radarr automatically.

### Bazarr (http://localhost:6767)

1. **Settings → General → Security**: set a username/password.
2. **Settings → Sonarr**: enable, Address `sonarr`, Port `8989`, API key from Sonarr. Test, Save.
3. **Settings → Radarr**: enable, Address `radarr`, Port `7878`, API key from Radarr. Test, Save.
4. **Settings → Languages**: create a language profile, and **Settings → Providers**: add subtitle providers.

### Jellyfin (http://localhost:8096)

1. Follow the setup wizard and create your admin user.
2. Add libraries (Jellyfin sees the media folder as `/data`):
   - Movies → `/data/movies`
   - Shows → `/data/tv-shows`
   - Music → `/data/music` (optional)

You're done. Add a show in Sonarr or a movie in Radarr to test the whole flow.

## Everyday commands

```bash
docker compose ps                         # status
docker compose logs -f sonarr             # follow logs of one service
docker compose restart radarr             # restart one service
docker compose down                       # stop everything
docker compose pull && docker compose up -d && docker image prune -f   # update
```

## Backups

Everything you configured lives in `CONFIG_ROOT` (default `./configs`). Stop the stack and copy that folder to back it up or move it to another machine:

```bash
docker compose down
tar czf arr-configs-backup.tar.gz configs
docker compose up -d
```

## Optional: route qBittorrent through a VPN

To make all torrent traffic go through a VPN (with a kill switch), add a [gluetun](https://github.com/qdm12/gluetun) service and attach qBittorrent to it. Example:

```yaml
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: arr-suite-gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - VPN_SERVICE_PROVIDER=your-provider   # see gluetun wiki for your provider
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=...
      - TZ=${TZ}
    ports:
      - ${QBITTORRENT_PORT}:8080              # qBittorrent web ui now goes through gluetun
      - ${QBITTORRENT_TORRENT_PORT}:6881
      - ${QBITTORRENT_TORRENT_PORT}:6881/udp
    restart: unless-stopped
```

Then in the `qbittorrent` service: remove its `ports:` section and add `network_mode: "service:gluetun"`. Sonarr/Radarr must then use host `gluetun` (port `8080`) for the download client.

## Optional: use with an existing reverse proxy

If you run a reverse proxy (nginx, Traefik, ...) on its own Docker network, add this to the end of `docker-compose.yml` so the stack joins it:

```yaml
networks:
  default:
    name: your_proxy_network
    external: true
```
