# Specialized Security Domains Lab: IoT, Mobile, and ICS

A hands-on defensive security assessment across three domains — Internet of Things (IoT), Mobile, and Industrial Control Systems (ICS) — performed against intentionally vulnerable training targets and simulated devices in an isolated lab environment.

## Executive Summary

The assessment produced eight documented findings across three domains. The highest-impact results:

- **Critical (IoT):** A default credential for the training firmware's `iotgoatuser` account was recovered in under one second from the extracted `/etc/shadow` using the leaked Mirai botnet wordlist, demonstrating why firmware-wide default credentials remain the dominant IoT compromise vector.
- **High (IoT):** The firmware uses MD5-crypt password hashing, and MQTT traffic (including door-lock state) is readable in plain text until TLS and authentication are enabled.
- **High (ICS):** Modbus write commands are accepted with zero authentication or encryption; only the honeypot's internal memory map prevented a successful write.
- **High (ICS, tooling):** Docker Desktop for macOS silently fails to enforce loopback-only binding on port-remapped publishes. A control believed in place (network isolation) was absent for two of three published ports and was only discovered through independent external verification. See [Finding ICS-01](docs/03-ics.md#finding-ics-01-docker-desktop-for-macos-fails-to-enforce-loopback-only-binding-on-port-remapped-publishes).

Full findings tables: [notes/](notes/). Detailed methodology and evidence: [docs/](docs/).

## Rules of Engagement

- Only systems owned or explicitly authorized for testing were used.
- Shodan was used passively only: searching and reading summary statistics, with no login, scanning, or interaction with any listed device.
- All honeypots and vulnerable applications were kept isolated from the internet, bound to loopback or a host-only network.

## Lab Environment

| Component | Details |
|---|---|
| Host | macOS, Docker Desktop |
| VM / secondary host | Kali Linux (used for cross-machine network exposure testing) |
| IoT target | OWASP IoTGoat firmware (Raspberry Pi 2 image) |
| IoT protocol target | Eclipse Mosquitto MQTT broker |
| Mobile targets | DIVA (Damn Insecure and Vulnerable App), OWASP MASTG UnCrackable Level 1 |
| Mobile emulator | Android Studio AVD, Pixel device, Google APIs system image |
| ICS target | Conpot honeypot (simulated Siemens S7-300 PLC), Docker image `honeynet/conpot` |

Setup and tooling reference: [docs/setup.md](docs/setup.md).

## Status Summary

| # | Task | Status |
|---|---|---|
| 1 | Shodan statistics recorded (passive only) | Complete |
| 2 | Firmware extracted and findings listed | Complete |
| 3 | MQTT traffic captured in plain text | Complete |
| 4 | MQTT secured with authentication and TLS | Complete |
| 5 | MobSF report generated and reviewed | Complete |
| 6 | Frida server running and Objection connected | Complete |
| 7 | Drozer attack surface listed | Complete |
| 8 | Conpot running and scanned | Complete |
| 9 | Modbus traffic captured and write filter created | Complete |
| 10 | Zone and conduit table completed | Complete |

## Documentation

| Section | Contents |
|---|---|
| [IoT Security](docs/01-iot.md) | Shodan exposure assessment, IoTGoat firmware analysis, MQTT testing and controls |
| [Mobile Security](docs/02-mobile.md) | MobSF static analysis, Frida/Objection dynamic testing, Drozer component audit, MASVS mapping |
| [ICS Security](docs/03-ics.md) | Conpot deployment, Modbus/S7comm analysis, Docker Desktop binding finding, IEC 62443 controls, NIST SP 800-82 assessment |
| [Setup Reference](docs/setup.md) | Environment preparation and tool verification |
| [Screenshot Inventory](docs/screenshot-inventory.md) | Original-to-target filename mapping for all evidence |

## Findings Index

| ID | Domain | Title | Severity |
|---|---|---|---|
| IOT-01 | IoT | Weak password hashing algorithm (MD5-crypt) | High |
| IOT-02 | IoT | Default credential recovered (Mirai wordlist) | Critical |
| IOT-03 | IoT | Hardcoded secrets, configuration files, and startup services | TBD |
| IOT-04 | IoT | MQTT broker accepts unauthenticated, unencrypted connections by default | High |
| MOB-01..n | Mobile | See [mobile findings](notes/mobile-findings.md) | TBD |
| ICS-01 | ICS | Docker Desktop for macOS fails to enforce loopback-only binding on port-remapped publishes | High |
| ICS-02 | ICS | Modbus has no authentication or encryption; write commands processed without credentials | High |

## Repository Structure

```text
security-domains-lab/
├── README.md            (this file: executive summary and index)
├── .gitignore
├── docs/                (detailed methodology and evidence per domain)
├── notes/               (findings tables in report-template format)
├── screenshots/         (numbered evidence images per domain)
├── configs/             (secure.conf: Mosquitto TLS/auth template, no secrets)
└── report/              (final consolidated report)
```

## Notes on Evidence Handling

- No firmware images, APK files, password files, or private keys are committed to this repository (see `.gitignore`).
- Public IP addresses and account credentials have been excluded or blurred in all screenshots.
- Placeholder fields marked `[fill in]` indicate values that depend on the specific screenshot content and should be completed with the exact figures observed during testing.
