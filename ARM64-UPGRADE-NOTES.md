# ARM64 runtime support for docker-openwisp

This branch adds native `linux/arm64` support to the upstream stack. It does not
change application behaviour, image contracts, service names, volumes, ports or
environment variables: upstream stays exactly as published, and everything below
is additive.

## Why a fork is needed

Four images referenced by the upstream compose file publish no `linux/arm64`
manifest. On an aarch64 host `docker pull --platform linux/arm64` still exits 0
for them: the engine silently resolves the amd64 variant, and the container then
runs under `qemu-user`, where it crashes with
`x86_64-binfmt-P: QEMU internal SIGSEGV {code=MAPERR, addr=0x20}` or exit 139.

| Upstream reference | Used here | Reason |
|---|---|---|
| `lisenet/openvpn:2.7.7` | `images/openwisp_openvpn/Dockerfile.arm64` | no arm64 manifest |
| `postgis/postgis:15-3.4-alpine` | `images/postgis/Dockerfile.arm64` | no arm64 manifest |
| `redis:alpine` | `redis:7-alpine` | alpine tag has no arm64 |
| `influxdb:1.8-alpine` | `influxdb:1.8` | alpine tag has no arm64 |

The two Dockerfiles reuse the same runtime packages, environment variables and
entrypoints as the upstream images, and only replace the base image:

- `openvpn`: `alpine:3.22` plus `openvpn iptables bash easy-rsa openssl`, keeping
  the upstream `ENV OPENVPN/EASYRSA/...` block and the rest of the file verbatim.
  Consequence: the packaged OpenVPN version follows Alpine (2.6.x) instead of the
  vendor image (2.7.x).
- `postgis`: `postgres:15-bookworm` plus the Debian PostGIS packages, keeping the
  stock `postgres` entrypoint so no `entrypoint:`/`command:` override is needed.

`docker-compose.arm64.yml` is the only other addition: it pins
`platform: linux/arm64` for the locally built services, points `openvpn` and
`postgres` at the two local Dockerfiles, and swaps the two alpine tags above.

## Build order (this matters)

Three services (`dashboard`, `api`, `websocket`) are declared as
`FROM openwisp/openwisp-base:${OPENWISP_VERSION}`. If you let
`docker compose build` handle them, it resolves that tag against Docker Hub,
pulls the published **amd64** base image and overwrites the local arm64 one.
Build them explicitly against a private tag instead:

```bash
# 1) the shared base, tagged privately so the registry cannot shadow it
docker build --no-cache -t openwisp/openwisp-base:edge  -f images/openwisp_base/Dockerfile images/
docker tag openwisp/openwisp-base:edge openwisp/openwisp-base:local-arm64

# 2) the three python services, pointing at the private tag
for s in dashboard api websocket; do
  docker build --no-cache --build-arg OPENWISP_VERSION=local-arm64 \
    -f images/openwisp_$s/Dockerfile -t openwisp/openwisp-$s:edge images/
done

# 3) postgis and the remaining services through compose
docker build --no-cache -f images/postgis/Dockerfile.arm64 -t openwisp/postgis:arm64-local images/
docker compose -f docker-compose.yml -f docker-compose.arm64.yml build --no-cache \
  nginx freeradius postfix openvpn
```

Select the native builder first. A `docker-container` builder produces amd64
content even when `--platform linux/arm64` is passed; on an aarch64 host the
`docker` driver builds natively and needs no `--platform`:

```bash
docker buildx use default
docker buildx ls | grep '\*'
```

## Starting the stack

```bash
docker compose -f docker-compose.yml -f docker-compose.arm64.yml up -d
```

Two startup details worth knowing:

- Recreating `postgres` changes its container IP; `nginx` caches upstream
  addresses at start, so a `502` on every route means the proxy must be restarted
  (or recreated) after the backends settle.
- Editing `.env` requires `up -d --force-recreate nginx`; `docker compose restart`
  keeps the old container environment and does not re-render the vhost config.

## Verification

Image metadata is not evidence. Run every image and require `aarch64`:

```bash
for s in base dashboard api websocket nginx freeradius postfix openvpn; do
  printf '%-12s ' $s
  docker run --rm --entrypoint /bin/sh openwisp/openwisp-$s:edge -c 'uname -m'
done
docker run --rm --entrypoint /bin/sh openwisp/postgis:arm64-local -c 'uname -m'
```

Then check the runtime, not just the container state:

```bash
docker compose -f docker-compose.yml -f docker-compose.arm64.yml ps
curl -sk -H 'Host: <dashboard-domain>' https://localhost/admin/login/ -o /dev/null -w '%{http_code}\n'  # 200
docker exec <dashboard-container> sh -c "uname -m"                                                    # aarch64
```

A stack where every container reports `Up` can still serve 502, and a stack that
serves 200 can still be running emulated layers, so both checks are required.

## Configuration stays upstream

Domain names, credentials and feature flags belong in `.env`, which is runtime
configuration and is not part of this branch. `ALLOWED_HOSTS`, the cookie domains
and the CSRF trusted origins are derived from `DASHBOARD_DOMAIN`/`API_DOMAIN` by
upstream settings; do not reintroduce local settings hooks to work around a host
mismatch. Access the deployment through the configured domain.

## Upgrade checklist

For every future sync with upstream:

1. Re-apply this branch (or rebase it) and re-read
   `images/openwisp_openvpn/Dockerfile` and `docker-compose.yml`; if upstream
   changed either, regenerate the matching `Dockerfile.arm64` from it so the only
   difference stays the base image.
2. Re-run the build order above with `--no-cache`.
3. Run the verification loop; any `x86_64` result means the amd64 base leaked in.
4. Restart the proxy after the backends are recreated.
