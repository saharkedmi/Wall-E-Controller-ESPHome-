# Wall-E Controller (ESPHome)

An interactive IoT controller shaped like the robot Wall-E (3D-printed enclosure, printed on a Bambu Lab printer). The project is built on an **ESP32-C3** running **ESPHome**, and its purpose is to protect kids' eyes from sitting too close to the TV: the controller measures distance in real time, filters out the noise/"ghost readings" typical of ultrasonic sensors, and automatically triggers mute/restore volume scripts in Home Assistant when it detects a stable presence too close to the screen.

## Table of Contents

- [Hardware](#hardware)
- [Wiring](#wiring)
- [Filtering Algorithm](#filtering-algorithm)
- [Setup](#setup)
- [Home Assistant Integration](#home-assistant-integration)
- [Adapting to Different TV Brands](#adapting-to-different-tv-brands)
- [Print Files (STL)](#print-files-stl)
- [Security and Secrets](#security-and-secrets)

## Hardware

| Component | Details |
|---|---|
| Main controller | ESP32-C3 (`esp32-c3-devkitm-1` dev board) |
| Distance sensor | HC-SR04 ultrasonic sensor |
| Enclosure | 3D-printed Wall-E shaped case |

## Wiring

| Pin | Role |
|---|---|
| GPIO2 | `trigger_pin` (sends pulse) |
| GPIO3 | `echo_pin` (receives echo) |

## Filtering Algorithm

Ultrasonic sensors are prone to "ghost readings" and noisy values, so the controller runs internal logic (`globals`) every 0.8 seconds to avoid false triggers:

1. **Operating window (zone)** – valid detection only in the 35cm to 1.2m range. `0.0` readings are filtered out immediately.
2. **Motion vs. static object** – if the distance doesn't change by more than 3cm over more than 6 consecutive samples, the system assumes it's a static object (a wall, furniture) rather than a child, and resets the presence buffer.
3. **Decision debounce buffer** – 3 consecutive in-range samples are required to mute, and 3 consecutive out-of-range samples are required to restore, so volume doesn't flicker.
4. **Time window** – the controller syncs to an SNTP server and only allows muting between 06:00 and 22:00.

## Setup

1. Clone the repository:
   ```bash
   git clone https://github.com/saharkedmi/Wall-E-Controller-ESPHome-.git
   cd Wall-E-Controller-ESPHome-
   ```
2. Copy `secrets.yaml.example` to `secrets.yaml` and fill in your real values (WiFi, API key, OTA password, etc.):
   ```bash
   cp secrets.yaml.example secrets.yaml
   ```
3. Generate a valid API encryption key with:
   ```bash
   esphome generate-key wall-e-controller
   ```
4. Flash the controller:
   ```bash
   esphome run wall-e-controller.yaml
   ```

> `secrets.yaml` is listed in `.gitignore` and should never be committed to git.

## Home Assistant Integration

The controller doesn't control the TV directly — it calls two `script`s defined in Home Assistant itself, and all the logic for "how to actually mute *your* specific TV" lives there, not in the firmware:

- `script.wall_e_volume_mute`
- `script.wall_e_volume_restore`

A generic example of these scripts, based on a standard Home Assistant `media_player` entity:

```yaml
script:
  wall_e_volume_mute:
    sequence:
      - service: media_player.volume_mute
        target:
          entity_id: media_player.living_room_tv
        data:
          is_volume_muted: true

  wall_e_volume_restore:
    sequence:
      - service: media_player.volume_mute
        target:
          entity_id: media_player.living_room_tv
        data:
          is_volume_muted: false
```

This split (the ESP32 calls a `script`, and the `script` is what talks to the TV) is intentional: it lets you swap TVs or control methods without touching the controller's firmware at all.

## Adapting to Different TV Brands

Since the ESP32 doesn't "know" anything about the TV brand (it only calls two generic scripts), brand-specific adaptation happens entirely on the Home Assistant side, inside the scripts. Some common options depending on TV type:

- **HDMI-CEC** (any TV that supports CEC, most newer models) – Home Assistant's [HDMI-CEC integration](https://www.home-assistant.io/integrations/hdmi_cec/), or a device like the Pulse-Eight USB-CEC adapter. Works well for generic mute/unmute regardless of brand.
- **Samsung (Tizen)** – Home Assistant's built-in [Samsung TV integration](https://www.home-assistant.io/integrations/samsungtv/), which exposes a `media_player` entity supporting `volume_mute`.
- **LG (webOS)** – the [LG webOS Smart TV integration](https://www.home-assistant.io/integrations/webostv/), also exposing a `media_player` with mute support.
- **Sony (Bravia)** – the [Sony Bravia TV integration](https://www.home-assistant.io/integrations/braviatv/).
- **Android TV / Google TV** (Sony, TCL, Philips, etc.) – the [Android Debug Bridge (ADB) integration](https://www.home-assistant.io/integrations/androidtv/), which controls volume as well as other actions (pause, navigation).
- **"Dumb" TVs without WiFi/CEC support** – a universal infrared remote. You can use [Broadlink](https://www.home-assistant.io/integrations/broadlink/), or even a second ESP32 with ESPHome's `remote_transmitter` sending IR volume-down/volume-up codes, wrapped in a `script` that simulates muting (e.g., counting "volume down" presses down to 0, while storing the prior value for restore).
- **External speaker / separate sound system** – if audio doesn't come from the TV itself but from a soundbar or home theater system, point the `entity_id` in the scripts at that device instead (for example a Sonos integration, or another general `media_player`).

**The key point**: in all of these cases, `wall-e-controller.yaml` and the filtering logic in the firmware stay exactly the same. You only need to edit the two Home Assistant scripts and point them at the right `media_player` entity (or integration) for your TV.

## Print Files (STL)

The STL files for the Wall-E enclosure haven't been uploaded to this repository yet — they'll be added later under an `stl/` directory.

## Security and Secrets

The original `secrets.yaml` file (including the WiFi password, API encryption key, OTA password, and AP password) is **not** included in this repository. It's been replaced with `!secret` references inside `wall-e-controller.yaml`, and `secrets.yaml.example` contains only placeholder values. Anyone who clones this project needs to create their own `secrets.yaml` with their real values, and make sure that file is never pushed to git (it's already blocked by `.gitignore`).
