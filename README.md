# HealthNote OS - Privacy-First Health Tracking on Nostr

<p align="center">
  <img src="https://raw.githubusercontent.com/HealthNoteLabs/HealthNote-OS/master/assets/healthnote_logo_banner.png" alt="HealthNote OS Banner" width="600"/>
</p>

**HealthNote OS** is a fork of the esteemed [Espruino](https://www.espruino.com/) JavaScript interpreter, repurposed and streamlined for a singular mission: **to create a privacy-preserving, Nostr-native firmware for health and fitness tracking.**

Our vision is to empower individuals with full ownership and control over their sensitive health data. By leveraging the decentralized nature of Nostr and the robust NIP-101h family of specifications, HealthNote OS aims to provide a secure and transparent way to monitor personal well-being without compromising on privacy.

## Core Goals & Philosophy

*   **Privacy by Design:** All sensitive health metrics are encrypted on-device (NIP-44) before being transmitted.
*   **Nostr Native:** Data is formatted and signed as NIP-101h events, ready for publishing to any Nostr relay or specialized Blossom servers.
*   **User Sovereignty:** Utilizes NIP-26 delegation, allowing the device to publish on behalf of the user's Nostr identity without needing direct access to private keys post-pairing.
*   **Open & Transparent:** Built on the open-source Espruino foundation, encouraging community audit and contribution.
*   **Focused & Lightweight:** Stripped of non-essential features to maximize resources for core health tracking and Nostr integration.

## Current Status & Roadmap (v0.1 "Genesis")

HealthNote OS is currently focused on delivering a minimal viable product (MVP) centered around:

1.  **Core Metrics:** Reliable collection of heart rate, step count, and battery level.
2.  **Secure Storage & Sync:** On-device buffering of encrypted, signed NIP-101h events.
3.  **BLE Hand-off:** A dedicated GATT service for securely transferring data to a companion phone application.
4.  **Minimal UI:** Basic on-watch display of key metrics and sync status.

For a detailed breakdown of features, technical architecture, and development progress, please see our **[HealthNote Implementation Guide](HealthNote-Implementation-Guide.md)**.

## Building HealthNote OS

HealthNote OS inherits its build system from Espruino. Refer to the original [Espruino Build Process](README_BuildProcess.md) and [Building Espruino](README_Building.md) documents for general guidance.

To build for the reference hardware (Bangle.js 2 equivalent):

```bash
# Ensure ARM GCC toolchain and make are in your PATH
make clean
make BOARD=BANGLEJS2 # This will produce espruino_HEALTHNOTE_OS-0.1_banglejs2.hex
```

*(Note: The `BOARD=BANGLEJS2` definition now points to HealthNote OS specific configurations within `boards/BANGLEJS2.py`)*

## Contributing

We welcome contributions from developers who share our passion for privacy, health, and decentralized technologies. Whether it's through code, documentation, testing, or NIP proposals, your input is valuable. Please review the [HealthNote Implementation Guide](HealthNote-Implementation-Guide.md) and the original [Espruino CONTRIBUTING.md](CONTRIBUTING.md) to get started.

## Acknowledgements

HealthNote OS stands on the shoulders of giants, primarily the Espruino project and its vibrant community. We are immensely grateful for their pioneering work in bringing JavaScript to microcontrollers. Our goal is to extend this powerful platform into the burgeoning world of decentralized personal data.

## License

HealthNote OS is distributed under the Mozilla Public License v2.0, the same as Espruino. Please see the [LICENSE](LICENSE) file for details.
