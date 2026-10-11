# MQTT Publishing

Bambuddy can publish events to an external MQTT broker, enabling integration with home automation platforms like **Home Assistant**, **Node-RED**, and other MQTT-based systems.

## Configuration

Navigate to **Settings → Network → MQTT Publishing** to configure the connection.

### Settings

| Setting | Description | Default |
|---------|-------------|---------|
| **Enable MQTT** | Toggle MQTT publishing on/off | Off |
| **Broker Hostname** | MQTT broker address (IP or hostname) | - |
| **Port** | Broker port | 1883 (8883 with TLS) |
| **Username** | Authentication username (optional) | - |
| **Password** | Authentication password (optional) | - |
| **Topic Prefix** | Prefix for all published topics | `bambuddy` |
| **Use TLS** | Enable TLS/SSL encryption | Off |

!!! tip "Port Auto-Population"
    When you toggle TLS on/off, the port field automatically updates:

    - TLS enabled → Port 8883
    - TLS disabled → Port 1883

## Published Topics

All topics are prefixed with your configured topic prefix (default: `bambuddy`).

### Status

| Topic | Description | Retained |
|-------|-------------|----------|
| `bambuddy/status` | Online/offline status | Yes |

### Printer Events

| Topic | Description | Retained |
|-------|-------------|----------|
| `bambuddy/printers/{serial}/status` | Real-time printer state (throttled to 1/sec) | Yes |
| `bambuddy/printers/{serial}/print/started` | Print job started | No |
| `bambuddy/printers/{serial}/print/completed` | Print job completed successfully | No |
| `bambuddy/printers/{serial}/print/failed` | Print job failed | No |
| `bambuddy/printers/{serial}/ams/changed` | AMS filament changed | No |
| `bambuddy/printers/{serial}/plate_clear` | Plate-clear gate raised or released | Yes |
| `bambuddy/printers/{serial}/offline` | Printer has been disconnected for a minute | No |
| `bambuddy/printers/{serial}/online` | Printer reconnected after an `offline` event | No |
| `bambuddy/printers/{serial}/filament/low` | The spool in an AMS slot crossed its low-stock threshold | No |

### Queue Events

| Topic | Description | Retained |
|-------|-------------|----------|
| `bambuddy/queue/job_added` | Job added to print queue | No |
| `bambuddy/queue/job_started` | Queued job started printing | No |
| `bambuddy/queue/job_completed` | Queued job completed | No |

### Maintenance Events

| Topic | Description | Retained |
|-------|-------------|----------|
| `bambuddy/maintenance/alert` | Maintenance due or nearly due, after a print; `current_value` is the hours (or days) since it was last done, `threshold` its interval, `unit` says `hours` or `days` | No |
| `bambuddy/maintenance/reset` | Maintenance counter reset | No |

### Smart Plug Events

| Topic | Description | Retained |
|-------|-------------|----------|
| `bambuddy/smart_plugs/on` | A plug was switched on, by hand or automatically (auto-on, schedule, queue); also sent when it already was on | No |
| `bambuddy/smart_plugs/off` | A plug was switched off, by hand or automatically (auto-off, schedule, failure detection); also sent when it already was off | No |
| `bambuddy/smart_plugs/energy` | Power and energy of each enabled plug, once a minute | No |

### Archive Events

| Topic | Description | Retained |
|-------|-------------|----------|
| `bambuddy/archive/created` | New archive created | No |
| `bambuddy/archive/updated` | Archive updated | No |

## Payload Format

All payloads are JSON objects. Here's an example printer status payload:

```json
{
  "printer_id": 1,
  "printer_name": "X1C-1",
  "printer_serial": "00M09C411500579",
  "timestamp": "2026-01-13T12:00:00.000000",
  "connected": true,
  "state": "PRINTING",
  "progress": 45.5,
  "remaining_time": 3600,
  "layer_num": 150,
  "total_layers": 300,
  "current_print": "benchy.3mf",
  "subtask_name": "Benchy",
  "temperatures": {
    "bed": 60.0,
    "bed_target": 60.0,
    "nozzle": 220.0,
    "nozzle_target": 220.0,
    "chamber": 35.0
  },
  "wifi_signal": -55,
  "chamber_light": true,
  "speed_level": 2,
  "cooling_fan_speed": 100,
  "big_fan1_speed": 50,
  "big_fan2_speed": 50,
  "awaiting_plate_clear": false
}
```

### Plate Clear

`awaiting_plate_clear` is a Bambuddy-side flag, not printer telemetry: the printer itself only ever reports `RUNNING`, `PAUSE`, `FAILED`, `FINISH` and `IDLE`. It goes `true` when a print reaches a terminal state and stays `true` until someone confirms the build plate is free — the queue will not dispatch the next job in the meantime. See [Print Queue](print-queue.md) for the gate itself.

Because the status topic only refreshes when the printer pushes telemetry — which stops entirely once [Auto Off](smart-plugs.md) powers the printer down — every transition is also published on its own **retained** topic:

