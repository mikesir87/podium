# Podium reporter kit (experimental)

A [Docker Sandboxes](https://docs.docker.com/ai/sandboxes/) kit that installs and configures the `podium-reporter` Claude Code plugin at
sandbox startup, so every Claude Code session run inside the sandbox shows up on a Podium fleet
console running on (or reachable from) the host — without the Podium auth token ever being stored
inside the sandbox.

This is a first draft, built against the Docker Sandboxes kit guides for connecting to a service
without exposing credentials and for making a kit configurable, plus one locally-inspected example
(`/etc/durable-startup.d/` inside a real sandbox). It has not yet been exercised end to end with a
real `sbx create` — see "Known unknowns" below.

## What it does

- **Network policy**: opens exactly the routes `podium-reporter` calls — `GET`/`POST`/`PUT` under
  `/api/fleet/**` on the configured Podium host and port. Nothing else on that host, and no other
  host at all.
- **Credential**: declares a proxy-managed secret named `podium-token`. The plugin's `token` option
  is set to the placeholder in `$PODIUM_API_TOKEN`; the sandbox's egress proxy swaps in the real
  value (from `sbx secret set podium-token` on the host) only on requests to the configured Podium
  host. The real token is never written to the sandbox's disk or process environment.
- **Lifecycle (startup)**: on every boot, non-interactively runs `claude plugin install
  podium-reporter --marketplace eengelking/skills` with `serverUrl`/`userName` baked in from the
  kit's args, then `claude plugin configure podium-reporter@eengelking-marketplace --values-stdin`
  to store the placeholder token in the plugin's secure config.
- **Agent context**: tells the agent reporting is active and proxy-managed, and not to print the
  placeholder token or ask the user for the real one.

## Prerequisites on the Podium server

The server needs to be reachable from the sandbox and to accept the `host.docker.internal` name
(see the main [README](../../README.md#several-people-on-one-podium) and
[DECISIONS 33](../../docs/DECISIONS.md)):

```sh
HOST=0.0.0.0 PODIUM_TOKEN=<a-real-token> PODIUM_ALLOWED_HOSTS=host.docker.internal \
  npm run start -w server
```

A loopback-only bind (the default) will not see traffic arriving from the sandbox's gateway.

## Using the kit

One-time, on the host:

```sh
sbx secret set podium-token
# paste the real Podium token (the admin secret, or a per-person token from Admin)
```

Create (or update) a sandbox with the kit:

```sh
sbx create docker/sbx-kit-shell:1.0.0 --name my-sandbox --kit ./kits/podium-reporter \
  --kit-arg userName="Your Name"
```

Approve the credential binding when prompted. `serverHost` and `serverPort` default to
`host.docker.internal` and `4317`; override with `--kit-arg serverHost=...` /
`--kit-arg serverPort=...` if Podium runs elsewhere. `userName` defaults to empty, which reports
with the shared admin secret instead of a per-person token.

The first Claude Code session started in the sandbox should appear on the Podium wall once it
sends its first prompt.

## Known unknowns (flag these when testing)

- **Port inside `network-policy`'s `hosts` vs `credential`'s `inject.domain`.** The hosts entry is
  written as `host:port` (`host.docker.internal:4317`); the credential's `inject.domain` is written
  as a bare host, per every example in the capability spec. Unconfirmed whether that's actually
  required, or whether `inject.domain` needs the port too.
- **Whether a `startup` lifecycle hook can read a `phase: runtime` credential's env var.** It's
  modeled here the same way the sandbox's own built-in MCP-gateway registration script does it
  (reads `$MCP_SENTINEL_TOKEN_NAME` from a startup-dispatched script), which is believed to carry
  over, but this kit hasn't been run yet to confirm it for a custom `credential@1` of our own.
- **Re-running `claude plugin install` on every boot.** Mirrors the built-in kit's own pattern
  (`claude mcp add ... || true`), but hasn't been checked for noisy output or any cost on a sandbox
  with a slow network path to the marketplace.
