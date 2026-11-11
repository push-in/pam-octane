<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset=".github/assets/banner-dark.svg">
  <img src=".github/assets/banner-light.svg" width="880" alt="pam-octane: Laravel Octane lifecycle on PAM's native transport, powered by Rust and Tokio.">
</picture>

**Laravel Octane lifecycle, powered by Rust and Tokio.**<br>
Connect PAM's native HTTP transport to Octane while Octane keeps ownership of Laravel isolation and worker lifecycle.

[![Packagist](https://img.shields.io/packagist/v/pushinbr/pam-octane?label=packagist&color=f53003)](https://packagist.org/packages/pushinbr/pam-octane)
![PHP 8.5](https://img.shields.io/badge/php-8.5-777bb4)
[![License](https://img.shields.io/github/license/push-in/pam-octane?color=1f6feb)](LICENSE)

[Install](#install) · [Commands](#quick-example) · [Features](#features) · [Benchmark](#benchmark) · [Upgrading](#upgrading) · [How it works](#how-it-works) · [Docs](#docs) · [Issues](https://github.com/push-in/pam-octane/issues)

</div>

<a id="install"></a>
## <picture><img src=".github/assets/icons/terminal.svg" width="22" height="22" alt=""></picture> Install

In any Laravel 12 or 13 application, with stock `laravel/octane`:

```sh
composer require laravel/octane pushinbr/pam-octane
php artisan octane:install --server=pam
php artisan octane:start
```

`octane:install --server=pam` finds the `pam` binary (`PATH`, the project root or `~/.local/bin`) or, after confirmation, installs it with PAM's official [install script](https://github.com/push-in/pam/releases/latest/download/install.sh), which verifies the release checksum. It then writes `OCTANE_SERVER=pam` to `.env` and publishes `config/octane.php`, exactly as for Octane's other servers. PAM runs on Linux and macOS; on Windows, run the application in WSL 2 or Docker.

Nothing else changes: Octane stays unmodified, and this package teaches its commands the `pam` server through its service provider.

<a id="quick-example"></a>
## <picture><img src=".github/assets/icons/code.svg" width="22" height="22" alt=""></picture> Commands

```sh
php artisan octane:start --server=pam   # or OCTANE_SERVER=pam in .env / config/octane.php
php artisan octane:reload --server=pam  # zero-downtime: every worker is replaced make-before-break
php artisan octane:status --server=pam
php artisan octane:stop --server=pam
```

`octane:start` takes Octane's usual options: `--host`, `--port`, `--workers`, `--task-workers`, `--max-requests`, `--watch` and `--poll`. It runs a foreground `pam start` cluster, prints each request in `local`, and stops it on `Ctrl+C`, like the Swoole and FrankenPHP servers. Every command also runs as `pam artisan octane:start` and so on.

The PAM-managed commands of earlier releases keep working unchanged. They start a detached application under PAM's supervisor (`pam ps`, `pam logs`, `pam monit`):

```sh
pam octane:start     # pam up artisan ... -- pam:octane
pam octane:status
pam octane:reload
pam octane:stop
```

Arguments of `pam octane:start` before `--` go to the supervisor, arguments after it to the server.

<a id="features"></a>
## <picture><img src=".github/assets/icons/rocket.svg" width="22" height="22" alt=""></picture> Features

Every Octane feature works on PAM, including the ones Octane only offers on Swoole:

- **`Octane::table()`.** Tables from `octane.tables` live in shared memory: every worker sees the same rows, and the rows survive reloads and worker recycling. Values larger than their column throw `ValueTooLargeForColumnException`, like Swoole.
- **The `octane` cache store** (`Cache::store('octane')`, `Cache::interval()`) is shared by every worker when task workers run, or with `PAM_OCTANE_SHARED_CACHE=true`.
- **`Octane::concurrently()` and task dispatch** run on PAM task workers: `octane:start --task-workers=N` (or `auto`, one per CPU). Task workers are opt-in; without them tasks run sequentially, as on RoadRunner and FrankenPHP.
- **`Octane::tick()`** and cache intervals run on the task workers.
- Octane's request sandbox, lifecycle events and listeners, and the real `Laravel\Octane\Worker`.
- Static files from `public/` (Vite builds, `storage:link`) with ETag, `Range` and `octane.static_file_headers`.
- `octane.max_execution_time`: a request past it is answered `504` and its worker replaced, even when it blocks.
- Zero-loss `octane:reload` and `--max-requests` recycling: PAM's master keeps every listening socket.
- Uploads streamed to disk, so a 1 GB upload works with the default `memory_limit`.
- `--watch` (Node and chokidar, like Octane's other servers).
- PAM extras: the post-boot zygote (`PAM_PREFORK_BOOT=app` boots Laravel once and forks every worker from it), the edge page cache served from Rust, route-aware worker pools, and PAM's live metrics (`pam top`).

Tested with laravel/octane 2.19 and 2.24 on Laravel 12 and 13: laravel/octane's own suite with the package installed (0 failures, every request and response cross-checked through the PAM client), Octane's 35 Swoole-only table and cache store tests ported to PAM tables, and an end-to-end run in a fresh application (install, start, a request, three reloads under load with no failed request, stop).

<a id="benchmark"></a>
## <picture><img src=".github/assets/icons/grid.svg" width="22" height="22" alt=""></picture> Benchmark

The same Laravel application on the four Octane servers, median of 5 rounds ([push-in/octane-benchmarks@14e42fb](https://github.com/push-in/octane-benchmarks/tree/14e42fb9a3863cd1aac6827ac728fbd2ff265a34), PAM 2.2.15):

| Requests per second | PAM | FrankenPHP | Swoole | RoadRunner |
| --- | ---: | ---: | ---: | ---: |
| hello | **11,191** | 9,826 | 8,310 | 6,323 |
| json | **10,530** | 9,479 | 8,161 | 6,140 |
| db | **8,854** | 8,260 | 7,003 | 5,205 |
| view | **9,456** | 8,329 | 6,963 | 5,443 |

PAM has the highest median throughput on every endpoint (7 to 14 % above FrankenPHP) and the lowest median p99 on `hello`, `json` and `view`. FrankenPHP is steadier: its p99 stays within about 1.6 ms across rounds, while PAM's varies more between rounds, and on `db` FrankenPHP's p99 (12.77 ms) is lower than PAM's (14.48 ms); Swoole has the lowest `db` p99. The numbers only hold relative to each other on that machine; the repository has the raw output, the method and the scripts to run it on your own hardware.

<a id="upgrading"></a>
## <picture><img src=".github/assets/icons/layers.svg" width="22" height="22" alt=""></picture> Upgrading from 1.3

- No breaking changes: `pam octane:start|status|reload|stop`, `pam artisan pam:octane`, `PamOctane::table()` and the configuration keep working.
- New: `php artisan octane:start --server=pam` and the rest of Octane's commands. `laravel/octane#1188` will not be merged into Octane: this package is the supported way to run Octane on PAM, and you do not need the `push-in/octane` fork. Remove it from `composer.json` (a `repositories` entry pointing at push-in/octane, or `laravel/octane: dev-2.x-pam`) and require stock `laravel/octane ^2.19`.
- `Octane::table()` now works on PAM with stock Octane, so code written for Swoole runs unchanged; `PamOctane::table()` remains available.
- To switch an application from the PAM-managed commands, stop it with `pam octane:stop`, set `OCTANE_SERVER=pam` and start it with `php artisan octane:start` (under systemd, Supervisor or a container entrypoint). Both run the same server code.
- `octane.serve_static_files` and `octane.static_file_headers` are now honoured besides the `pam-octane.*` keys.
- `octane:start --server=pam` requires PAM 2.2.15 or newer (it warns on older versions); `pam self-update` updates PAM.

<a id="how-it-works"></a>
## <picture><img src=".github/assets/icons/layers.svg" width="22" height="22" alt=""></picture> How it works

PAM Octane connects PAM's native HTTP transport directly to Laravel Octane's worker. Octane continues to own application isolation, lifecycle events and request cleanup; PAM owns PHP Embed, networking, streaming, supervision and Tokio-powered infrastructure.

```text
HTTP → PAM / Tokio → PamClient → Octane Worker → Laravel
HTTP ← PAM / Tokio ← PamClient ← Octane Worker ← Laravel
```

PAM already embeds PHP and provides a persistent native HTTP runtime. This package deliberately does not duplicate Octane's Laravel lifecycle. It adapts PAM's request and response objects to Octane's public `Client` contract and then uses the real `Laravel\Octane\Worker`. That gives an application:

- one warm Laravel application per worker;
- Octane's request sandbox and lifecycle events;
- PAM's bounded streaming and client backpressure;
- correct cookies, uploads, `HEAD`, streamed and binary responses;
- one safe Laravel execution slot per worker;
- normal Laravel package discovery and configuration.

| | |
| --- | --- |
| **Role** | Octane server adapter |
| **Boundary** | Octane remains the lifecycle authority; this is not a Laravel fork |

<a id="configure"></a>
## <picture><img src=".github/assets/icons/terminal.svg" width="22" height="22" alt=""></picture> Configure

To customize limits, publish the small configuration file:

```bash
pam artisan vendor:publish --tag=pam-octane-config
```

Environment variables include:

```dotenv
PAM_HOST=127.0.0.1
PAM_PORT=8000
PAM_REQUEST_TIMEOUT_MS=30000      # default: octane.max_execution_time
PAM_MAX_BODY_BYTES=2097152
PAM_MAX_RESPONSE_BYTES=268435456
```

With PAM 2.2.13 or newer:

- Uploads are streamed to temporary files, like PHP-FPM: a 1 GB upload works with the default `memory_limit`. PHP's `upload_max_filesize`, `post_max_size`, `max_file_uploads` and `upload_tmp_dir` apply, `is_uploaded_file()`/`move()` work, and the files are deleted when the request ends. `PAM_MAX_BODY_BYTES` only bounds bodies held in memory.
- The request timeout follows Octane's `octane.max_execution_time` (`0` disables it; `PAM_REQUEST_TIMEOUT_MS` overrides it). A request past it is answered `504 Gateway Timeout`, even when the handler blocks, and its worker is replaced.
- `octane:reload` and `--max-requests` recycling are make-before-break and drop no requests. With PAM 2.2.15 the PAM master keeps every listening socket and a replacement worker adopts it, so no listener with queued connections is ever closed: no `net.ipv4.tcp_migrate_req` setting is needed.

With PAM 2.2.15 or newer:

- **Post-boot zygote.** On Linux, `pam octane:start` (and `php artisan octane:start` with `PAM_PREFORK_BOOT=app`) boots Laravel once and forks every worker (new, recycled or replacing a crashed one) from the booted application: workers are ready in a few milliseconds and share the booted memory copy-on-write. Database and Redis connections opened while booting are closed before the fork; register `\Pam\Fork::beforeFork()` / `\Pam\Fork::afterFork()` for anything else to reset per process. `PAM_PREFORK_BOOT=engine` boots Laravel in every worker instead.
- **Edge page cache.** `PAM_RESPONSE_CACHE=true` lets PAM serve whole pages from Rust without entering PHP. Mark routes with `Pam\Octane\Http\Middleware\CacheResponse` (alias `pam.cache:ttl,stale,tags...`) and purge with `Pam::purge()`, `Pam::purgeUrl()`, `Pam::purgeAll()` or the `PurgesPageCache` model trait. Requests with `Cookie` or `Authorization` and responses that set cookies are never cached by default.
- **Tables, shared cache, tasks and ticks.** Octane tables (`octane.tables`) live in shared memory (`Octane::table()`, `PamOctane::table()`), `--task-workers=N` runs `Octane::concurrently()`, task dispatch, `Octane::tick()` and cache intervals, and the `octane` cache store is shared by every worker when task workers run.

Details for each: the [package README in the PAM repository](https://github.com/push-in/pam/blob/main/packages/octane/README.md).

The readiness report against Laravel Octane's own test suite, Laravel 12 and 13, a real ecosystem app and a benchmark is in [READINESS.md](https://github.com/push-in/pam/blob/main/packages/octane/READINESS.md).

`maxConcurrentRequests` is intentionally fixed to `1` for Laravel. Framework managers and facades contain process-global mutable state. Scale safely with multiple supervised PAM workers instead of overriding that boundary.

Supervised clusters, route-aware worker pools, systemd, troubleshooting and the quality suite are in [Production](docs/production.md).

<a id="compatibility"></a>
## <picture><img src=".github/assets/icons/grid.svg" width="22" height="22" alt=""></picture> Compatibility

| PHP | Laravel | Octane |
| --- | --- | --- |
| 8.5 (default) | 12.x | 2.19+ |
| 8.5 (default) | 13.x | 2.19+ |
| 8.4 (supported) | 12.x | 2.19+ |
| 8.4 (supported) | 13.x | 2.19+ |

PAM embeds its own PHP 8.5, so the server always runs on 8.5; the host PHP only runs `php artisan`. Octane's commands need the `pcntl` extension there, as for every Octane server.

<a id="runtime-capabilities"></a>
## <picture><img src=".github/assets/icons/rocket.svg" width="22" height="22" alt=""></picture> Runtime capabilities

- generational reload, status, stop and control-plane health;
- route-aware worker pools with crash and memory isolation;
- cooperative Fiber scheduling and HTTP/Redis clients;
- isolated bounded PDO and process pools for blocking work;
- production response cache with tags, authenticated invalidation, collapse and stale-while-revalidate;
- bounded route metrics plus Pulse, Telescope and OpenTelemetry integration.

<a id="support"></a>
## <picture><img src=".github/assets/icons/shield.svg" width="22" height="22" alt=""></picture> Support and repository model

Usage questions belong in PAM GitHub Discussions. Reproducible defects use the main PAM issue tracker, and vulnerabilities must use private GitHub Security Advisories as described in [SECURITY.md](SECURITY.md).

PAM Octane is developed in `packages/octane` alongside the runtime so transport and lifecycle changes can be tested atomically. It remains an independent, optional Composer package: applications that do not use Laravel or Octane do not install it.

<a id="docs"></a>
## <picture><img src=".github/assets/icons/book.svg" width="22" height="22" alt=""></picture> Docs

- [Laravel on PAM](https://push-in.github.io/pam-docs/laravel/overview/)
- [Production](docs/production.md): supervised cluster, worker pools, systemd, troubleshooting, quality
- [Changelog](CHANGELOG.md)
- [Contributing](CONTRIBUTING.md)

<a id="pam-ecosystem"></a>
## <picture><img src=".github/assets/icons/plug.svg" width="22" height="22" alt=""></picture> Part of the PAM ecosystem

<p align="center"><picture>
  <source media="(prefers-color-scheme: dark)" srcset=".github/assets/pam-wordmark-dark.svg">
  <img src=".github/assets/pam-wordmark-light.svg" height="56" alt="PAM">
</picture></p>

PAM runs PHP natively on servers, desktops and phones. This repository is one piece of it.

| [**PAM**](https://github.com/push-in/pam) | [**PAM Desktop**](https://github.com/push-in/pam-desktop) | [**PAM Native**](https://github.com/push-in/pam-native) | [**Ecosystem**](https://github.com/push-in/pam#ecosystem) |
| --- | --- | --- | --- |
| the runtime and CLI | native desktop apps in PHP | native Android and iOS apps in PHP | every official package and plugin |

<sub>Docs: [push-in.github.io/pam-docs](https://push-in.github.io/pam-docs/introduction/) · [Issues](https://github.com/push-in/pam-octane/issues)</sub>

<p align="center"><sub><a href="LICENSE">Apache License 2.0</a> · Made in Brazil by Pushin</sub></p>