```json
{
  "printer_id": 1,
  "printer_name": "X1C-1",
  "printer_serial": "00M09C411500579",
  "awaiting": true,
  "timestamp": "2026-01-13T12:00:00.000000"
}
```

Subscribe to `bambuddy/printers/+/plate_clear` and you get the current state of every printer the moment you connect, then one message per change. Acknowledge from your own automation with `POST /api/v1/printers/{id}/clear-plate` (see the [API reference](../reference/api.md)).

!!! tip "Notification too"
    The same transition can raise a push notification — enable **Plate Clear Required** on a notification provider. It is off by default because it fires after every print, at the same moment as the print-complete notification. See [Notifications](notifications.md).

### Printer Offline and Online

`offline` is sent when a printer has stayed disconnected for a minute, so a short network blip sends nothing. `online` is sent when that printer connects again, and only after an `offline` went out, so an `online` never arrives on its own. If the broker is unreachable when the printer comes back, `online` follows once the relay has reconnected. Both carry the printer:

```json
{
  "printer_id": 1,
  "printer_name": "X1C-1",
  "printer_serial": "00M09C411500579",
  "timestamp": "2026-01-13T12:00:00.000000"
}
```

### Low Filament

Sent once when the spool assigned to an AMS slot drops below its low-stock threshold (the spool's own, or the global one, 20% by default; with Spoolman always the global one), at the same moment as the **Low Filament** notification. It is sent again only after the slot has gone back above the threshold, for example with a new spool, or after the relay was switched off and on again, which sends every slot that is low at that point. If the relay is only disconnected at that moment, the event follows once it reconnects.

```json
{
  "printer_id": 1,
  "printer_name": "X1C-1",
  "printer_serial": "00M09C411500579",
  "slot": "A1",
  "remaining_percent": 15,
  "color": "Jade White",
  "spool_id": 42,
  "timestamp": "2026-01-13T12:00:00.000000"
}
```

`color` is the spool's colour name, or `null` when it has none. `spool_id` is Bambuddy's spool ID, or Spoolman's when Spoolman manages the inventory. Only slots with an assigned spool are checked.

### Smart Plug Energy

Every enabled plug's readings are published once a minute while the relay is connected. A value the plug doesn't report is `null`. A plug that gives no reading at all is left out (a Home Assistant plug without a power value counts as that), and so is an MQTT plug that has stopped sending messages. An MQTT plug has a single energy value, the one from its energy topic, which is given as both `energy_today_kwh` and `energy_total_kwh`.

```json
{
  "plug_id": 3,
  "plug_name": "X1C plug",
  "power_watts": 182.4,
  "energy_today_kwh": 1.27,
  "energy_total_kwh": 315.8,
  "timestamp": "2026-01-13T12:00:00.000000"
}
```

## Home Assistant Integration

### MQTT Discovery

Bambuddy does not currently support Home Assistant MQTT discovery. You'll need to configure sensors manually.

### Example Sensor Configuration

Add to your `configuration.yaml`:

```yaml
mqtt:
  sensor:
    - name: "X1C Print Progress"
      state_topic: "bambuddy/printers/00M09C411500579/status"
      value_template: "{{ value_json.progress }}"
      unit_of_measurement: "%"

    - name: "X1C State"
      state_topic: "bambuddy/printers/00M09C411500579/status"
      value_template: "{{ value_json.state }}"

    - name: "X1C Bed Temperature"
      state_topic: "bambuddy/printers/00M09C411500579/status"
      value_template: "{{ value_json.temperatures.bed }}"
      unit_of_measurement: "°C"
      device_class: temperature
```

### Node-RED Example

Use an **MQTT In** node subscribed to `bambuddy/#` to receive all events, then route them based on topic:

```
[MQTT In] → [Switch (by topic)] → [Function/Dashboard]
```

## TLS/SSL Configuration

Bambuddy supports TLS connections with self-signed certificates:

- Certificates are **not verified** (allows self-signed certs)
- Connection is still encrypted
- Suitable for home networks

For production environments, use properly signed certificates from your broker.

## Troubleshooting

### Connection Status

The connection status is shown in the Settings page:

- **Green dot** = Connected
- **Red dot** = Disconnected or error

### Common Issues

| Issue | Solution |
|-------|----------|
| "Not authorized" | Check username/password |
| Connection refused | Verify broker address and port |
| TLS errors | Ensure broker supports TLS on the configured port |
| No messages received | Check your subscriber's topic filter matches |

### Testing with Mosquitto

```bash
# Subscribe to all Bambuddy topics
mosquitto_sub -h your-broker -t "bambuddy/#" -v

# With authentication
mosquitto_sub -h your-broker -u username -P password -t "bambuddy/#" -v

# With TLS
mosquitto_sub -h your-broker -p 8883 --cafile ca.crt -t "bambuddy/#" -v
```
