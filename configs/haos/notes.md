# Home Assistant OS — host notes
# Host: hao.l  (Home Assistant OS, version 2026.6.3)
# NOTE: This host is HAOS, not a generic Docker host. There is no shell-level
# docker/systemd; administration happens through the Home Assistant CLI
# (`ha`) over SSH as root. Per the project rules we describe capabilities in
# prose and deliberately do NOT include configuration.yaml, secrets.yaml, or
# .storage/ contents (they contain long-lived tokens, coordinates, and
# device IDs).

## Addons (official core repository)
- Terminal & SSH (10.3.0) — remote admin
- Samba share (12.6.1) — folder access from the LAN
- File editor (6.1.0) — in-browser config editing

## Integrations / custom components
- HACS with Lovelace card-mod (frontend customization)
- Tuya / Sonoff / localtuya / tuya_local — smart switches and plugs
  controlled over the local network
- default_config, recorder, notify, rest_command, packages

## Home Assistant packages
- `packages/prayer_times.yaml` — prayer-time sensors, the same data source
  as the self-hosted Mawaqit API on home.l

## Automations & scripts (themes, not full dumps)
- A voice-interactive robot ("robo") with a suite of scripts: broadcast
  audio/TTS, attention-then-play, send messages, and a Quran reading flow
  (including a "read from slider" variant and volume control)
- Switch on/off automations against the Tuya/Sonoff devices
- Notifications to a mobile phone (companion app) and to the robot's
  devices
- ~10 custom scripts driving the above

## Notes
- Automations reference devices by ID; device/host identifiers are redacted
  here on purpose.
- HAOS runs its own Supervisor; no containers are managed from this
  repository.
