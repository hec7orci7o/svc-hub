# Connecting more Raspberry Pi boards to Komodo (Periphery)

`lab53` already carries its own Periphery bundled in `core/docker-compose.yml`.
For the rest of the fleet (`lab54`, `lab55`, `lab56`, `lab57`, `lab58`), Komodo v2 recommends
installing Periphery as a systemd service instead of a container — it avoids Docker-in-Docker
permission headaches and is the "recommended and simplest" method per the official docs.

Source: https://komo.do/docs/setup/connect-servers

## Steps

1. **Create an onboarding key** in the Komodo Core UI (`http://lab53.local:9120`), in the
   Servers section. It's a one-time-use key for the first handshake; no need to manage keys
   manually afterward (Core/Periphery rotate their own keys).

2. **Install Periphery on the new Pi** with the official script (example with `lab56`, which
   runs n8n — repeat for `lab54`, `lab55`, `lab57`, `lab58`, changing `--connect-as`):

   ```sh
   curl -sSL https://raw.githubusercontent.com/moghtech/komodo/main/scripts/setup-periphery.py \
     | python3 - \
     --core-address="http://lab53.local:9120" \
     --connect-as="lab56" \
     --onboarding-key="O-..."

   sudo systemctl enable periphery
   ```

   `--connect-as` must match the `name` of the corresponding `[[server]]` entry in
   `komodo/servers.toml` (`lab54`, `lab55`, `lab56`, `lab57`, or `lab58`).

3. **Confirm the connection status** in the Komodo Core UI (should show `OK`).

Repeat for each additional Raspberry Pi. The script can be safely re-run after each Komodo
release to update the Periphery version.
