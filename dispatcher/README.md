🇧🇷 [Ler em português](../docs/pt-BR/dispatcher.md)

# Dispatcher

`dispatcher/src/` is **generated, not checked in** (see `.gitignore`). On first
`./aem start dispatcher` (or `./aem start full`), the CLI:

1. Extracts Adobe's Dispatcher Tools from your SDK zip in `sdk/`.
2. Copies the `src/` baseline that ships inside the Dispatcher Tools
   themselves into `dispatcher/src/` — a complete, working default
   farm/vhost pair (immutable `default_*` files, `enabled_farms`/
   `enabled_vhosts` already symlinked to `available_farms`/
   `available_vhosts`), exactly as Adobe packages it.
3. Runs `docker_run.sh` against that folder, pointed at your configured
   `AEM_PUBLISH_PORT` and `DISPATCHER_PORT` from `.env`.

This gives you a working Dispatcher out of the box with **no application
knowledge** — it is infrastructure, not your project's real caching/filter
rules.

## Using your real project's dispatcher config instead

If your AEM project already has its own `dispatcher/src/` (from the Cloud
Manager archetype), point at it instead of the generated baseline:

```env
# in .env
DISPATCHER_SRC_DIR=/absolute/path/to/your-project/dispatcher/src
```

This repository never reads or modifies your project's files — it only
passes the path to Adobe's `docker_run.sh`.
