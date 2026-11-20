<p align="center">
  <img src="docs/assets/mondo-logo.svg" alt="MONDO Robotics" width="128">
</p>

<h1 align="center">MONDO Beni SDK</h1>

<p align="center">Build applications that connect to Beni robots over your local network.</p>

<p align="center">
  <a href="docs/MONDO%20SDK%20User%20Guide.en.md">User Guide</a> ·
  <a href="docs/protocol-reference.en.md">Protocol Reference</a> ·
  <a href="docs/hms-fault-codes.en.md">Fault Codes</a> ·
  <a href="https://discord.gg/smcZyv5efa">Discord</a>
</p>

---

# Internal Testing Only — Not Available in Public Releases

## The app entry point and firmware capabilities described here are currently available only to authorized test users.

## Interested in testing? Join our [Discord community](https://discord.gg/smcZyv5efa) and wait for test-access announcements.

## Documentation

Start with the user guide for access setup, then use the protocol reference when implementing your client.

| Resource | English | Chinese |
| --- | --- | --- |
| SDK User Guide | [Read the guide](<docs/MONDO SDK User Guide.en.md>) | [Read the guide](<docs/MONDO SDK使用简介.md>) |
| Protocol Reference | [Read the reference](docs/protocol-reference.en.md) | [Read the reference](docs/protocol-reference.md) |
| HMS Fault Codes | [Open fault code table](docs/hms-fault-codes.en.md) | [Open fault code table](<docs/HMS 故障码参考表.md>) |

All guides and reference tables can be read directly on GitHub or in a Markdown preview.

## Start Here

1. Obtain access to the internal test and the required app and firmware versions.
2. Follow the user guide to generate credentials, configure Wi-Fi, and enable SDK access.
3. Connect your SDK host and robot over a reachable local network.
4. Use the protocol reference for message topics, fields, responses, and operating conditions.

## Connection Overview

- **Authentication and signaling:** TLS pairing authentication and SDP exchange over TCP 1024.
- **Application messages:** Zenoh and Protobuf over a WebRTC DataChannel.
- **Video preview:** H.265 over a WebRTC media track, sharing UDP 1024 with the DataChannel.

## Community

Join the [MONDO Discord community](https://discord.gg/smcZyv5efa) for updates about SDK testing and access.

Copyright @ MondoTech LLC. All rights reserved.

The protocol documentation and code examples are provided “as is,” without warranties of any kind. To the fullest extent permitted by applicable law, the copyright holder and contributors disclaim liability for any loss or damage arising from the use of these materials or from building or using any SDK based on them, including AI-generated SDKs.
