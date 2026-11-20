# axs-battery-check — agent guide

This repository starts from [tarekrached/esphome-sram-axs](https://github.com/tarekrached/esphome-sram-axs), a working ESPHome external component and SRAM AXS protocol investigation. The project direction is to grow it into a standalone, AC-powered monitor for battery readiness across two bicycles. Home Assistant and cloud connectivity must not be required. Read this guide before changing code; do not treat proposed architecture or unverified device behavior as implemented fact.

## Project Goals

- Provide a quick pre-ride view of the battery condition of components on both bikes, with clear per-bike and whole-system readiness.
- Retain the last valid battery reading and its timestamp. Distinguish fresh, stale, unavailable, unknown, and low states; device activity alone is not a fresh battery reading.
- Identify devices reliably using evidence-backed identifiers and group them by bike.
- Support SRAM AXS advertisements, standard GATT Battery Service where actually available, and manufacturer-specific reporting when necessary. Do not assume every device reports battery data or supports a standard service.
- Do not present an estimate as a measured percentage. Some devices, especially coin-cell devices, may only support a qualitative condition or voltage-based warning.
- Eventually provide a local HTTP dashboard/API and support a small physical display with automatic activity-driven wake, manual wake, and configurable timeout.
- Keep the ESP32 continuously operational for BLE monitoring while any eventual display is off. No Home Assistant, cloud service, or premature hardware purchase is a requirement.

## Bikes And Inventory

The requested initial inventory is eight components. Treat these names as configuration/profile data, not protocol identities.

| Bike | Components |
|---|---|
| **Dean** | SRAM AXS rear derailleur; left and right SRAM AXS shifters; Sigeyi AXO SRAM Road Power Meter |
| **Zinn** | SRAM AXS rear derailleur; left and right SRAM AXS shifters; Wahoo RPM wheel-mounted speed sensor |

Neither bike has a front derailleur. The architecture should leave room for a Bluetooth heart-rate monitor, whose model is not selected. The Sigeyi and Wahoo battery-reporting capabilities and protocols have not been established in this repository; investigate them before promising a reading or implementing a driver.

## Hardware And Runtime

The intended initial development board is an M5Stack ATOM Lite (grey), specified as an original ESP32 with 4 MB flash, BLE, 2.4 GHz Wi-Fi, USB-C power, integrated button, and RGB LED. It is intended to run continuously from a USB wall adapter. Verify the actual board, firmware/runtime support, and available memory before relying on board-specific details.

No display model or display wake mechanism has been selected. Do not choose one on the user's behalf. An HTML/JavaScript dashboard may require a separate browser-capable display device; do not assume an ESP32 can render a web page on a simple SPI/I2C panel or that JavaScript alone can power a display on. Treat display hardware, interface, and power control as unresolved.

The existing application is ESPHome/C++ with a Python ESPHome configuration module and a CPython macOS scanner using Bleak. The long-term preference is to investigate MicroPython as the embedded application language, not to assume a migration is approved or feasible. Check RAM, BLE scanning/connection behavior, Wi-Fi coexistence, async support, HTTP serving, and dependency compatibility on the ATOM Lite before deciding. CPython packages do not automatically run under MicroPython.

## Existing Implementation And Boundaries

- `components/sram_axs/` is the current working ESPHome external component. It passively listens through `esp32_ble_tracker`, matches SRAM devices by serial, decodes supported SRAM advertisement fields, publishes optional sensors, and can restore last-known state from NVS.
- The current listener does not connect over GATT. It is not a multi-bike registry/readiness engine and does not implement the proposed standalone API, dashboard, or display wake/control.
- `example.yaml` is the ESPHome example and must remain consistent with the README quickstart when its configuration changes. It may use ESPHome/Home Assistant facilities; that does not make Home Assistant a requirement for the target appliance.
- Preserve the working ESPHome baseline while investigating alternatives. Do not replace its architecture or discard existing behavior as a preliminary step. Make any later port incremental and compare behavior against the baseline.
- Keep changes within the existing repository's conventions and avoid introducing a new runtime or dependency before a small feasibility check demonstrates its value.

## Protocol Evidence And Bluetooth Rules

- **`facts.yaml` is the source of truth for SRAM AXS protocol numbers and confidence.** Component offsets, UUIDs, constants, and field interpretations must agree with it. Keep confirmed observations distinct from hypotheses; never promote a hypothesis without supporting capture evidence.
- **`docs/protocol.md` is the human-readable SRAM teardown.** Keep it consistent with `facts.yaml`, including uncertainty and caveats, in the same change when protocol knowledge changes.
- Apply the same evidence discipline to future device drivers: investigate and document the actual protocol before implementation. Do not invent manufacturer behavior or infer an exact percentage from voltage or qualitative status.
- Malformed, incomplete, or unknown packets must be handled safely. Bounds-check before reading fields; missing data is not zero, and an unavailable reading is not a low battery.
- Prefer passive scanning and avoid unnecessary connections. Use GATT only when evidence shows it is needed to obtain battery information. Document connection duration, battery cost, possible interference with bike computers/apps, and concurrency limits before adopting a connection-based approach.
- Do not assume an advertisement, movement, or a button press guarantees a fresh battery reading. A wake/activity event and a valid battery update are separate state transitions.
- Avoid treating repeated advertisements from an already-active component as new bicycle activity. Activity detection and display wake should be based on deliberate transitions or a documented debounce/window, with tests for repeated packets.
- Use identifiers appropriate to each protocol. Do not assume BLE addresses are stable, visible, or sufficient; macOS CoreBluetooth uses opaque addresses, and device identifiers may differ by stack.
- Raw captures stay out of version control (`captures/` is intended to be ignored); they can contain serials and other identifying details. Put masked excerpts and conclusions in protocol documentation. A decode correction should include the relevant masked evidence.
- The existing macOS `tools/mac-scan.py` uses Bleak and is a discovery/feasibility tool; its CoreBluetooth output may coalesce or rate-limit data. `tools/pi-capture.sh`/`btmon` is the documented lower-level capture path. Do not treat either tool's presentation as universal BLE framing.

## Proposed Architecture And Roadmap

Favor modules with clear responsibilities, but treat the following as logical boundaries, not a mandatory file layout:

- BLE scan/discovery and, only where needed, connection management
- Independent protocol decoders for SRAM AXS, Sigeyi AXO, Wahoo RPM, and future sensors
- Device registry and configurable bicycle profiles
- Normalized battery readings, status, timestamps, freshness, and activity state
- Per-bike and system readiness evaluation with configurable thresholds by device type
- Configuration persistence, local HTTP/JSON API, browser dashboard, and button/LED/display controls

Keep protocol decoding separate from ESP32-specific radio operations where practical so packet parsing and status logic can be tested on a Mac. Keep hardware-specific code isolated. Do not build abstractions before there are multiple concrete consumers.

Suggested incremental order:

1. Understand the repository and establish the ESPHome baseline on the ATOM Lite.
2. Verify the existing SRAM decoder against the documented protocol and captures.
3. Experiment with Python BLE scanning on macOS and build reusable, unit-tested protocol decoding logic.
4. Evaluate MicroPython on the target board for memory, BLE, Wi-Fi, async, and HTTP requirements.
5. Port incrementally only if MicroPython proves suitable; preserve a usable ESPHome path during evaluation.
6. Investigate and add Sigeyi AXO and Wahoo RPM support based on actual observed protocols and battery data.
7. Add device profiles, multi-bike status, normalized freshness, thresholds, and readiness aggregation.
8. Add the standalone HTTP API and mobile-friendly dashboard; keep API paths such as `/api/bikes`, `/api/batteries`, and `/api/activity` provisional until designed.
9. Add activity transitions and display wake events, then integrate display hardware after it is selected.

This is a proposal, not a mandate to rewrite or implement all stages in one change.

## Development And Coding Conventions

- Preferred environment: macOS, Visual Studio Code, OpenAI Codex, Git, and Python virtual environments.
- Prefer clear, maintainable Python; use type hints where supported and practical, descriptive names, and small focused functions. Explain unfamiliar Python idioms when introducing them.
- Keep CPython development/test code distinct from MicroPython firmware code when their runtimes or dependencies differ. Avoid assuming CPython libraries work on-device.
- Prefer asynchronous, nonblocking BLE and network work. Do not block scanning while serving HTTP requests or updating a display.
- Minimize dependencies and persistent flash writes. Do not put Wi-Fi credentials or other secrets in source control.
- Use explicit state models for measured value, qualitative status, freshness, availability, activity, and readiness. Keep unavailable/unknown/stale distinct from low.
- Thresholds must be configurable by device type where appropriate and supported by documented evidence. Clearly label empirically chosen guidance; do not imply manufacturer endorsement.
- Follow the style of the touched language and neighboring code. Existing C++ uses ESPHome conventions; do not introduce unrelated formatting or refactors.
- Prefer readable code over clever abstractions. Keep the existing ESPHome config/documentation and the README quickstart in sync when changing user-facing configuration.

## Testing And Validation

Prioritize tests that run on the development Mac without the ESP32. Use pytest where appropriate, especially for reusable CPython protocol and readiness logic. Add focused tests for:

- SRAM packet parsing and device identification
- Battery status/threshold interpretation without fabricated percentages
- Reading freshness and stale/unavailable distinction
- Bike and system readiness aggregation
- Activity transitions, repeated-advertisement debounce, and wake-event generation
- Malformed, truncated, and incomplete BLE data

Use synthetic packets for edge cases and appropriately captured, masked packets for observed protocol behavior. A passing host test does not validate radio behavior, MicroPython compatibility, memory fit, or hardware operation; identify which layer was actually verified. Preserve existing ESPHome validation and hardware workflow when modifying that implementation. Do not claim a test suite or build command exists unless it has been confirmed in the repository.

## Repository Map

| Path | Responsibility |
|---|---|
| `components/sram_axs/` | Existing ESPHome SRAM AXS component (Python config plus C++ listener/decoder) |
| `facts.yaml` | SRAM AXS protocol constants, observations, and confidence notes |
| `docs/protocol.md` | Human-readable SRAM AXS advertisement teardown and capture guidance |
| `tools/mac-scan.py` | macOS BLE discovery scanner using Bleak |
| `tools/pi-capture.sh` | Linux/Pi BLE capture helper using `btmon` |
| `example.yaml` | Worked ESPHome configuration |
| `README.md` | User-facing installation, setup, and project overview |

## Prior Art

[This project is based on tarekrached/esphome-sram-axs](https://github.com/tarekrached/esphome-sram-axs); its existing ESPHome component and SRAM AXS protocol investigation are the working baseline and should remain credited as the project evolves.

[ShannoG/home-assistant-sram-axs](https://github.com/ShannoG/home-assistant-sram-axs) is a passive HA-side advertisement decoder; [karl-petter/sram-axs-for-ha](https://github.com/karl-petter/sram-axs-for-ha) uses GATT reads. They are useful adjacent references, not proof that all devices expose the same fields or connection behavior. Credit relevant work, compare evidence carefully, and report findings that affect their decodes upstream when useful.

### Hardware

I am building for a M5Stack Official ATOM Lite Compact ESP32 IoT Development Kit
