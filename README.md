# Home Assistant

Home Assistant Core running natively on macOS (Intel Mac mini home server), managed with uv.

## Setup

After cloning:

1. Install custom components:
   ```bash
   bin/setup
   ```
2. Install Python dependencies:
   ```bash
   uv sync
   ```
3. Start HA from the terminal:
   ```bash
   bin/start
   ```
4. Open `http://localhost:8123` and complete onboarding
5. Add integrations via Settings → Devices & Services → Add Integration:
   - **Eight Sleep** — search "Eight Sleep", sign in with your Eight Sleep account
   - **Dyson** — search "Dyson", sign in with your Dyson account
   - **HomeKit Bridge** — search "HomeKit", configure as bridge mode, select which domains to expose (climate, fan, humidifier). Once created, a pairing QR code and pin are shown in "Notifications". On your iPhone, open Apple Home → Add Accessory → scan the code or enter the pin.
6. Stop HA from the terminal:
   ```bash
   bin/stop
   ```
7. Set up the LaunchDaemon for auto-start:

   Create the plist:

   ```xml
   sudo tee /Library/LaunchDaemons/org.nixos.home-assistant.plist > /dev/null << 'EOF'
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
   <plist version="1.0">
   <dict>
     <key>EnvironmentVariables</key>
     <dict>
       <key>PATH</key>
       <string>/run/current-system/sw/bin:/nix/var/nix/profiles/default/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
       <key>HOME</key>
       <string>/Users/camen</string>
     </dict>
     <key>KeepAlive</key>
     <dict>
       <key>PathState</key>
       <dict>
         <key>/Users/camen/Projects/home-assistant/bin/start</key>
         <true/>
       </dict>
     </dict>
     <key>Label</key>
     <string>org.nixos.home-assistant</string>
     <key>ProgramArguments</key>
     <array>
       <string>/bin/sh</string>
       <string>-c</string>
       <string>/bin/wait4path /nix/store &amp;&amp; exec /bin/bash -c 'test -x /Users/camen/Projects/home-assistant/bin/start &amp;&amp; exec /Users/camen/Projects/home-assistant/bin/start'</string>
     </array>
     <key>RunAtLoad</key>
     <true/>
     <key>StandardErrorPath</key>
     <string>/tmp/home-assistant.stderr.log</string>
     <key>StandardOutPath</key>
     <string>/tmp/home-assistant.stdout.log</string>
     <key>WorkingDirectory</key>
     <string>/Users/camen/Projects/home-assistant</string>
   </dict>
   </plist>
   EOF
   ```

   Set permissions and start:

   ```
   sudo chown root:wheel /Library/LaunchDaemons/org.nixos.home-assistant.plist
   ```

   ```
   sudo launchctl bootstrap system /Library/LaunchDaemons/org.nixos.home-assistant.plist
   ```

The `config/` directory is gitignored — it contains credentials, device pairings, and database files that are created during onboarding and integration setup.

## Integrations

- **Eight Sleep** — custom component (`lukas-clarke/eight_sleep`), config-flow only. Vendored under `config/custom_components/eight_sleep/` and tracked in git because of a local patch — see [Apple Home off command re-enables the bed](#apple-home-off-command-re-enables-the-bed).
- **Dyson** — custom component (`libdyson-wg/ha-dyson`), config-flow only
- **HomeKit** — built-in, bridge mode on port 21064

## Dev

Run directly from the terminal:

```
bin/start    # Start HA
bin/stop     # Stop HA
bin/setup    # Install custom components
bin/update   # Update HA version
```

Logs go to stderr in the terminal.

## Server

HA runs as a macOS LaunchDaemon so it starts at boot, before login.

### Plist

The plist lives at `/Library/LaunchDaemons/org.nixos.home-assistant.plist`. It runs `bin/start` as root, which immediately drops to user `camen` via `sudo -u camen` before launching hass. See [Setup](#setup) for the full plist and installation.

### Managing

```
sudo launchctl bootstrap system /Library/LaunchDaemons/org.nixos.home-assistant.plist
```

Start the server.

```
sudo launchctl bootout system/org.nixos.home-assistant
```

Stop the server.

```
tail -f /tmp/home-assistant.stderr.log
```

Tail the logs.

Shell aliases are also available: `ha-start`, `ha-stop`, `ha-restart`, `ha-log`.

### Logs

- `/tmp/home-assistant.stderr.log` — all warnings, errors, and debug output
- `/tmp/home-assistant.stdout.log` — unused
- `config/home-assistant.log` — HA's internal log file (warnings and above)

### Debugging HomeKit

Check if the HomeKit bridge is advertising via mDNS:

```
dns-sd -B _hap._tcp local.
```

The bridge should appear as `HASS Bridge` within a few seconds. Ctrl-C to stop.

## Errors

### HomeKit bridge not visible in Apple Home / "No Response"

**Symptom:** Apple Home shows "No Response". The HomeKit bridge is not advertising via mDNS. `dns-sd -B _hap._tcp local.` does not show `HASS Bridge`. Logs show:

```
[zeroconf] Error with socket 15 (('0.0.0.0', 5353))): [Errno 65] No route to host
OSError: [Errno 65] No route to host
```

**Cause:** macOS Local Network Privacy silently blocks mDNS multicast (UDP to 224.0.0.251:5353) for processes spawned by LaunchAgents. This is not a network issue — multicast works fine from a terminal because terminal apps (Ghostty, Terminal.app, iTerm) are exempt.

**Fix:** HA must run as a LaunchDaemon (root), which is exempt from Local Network Privacy. `bin/start` drops to user `camen` via `sudo -u camen` so hass itself never runs as root. A LaunchAgent will not work regardless of codesigning, .app bundle wrapping, or plist configuration.

### Apple Home "off" command re-enables the bed

**Symptom:** An Apple Home automation that sets an Eight Sleep climate accessory to "off" appears to take effect for a moment but the bed stays heating (or quickly turns back on). The target temperature dial in the Home app stays at its previous value.

**Cause:** HomeKit doesn't send a single "off" command for a thermostat accessory — Apple Home pairs every `set_hvac_mode: off` with a follow-up `set_temperature` call that carries `hvac_mode: off` alongside the target temperature. The upstream `lukas-clarke/eight_sleep` integration's `async_set_temperature` ignores the `hvac_mode` kwarg and calls `pyEight.user.set_heating_level`, which unconditionally calls `turn_on_side()` before writing the level. The off command lands, then the temperature call immediately re-enables the side. A bare `set_temperature` on a bed that's currently off (e.g. the Home app sending only the new target on a side that was already off) hits the same code path.

**Fix:** Patched locally in `config/custom_components/eight_sleep/climate.py`. `async_set_temperature` now short-circuits when `hvac_mode: off` is passed in the call or when the bed is already off — it stores the new target and writes state without pushing to the device. `async_set_hvac_mode(HEAT_COOL)` calls `set_heating_level` directly instead of recursing through `async_set_temperature`, so the turn-on flow still restores the stored target.
