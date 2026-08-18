# Immich CORS Proxy

A small nginx sidecar that adds CORS support in front of an existing Immich
server.

The proxy is intentionally configured for **public CORS**:

```text
Access-Control-Allow-Origin: *
```

It is useful when a browser application such as WallPanel needs to call the
Immich API from a different origin.

## How to use this with the official/personal Immich Docker Compose

This repository is **not a replacement for your Immich Compose file**.
Keep your existing Immich deployment unchanged and add the proxy service to
the same Compose project.

### 1. Copy the nginx configuration

Copy this repository's directory:

```text
nginx/immich.conf
```

into the directory containing your existing Immich `docker-compose.yml`, so
your layout becomes:

```text
your-immich-folder/
├── docker-compose.yml
└── nginx/
    └── immich.conf
```

### 2. Add the proxy service to your existing Compose file

Under `services:` in your existing Immich `docker-compose.yml`, add:

```yaml
  immich-cors-proxy:
    image: nginx:alpine
    container_name: immich-cors-proxy
    restart: unless-stopped
    ports:
      - "2284:80"
    volumes:
      - ./nginx/immich.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - immich-server
```

Do **not** create a second `immich-server` service. The proxy expects your
existing Immich service to be named:

```text
immich-server
```

This is the normal service name in the Immich Compose setup. If your existing
Compose file uses a different service name, change this line in
`nginx/immich.conf`:

```nginx
proxy_pass http://immich-server:2283;
```

to match your actual Immich server service name.

### 3. Start the proxy

From the directory containing your existing Immich Compose file:

```bash
docker compose up -d immich-cors-proxy
```

To force recreation after changing the nginx config:

```bash
docker compose up -d --force-recreate immich-cors-proxy
```

### 4. Point WallPanel at the proxy

If Immich normally runs on:

```text
http://YOUR-NAS-IP:2283
```

use:

```text
http://YOUR-NAS-IP:2284
```

for browser clients that need CORS.

The proxy forwards requests internally to:

```text
http://immich-server:2283
```

and adds the required CORS headers.

## CORS headers

The proxy allows:

- Any origin
- `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`
- `Authorization`
- `X-API-Key`
- `Content-Type`
- `Range`
- Other common browser request headers

The proxy also exposes media-related response headers such as `Content-Range`
and `Accept-Ranges`.

## Test the preflight request

From another machine on your network:

```bash
curl -i -X OPTIONS \
  http://YOUR-NAS-IP:2284/api/server/version \
  -H "Origin: http://example.com" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: x-api-key"
```

You should get:

```text
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: *
```

## Security note

This configuration intentionally allows requests from **any web origin**.
That is appropriate for a CORS gateway on a trusted home network, but it does
not make Immich public on the Internet safely.

Do not expose port `2284` directly to the Internet unless you also have
appropriate network restrictions, authentication, and TLS in front of it.

The proxy does not bypass Immich authentication. Clients still need whatever
Immich authentication/API key is required by the endpoint.
