# ESP32PLC-DeviceLibrary

A library of remote device type configurations for the [ESP32PLC](https://github.com/richcj10/-ESP32PLC) project. The ESP32PLC web UI fetches `devices.json` directly from this repo at runtime to populate device presets during a bus scan.

---

## How It Works

When the ESP32PLC web UI scans the Modbus bus and discovers an unknown device type, it offers a **Fetch** button. Clicking it pulls `devices.json` from this repo, finds the matching `typeId`, and adds the device — pre-configured with its register map, poll rates, and MQTT topics — to `Remote.json` on the device.

The URL the firmware targets is:
```
https://raw.githubusercontent.com/richcj10/ESP32PLC-DeviceLibrary/main/devices.json
```

---

## `devices.json` Structure

The root file is a JSON object containing a `devices` array. Each entry describes one device type — no instance-specific data (like `address`) is included; the UI injects the scanned address at add time.

```json
{
  "devices": [
    {
      "name": "DeviceName",
      "typeId": 1,
      "swVersion": 1,
      "groups": [ ... ]
    }
  ]
}
```

### Device Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Human-readable device name |
| `typeId` | uint8 | Unique device type ID — must match register 1 on the physical device |
| `swVersion` | uint16 | Expected firmware version — used for version-mismatch detection |
| `groups` | array | One or more register groups (see below) |

### Group Fields

Each entry in `groups` defines a set of registers that are polled together.

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Group label shown in the UI |
| `fc` | uint8 | Modbus function code (`4` = Read Input Registers, `3` = Read Holding Registers) |
| `startReg` | uint16 | First register address to read |
| `count` | uint8 | Number of registers to read |
| `pollMs` | uint32 | Poll interval in milliseconds |
| `scale` | float | Divide raw register value by this to get the display value (e.g. `100` → divide by 100) |
| `units` | string | Unit label appended to values (e.g. `"A"`, `"°C"`, `""`) |
| `regs` | array | Register name strings, one per register in read order |
| `mqttEnable` | bool | Whether to publish this group to MQTT |
| `mqttTopic` | string | MQTT topic base for this group |
| `writes` | array | Optional write groups (see below) |

### Write Group Fields

Entries in `writes` define Modbus write operations, typically triggered via MQTT.

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Write operation label |
| `fc` | uint8 | Modbus function code (`16` = Write Multiple Registers, `5` = Write Single Coil) |
| `startReg` | uint16 | First register address to write |
| `count` | uint8 | Number of registers to write |
| `regs` | array | Register name strings or integer literals (fixed values sent as-is) |
| `defaults` | array | Default uint16 values written when no MQTT payload overrides them |
| `mqttTopic` | string | MQTT topic this write listens on for trigger payloads |

---

## Example Entry

```json
{
  "name": "OCC-TOL",
  "typeId": 2,
  "swVersion": 4,
  "groups": [
    {
      "name": "Occupancy",
      "fc": 4,
      "startReg": 2,
      "count": 7,
      "pollMs": 1500,
      "scale": 100,
      "units": "",
      "regs": ["VIN", "Temp1", "Temp2", "Humid1", "Humid2", "Pres", "Lux"],
      "mqttEnable": true,
      "mqttTopic": "remote/OCC",
      "writes": [
        {
          "name": "LED",
          "fc": 16,
          "startReg": 0,
          "count": 3,
          "regs": [18, "R", "GB"],
          "defaults": [18, 0, 0],
          "mqttTopic": "remote/OCC/led/set"
        }
      ]
    }
  ]
}
```

---

## Adding a New Device

1. Add a new entry to the `devices` array in `devices.json` with a unique `typeId`.
2. Add a matching standalone file under `DeviceConfigs/<DeviceName>.json` for reference.
3. Open a PR — once merged to `main`, the ESP32PLC UI will pick it up automatically on the next Fetch.

---

## Device Type IDs

| typeId | Device |
|--------|--------|
| 2 | OCC-TOL (Occupancy / Environmental Sensor) |
| 5 | CurrentSensor |
| 13 | Weather Station |
