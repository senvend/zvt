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
                   |                SENVEND Device                 |
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
- **Transparent protocol** — The proxy does not alter the ZVT wire format. Commands and responses are forwarded as-is, with the exception of age verification (see [Age Verification](#age-verification)).
- **Command filtering** — Not all ZVT commands are allowed through the proxy. Certain commands (e.g. Initialization, ResetTerminal, ChangePassword) are rejected with an Abort response. See [Command Handling](#command-handling) for details.
- **Standard ZVT flow** — The ACK / response exchange follows the [ZVT specification][zvt-spec] exactly. The ECR sends a command, receives an ACK (or error), then receives intermediate status and a terminal response (Completion or Abort), acknowledging each step as usual.

## Command Handling

The proxy classifies every incoming APDU by its command tag (class + instruction byte) and applies one of three policies:

### Forwarded commands

These commands are forwarded to the PT and their responses relayed back to the ECR. Some commands have their responses parsed by SENVEND for monitoring purposes, but the raw bytes are always forwarded transparently.

| Command | Tag | ZVT Spec |
|---------|-----|----------|
| Authorization | `06 01` | 2.2 |
| Pre-Authorization / Reservation | `06 22` | 2.8 |
| Partial Reversal | `06 23` | 2.10 |
| Pre-Auth Reversal | `06 25` | 2.11 |
| Diagnosis | `06 70` | 2.28 |
| Print System Configuration | `06 1A` | 2.38 |
| Read Card | `06 C0` | 2.40 |
| Activate Service Mode | `08 01` | 2.56 |
| Abort | `06 B0` | 2.12 |

### Conditionally forwarded commands

These commands are only allowed when the corresponding feature is **delegated to the ECR** (i.e. not managed by SENVEND). By default, when proxy mode is enabled, Registration and End-of-Day are delegated to the ECR. See [Configuration](#configuration) for details.

| Command | Tag | ZVT Spec | Requires |
|---------|-----|----------|----------|
| Registration | `06 00` | 2.1 | Registration delegated to ECR |
| Log-Off | `06 02` | 2.3 | Registration delegated to ECR |
| End-of-Day | `06 50` | 2.6 | End-of-Day delegated to ECR |

### Denied commands

All commands not listed above are rejected immediately with `FunctionNotPossible` (`84 83`). This includes but is not limited to:

| Command | Tag | ZVT Spec | Note |
|---------|-----|----------|------|
| Initialization | `06 93` | 2.26 | Terminal ID is managed by SENVEND. |
| Set Terminal ID | `06 1B` | 2.39 | Terminal ID is managed by SENVEND. |
| Reset Terminal | `06 18` | 2.43 | |
| Change Password | `06 95` | 2.27 | |
| Select Language | `08 30` | 2.36 | Not yet supported by Verifone. Proxy support planned for Q2 2026. |

Any unknown or unrecognized command tag is also denied.

## Age Verification

When an Authorization (`06 01`) or Reservation (`06 22`) command contains a `minimum_age` TLV (tag `1F6B`, see [ZVT spec][zvt-spec] section 9.4.2), the proxy intercepts the command and triggers an age verification flow on the SENVEND device before forwarding it to the PT.

### Flow from the ECR's perspective

1. **ECR sends** Authorization or Reservation with `minimum_age` TLV (e.g. `18` for age 18+).
1. **Proxy responds** with an immediate ACK (`80 00`).
1. **Age verification** takes place on the SENVEND device. This may take up to several minutes depending on the verification method.
1. **On approval:** The command is forwarded to the PT. The normal ZVT response exchange proceeds — the ECR receives IntermediateStatusInformation, StatusInformation, and Completion as usual. The `age_verification_result` TLV (tag `1F6C`) is injected into the StatusInformation with value `0x01` (MinimumAgeReached).
1. **On denial or timeout:** The ECR receives an Abort (`06 1E`).

### Important notes for integrators

- The ECR receives the ACK immediately, but the first PT response may be delayed while age verification is in progress. Adjust any response timeouts accordingly.
- The ECR can send an Abort (`06 B0`) during the verification period to cancel the transaction.
- On a successful payment, the StatusInformation response will contain `age_verification_result` (tag `1F6C`) with value `0x01` — this is injected by the proxy and was not sent by the PT.

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

## Help Us Improve

Feel free to open issues if something is missing, unclear or not working as expected.
You are also welcome to contribute directly by opening Pull Requests.
If you need any help during your integration and you want to handle it confidentially, please reach out to us via api@senvend.com.

[zvt-spec]: https://www.terminalhersteller.de/downloads.aspx
