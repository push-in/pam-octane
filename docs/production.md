# PAM Octane in production

Moved out of the README; nothing was removed.

## Production

The package owns the Octane bridge; the PAM binary owns process supervision.
For a supervised production cluster, run Artisan as the worker entrypoint:

```bash
pam start artisan \
  --workers 8 \
  --max-requests 100000 \
  --admin-address 127.0.0.1:3010 \
  -- pam:octane --host=0.0.0.0 --port=8000
```

PAM then provides crash recovery, health checks, worker recycling and
generational `SIGHUP` reloads.

Octane's own command runs the same server in the foreground, which suits
systemd, Supervisor and container entrypoints:

```bash
OCTANE_SERVER=pam PAM_PREFORK_BOOT=app php artisan octane:start \
  --host=0.0.0.0 --port=8000 --workers=8 --max-requests=100000
```

`php artisan octane:reload` (or `kill -HUP` on the `pam start` process) reloads
every worker without dropping a request, and `php artisan octane:stop` drains
and stops it. `PAM_PREFORK_BOOT=app` boots Laravel once and forks every worker
from the booted application. Queue workers, Horizon and the scheduler remain
separate supervised processes.

For systemd, copy the hardened
[`pam-octane.service`](https://github.com/push-in/pam/blob/main/packaging/pam-octane.service)
unit and place deployment-specific values in
`/etc/pam/pam-octane.env`. The unit binds application HTTP to loopback and keeps
the control plane on `127.0.0.1:3010` by default.

Workloads with different latency or memory profiles can run in isolated pools:

```bash
pam octane:start \
  --ingress-address=0.0.0.0:8000 \
  --pool=api=8@/api,/graphql \
  --pool=web=4@* \
  -- --host=0.0.0.0 --port=8000
```

The Rust ingress streams HTTP and WebSocket traffic, uses segment-aware longest
prefix routing, and keeps each pool's Laravel container, PHP heap, OPcache and
restart boundary independent. Exactly one `*` fallback is required. Terminate
TLS/HTTP3 at the edge when pools are enabled; internal traffic is loopback-only.

## Troubleshooting

Run these commands from the Laravel project root before opening a report:

```bash
pam --version
pam doctor .
pam artisan about
pam octane:status            # or: php artisan octane:status --server=pam
```

- If the package reports that the PAM runtime is missing, replace `php artisan`
  with `pam artisan` or use `pam octane:start`.
- If a worker grows after repeated requests, reproduce with one worker, inspect
  request-scoped singletons and run the soak contract before lowering
  `--max-requests`.
- If requests stall, identify blocking PDO, filesystem, HTTP or extension calls;
  use PAM's cooperative clients, an isolated process pool or additional workers.
- If a reverse proxy cannot connect, keep the PAM listener private but bind it to
  the interface reachable by that proxy. Never expose the admin listener publicly.

Sanitize diagnostics: remove credentials, cookies, authorization headers,
private URLs and application data.

## Quality

```bash
tests/tools/fetch-pam-runtime.sh   # PAM's PHP sources, for PHPStan outside PAM
composer analyse               # PHPStan, level 9
pam test                       # the suite on PAM's runtime (php-cli skips PAM-only tests)
tests/tools/octane-suite.sh        # laravel/octane's own suite with this package
docker run --rm -v "$PWD:/package:ro" php:8.5-cli bash /package/tests/tools/e2e-smoke.sh 13
```

The suite checks request marshalling, output buffers, repeated headers, cookies,
streaming, `HEAD`, package discovery and invalid context handling. Integration
tests in the PAM repository cover the native transport and Laravel runtime.
