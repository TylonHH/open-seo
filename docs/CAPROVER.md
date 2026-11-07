# CapRover

CapRover runs Swarm services, not Compose files, so the upstream standalone
stack (`deploy/standalone/compose.yaml`) is deployed as several CapRover apps on
the `captain-overlay-network`. Service discovery uses CapRover's
`srv-captain--<app>` names instead of Compose service names.

## Images

`.github/workflows/publish-container.yml` publishes two linux/amd64 images on
every push to `main` and on `v*` tags, tagged `latest`, `sha-<commit>` and the tag:

- `ghcr.io/tylonhh/open-seo-standalone` from `deploy/standalone/Dockerfile`
  (current celld runtime; use this)
- `ghcr.io/tylonhh/open-seo` from `deploy/docker/Dockerfile`
  (legacy Wrangler/Miniflare single container; kept only for rollback)

Pin deployments to the digest printed in the workflow summary.

## Fork adaptations (kept minimal)

- `deploy/standalone/init.sh`: `OPENSEO_RUNTIME_HOST` (default `runtime`)
  rewrites the nginx upstream when writing `/web-config/default.conf`.
- `scripts/celld/runtime/configure.mjs`: `CRAWL_GATEWAY_URL` env overrides the
  default `http://crawl-gateway:8090`.

Everything else is upstream as-is.

## Apps

| CapRover app | Image | Command | Volumes |
| --- | --- | --- | --- |
| `open-seo-storage` | `rustfs/rustfs:1.0.3@sha256:9ca955...` | default | `open-seo-storage:/data` |
| `open-seo-runtime` | open-seo-standalone | default (`run.sh`) | `open-seo-runtime:/data`, `open-seo-config:/config` |
| `open-seo-crawl-gateway` | open-seo-standalone | `node /opt/standalone/crawl-gateway.mjs` | `open-seo-config:/config` |
| `open-seo` (public URL, HTTPS, Basic auth) | `nginx:1.28-alpine` | default | `open-seo-web-config:/etc/nginx/conf.d` |

Environment for `open-seo-runtime` follows `compose.yaml` with
`S3_ENDPOINT=http://srv-captain--open-seo-storage:9000`.

## Init

`init.sh` must reach the runtime's internal listener `127.0.0.1:8081`
(`network_mode: service:runtime` in Compose). On CapRover run it once as a
one-shot container that joins the runtime task's network namespace:

```sh
docker run --rm --network container:$(docker ps -q -f name=srv-captain--open-seo-runtime) \
  -v open-seo-config:/config -v open-seo-web-config:/web-config \
  -e OPENSEO_RUNTIME_HOST=srv-captain--open-seo-runtime \
  -e CRAWL_GATEWAY_URL=http://srv-captain--open-seo-crawl-gateway:8090 \
  --env-file /root/open-seo/init.env \
  ghcr.io/tylonhh/open-seo-standalone@sha256:... sh /opt/standalone/init.sh
```

`init.env` holds the `x-storage` variables, `AUTH_MODE`, `BETTER_AUTH_URL` and
optional provider keys (root-only file). Re-run after every image update
(deploy + migrations). The legacy import (`scripts/celld/migrate-legacy.mjs`)
runs the same way from the `build` stage image with the old volume mounted
read-only at `/legacy`.