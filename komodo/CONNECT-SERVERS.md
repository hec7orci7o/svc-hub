# Connecting Raspberry Pi boards to Komodo (Periphery)

Every board (`lab53`-`lab58`, including the one running Core itself) uses the same systemd
install — no bundled-container special case. Komodo v2 recommends systemd over a container for
Periphery; it avoids Docker-in-Docker permission headaches and is the "recommended and simplest"
method per the official docs.

Source: https://komo.do/docs/setup/connect-servers

**Migrating `lab53` from the old bundled setup:** the `periphery` service used to live in
`komodo/core/docker-compose.yml`, sharing Core's `keys` volume — that's gone now (confirmed in
production: two Periphery agents both claiming the `lab53` identity, one via that container and
one installed by hand via systemd, made the connection drop every few minutes). Redeploy the
Stack first so the removed service is torn down, then follow the steps below for `lab53` like
any other board.

## Steps

1. **Create an onboarding key** in the Komodo Core UI (`http://192.168.1.53:9120`), in the
   Servers section. It's a one-time-use key for the first handshake; no need to manage keys
   manually afterward (Core/Periphery rotate their own keys).

   `--core-address` below must be the direct IP, never the Traefik-fronted domain — Periphery
   has no LAN DNS for it and fails outright (confirmed in production: `Name or service not
   known`). This is also why `komodo/servers.toml` has no `address` field: Periphery dials
   Core (outbound), not the other way around — setting one signals the opposite mode.

2. **Install Periphery** with the official script (example with `lab56`, which runs n8n —
   repeat for every board, changing `--connect-as`). Remove any existing config first — a
   leftover `periphery.config.toml` from a previous attempt (stale key/state) blocks a clean
   handshake even with a fresh onboarding key, confirmed in production:

   ```sh
   sudo rm -f /etc/komodo/periphery.config.toml
   curl -sSL https://raw.githubusercontent.com/moghtech/komodo/main/scripts/setup-periphery.py \
     | sudo python3 - \
     --core-address="http://192.168.1.53:9120" \
     --connect-as="lab56" \
     --onboarding-key="O-..."

   sudo systemctl enable periphery
   ```

   `--connect-as` must match the `name` of the corresponding `[[server]]` entry in
   `komodo/servers.toml` (`lab53`, `lab54`, `lab55`, `lab56`, `lab57`, or `lab58`).

3. **Confirm the connection status** in the Komodo Core UI (should show `OK`), then remove
   `onboarding_key` from `/etc/komodo/periphery.config.toml` and restart the service — left in
   place, Periphery re-attempts the (single-use) onboarding flow on every reconnect instead of
   its own key pair, and once that key is spent, reconnection breaks outright (confirmed in
   production: `Matching Server onboarding key not found`).

   ```sh
   sudo sed -i '/^onboarding_key/d' /etc/komodo/periphery.config.toml
   sudo systemctl restart periphery.service
   ```

4. **Disable remote shell access** — if Komodo Core is ever compromised, this is what would
   otherwise let an attacker get an interactive shell on the board (host or into any container)
   through Periphery, not just run Komodo's normal structured actions:

   ```sh
   sudo sed -i \
     -e 's/^disable_terminals = false/disable_terminals = true/' \
     -e 's/^disable_container_terminals = false/disable_container_terminals = true/' \
     /etc/komodo/periphery.config.toml
   sudo systemctl restart periphery.service
   ```

Repeat for each board. The same steps (including the config removal) also apply to reconnecting
a board after its `[[server]]` got deleted and recreated in Komodo.
