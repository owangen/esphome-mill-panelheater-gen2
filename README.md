# ESPHome Mill Panel Heater Gen2

An external ESPHome component for Mill panel heaters that use the second-generation UART protocol.

This component was developed by reverse-engineering the serial traffic of a Mill Gen2 panel heater. The protocol is undocumented by the manufacturer, so compatibility with other models or firmware versions is not guaranteed.

## Installation

Add the repository as an external component in your ESPHome configuration:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/owangen/esphome-mill-panelheater-gen2
      ref: main
    components: [mill_panelheater_gen2]
```

Pin the `ref` to a release tag or commit when reproducible builds are important.

## UART requirements

The component requires a two-way UART connection and validates these settings:

- 9600 baud
- 8 data bits
- No parity
- 1 stop bit
- Both RX and TX connected

Example:

```yaml
uart:
  id: uart_bus
  tx_pin: GPIO17  # Change for your board
  rx_pin: GPIO16  # Change for your board
  baud_rate: 9600
  data_bits: 8
  parity: NONE
  stop_bits: 1
```

The GPIO pins and any required level shifting or isolation depend on the heater and ESP board. Verify the electrical interface before connecting anything.

## Minimal configuration

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/owangen/esphome-mill-panelheater-gen2
      ref: main
    components: [mill_panelheater_gen2]

uart:
  id: uart_bus
  tx_pin: GPIO17
  rx_pin: GPIO16
  baud_rate: 9600
  data_bits: 8
  parity: NONE
  stop_bits: 1

climate:
  - platform: mill_panelheater_gen2
    name: Mill Panel Heater
    uart_id: uart_bus
```

## Estimated power sensor

The heater protocol reports whether the heater is heating, but it does not provide a measured wattage. To expose an estimated power sensor, configure both `rated_power` and `power`:

```yaml
climate:
  - platform: mill_panelheater_gen2
    name: Mill Panel Heater
    uart_id: uart_bus
    rated_power: 900
    power:
      name: Mill Panel Heater Estimated Power
```

The sensor publishes `rated_power` while the reported action is heating and `0 W` while idle or off. It is an estimate, not an electrical measurement. Do not use it for protection, billing, or safety functions.

## Supported features

- Climate modes: `OFF` and `HEAT`
- Target temperature control from 5 °C through 35 °C
- Current temperature reporting
- Heating/idle/off climate action reporting
- Power-on and power-off commands
- Estimated power sensor using a configured rated power
- Communication timeout warning when no valid status frame is received for 150 seconds

## Known limitations

- `restore_mode` is not implemented yet.
- Temperatures are handled at one-degree protocol resolution.
- The power sensor is estimated from the configured rated power; there is no real-time power measurement.
- The implementation supports the observed Gen2 protocol frames only. Other Mill models, hardware revisions, or firmware versions may use different frames.
- Commands are sent and the component waits for a status frame to confirm the resulting state.
- This is not a safety controller and does not provide a hardware disconnect or independent over-temperature protection.

## Reverse-engineering notes

The following observations are based on captured traffic and are not an official Mill protocol specification:

- Serial settings are 9600 8N1.
- Frames start with `0x5A` and end with `0x5B`.
- The declared frame length is carried in the frame and is used to delimit reception.
- Status frames use command type `0xC9` and are 17 bytes including framing.
- Status data contains target temperature, current temperature, mode, and action fields.
- Power and temperature commands use observed payload types `0x06` and `0x22` respectively.
- The checksum is the 8-bit sum of the command/status payload bytes before the checksum field.

The implementation rejects invalid lengths, unsupported status values, invalid checksums, and an incorrect frame terminator. The test suite documents the observed frames in executable form.

## Tests

The existing ESPHome tests are retained under `tests/components/mill_panelheater_gen2`. They cover command framing, status parsing, checksum and terminator validation, recovery from incomplete frames, temperature limits, state confirmation, and estimated-power updates. They are intended to run with ESPHome's native component test harness.

## Electrical safety warning

Mill panel heaters are mains-powered appliances. Do not connect an ESP GPIO, USB-UART adapter, or other low-voltage electronics to mains wiring or heater terminals. Only connect to a confirmed low-voltage serial interface using the correct voltage levels, isolation, and wiring for the specific heater. Disconnect mains power before opening the appliance. Installation and modification should be performed by a qualified electrician or suitably competent person. You are responsible for safe installation, damage prevention, and compliance with local electrical rules.

## License

This project is provided under the MIT License. See [LICENSE](LICENSE).
