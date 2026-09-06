# DeepSeek Harness SDK for Workshop

This SDK provides DeepSeek Harness (`dsh`), an open-source, plugin-based agent
harness from DeepSeek AI where "everything is a plugin". The `dsh` CLI and a
bundled Node.js runtime are prebuilt into the SDK image, so `dsh` is on `PATH`
inside the workshop with no runtime install step. All harness user data —
profiles, sessions, and configuration under `~/.dsh` — is persisted on the host
across workshop updates, and the web UI port is exposed through a tunnel so you
can open it from the host.

This SDK ships only the harness runtime; it does not include model
credentials. `dsh` talks to a model backend (for example the DeepSeek API), so
you supply a key at runtime — see [Prerequisites](#prerequisites-project-layout).

---

## Reference workshop

A minimal workshop:

```yaml
# workshop.yaml
name: dev
base: ubuntu@24.04
sdks:
  - name: system
    plugs:
      dsh-web:
        interface: tunnel
        endpoint: localhost:3080
  - name: dsh
    channel: latest/edge
    plugs:
      dsh-web:
        interface: tunnel
        endpoint: localhost:3080

actions:
  web: |
    dsh web
```

n.b. Don't forget to run `workshop connect dev/system:dsh-web dev/dsh:dsh-web` to activate the connection.

This demonstrates booting the DeepSeek Harness web UI. Running the `web` action
starts the server on `http://127.0.0.1:3080` inside the workshop, reachable from
the host through the `dsh-web` tunnel. The `base` may be any of `ubuntu@22.04`,
`ubuntu@24.04`, or `ubuntu@26.04` — the SDK ships a build for each.

---

## Snap channels and (in)stability

The SDK is currently only published to the `latest/edge` track.

Upstream's DeepSeek Harness releases are all marked **Pre-release**, they have
not reached a stable release cadence yet. Upgrades between these pre-release
versions may not be smooth, and may even require a `workshop destroy` followed
by `workshop launch` to recover from incompatibilities between versions.

Because of that, we intentionally do not publish to `latest/stable` — data-loss
and breakage are not acceptable on a stable track. Once upstream starts
producing stable releases, we will publish to `latest/stable` and keep
`latest/edge` for on-going pre-release work.

---

## Using the SDK

### Prerequisites, project layout

1. `dsh` needs a model backend and this SDK ships no credentials. Provide a
   DeepSeek API key (or another backend supported by the harness) before
   starting a session. Export it in the workshop shell or persist it in a
   profile under `~/.dsh` (which survives workshop updates):

   ```bash
   # inside the workshop
   export DEEPSEEK_API_KEY=sk-...
   ```

2. The directory you launch `dsh` from is the default workspace root, so run it
   from `/project` (your mounted sources) to let the agent operate on your code.

3. On first use, the `web` and `headless` profiles auto-initialize from shipped
   templates into `$DSH_HOME/profiles/` (`~/.dsh/profiles/`). This directory is
   persisted, so the initialization happens once and survives workshop updates.

### Run the web UI

Once the workshop is ready, from the host:

```bash
workshop run web
```

Or drive it manually from inside the workshop:

```bash
workshop shell
# inside the workshop, in your project directory:
dsh web                 # serves http://127.0.0.1:3080
dsh web --port 8080     # app flags pass through to the web profile
```

The server binds `127.0.0.1:3080` inside the workshop; the `dsh-web` tunnel
slot forwards that port so you can open the UI from the host.

### Run a headless job

From within the workshop shell:

```bash
workshop shell
# inside the workshop, in your project directory:
dsh --profile headless "summarize the failing tests and propose a fix"
```

This boots a single fresh persisted session, prints the final answer, and
exits. Session state is written under `~/.dsh` and therefore survives workshop
updates.

### Manage plugins

`dsh plugin --profile <name> <pnpm args>` installs out-of-tree plugins into a
profile by forwarding to `pnpm`. This SDK exposes `pnpm` on `PATH` (via the
bundled Node.js `corepack`) so the workflow works without an extra SDK:

```bash
dsh plugin --profile web add <some-dsh-plugin>
```

Installed plugins live in the profile's `node_modules` under `~/.dsh`, so they
persist across workshop updates.

### Verify

```bash
# On the host:
workshop info        # the SDK health line should read "okay"
workshop shell
```

```bash
# Inside the workshop:
dsh --help           # prints the launcher command grammar and exits 0
```

---

## Known limitations

### "Open configuration file" cannot open on the host

The Settings pane's **Open configuration file** action does not work in this
SDK and fails with *"Could not open configuration file"*.

The action is a *host-side* convenience: when clicked, the `dsh` process
running **inside the workshop** resolves the container-local settings path
(`~/.dsh/...`) and spawns a native text editor on its own host via
`xdg-open` / the desktop file association. In the workshop topology that can
never succeed:

- The workshop container is headless — it has no display server and does not
  ship `xdg-open`, so the spawn fails immediately.
- The path handed to the editor is a container path that does not exist on the
  host machine, and the browser never receives a host-usable path to open.

Upstream, the button's visibility is gated on *loopback detection* (the browser
reached `dsh web` over a loopback address) rather than on whether a native
opener is actually available, so a tunneled browser makes it appear even though
it can never work here. This cannot be disabled from a runtime plugin or the
SDK — it requires an upstream fix. See
[deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness).

**Workaround:** edit the settings file from inside the workshop instead:

```bash
workshop shell
# inside the workshop:
cat ~/.dsh/settings.yaml      # or open it with the editor of your choice
```

---

## Plugs (resources this SDK consumes)

### `dsh-home`

- Interface: `mount`
- Workshop target: `/home/workshop/.dsh`
- Purpose: persists the single DeepSeek Harness home root — profiles, sessions,
  installed plugins, and configuration — so agent state and setup survive
  workshop updates.

## Slots (resources this SDK provides)

### `dsh-web`

- Interface: `tunnel`
- Endpoint: `3080`
- Purpose: exposes the `dsh web` UI port so the host can reach the harness web
  interface running inside the workshop.

---

## Documentation and guidance

- [DeepSeek Harness repository](https://github.com/deepseek-ai/deepseek-harness)
- [Cordis](https://github.com/cordiverse/cordis), the plugin runtime that powers the harness
- [Workshop documentation](https://documentation.ubuntu.com/workshop/)

---

## Community and support

- [DeepSeek Harness GitHub Discussions](https://github.com/deepseek-ai/deepseek-harness/discussions)
  and [Discord community](https://discord.gg/Ycq5dCaS4).
- Please review our [Code of Conduct](https://ubuntu.com/community/ethos/code-of-conduct)
  before participating.

---

## Contributions

All contributions, including code, documentation updates, and issue reports,
are welcome!

- Open issues or pull requests on the SDK repository.
- For the harness itself, see the upstream
  [CONTRIBUTING guide](https://github.com/deepseek-ai/deepseek-harness/blob/main/CONTRIBUTING.md).

---

## License and copyright

Copyright 2026 Canonical Ltd. The SDK packaging is distributed under the
Apache License 2.0.

The packaged DeepSeek Harness is Copyright 2026 DeepSeek AI, distributed under
the [MIT License](https://github.com/deepseek-ai/deepseek-harness/blob/main/LICENSE).
