# LocoNet MQTT Gateway — Releases

Installable releases of the LocoNet ⇄ MQTT gateway. It bridges a Digitrax LocoNet interface
(DCS52, PR4 or similar, over USB serial) to an MQTT broker so turnouts can be thrown and block
occupancy read from any MQTT client. Built for [LocoPanel](https://github.com/LocoPanel).

## Install

On a Raspberry Pi (or any Debian/Ubuntu machine) with the Digitrax controller plugged in:

```sh
curl -fsSL https://github.com/LocoPanel/loconet-gateway-releases/releases/latest/download/get.sh | sudo bash
```

The installer lists the serial devices it finds and asks which one is the Digitrax controller
(stable `/dev/serial/by-id/` names are shown first). It then installs Mosquitto, sets up a Python
virtualenv, and enables the `loconet-mqtt` systemd service. Running it again upgrades in place and
keeps your configuration.

Options go after `-s --`:

```sh
curl -fsSL https://github.com/LocoPanel/loconet-gateway-releases/releases/latest/download/get.sh \
  | sudo bash -s -- --no-mosquitto --serial-port /dev/ttyACM0
```

| Option | Effect |
|---|---|
| `--serial-port /dev/DEVICE` | Use this device and skip the prompt |
| `--no-mosquitto` | Don't install Mosquitto (use an existing or remote broker) |

**Requirements:** Debian/Ubuntu-based OS with `apt` and `systemd`, root access, internet access.

## MQTT topics

| Direction | Topic | Payload |
|---|---|---|
| Command → LocoNet | `track/turnout/cmd` | `42:T` throw, `42:C` close, `42:Q` report position |
| Command → LocoNet | `track/sensor/cmd` | `Q` asks all detectors to report again |
| LocoNet → MQTT | `track/turnout/status` | `42:T` / `42:C` |
| LocoNet → MQTT | `track/sensor/status` | `42:1` / `42:0` (switch-input feedback) |
| LocoNet → MQTT | `track/block/status` | `42:1` occupied / `42:0` clear (e.g. BDL16x detectors) |
| Gateway → MQTT | `track/gateway/status` | `online` / `offline` (retained) |

`online` is published only while both the MQTT broker and the LocoNet serial port are up, so clients can
treat everything as unknown otherwise. Detector state is requested whenever the serial port opens, so
block occupancy is current after a restart (takes about 10 s).

Try it:

```sh
mosquitto_pub -t track/turnout/cmd -m 5:T
mosquitto_sub -t 'track/#' -v
```

## Configuration

Settings live in `/etc/loconet-gateway.env` (created on first install, never overwritten). Uncomment and
edit, then `sudo systemctl restart loconet-mqtt`.

| Variable | Default |
|---|---|
| `LOCONET_SERIAL_PORT` | `/dev/ttyACM0` |
| `LOCONET_BAUD_RATE` | `57600` |
| `LOCONET_MQTT_HOST` / `LOCONET_MQTT_PORT` | `localhost` / `1883` |
| `LOCONET_MQTT_USER` / `LOCONET_MQTT_PASSWORD` | unset |
| `LOCONET_CMD_TOPIC` / `LOCONET_STATUS_TOPIC` | `track/turnout/cmd` / `track/turnout/status` |
| `LOCONET_SENSOR_CMD_TOPIC` / `LOCONET_SENSOR_TOPIC` | `track/sensor/cmd` / `track/sensor/status` |
| `LOCONET_BLOCK_TOPIC` | `track/block/status` |
| `LOCONET_AVAIL_TOPIC` | `track/gateway/status` |
| `LOCONET_QUERY_SENSORS` | `1` (set `0` to skip the startup detector query) |
| `LOCONET_IGNORE_SENSORS` | unset. Sensor numbers to leave out of block occupancy, e.g. `1-16`. Turnout decoders (DS54...) report turnout N's position as sensors 2N-1 and 2N. |

## Operate

```sh
systemctl status loconet-mqtt
journalctl -u loconet-mqtt -f
```

## Troubleshooting

- **`Cannot open /dev/ttyACM0`** in the logs: wrong device or controller not powered. Re-run the installer
  to pick again, or set `LOCONET_SERIAL_PORT`. `ls /dev/serial/by-id/` shows what is connected.
- **Permission denied on the port:** the service runs as user `loconet` in the `dialout` group; the
  installer sets this up.
- **Gateway shows `offline`:** the broker or the serial port is down; the service retries automatically.
- **Phantom block occupancy from turnout decoders:** set `LOCONET_IGNORE_SENSORS` (see above).

## Update and uninstall

Update by re-running the install command. To remove the service and files (Mosquitto and your
`/etc/loconet-gateway.env` are left alone):

```sh
curl -fsSL https://github.com/LocoPanel/loconet-gateway-releases/releases/latest/download/loconet-gateway.tar.gz | tar -xz -C /tmp
sudo /tmp/loconet-gateway/uninstall.sh
```

## Release contents

Each release has `get.sh` (the bootstrap script) and `loconet-gateway.tar.gz` (gateway, installer,
systemd unit, example config). Review `get.sh` before piping it to a shell if you prefer.

Source is private. Licensed under MIT, see [LICENSE](LICENSE).
