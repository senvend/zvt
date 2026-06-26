# SENVEND ZVT Integration Guide

> **Note:** ZVT Proxy Mode is a new feature and the API may not be fully stabilized yet.
> If you encounter any issues or have suggestions, please [let us know](#help-us-improve).

## Definitions

| Term | Description |
|------|-------------|
| **ECR** | Electronic Cash Register — used throughout this document as a general term for any client connecting to the proxy, whether it is a POS system, vending machine, or traditional cash register. |
| **PT** | Payment Terminal — the SENVEND payment terminal that processes card payments. |
| **Proxy** | The SENVEND ZVT Proxy — a TCP server on the SENVEND device that accepts ZVT connections from an external ECR and forwards allowed commands to the PT. |
| **APDU** | Application Protocol Data Unit — a single ZVT message on the wire, consisting of a class byte, instruction byte, and payload. |

## Overview

ZVT Proxy Mode allows an external ECR to communicate with a payment terminal through the SENVEND middleware via standard [ZVT protocol][zvt-spec]. Instead of talking directly to the payment terminal, the ECR connects to a TCP server exposed by SENVEND, which inspects, validates, and forwards ZVT commands to the terminal.

This enables third-party systems to issue ZVT commands directly while SENVEND retains the ability to monitor, manage, and extend the payment flow — including telemetry, age verification, and terminal lifecycle management.

The proxy speaks **native ZVT over TCP** — no proprietary framing or translation layer. Any ECR implementation that follows the [ZVT specification][zvt-spec] can connect to the proxy as if it were a payment terminal.

## Architecture

```
                   +-----------------------------------------------+
                   |            SENVEND device (physical)          |
                   |                                               |
  +-------+  TCP   |   +---------------------+   TCP   +------+    |
  |  ECR  |<------>|   |   SENVEND Proxy     |<------->|  PT  |    |
  |       | :20000 |   |   (ZVT Proxy Mode)  |         |      |    |
  +-------+        |   +---------------------+         +------+    |
                   |                                               |
                   +-----------------------------------------------+
```

The proxy sits between the ECR and the PT on the same device. Both connections use TCP with the standard ZVT wire format.

- **ECR → Proxy** — The ECR connects to the proxy's TCP port (default `20000`). This is the only endpoint the ECR needs to know about.
- **Proxy → PT** — The proxy maintains a TCP connection to the payment terminal via `localhost`.

### Key characteristics

- **Single client** — The proxy accepts one ECR connection at a time. A second client will be accepted only after the first disconnects.
- **Transparent protocol** — The proxy does not alter the ZVT wire format. Commands and responses are forwarded as-is, with the exception of [Age Verification](#age-verification).
- **Command filtering** — Not all ZVT commands are allowed through the proxy. Certain commands (e.g. Initialisation, Reset Terminal, Change Password) are rejected with an Abort response. See [Command Handling](#command-handling) for details.
- **Standard ZVT flow** — The ACK / response exchange follows the [ZVT specification][zvt-spec] exactly. The ECR sends a command, receives an ACK (or error), then receives intermediate status and a terminal response (Completion or Abort), acknowledging each step as usual.

## Command Handling

The proxy classifies every incoming APDU by its command tag (class + instruction byte) and applies one of three policies:

### Forwarded commands

These commands are forwarded to the PT and their responses relayed back to the ECR. Some commands have their responses parsed by SENVEND for monitoring purposes, but the raw bytes are always forwarded transparently.

| Command | Tag | ZVT Spec |
|---------|-----|----------|
| Authorization | `06 01` | 2.2 |
| Pre-Authorisation / Reservation | `06 22` | 2.8 |
| Book Total | `06 24` | 2.13 |
| Partial Reversal | `06 23` | 2.10 |
| Pre-Authorisation Reversal | `06 25` | 2.14 |
| Refund | `06 31` | 2.15 |
| Diagnosis | `06 70` | 2.18 |
| Print System Configuration | `06 1A` | 2.47 |
| Read Card | `06 C0` | 2.22 |
| Activate Service-Mode | `08 01` | 2.57 |
| Abort | `06 B0` | 2.24 |

### Conditionally forwarded commands

These commands are only allowed when the corresponding feature is **delegated to the ECR** (i.e. not managed by SENVEND). By default, when proxy mode is enabled, Registration and End-of-Day are delegated to the ECR. See [Configuration](#configuration) for details.

| Command | Tag | ZVT Spec | Requires |
|---------|-----|----------|----------|
| Registration | `06 00` | 2.1 | Registration delegated to ECR |
| Log-Off | `06 02` | 2.25 | Registration delegated to ECR |
| End-of-Day | `06 50` | 2.16 | End-of-Day delegated to ECR |

### Denied commands

All commands not listed above are rejected immediately with "function not possible" (`84 83`). This includes but is not limited to:

| Command | Tag | ZVT Spec | Note |
|---------|-----|----------|------|
| Initialisation | `06 93` | 2.19 | Terminal-ID is managed by SENVEND. |
| Set Terminal-ID | `06 1B` | 2.48 | Terminal-ID is managed by SENVEND. |
| Reset Terminal | `06 18` | 2.46 | |
| Change Password | `06 95` | 2.51 | |
| Select Language | `08 30` | 2.38 | Not yet supported by Verifone. Proxy support planned for Q2 2026. |

Any unknown or unrecognized command tag is also denied.

## Age Verification

When an Authorization (`06 01`) or Reservation (`06 22`) command contains an Age verification control TLV (tag `1F6B`, see [ZVT spec][zvt-spec] section 9.4.2), the proxy intercepts the command and triggers an age verification flow on the SENVEND device before forwarding it to the PT.

### Flow from the ECR's perspective

1. **ECR sends** Authorization or Reservation with Age verification control TLV (e.g. `18` for age 18+).
1. **Proxy responds** with an immediate ACK (`80 00`).
1. **Age verification** takes place on the SENVEND device. This may take up to several minutes depending on the verification method. During this period, the proxy may send Intermediate Status Information (`04 FF`) messages to inform the ECR about the verification progress — see [Intermediate status during age verification](#intermediate-status-during-age-verification) below.
1. **On approval:** The command is forwarded to the PT. The normal ZVT response exchange proceeds — the ECR receives Intermediate Status Information, Status Information, and Completion as usual. The Age verification result TLV (tag `1F6C`) is injected into the Status Information with value `0x01` (minimum age reached).
1. **On denial or timeout:** The ECR receives an Abort (`06 1E`).

### Intermediate status during age verification

During the age verification period, the proxy may send Intermediate Status Information (`04 FF`) messages to keep the ECR informed about the progress. These look identical to regular intermediate status messages from the PT.

Each message contains:

- **Status byte:** "Please wait" (`0x0E`)
- **Text line** — a human-readable message describing the event
- **Age verification result** (TLV tag `1F6C`) — a machine-readable result code

| Event | Text | `1F6C` value |
|-------|------|--------------|
| Success | `Age verified, starting payment...` | `0x01` (minimum age reached) |
| Retry | `Age verification failed, please retry` | `0x00` (minimum age not reached) |

The full list of defined `1F6C` values is:

| Value | Meaning |
|-------|---------|
| `0x00` | minimum age not reached |
| `0x01` | minimum age reached |
| `0x02` | customer card does not support age verification |
| `0x03` | verification not supported |

> **Note:** The proxy currently only sends `0x00` and `0x01`. `0x00` is used for any unsuccessful verification attempt (including card read errors or user cancellations on the SENVEND device) — it does not necessarily mean the customer is underage. A future release will reflect the SENVEND contract status and may also return `0x02` (customer card does not support age verification).

**ECR behavior:**

- The ECR **must ACK** each Intermediate Status Information with a standard PT ACK (`80 00`), as with any ZVT intermediate status message.
- The "retry" message is **informational only**. It indicates that a verification attempt did not succeed, but the verification session remains active and the user can try again on the SENVEND device.
- The ECR **may** abort the transaction at this point by sending an Abort (`06 B0`), but we **recommend against it**. Let the user retry on the SENVEND device; use the message for display or logging purposes only.
- The session ends naturally when the user succeeds, aborts on the SENVEND device, or the timeout expires.
- ECRs should rely on the `1F6C` TLV value, not the text. The text is currently English only; localized messages tied to the PT's configured language (Select Language) are planned for a future release.

**When these messages are sent:**

By default, the proxy only sends these messages if the ECR requested intermediate status information during its Registration by setting the "ECR requires intermediate status-Information" bit in the `<config-byte>` (see [ZVT spec][zvt-spec] section 2.1). This matches standard ZVT behavior.

SENVEND can override this behavior per device via the [age verification status policy](#age-verification-status-policy). Contact SENVEND to adjust this setting.

The ECR's intermediate status preference persists across TCP client disconnects. Once the ECR has registered with the proxy (setting the "ECR requires intermediate status-Information" bit in its `<config-byte>`), the proxy remembers this preference until an explicit Log-Off (`06 02`) or a subsequent Registration (`06 00`) overwrites it. This matches real PT behavior: if the ECR's TCP connection drops and it reconnects without re-registering, the proxy continues to honor the previously registered setting.

### Important notes for integrators

- The ECR receives the ACK immediately, but the first PT response may be delayed while age verification is in progress. Adjust any response timeouts accordingly.
- The ECR can send an Abort (`06 B0`) during the verification period to cancel the transaction.
- On a successful payment, the Status Information response will contain Age verification result (tag `1F6C`) with value `0x01` — this is injected by the proxy and was not sent by the PT.
- To receive progress updates during age verification, the ECR should set the "ECR requires intermediate status-Information" bit in its Registration `<config-byte>`.

## Configuration

ZVT Proxy Mode is currently configured by SENVEND. To enable proxy mode on your device, please contact us at [api@senvend.com](mailto:api@senvend.com). Self-service configuration via [my.senvend.com](https://my.senvend.com) is planned for the future.

### Proxy server

| Option | Description | Default |
|--------|-------------|---------|
| Enable | Enable or disable the proxy TCP server. | Disabled |
| Port | TCP port the proxy listens on for incoming ECR connections. | `20000` |

### Feature delegation

When proxy mode is enabled, certain features are **delegated to the ECR** by default — meaning the ECR is responsible for managing them via the corresponding ZVT commands. SENVEND can override these defaults per device.

| Feature | Default (proxy enabled) | Effect when delegated to ECR |
|---------|------------------------|------------------------------|
| Registration | Delegated to ECR | ECR must send Registration (`06 00`). |
| End-of-Day | Delegated to ECR | ECR must send End-of-Day (`06 50`). |

When a feature is **not** delegated to the ECR, SENVEND manages it automatically and the corresponding commands are rejected by the proxy.

### Age verification status policy

Controls whether the proxy sends Intermediate Status Information during age verification. See [Intermediate status during age verification](#intermediate-status-during-age-verification) in the Age Verification section for what these messages contain.

| Policy | Description |
|--------|-------------|
| `Enabled` | Always send status updates during age verification. |
| `Disabled` | Never send status updates during age verification. |
| `Registration` (default) | Send only if the ECR requested intermediate status information in its Registration `<config-byte>`. |

## CLI Testing Tool

`zvt_cli` is a small command-line tool for talking to the [SENVEND ZVT Proxy](#overview) — or any payment terminal — over ZVT/TCP. It is handy during integration for sending commands (Authorization, Reservation, Read Card, …) by hand and inspecting the responses.

Prebuilt binaries are published on the [Releases page](https://github.com/senvend/zvt/releases):

| Platform | Asset |
|----------|-------|
| Linux (x86_64) | `zvt_cli-<version>-linux-x86_64` |
| Linux (aarch64) | `zvt_cli-<version>-linux-aarch64` |
| Windows (x86_64) | `zvt_cli-<version>-windows-x86_64.exe` |

Run `zvt_cli --help` to list the available commands and options.

> **Note:** These are experimental testing builds intended for development and integration only — not for production use.

## Troubleshooting

### Why does the PT send dial-up commands (`06 D8` / `06 D9` / `06 DA` / `06 DB`)?

Because your ECR advertised them. During Registration (`06 00`), the permitted-commands list (TLV tag `26`) tells the PT which PT→ECR commands the ECR is willing to receive. If that list includes the dial-up / DFÜ commands, the PT assumes the ECR provides a dial-up channel and will route host communication through the ECR instead of using its own connection:

| Command | Tag | ZVT Spec |
|---------|-----|----------|
| Dial-Up | `06 D8` | 3.8 |
| Transmit Data via Dial-Up | `06 D9` | 3.10 |
| Receive Data via Dial-Up | `06 DA` | 3.11 |
| Hang-Up | `06 DB` | 3.9 |

Most ECRs do not implement dial-up forwarding, so these sessions fail and the transaction ends with result code `9B`. **Fix:** remove the dial-up tags from the tag `26` permitted-commands container in your Registration so the PT keeps using its own host connection.

## Help Us Improve

Feel free to open issues if something is missing, unclear or not working as expected.
You are also welcome to contribute directly by opening Pull Requests.
If you need any help during your integration and you want to handle it confidentially, please reach out to us via api@senvend.com.

[zvt-spec]: https://www.terminalhersteller.de/downloads.aspx
