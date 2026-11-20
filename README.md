<p align="center">
  <img src="docs/assets/mondo-logo.svg" alt="MONDO Robotics" width="128">
</p>

<h1 align="center">MONDO Beni SDK</h1>

<p align="center">Control and stream video from your Beni robot over your local network.</p>

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

Start with the **SDK User Guide** to pair your robot and understand how the connection works, then use the **Protocol Reference** when implementing your client.

| Resource | English | 中文 |
| --- | --- | --- |
| SDK User Guide | [Read the guide](<docs/MONDO SDK User Guide.en.md>) | [阅读使用简介](<docs/MONDO SDK使用简介.md>) |
| Protocol Reference | [Read the reference](docs/protocol-reference.en.md) | [查看协议手册](docs/protocol-reference.md) |
| HMS Fault Codes | [Open fault code table](docs/hms-fault-codes.en.md) | [查看故障码参考表](<docs/HMS 故障码参考表.md>) |

All guides and reference tables render directly on GitHub or in any Markdown viewer.

## Getting Started

1. Make sure you have internal test access along with the matching Mondo app and robot firmware versions.
2. Follow the **SDK User Guide** to generate SDK credentials, connect the robot to Wi-Fi, and turn on SDK mode.
3. Put your development machine (Linux / WSL) and the robot on the same reachable local network (with TCP and UDP port `1024` allowed).
4. Implement your client using the **Protocol Reference** for topics, Protobuf fields, responses, and operating rules.

## How the Connection Works

- **Authentication & Signaling (`TCP 1024`):** Your client authenticates with a TLS 1.3 external PSK derived from the pairing credentials, exchanges WebRTC SDP over Zenoh, and then closes the TCP connection.
- **Commands & Telemetry (`WebRTC / UDP 1024`):** Zenoh runs over a reliable, ordered binary WebRTC DataChannel (`beni`) carrying Protobuf messages.
- **Live Video (`WebRTC / UDP 1024`):** H.265 video streams over a WebRTC media track sharing the same UDP port `1024`, with RTCP handling bitrate adaptation and keyframe requests.

The robot accepts **one** SDK client connection at a time.

## Community

Join the [MONDO Discord community](https://discord.gg/smcZyv5efa) for SDK updates, testing announcements, and developer discussions.

---

Copyright © MondoTech LLC. All rights reserved.

The protocol documentation and code examples are provided “as is,” without warranties of any kind. To the fullest extent permitted by applicable law, the copyright holder and contributors disclaim liability for any loss or damage arising from the use of these materials or from building or using any SDK based on them, including AI-generated SDKs.
