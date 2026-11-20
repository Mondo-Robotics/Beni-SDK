# AUTHORIZED TEST USERS ONLY — NOT AVAILABLE IN PUBLIC RELEASES

## MONDO SDK is currently in internal testing. The **app entry point and related firmware capabilities described here are not available in public releases**. Access is limited to users who have been granted testing permission.

## **To express interest in testing, join our [Discord community](https://discord.gg/smcZyv5efa) and wait for further announcements about test access and permissions.**

# Beni SDK Open Protocol Reference

> 85 Open Topics · Standard Zenoh + Protobuf · LAN Developer Reference

[中文版本](protocol-reference.md) ｜ [MONDO SDK User Guide](<MONDO SDK User Guide.en.md>) ｜ [HMS Fault Codes](hms-fault-codes.en.md)

---

## Quick Start

1. Connect your development machine (Linux / WSL) and robot to the same local network, allowing both **TCP 1024** and **UDP 1024**.
2. Generate and export the SDK credentials JSON in the Mondo app, connect the robot to Wi-Fi, and enable SDK mode (see the [User Guide](<MONDO SDK User Guide.en.md>)).
3. Connect to `local_ip` from the credentials (or the robot's current LAN IP): **TCP 1024** is used only for TLS 1.3 PSK authentication and SDP exchange, then closed; application messages and H.265 video run over **WebRTC / UDP 1024**.

<details>
<summary>Expand full connection, messaging, and video preview checklist</summary>

### 1. Authentication & Signaling (`TCP 1024`)

1. Make sure SDK mode is enabled in the app and your credentials are valid. The robot accepts **only one SDK client at a time**.
2. Connect to `TCP 1024` on the robot's LAN IPv4 address and perform a **TLS 1.3 external PSK** handshake:
   - Derive a 32-byte shared secret via X25519 using your SDK private key and the robot's `device_public_key`.
   - Derive a 32-byte PSK with **HKDF-SHA256** (`salt` = raw 16-byte `pair_id`, `info` = ASCII `Beni LAN TLS 1.3 external PSK v1`).
   - Use PSK identity `Beni-SDK-v1` and cipher suite `TLS_AES_128_GCM_SHA256` with ephemeral key exchange (no CA or device certificate required).
   - Verify that external PSK mode was negotiated. TLS handshake timeout is 3 s; total signaling timeout is 30 s.

### 2. Establish WebRTC (`UDP 1024`)

3. Create an **H.265 receive video track** (or **2 H.265 receive video tracks** if you also want the stereo pair stream: track 1 for the main camera `beni`, track 2 for stereo `beni-stereo`) and a **reliable, ordered, binary DataChannel** labeled `beni` (no lifetime or max-retransmit limits).
4. Wait until ICE candidate gathering completes. Over the authenticated TLS stream, start Zenoh, **subscribe to `device/sdk/webrtc/answer` first, then publish** the full UTF-8 SDP offer to `app/sdk/webrtc/offer` with encoding `application/octet-stream` (no session attachment needed).
5. The robot replies with a full UTF-8 SDP answer; both peers verify DTLS certificates against the authenticated SDP fingerprints. TCP `1024` closes immediately after SDP exchange.
6. The robot advertises only LAN IPv4/UDP `1024` candidates (no public STUN/TURN or ICE TCP) and requires BUNDLE + RTCP-mux. If the DataChannel does not open within **30 seconds**, the robot tears down the connection.

### 3. Application Messages (DataChannel)

7. Once the DataChannel is open, start a Zenoh session over it and subscribe to `beni/session`. Every 250 ms, the robot pushes a 17-byte payload: **a 16-byte random connection ID followed by a reserved `0` byte**. Attach that 16-byte connection ID to every application PUT sent to the robot.
8. Publish to `app` + canonical topic; subscribe to `device` + canonical topic. Payloads use Protobuf with encoding `application/protobuf`.
9. Requests with responses require a nonzero `header.seq_num`, matched by current session, response key, and sequence number. One-way joystick messages (`/cmd/rc/joystick`) must **omit `header` completely**.
10. Status updates may have gaps and requests may time out; never auto-retry non-idempotent actions. On disconnect, re-authenticate, exchange new SDP, and obtain a fresh connection ID—never reuse old IDs or sequence numbers.

### 4. Video Preview (WebRTC H.265 Track)

11. Video preview starts **off** on every new session. Subscribe to `device/cmd/vt/sdk_preview_rsp`, then send `app/cmd/vt/sdk_preview_req`: use `version=1` for main-camera control (`operation=0` query or `1` set, `enabled=0` off or `1` on), or `version=2` with `stream_mask` to select main camera (`bit0`) and 640×960 stacked stereo (`bit1`), with `session_id` set to the 16-byte connection ID.
12. Receive H.265 video on the WebRTC media track (RTP/SRTP); RTCP handles reception quality and keyframe feedback. Wait for a keyframe before decoding when starting or resuming preview.
13. Stopping preview halts new frames (in-flight frames may still arrive) while keeping the DataChannel open. After a full WebRTC disconnect, reconnect and re-enable preview explicitly.

### 5. Authorization & Safety

14. Authorization is checked continuously. Disabling SDK or revoking/rotating credentials ends the session immediately. Being connected does not guarantee every command will run—mode, state, and safety checks still apply.
15. Send the full 64-bit joystick value. Control input times out after **500 ms** without updates. Always keep personnel on site when operating a physical robot.

</details>

---

<a id="camera-mode-guide"></a>

## Available Actions in Camera Mode

Subscribe to [Scheduler status](#t/status/scheduler/status) first: `current_mode=1` indicates the Camera task, and `sub_mode` selects the behavior:

| Submode | Driving & actions |
| --- | --- |
| `0` · Manual | Joystick controls driving; `jump_cmd` and `flip_cmd` request jumps and flips. Note: manual actions may be blocked while autonomous interaction or replay fallback holds control, so `sub_mode=0` alone does not guarantee execution. |
| `1` · Spotlight | Gimbal tracks the target while joystick drives the robot and triggers jumps/flips (subject to state and cooldown limits). A target-selection ack only confirms request acceptance—check scheduler status to verify Spotlight is active. |
| `2` · Follow | The follow policy controls movement. Policies with auto-jump may jump on their own; generic manual jump/flip requests from Manual/Spotlight are not accepted here. |

**Before sending motion commands:**
- Confirm robot status is `Active` with `abnormal_flags=0`.
- Confirm scheduler status shows standing (`is_locked=0`), no emergency stop (`is_paused=0`), and Camera Manual or Spotlight submode.
- Joystick commands are one-way and jumps/flips have no separate execution ack—verify outcomes from status telemetry. Ordinary jumps can also be requested while stationary. See [`/cmd/rc/joystick`](#t/cmd/rc/joystick) for frame cadence and cooldowns.

**Low-battery restrictions:**
When the robot reports a below-25% or below-10% battery warning, new jumps and front/back/left/right flips are disabled. Check power status and HMS faults, charge the robot, and confirm warnings have cleared before retrying jumps or flips.

---

## Protobuf Framing

Application messages have two outer layers: `header` (tag 1) and `data` (tag 2). Field numbers in the tables below refer to **Protobuf tags inside `data`** (only externally transmitted fields are listed). Empty requests must still encode `data` as an empty submessage (`12 00`); `/cmd/rc/joystick` must omit `header` completely ([see encoding example](#t/cmd/rc/joystick)). Live video uses the WebRTC H.265 media track rather than Protobuf.

---

<a id="t/cmd/rc/joystick"></a>

### `/cmd/rc/joystick` Joystick command

- **Direction:** Client → Robot (Publish)

[Camera modes and action requirements](#camera-mode-guide)

- **Protobuf message:** `md.protocol.cmd.RcJoystick`

**Encoding example (serialization check only):** `data.joystick = 1000`; the entire Protobuf `header` must be omitted. Firmware also rejects an empty header.

```text
12 03 08 E8 07
```

These bytes are the complete Protobuf payload. Set the Zenoh key, encoding, and 16-byte session attachment separately.

- **Message key:** `app/cmd/rc/joystick`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| joystick | u64 | 1 | 64-bit bitfield; see the allocation below. |

#### Bitfield · bit 0 is the least significant bit

| Field | Bits | Name | Meaning |
| --- | --- | --- | --- |
| joystick | 0–10 | channel0 | Left stick vertical axis: forward/reverse speed |
| joystick | 11–21 | channel1 | Left stick horizontal axis: turning speed |
| joystick | 22–32 | channel2 | Screen horizontal input: gimbal yaw |
| joystick | 33–38 | channel3 | Pitch input: 0..62, center 31; bit60 selects angle reference or instantaneous stick input. |
| joystick | 39–44 | roll | Roll adjustment |
| joystick | 45–49 | height | Height adjustment |
| joystick | 50 | jump_cmd | Jump command |
| joystick | 51–53 | flip_cmd | Swipe-triggered flip |
| joystick | 54–55 | max_speed | Speed gear / maximum speed |
| joystick | 56–57 | nav_flag | Navigation level: 0 off, 1 obstacle avoidance, 2 automatic jumping |
| joystick | 58 | ispause | Emergency stop |
| joystick | 59 | islock | 0 request standing, 1 request folding |
| joystick | 60 | pitch_input_is_stick | 0=legacy angle reference, 1=instantaneous pitch stick input. After confirming version 2 support, this controls pitch angular velocity in Manual and adjusts framing in Spotlight and Follow. |
| joystick | 61 | third_person_mode | Third-person mode |
| joystick | 62 | jump_mode | Jump mode |
| joystick | 63 | no_liveview | Preview switch (inverted) |

#### Keyboard-triggered jumps and flips (Camera Manual/Spotlight)

flip_cmd (bits 51–53): 0 no action, 1 front flip, 2 back flip, 3 left flip, 4 right flip. Trigger flips separately with jump_cmd=0 and jump_mode=0. For an ordinary jump set jump_cmd (bit 50) to 1 and jump_mode (bit 62) to 0; for a standing long jump set both in the same frame. A standing long jump requires both desired and actual translational speed below 0.1 m/s; otherwise an ordinary jump is used.

Send control messages at a recommended 50 Hz. Trigger once per key press and ignore keyboard auto-repeat. Hold the jump flag for 10 frames and clear it on frame 11; hold a flip direction for 9 frames and clear it on frame 10. Change only action bits, preserving other joystick state. The robot may execute within the first few frames. Camera-task cooldowns are approximately 0.42 seconds after accepting a manual jump and 2.5 seconds after a flip. New actions during cooldown are not queued.

---

<a id="t/cmd/audio/op_req"></a>

### `/cmd/audio/op_req` Start or stop real-time voice communication

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.AudioOpReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/audio/op_req`
- **Response:** [/cmd/audio/op_rsp](#t/cmd/audio/op_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| operation_type | u8 | 1 | Voice operation: 0=start, 1=stop. Other values are not exposed by this reference. |

#### Voice communication switch

Before speaking from the computer to the robot, send operation_type=0 (START_RECORD). Wait for ret_code=0 on device/cmd/audio/op_rsp before transmitting audio. Send 1 (STOP_RECORD) to finish. Normal voice communication uses only 0 and 1.

---

<a id="t/cmd/audio/op_rsp"></a>

### `/cmd/audio/op_rsp` Voice operation response

- **Direction:** Robot → Client (Subscribe)

Voice operation response. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.AudioOpRsp`
- **Message key:** `device/cmd/audio/op_rsp`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | Voice operation result: 0=success; nonzero=failure, do not start transmitting audio. |

---

<a id="t/data/audio/playback_data"></a>

### `/data/audio/playback_data` Send live audio for playback through the robot speaker; enable voice communication first.

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.data.AudioPlaybackData`
- **Message key:** `app/data/audio/playback_data`

Send live audio for playback through the robot speaker. Publish to app/data/audio/playback_data. First send /cmd/audio/op_req (operation_type=0); after a successful response, continuously send audio frames.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| encoding_type | u8 | 1 | Audio encoding: 0=Opus. |
| sample_rate | u32 | 2 | Sample rate in Hz. The sender uses 48000; the receiver decodes using the reported value. |
| channels | u8 | 3 | Channel count; the current transmit format is 1 (mono). |
| frame_duration_ms | u16 | 4 | Audio frame duration in milliseconds; the current transmit format is 20. |
| timestamp | u32 | 5 | Audio frame timestamp in milliseconds, for ordering within the audio stream, not directly a calendar date/time. |
| seq_num_audio | u32 | 6 | Audio frame sequence number, incremented each frame to check continuity. |
| data_size | u32 | 7 | Valid encoded byte count (<=960). |
| data | u8 [960] | 8 | Encoded bytes for one audio frame; use only the valid length specified by data_size. |

#### Computer microphone → robot speaker

After enabling voice communication, send one Protobuf message every 20 ms: encoding_type=0 (Opus), sample_rate=48000, channels=1, frame_duration_ms=20. One frame corresponds to 960 mono PCM input samples. data contains one encoded Opus payload, 1–960 bytes long; data_size must equal its actual length without padding. timestamp uses milliseconds and seq_num_audio increments each frame. This is a one-way stream; the robot discards audio if communication is disabled or the format is invalid.

---

<a id="t/status/audio/status"></a>

### `/status/audio/status` Audio capture status and recording-response capability

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.AudioStatus`
- **Message key:** `device/status/audio/status`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| record_enabled | u8 | 1 | Whether audio capture/recording is active: 0=inactive, 1=active. This is separate from the video audio-track preference and does not by itself confirm video recording. |
| record_executor_version | u8 | 2 | 2 indicates recording-action execution with final-state feedback. |

---

<a id="t/cmd/camera/capture_controls_set_req"></a>

### `/cmd/camera/capture_controls_set_req` Update selected camera parameters using update_mask and physical values rather than preset indices

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.CaptureControlsSetReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/camera/capture_controls_set_req`

Update selected camera parameters using update_mask and physical values, not preset indices. Validation uses the final combination of current settings and requested changes. An empty mask, unknown bits/modes, or invalid combinations reject the whole request without changes. Selected manual shutter/ISO/WB values must be nonzero and meet device ranges and steps; automatic modes ignore numeric values. Unsupported or out-of-range settings must not be silently substituted. Read status for effective values. A real overall-mode change resets shutter, ISO, and WB to auto and EV to zero; entering professional mode then applies the requested parameters. Repeating the same overall mode does not reset settings. A final automatic overall mode cannot request manual parameters. When shutter and ISO are both manual, EV is disabled and reset to zero. Returning either to auto enables EV at zero; the same request may set a new EV. Explicitly updating EV when the final state disables it rejects the request. ev_mode is read-only. Responses echo the standard header sequence; no new request_id is added. Multiple parameter groups may succeed only partially.

- **Response:** [/cmd/camera/capture_controls_set_rsp](#t/cmd/camera/capture_controls_set_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| update_mask | u32 | 1 | Update bits: bit0=overall professional mode, bit1=shutter, bit2=ISO, bit3=white balance, bit4=exposure compensation. Combine with bitwise OR; other bits are 0. |
| is_pro_mode | bool | 2 | Overall professional mode: true=professional, false=automatic. Updated only when update_mask bit0 is set. |
| shutter_mode | camera_parameter_mode | 3 | Shutter mode: 1=auto, 2=manual. Do not set 0 (unknown). |
| shutter_ns | u64 | 4 | Manual exposure time in ns; ignored for automatic shutter. |
| iso_mode | camera_parameter_mode | 5 | ISO mode: 1=auto, 2=manual. Do not set 0 (unknown). |
| iso | u32 | 6 | Manual ISO value; ignored for automatic ISO. |
| wb_mode | camera_parameter_mode | 7 | White-balance mode: 1=auto, 2=manual. Do not set 0 (unknown). |
| wb_kelvin | u32 | 8 | Manual color temperature in K; ignored for automatic white balance. |
| ev_compensation | f32 | 9 | Exposure compensation in EV. Reject NaN/Inf. Current range: -3..+3 in 1/3 EV steps; distance from the nearest valid step must be at most 0.000001 EV. |

---

<a id="t/cmd/camera/capture_controls_set_rsp"></a>

### `/cmd/camera/capture_controls_set_rsp` Camera parameter response; standard header sequence echoes the request

- **Direction:** Robot → Client (Subscribe)

Camera parameter response, echoing the request sequence. result=0 means all selected groups were accepted, not that the next frame has stabilized. applied_mask and failed_mask identify successful and failed groups; they do not overlap and their OR equals the requested update_mask. Invalid or busy requests mark all selected groups as failed. Partial execution success returns 5. Rollback after failure is not guaranteed; read status for actual settings.

- **Protobuf message:** `md.protocol.cmd.CaptureControlsSetRsp`
- **Message key:** `device/cmd/camera/capture_controls_set_rsp`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| result | capture_controls_set_result | 1 | Result: 0=success, 1=invalid parameters, 2=unsupported, 3=busy, 4=setting failed, 5=partial success. Success does not mean the next frame has stabilized. |
| applied_mask | u32 | 2 | Successfully applied groups. Bits: bit0=overall professional mode, bit1=shutter, bit2=ISO, bit3=white balance, bit4=exposure compensation; combine with OR, other bits 0. |
| failed_mask | u32 | 3 | Groups not successfully applied. Bits: bit0=overall professional mode, bit1=shutter, bit2=ISO, bit3=white balance, bit4=exposure compensation; combine with OR, other bits 0. |

---

<a id="t/cmd/camera/record_req"></a>

### `/cmd/camera/record_req` Recording request

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.CameraRecordReq`
- **Response timeout:** **80000 ms**
- **Message key:** `app/cmd/camera/record_req`

Recording request. Check camera_status.record_response_version: version 1 only checks admission, then the caller sends the legacy recording command; version 2 executes recording and returns the final state via camera_record_rsp. Version 2 callers must not additionally send unified_cmd. Timeout means the result is unknown: do not automatically resend or issue the opposite operation.

- **Response:** [/cmd/camera/record_rsp](#t/cmd/camera/record_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| action | u8 | 1 | Recording action: 1=start, 2=stop. |
| request_id | u64 | 2 | Request identity within the app connection session; echoed for response matching. |

---

<a id="t/cmd/camera/record_rsp"></a>

### `/cmd/camera/record_rsp` Request sequence and request_id echo the request identity

- **Direction:** Robot → Client (Subscribe)

The request sequence and request_id echo the request identity. In version 1, ret_code=0 only means admission succeeded. In version 2 it means the actual action completed: start requires state=2, stop requires state=0. Starting, stopping, and queuing do not count as success. Failure/cancellation returns a nonzero code; state is the actual state at response time. Recording overview is reported on /status/camera/status.

- **Protobuf message:** `md.protocol.cmd.CameraRecordRsp`
- **Message key:** `device/cmd/camera/record_rsp`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| action | u8 | 1 | Echoed action: 1=start, 2=stop. |
| request_id | u64 | 2 | Echoed request identity |
| ret_code | u8 | 3 | Result: 0=success, 1=invalid request, 2=state conflict, 3=storage busy, 4=storage unavailable, 5=start failed, 6=stop failed, 7=service unavailable, 8=start sound failed, 9=operation canceled, 10=insufficient space, 11=main camera abnormal, 12=storage in use by computer. Interpret success according to the recording-response version. |
| state | u8 | 4 | Actual recording state at response time: 0=idle, 1=starting, 2=recording, 3=stopping. Version 2 start success requires state 2; stop success requires state 0. |

---

<a id="t/cmd/camera/unified_cmd"></a>

### `/cmd/camera/unified_cmd` Camera controls: photo, zoom, stabilization, legacy capture parameters, highlights, and recording audio

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.CameraUnifiedCmd`
- **Message key:** `app/cmd/camera/unified_cmd`

Publish camera commands to app/cmd/camera/unified_cmd. cmd_type selects the operation; fill its associated values. This message has no separate success response. Successful transmission only confirms sending; verify effective settings using device/status/camera/status, device/status/camera/capture_controls, or the relevant status.

For device compatibility, select only one cmd_type per message; do not OR operations together. Encode all 16 data fields; initialize unused fields as in the example below. Photo capture, RAW capture, and highlight marking are one-shot actions. Do not send them in a loop or retry automatically because no separate response arrived.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| cmd_type | u16 | 1 | Operation selector; see below. Send one listed command value at a time. Keep unexposed bits at 0. |
| trigger_flags | u8 | 2 | Action trigger only: photo=1 (bit0); legacy recording start=2 (bit1), stop=0. Use 0 for other operations. |
| zoom_data | u8 | 3 | Zoom ratio ×10: 10=1.0×, 20=2.0×. Usually 10–20; some recording profiles accept only 10 or 20. Supported ranges and changes during recording depend on the device. |
| eis_enable | u8 | 4 | Electronic stabilization request: 0=off, 1=on. Some devices always enable stabilization and only support changing eis_level; this field cannot disable it. |
| eis_level | u8 | 5 | Stabilization level: 0=basic, 1=standard, 2=enhanced; supported levels depend on the device. Used only with cmd_type=0x0008. |
| horizon_lock | u8 | 6 | Not exposed as a configurable setting; set to 0. |
| storage_type | u8 | 7 | This command does not change storage location; set to 0. |
| raw_record_enable | u8 | 8 | With cmd_type=0x0040, 1=request one RAW frame, 0=no action. This is not continuous recording; it executes only when RAW capture is supported and allowed. |
| record_profile_id | u16 | 9 | This setting is not exposed in this reference; set to 0. |
| vt_config_index | u8 | 10 | This setting is not exposed in this reference; set to 0. |
| pro_mode_enable | u8 | 11 | With cmd_type=0x0080, 0=automatic, 1=professional. Submit shutter, ISO, EV, and white balance together; omitted values do not mean keep the current value. |
| exp_time_index | u8 | 12 | Legacy shutter index: 0=auto; see the mapping for 1–25 below. This is an index, not exposure nanoseconds. Used only with 0x0080. |
| iso_index | u8 | 13 | Legacy ISO index: 0=auto; interpret nonzero indices using the target device's preset definitions. Do not enter an ISO value directly or reuse indices across devices. To set an actual ISO value, use capture_controls_set_req on supported devices. |
| ev_index | u8 | 14 | Legacy exposure-compensation index: 0–18, with 9=0 EV. Target compensation is approximately (index−9)/3 EV, from −3 to +3 EV. Read status for quantized values. Used only with 0x0080; use 9 when unused. |
| wb_index | u8 | 15 | Legacy white-balance index: 0=auto; 1–81 correspond to 2000–10000 K, temperature=(index−1)×100+2000. Used only with 0x0080. |
| reserved | u8 | 16 | Reserved; set to 0 when sending and ignore when receiving. |

#### Operations and field values

| cmd_type | Purpose | Values and result verification |
| --- | --- | --- |
| 0x0001 (1) | Take photo | trigger_flags=1 requests one photo; completion of sending does not confirm the photo was saved. |
| 0x0002 (2) | Legacy recording start/stop | trigger_flags=2 to start, 0 to stop. Follow the recording-version rules below; do not bypass recording admission checks. |
| 0x0004 (4) | Zoom | Set zoom_data; verify the reported ratio in camera status zoom_data. |
| 0x0008 (8) | Electronic image stabilization | Set eis_enable and eis_level; verify eis_status in camera status. |
| 0x0040 (64) | Single-frame RAW capture | raw_record_enable=1; executes only when supported and allowed. This is not the normal photo or continuous-recording interface. |
| 0x0080 (128) | Legacy professional mode and capture parameters | Provide pro_mode_enable, exp_time_index, iso_index, ev_index, and wb_index together. Prefer capture_controls_set_req when supported. |
| 0x0100 (256) | Mark a recording highlight | Request a highlight in the current recording. Send only after confirming recording is active; behavior outside recording is not guaranteed. |
| 0x0800 (2048) | Recording audio switch | reserved=0 for mute, 1 for audio. Takes effect on the next recording; check record_audio_enabled in device/status/mic/status. |

Operations not listed here are not exposed. Horizon lock has no available implementation. This command cannot switch storage. Recording-profile and video-transmission-mode settings are not exposed; do not invoke them using other command bits.

#### Recording start/stop version rules

Read record_response_version in camera status first. Version 1: send /cmd/camera/record_req, wait for a matching successful admission response, then send 0x0002 through this command. Version 2: record_req executes the action and returns the final state; do not additionally send 0x0002. An undeclared version is a legacy compatibility case, not confirmation of the new response protocol. A timed-out request has an unknown result; do not automatically resend or reverse it.

#### Legacy capture parameters

0x0080 submits all capture settings together. To restore automatic mode use pro_mode_enable=0, exp_time_index=0, iso_index=0, ev_index=9, wb_index=0. Manual settings must use combinations supported by the target device; independent shutter/ISO modes and EV applicability may differ. Settings may succeed partially; successful transmission does not confirm the entire group was applied.

Shutter index 0=auto. Indices 1–25 map to 1/8000, 1/6400, 1/5000, 1/4000, 1/3200, 1/2500, 1/2000, 1/1600, 1/1250, 1/1000, 1/800, 1/640, 1/500, 1/400, 1/320, 1/250, 1/200, 1/160, 1/125, 1/100, 1/80, 1/60, 1/50, 1/40, 1/30 seconds. Actual exposure is limited by frame rate and device capabilities.

capture_controls_set_req uses actual shutter duration, ISO, color temperature, and EV, and provides a response. It covers capture parameters only, not photo/zoom operations. Use one parameter-setting protocol per change to avoid conflicting writes.

#### Example: set 1.0× zoom

These are field values before Protobuf encoding, not JSON to transmit directly. Encode as CameraUnifiedCmd and publish to app/cmd/camera/unified_cmd with encoding=application/protobuf and the current 16-byte connection ID in the attachment.

```json
{
  "header": {
    "seq_num": 1
  },
  "data": {
    "cmd_type": 4,
    "trigger_flags": 0,
    "zoom_data": 10,
    "eis_enable": 0,
    "eis_level": 0,
    "horizon_lock": 0,
    "storage_type": 0,
    "raw_record_enable": 0,
    "record_profile_id": 0,
    "vt_config_index": 0,
    "pro_mode_enable": 0,
    "exp_time_index": 0,
    "iso_index": 0,
    "ev_index": 9,
    "wb_index": 0,
    "reserved": 0
  }
}
```

---

<a id="t/status/camera/capture_controls"></a>

### `/status/camera/capture_controls` Authoritative complete camera-parameter snapshot

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.CaptureControlsStatus`
- **Message key:** `device/status/camera/capture_controls`

Published after configuration changes; actual values update with measurements. Supports initial subscription and reconnect recovery. Modes and numeric values are independent: modes reflect accepted settings; measurements may update later. User targets and preset indices are not reported. Invalid shutter/ISO/WB measurements are 0, never substituted with requested values. Temporarily unavailable measurements do not erase known modes; report unknown only when the mode itself cannot be determined. EV is effective exposure compensation, not metered scene EV. When disabled, its state is disabled and its value is 0.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| is_pro_mode | bool | 1 | Current professional mode: true=yes, false=no. |
| shutter_mode | camera_parameter_mode | 2 | Shutter mode: 0=unknown, 1=auto, 2=manual. |
| iso_mode | camera_parameter_mode | 3 | ISO mode: 0=unknown, 1=auto, 2=manual. |
| wb_mode | camera_parameter_mode | 4 | White-balance mode: 0=unknown, 1=auto, 2=manual. |
| ev_mode | camera_ev_mode | 5 | Exposure-compensation state: 0=unknown, 1=enabled, 2=disabled. Disabled when shutter and ISO are both manual. |
| shutter_ns | u64 | 6 | Current measured exposure time in ns; 0=temporarily unavailable. |
| iso | u32 | 7 | Current measured ISO; 0=temporarily unavailable. |
| wb_kelvin | u32 | 8 | Current measured color temperature in K; 0=temporarily unavailable. |
| ev_compensation | f32 | 9 | Effective exposure compensation in EV; 0 when disabled. |

---

<a id="t/status/camera/status"></a>

### `/status/camera/status` Camera status: recording, duration, resolution, frame rate, zoom, and more

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.CameraStatus`
- **Message key:** `device/status/camera/status`

The robot publishes camera status on device/status/camera/status. Use it for recording state/duration, photo and recording dimensions, video-stream settings, stabilization, zoom, and ambient light. It is a status notification, not a command or request acknowledgment. Verify changes through the corresponding response and subsequent status. For the new capture-parameter protocol, read actual shutter/ISO/WB/EV from device/status/camera/capture_controls.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| rec_state | u8 | 1 | Recording flag: 0=not formally recording, 1=formally recording; insufficient by itself to identify pre-recording. |
| cur_rec_width | u16 | 2 | Current recording width in pixels. |
| cur_rec_height | u16 | 3 | Current recording height in pixels. |
| cur_rec_bitrate | u8 | 4 | Reported recording bitrate in Mbps. |
| cur_snap_width | u16 | 5 | Current photo width in pixels. |
| cur_snap_height | u16 | 6 | Current photo height in pixels. |
| camera_width | u16 | 7 | Camera capture width in pixels, distinct from recording or streamed-video output width. |
| camera_height | u16 | 8 | Camera capture height in pixels, distinct from recording or streamed-video output height. |
| rec_frame_rate | u16 | 9 | Recording frame rate in fps. |
| vt_width | u16 | 10 | Streamed-video width in pixels. |
| vt_height | u16 | 11 | Streamed-video height in pixels. |
| vt_frame_rate | u16 | 12 | Streamed-video frame rate in fps. |
| vt_bitrate | u16 | 13 | Reported streamed-video bitrate in kbps, not the client's measured network throughput. |
| eis_status | u8 | 14 | Stabilization state or level; value definitions may vary by device version. Do not treat it as a universal Boolean switch. |
| pro_shutter_idx | u8 | 15 | Legacy shutter preset index, not exposure duration. Read capture_controls.shutter_ns for actual duration. |
| pro_iso_idx | u8 | 16 | Legacy ISO preset index, not an ISO value. Read capture_controls.iso for the actual value. |
| pro_ev_idx | u8 | 17 | Legacy EV preset index. Read capture_controls.ev_compensation for actual EV. |
| pro_wb_idx | u8 | 18 | Legacy white-balance preset index, not color temperature. Read capture_controls.wb_kelvin for actual temperature. |
| rec_duration_s | u16 | 19 | Duration in seconds. recording_state=2: current pre-recording window; =1: elapsed formal recording time excluding pre-record history; =0: 0. Timer restarts at formal recording start. 0 for no valid reading; maximum 65535. |
| zoom_data | u8 | 20 | Zoom ratio ×10, e.g. 10=1.0×, 20=2.0×. Supported range depends on the device; 0=unknown. |
| current_mode | u8 | 21 | Compatibility mode field; meaning depends on device version. 255=invalid. Do not use it to detect recording; read scheduler status for task mode. |
| current_preset | u8 | 22 | Compatibility preset ID; meaning depends on device version. 255=invalid. Do not infer a recording profile from it. |
| cur_lux | u32 | 23 | Estimated ambient illuminance in lux; 0=no valid estimate, not complete darkness. |
| lux_level | u8 | 24 | Ambient brightness level, increasing with light: 1=extremely dark (1–<5 lux), 2=very dark (5–<20), 3=dark (20–<50), 4=dim (50–<100), 5=normal (100–<300), 6=bright (300–<750), 7=very bright (750–<2000), 8=strong light (2000–<10000), 9=extreme light (≥10000); 255=invalid. |
| current_profile_id | u16 | 25 | Current recording-profile ID. Missing or 0 means the profile is unconfirmed. Check width, height, and frame rate; the ID is not itself a resolution. |
| recording_state | camera_recording_state | 26 | Recording state: 0=idle, 1=formal recording, 2=pre-recording or preparing to start. Idle does not guarantee file finalization. A missing field means unsupported reporting; do not overwrite rec_state with a default value. |
| record_response_version | u8 | 27 | Camera response version: 0=undeclared, 1=request admission check, 2=recording execution and final-state feedback. |

---

<a id="t/status/capability/voice_language"></a>

### `/status/capability/voice_language` Voice language and resource availability

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.VoiceLanguageStatus`
- **Message key:** `device/status/capability/voice_language`

Device language and prompt resources are reported at initialization, after language/resource changes, and every second. Current format version: 1. Do not reuse status after more than 3 seconds without updates, invalid timestamps, or out-of-order reports. Refresh after reconnecting. current_language selects spoken prompts and local voice-command recognition language; supported_mask only indicates prompt-resource availability. Default language without a saved choice is US English. If the choice cannot be read, language/resource fields are 0; unknown does not mean Chinese or English was selected.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| version | u16 | 1 | Status format version, currently 1; not an indication of voice readiness. |
| source_monotonic_ns | u64 | 2 | Device monotonic timestamp in ns when status was generated; not directly comparable to the client clock. |
| supported_mask | u8 | 3 | Available prompt resources: bit0=Simplified Chinese, bit1=US English; other bits 0. A zero mask means no prompt-resource package is available. This does not describe recognition-model availability. |
| current_language | u8 | 4 | Current language: 0=unknown, 1=Simplified Chinese, 2=US English; other values are unknown. Selects prompts and local voice-command recognition, not detection of the language currently spoken by the user. |

---

<a id="t/cmd/common/get_version_req"></a>

### `/cmd/common/get_version_req` Query device version information

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.CommonGetVersionReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/common/get_version_req`
- **Response:** [/cmd/common/get_version_rsp](#t/cmd/common/get_version_rsp)

---

<a id="t/cmd/common/get_version_rsp"></a>

### `/cmd/common/get_version_rsp` Device version response

- **Direction:** Robot → Client (Subscribe)

Device version response. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.CommonGetVersionRsp`
- **Message key:** `device/cmd/common/get_version_rsp`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | Version query result: 0=success; nonzero=failure. |
| major_version | u8 | 2 | Packed compatibility version byte: low 4 bits=minor version, high 4 bits=major version; see bit definitions. |
| minor_version | u8 | 3 | Separate minor-version byte; preserve independently from the packed version byte. |
| hardware_version | u8 [16] | 4 | Hardware version identifier, up to 16 bytes; not a chipset-model description. |
| loader_version | u8 [4] | 5 | Bootloader version encoding, 4 bytes; preserve raw bytes when no format is specified. |
| firmware_version | u8 [4] | 6 | Firmware version encoding, 4 bytes; preserve raw bytes when no format is specified. |
| is_mass_product | u8 | 7 | Production flag: bit0=1 for a production release; other bits reserved. |

#### Bitfield · bit 0 is the least significant bit

| Field | Bits | Name | Meaning |
| --- | --- | --- | --- |
| major_version | 0–3 | minor_version | Minor-version bits in the packed version. |
| major_version | 4–7 | major_version | Major-version bits in the packed version. |
| is_mass_product | 0 | is_mass_product | 1=production release, 0=non-production release. |
| is_mass_product | 1–31 | reserved | Reserved bits; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/common/ping_req"></a>

### `/cmd/common/ping_req` Connectivity probe request

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.CommonPingReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/common/ping_req`
- **Response:** [/cmd/common/ping_rsp](#t/cmd/common/ping_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| ping_data | u8 [16] | 1 | 16-byte probe data supplied by the client and echoed unchanged, for matching probes and checking connectivity. |

---

<a id="t/cmd/common/ping_rsp"></a>

### `/cmd/common/ping_rsp` Connectivity probe response and echo

- **Direction:** Robot → Client (Subscribe)

Connectivity probe response and echo. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.CommonPingRsp`
- **Message key:** `device/cmd/common/ping_rsp`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | Probe result: 0=success; nonzero=failure. |
| ping_data | u8 [16] | 2 | 16-byte probe data supplied by the client and echoed unchanged, for matching probes and checking connectivity. |

---

<a id="t/status/hms/active_snapshot"></a>

### `/status/hms/active_snapshot` Complete snapshot of current active faults

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.HmsActiveSnapshot`
- **Message key:** `device/status/hms/active_snapshot`

HMS active-fault snapshot response

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| at_offset | u64 | 1 | Fault event sequence number at snapshot generation, used to synchronize with the event stream. |
| count | u32 | 2 | Active fault count (<=256). |
| module_ids | u32 [256] | 3 | Fault module ID array; the first count entries are valid. |
| fault_codes | u32 [256] | 4 | Complete 32-bit fault-code array; indices correspond to the other fault arrays. |
| instance_ids | u32 [256] | 5 | Fault instance ID array; one fault code may have multiple instances. |
| severities | u32 [256] | 6 | Severity array: 0=info, 1=warning, 2=error, 3=fatal; indices correspond to the other fault arrays. |
| first_assert_ts | u64 [256] | 7 | First-occurrence time counter for each fault instance; preserve raw values without inferring an epoch. |
| last_update_ts | u64 [256] | 8 | Last-update time counter for each instance, using the same time base as first_assert_ts. |
| daemon_instance_id | u64 | 9 | Fault-state service instance ID. A change invalidates the previous instance; obtain a new complete snapshot. |
| state_revision | u64 | 10 | Current fault-state revision, used to reject old state and associate subsequent events. |
| recovery_state | hms_recovery_state | 11 | State validity: 0=invalid, 1=recovering, 2=ready. Only a valid, ready, complete snapshot can clear previous fault state. |
| ret_code | u32 | 12 | 0=success, 1=invalid filter, 2=capacity exceeded. A failure is not an empty fault list. |
| request_id | u64 | 13 | Corresponds to hms_query_active.request_id; the app accepts only the current query generation. |

Subscribe to fault snapshots and events before querying. Build complete state from a valid, ready snapshot in the same fault-service instance; events represent changes only. Query again after reconnecting, service-instance changes, event gaps, or invalid data. Missing messages do not mean faults have cleared.

---

<a id="t/status/hms/event_broadcast"></a>

### `/status/hms/event_broadcast` Fault event notification

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.HmsEventBroadcast`
- **Message key:** `device/status/hms/event_broadcast`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| offset | u64 | 1 | Fault event sequence number for detecting gaps and synchronizing with complete snapshots. |
| module_id | u16 | 2 | Fault module ID; complete fault code=(module_id<<16)\|fault_code. |
| fault_code | u16 | 3 | Low 16 bits of the fault code; combine with module_id before looking up the fault reference. |
| instance_id | u16 | 4 | ID of the fault instance that changed. |
| kind | u8 | 5 | Event kind: 1=asserted, 2=cleared, 3=updated. Events may be missed and do not replace a complete snapshot. |
| severity | u8 | 6 | Severity: 0=info, 1=warning, 2=error, 3=fatal. |
| timestamp_ms | u64 | 7 | Event time counter in milliseconds; do not infer an epoch. |
| daemon_instance_id | u64 | 8 | Fault-service instance ID, matching the snapshot field. |
| state_revision | u64 | 9 | Fault-state revision for this event. Query a snapshot after a gap or regression. |
| origin | u8 | 10 | Event origin: 0=invalid, 1=live change, 2=state resynchronization, 3=source-disconnection handling. |
| recovery_state | u8 | 11 | State validity: 0=invalid, 1=recovering, 2=ready. Events alone must not clear existing restrictions during recovery. |

Subscribe to fault snapshots and events before querying. Build complete state from a valid, ready snapshot in the same fault-service instance; events represent changes only. Query again after reconnecting, service-instance changes, event gaps, or invalid data. Missing messages do not mean faults have cleared.

---

<a id="t/status/hms/query_active"></a>

### `/status/hms/query_active` Query current faults for all modules or one module

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.status.HmsQueryActive`
- **Message key:** `app/status/hms/query_active`

HMS active-fault query request

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| filter_kind | u8 | 1 | Filter: 0=all faults, 1=one module; other values invalid. |
| _pad0 | u8 | 2 | Reserved field; always 0. |
| module_id | u16 | 3 | Target module ID when filter_kind=1; use 0 for all modules. |
| request_id | u64 | 4 | Query ID; active_snapshot must echo it unchanged. |

Subscribe to fault snapshots and events before querying. Build complete state from a valid, ready snapshot in the same fault-service instance; events represent changes only. Query again after reconnecting, service-instance changes, event gaps, or invalid data. Missing messages do not mean faults have cleared.

---

<a id="t/cmd/led/ctrl_oneway"></a>

### `/cmd/led/ctrl_oneway` Set the colors of 14 LEDs directly

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.LedCtrlOneway`
- **Message key:** `app/cmd/led/ctrl_oneway`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| seq | u32 | 1 | Increasing lighting-frame sequence number for continuity checks. |
| timestamp_us | u64 | 2 | Send timestamp in microseconds, for ordering and expiry checks. |
| rgbw_data | u32 [14] | 3 | Array of 14 LED colors, 32 bits each: (R<<24)\|(G<<16)\|(B<<8)\|W. Each component ranges from 0 to 255. |

---

<a id="t/status/media/catalog_changed"></a>

### `/status/media/catalog_changed` Media-list change notification

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.MediaCatalogChanged`
- **Message key:** `device/status/media/catalog_changed`

Notifies media-list changes so the client can refresh. storage_kind: 0=internal, 1=SD card, 255=unknown. change_reason: 1=recording stopped, 2=rescan complete, 3=photo complete, 4=storage recovered, 5=storage offline.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| revision | u64 | 1 | Media-list revision; query the list again when it changes. |
| storage_kind | u8 | 2 | Changed storage: 0=internal, 1=SD card, 255=unknown. |
| change_reason | u8 | 3 | Change reason: 1=recording stopped, 2=rescan complete, 3=photo complete, 4=storage recovered and available, 5=storage offline. |
| reserved | u16 | 4 | Reserved; set to 0 when sending and ignore when receiving. |
| mount_generation | u64 | 5 | Storage connection generation, identifying a new storage session after replacement or remount. |

---

<a id="t/cmd/mic/saved_list_req"></a>

### `/cmd/mic/saved_list_req` Request the saved microphone list again

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.MicSavedListReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/mic/saved_list_req`

Request the saved microphone list. The response acknowledges acceptance only; obtain list data from /status/mic/saved_list. Request the current list after subscribing; subsequent changes are pushed by the robot.

- **Response:** [/cmd/mic/saved_list_rsp](#t/cmd/mic/saved_list_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| reserved0 | u8 | 1 | Reserved; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/mic/saved_list_rsp"></a>

### `/cmd/mic/saved_list_rsp` Acceptance response for a saved-microphone-list request

- **Direction:** Robot → Client (Subscribe)

Acceptance response for a saved-microphone-list request. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.MicSavedListRsp`
- **Message key:** `device/cmd/mic/saved_list_rsp`

Acknowledges acceptance without list data. After the request, the robot publishes the current list on /status/mic/saved_list. Initial subscription does not replay the old list; explicitly request it once. Subsequent changes are pushed.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | Request result: 0=accepted; nonzero=rejected or failed. Read microphone status for the final audio-source switch result. |

---

<a id="t/cmd/mic/scan_req"></a>

### `/cmd/mic/scan_req` Start or stop scanning for nearby Bluetooth microphones

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.MicScanReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/mic/scan_req`

Start/stop microphone scanning; results use mic_scan_result. Only models verified for the current hardware and firmware are reported. Eligibility is not determined by the device name or brand.

- **Response:** [/cmd/mic/scan_rsp](#t/cmd/mic/scan_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| action | u8 | 1 | 0=stop scanning, 1=start scanning. |
| duration_s | u8 | 2 | Scan duration in seconds; 0=scan until explicitly stopped (used by the scan panel). |

---

<a id="t/cmd/mic/scan_rsp"></a>

### `/cmd/mic/scan_rsp` Acceptance response for a microphone scan request

- **Direction:** Robot → Client (Subscribe)

Acceptance response for a microphone scan request. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.MicScanRsp`
- **Message key:** `device/cmd/mic/scan_rsp`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | Request result: 0=accepted; nonzero=rejected or failed. Read microphone status for the final audio-source switch result. |

---

<a id="t/cmd/mic/source_select_req"></a>

### `/cmd/mic/source_select_req` Select the video audio source and update recording-audio settings

- **Direction:** Client → Robot (Publish)

Select the video audio source and update recording-audio settings. kind=0 disables capture and cancels an unfinished switch while keeping existing Bluetooth connections. A successful kind=1/2 switch enables recording audio; failure restores the previous setting. Audio-track changes apply to the next recording. The response acknowledges acceptance only; read /status/mic/status for the final result. Bluetooth connection and restoration of the previous Bluetooth source each wait at most 15 seconds.

- **Protobuf message:** `md.protocol.cmd.MicSourceSelectReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/mic/source_select_req`
- **Response:** [/cmd/mic/source_select_rsp](#t/cmd/mic/source_select_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| kind | u8 | 1 | 0=disable all audio capture, 1=enable built-in microphone, 2=enable the specified Bluetooth microphone; see mic_source_kind. |
| mac | u8 [6] | 2 | For kind=2, supply the target microphone's 6-byte Bluetooth address. For kind=0/1, use all zeros. |

---

<a id="t/cmd/mic/source_select_rsp"></a>

### `/cmd/mic/source_select_rsp` Acceptance response for an audio-source switch request

- **Direction:** Robot → Client (Subscribe)

Acceptance response for an audio-source switch request. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.MicSourceSelectRsp`
- **Message key:** `device/cmd/mic/source_select_rsp`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | Request result: 0=accepted; nonzero=rejected or failed. Read microphone status for the final audio-source switch result. |

---

<a id="t/status/mic/saved_list"></a>

### `/status/mic/saved_list` Saved microphone list push: the sole source of list data; battery, availability, additions, and removals are pushed without display-side polling

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.MicSavedList`
- **Message key:** `device/status/mic/saved_list`

The robot pushes all saved-list changes: battery, availability, additions, and removals; the display does not poll. entries are ordered by the first successful usable audio connection and save time, oldest first. Reconnects, source switches, and status changes preserve position. New devices and devices paired again after removal are appended. Existing order is preserved. mic_saved_list_req triggers one resend; mic_saved_list_rsp contains only ret_code, not list data.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| entries | mic_saved_device [32] | 1 | Saved microphones, at most 32 entries. Each contains address, name, connection state, battery-valid flag/value, and connection history; see the compound-field definition. |

#### Compound field: mic_saved_device

| Field | Field number | Description |
| --- | --- | --- |
| mac | 1 | 6-byte Bluetooth address used as the device identifier. |
| name | 2 | UTF-8 device name, without a string terminator. |
| state | 3 | 0=unknown, 1=currently unavailable, 2=visible in this scan, 3=connecting, 4=connected. Unavailable does not necessarily mean powered off. |
| battery_valid | 4 | 0=battery value unavailable, 1=valid. |
| battery_pct | 5 | Battery percentage 0–100; check battery_valid first. |
| last_success_seq | 6 | Sequence number of the most recent successful connection; not used for list order. |
| last_success_ms | 7 | Most recent successful connection time as Unix milliseconds; 0=time unavailable. |
| supported | 8 | Whether the current device software supports this model: 0=no, 1=yes. |

---

<a id="t/status/mic/scan_result"></a>

### `/status/mic/scan_result` Complete scan-results snapshot

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.MicScanResult`
- **Message key:** `device/status/mic/scan_result`

Complete scan-results snapshot; replace the previous results when a new snapshot arrives.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| scanning | u8 | 1 | Scanning in progress: 0=no, 1=yes. |
| entries | mic_scan_device [32] | 2 | Complete scan list, at most 32 entries. Each includes address, name, signal strength, model ID, and saved status. |

#### Compound field: mic_scan_device

| Field | Field number | Description |
| --- | --- | --- |
| mac | 1 | 6-byte Bluetooth address. |
| name | 2 | UTF-8 device name, without a string terminator. |
| rssi | 3 | Received signal strength in dBm. |
| model_id | 4 | Device model identifier for distinguishing models in scan results. |
| saved | 5 | 0=not saved, 1=already in the saved list. |

---

<a id="t/status/mic/status"></a>

### `/status/mic/status` Authoritative video-audio status: effective source, connecting target, latest event, and input level

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.MicStatus`
- **Message key:** `device/status/mic/status`

Video-audio status: effective source, connecting target, latest event, and input level. This status determines the final source-switch result; request responses acknowledge acceptance only. The recording-audio switch is read-only here; change it using camera_unified_cmd cmd_type bit11 and reserved. Changes apply to the next recording.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| active_kind | u8 | 1 | Effective source: 0=none, 1=built-in microphone, 2=Bluetooth microphone. Check record_audio_enabled to determine whether recording audio is enabled. |
| active_mac | u8 [6] | 2 | Bluetooth microphone address when active_kind=2; otherwise all zeros. |
| switching | u8 | 3 | 1=connecting to a target; the display shows a waiting indicator. |
| target_mac | u8 [6] | 4 | When switching=1, identifies the connecting target; when switching=0, identifies the device associated with the latest event. All zeros for a global audio-disable event. |
| last_event_kind | u8 | 5 | Latest event: 0=none, 1=switch succeeded, 2=switch failed/timed out, 3=switch canceled, 4=unexpected disconnect, 5=audio capture disabled, 6=built-in capture enabled. |
| last_event_reason | u8 | 6 | Failure reason, interpreted only for event 2: 0=none, 1=unknown device, 2=Bluetooth audio service unavailable, 3=authentication failed, 4=busy, 5=no response, 6=audio format not ready, 7=cannot open audio, 8=device removed, 9=recording-audio setting commit failed. |
| result_seq | u16 | 7 | Event sequence increments for each new event and wraps at 16 bits. Use it to avoid duplicate notifications. The initial snapshot initializes state; do not replay historical event popups. |
| input_valid | u8 | 8 | Whether audio input is currently available. 0 means no input and is the no-audio indicator, not necessarily a device fault. Do not substitute input_level==0: that may mean unavailable input or silence. |
| input_level | u8 | 9 | Audio level 0–100, invalid when input_valid=0. Mapped in dB: −60 dBFS=0, 0 dBFS=100. Represents the peak between reports, not linear amplitude. |
| record_audio_enabled | u8 | 10 | Recording-audio switch, read-only: 1=record sound, 0=mute (default). Set using camera_unified_cmd cmd_type bit11 and reserved. Successful sending does not confirm application; takes effect on the next recording. |

---

<a id="t/cmd/mission/request"></a>

### `/cmd/mission/request` Task request

- **Direction:** Client → Robot (Publish)

<a id="mission-submodes"></a>

#### Task mode selection

Publish to `app/cmd/mission/request`. Select mission_id first, then interpret sub_mode for that task. Use nonzero header.seq_num, version=1, and 9-byte mode_params.

| mission_id | sub_mode values | Usage |
| --- | --- | --- |
| 1 · Camera | 0 Manual; 1 Spotlight; 2 Follow | To switch between trailing and parallel following, keep sub_mode=2 and change mode_params[0]. |
| 3 · Pet | 0 Idle; 2 Return to base and dock | Confirm switching using the submode reported in scheduler status. |

#### Manual mode parameters (mission_id=1, sub_mode=0)

Byte indices start at 0. `mode_params[4]` is the fifth byte: it specifies the control orientation and is **not reserved**.

| Byte index | Field | Values and description |
| --- | --- | --- |
| 0 | drive_mode | 0=steer the chassis with the gimbal following; 1=control the gimbal heading with the chassis following the gimbal. |
| 1 | Reserved | Set to 0. |
| 2 | speed_radius_priority | 0=disabled; 1=enable speed reduction during turns. |
| 3 | Reserved | Set to 0. |
| 4 | control_orientation | 0=unspecified control orientation; 1=portrait; 2=landscape. |
| 5–8 | Reserved | Set to 0. |

**Portrait control (control_orientation=1) forces drive_mode=0, even when the request specifies drive_mode=1. For chassis-following-gimbal control, explicitly set drive_mode=1 and control_orientation=2.** Physical remote-controller input always uses chassis steering mode.

Complete request parameters for chassis-following-gimbal control: `version=1`, `mission_id=1`, `sub_mode=0`, with these nine raw bytes in `mode_params`:

```text
01 00 00 00 02 00 00 00 00
```

#### Camera request examples

For first-person Spotlight, use mission_id=1, sub_mode=1, and all-zero mode_params (no fixed orbit). When selecting a tracking target, confirm the tracking response before requesting a mode change.

For ordinary trailing and parallel following, use mission_id=1 and sub_mode=2. mode_params[0] is 0 or 1 respectively; [1–4]=distance in meters (little-endian f32), [5–6]=angle in degrees (little-endian i16), [7]=orbit direction, [8]=routing. Example: parallel following at 2 meters, angle 0, default routing uses these nine bytes: `01 00 00 00 40 00 00 00 00`. To return from Follow to Manual, send mission_id=1, sub_mode=0 and fill parameters using the Manual layout.

Subscribe to the [request response](#t/cmd/mission/request_ack) and match header.seq_num. After result_code=0, check current_mission_id and current_sub_mode. For ongoing display, use [Scheduler status](#t/status/scheduler/status) current_mode and sub_mode. In Follow, sub_mode_state[0] distinguishes trailing from parallel following.

[Camera modes and action requirements](#camera-mode-guide)

- **Protobuf message:** `md.protocol.cmd.MissionRequest`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/mission/request`

Request to start, switch, or stop a task. Exposed task IDs: 1=Camera, 3=Pet.

- **Response:** [/cmd/mission/request_ack](#t/cmd/mission/request_ack)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| version | u8 | 1 | Protocol version; currently 1. |
| mission_id | u8 | 2 | Exposed task IDs: 1=Camera, 3=Pet. |
| sub_mode | u8 | 3 | Interpret using the mission_id table above. Camera: 0=Manual, 1=Spotlight, 2=Follow. |
| mode_params | u8 [9] | 4 | Interpret by task ID, submode, and protocol version; see this entry's mode descriptions. |

---

<a id="t/cmd/mission/request_ack"></a>

### `/cmd/mission/request_ack` Task response reports task-lifecycle handling only, not completion of a specific action.

- **Direction:** Robot → Client (Subscribe)

Task response reports task-lifecycle handling only, not completion of a specific action. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

[View sub_mode values for each mission_id](#t/cmd/mission/request)

- **Protobuf message:** `md.protocol.cmd.MissionRequestAck`
- **Message key:** `device/cmd/mission/request_ack`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| result_code | u8 | 1 | 0=success, 1=task switch failed, 2=submode switch failed, 3=parameter settings failed. |
| current_mission_id | u8 | 2 | Currently effective mission_id. |
| current_sub_mode | u8 | 3 | Currently effective sub_mode. |
| error_code_lo | u8 | 4 | Low byte of error_code. |
| error_code_hi | u8 | 5 | High byte of error_code. |
| pitch_handoff | pitch_handoff | 6 | Optional held-pitch target returned after successful entry into Camera Manual. Accept only a value matching the current connection and request sequence; apply once before restoring legacy absolute-pitch control. Absence does not mean zero. held_pitch_rad is an absolute pitch target in radians. |

#### Compound field: pitch_handoff

| Field | Field number | Description |
| --- | --- | --- |
| held_pitch_rad | 1 | Absolute pitch target latched for this switch, in radians. Applies only to the matching successful request; not a delayed attitude measurement. |

---

<a id="t/cmd/nm/wifi_connect_req"></a>

### `/cmd/nm/wifi_connect_req` Provisioning step 1: submit Wi-Fi credentials and a provisioning ID

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.NmWifiConnectReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/nm/wifi_connect_req`

Step 1 submits ssid/psk/security and the provisioning uuid. The robot responds after accepting them. Wi-Fi mode does not change yet; its hotspot remains available. Then send wifi_provision_start_req; after its response, send wifi_provision_commit_req to connect. Query the result using wifi_provision_result_req with the same uuid.

- **Response:** [/cmd/nm/wifi_connect_rsp](#t/cmd/nm/wifi_connect_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| ssid_len | u8 | 1 | Valid SSID byte count: 1..32. |
| ssid | u8 [32] | 2 | Raw target SSID bytes, without trailing zero; usually UTF-8. |
| psk_len | u8 | 3 | Valid PSK byte count; 0 for an open network. |
| psk | u8 [64] | 4 | Wi-Fi password: WPA-PSK passphrase of 8..63 bytes or a 64-byte hexadecimal PSK; empty for open networks. |
| security | u8 | 5 | Security: 0=open, 1=WEP, 2=WPA, 3=WPA2, 4=WPA3, 5=WPA2/WPA3 mixed. |
| timeout_sec | u8 | 6 | Association+DHCP timeout for step 2, in seconds. 0 uses the device default of 15s; values >60 are treated as 60. |
| uuid | u8 [16] | 7 | Provisioning uuid: a new random 16-byte session ID generated by the app per attempt. Stored with the result for lookup through /cmd/nm/wifi_provision_result_req. |
| resv | u8 [4] | 8 | Reserved; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/nm/wifi_connect_rsp"></a>

### `/cmd/nm/wifi_connect_rsp` Provisioning credential acknowledgment

- **Direction:** Robot → Client (Subscribe)

Provisioning credential acknowledgment. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.NmWifiConnectRsp`
- **Message key:** `device/cmd/nm/wifi_connect_rsp`

Step-1 acceptance response. state=PENDING means credentials and uuid were received; ssid is echoed for verification. The hotspot remains available and connection to the target network has not started. Read results from /cmd/nm/wifi_provision_result_rsp.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| result | u8 | 1 | 0=credentials received, 1=busy and previous credentials unchanged, 2=invalid credentials. |
| ssid_len | u8 | 2 | Echoed SSID byte count; 0 when result != 0. |
| ssid | u8 [32] | 3 | Raw SSID bytes actually stored by the robot, allowing the app to verify receipt. |
| resv | u8 [6] | 4 | Reserved; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/nm/wifi_provision_commit_req"></a>

### `/cmd/nm/wifi_provision_commit_req` Commit provisioning and switch networks

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.NmWifiProvisionCommitReq`
- **Response timeout:** **1000 ms**
- **Message key:** `app/cmd/nm/wifi_provision_commit_req`

Commit network provisioning only after a successful prepare response for the same session ID. The robot then closes its hotspot and Bluetooth connection and connects to the target network. The current connection may immediately drop; start waiting for the result even if the commit response is not received.

- **Response:** [/cmd/nm/wifi_provision_commit_rsp](#t/cmd/nm/wifi_provision_commit_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| trigger_source | u8 | 1 | Source: 0=other, 1=device button, 2=client, 3=production setup. |
| resv | u8 [7] | 2 | Reserved; set to 0 when sending and ignore when receiving. |
| uuid | u8 [16] | 3 | Must match the successfully prepared provisioning session ID; retries with the same UUID are idempotent. |

---

<a id="t/cmd/nm/wifi_provision_commit_rsp"></a>

### `/cmd/nm/wifi_provision_commit_rsp` Provisioning commit response

- **Direction:** Robot → Client (Subscribe)

Provisioning commit response. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.NmWifiProvisionCommitRsp`
- **Message key:** `device/cmd/nm/wifi_provision_commit_rsp`

Diagnostic commit response; delivery before the network switch is not guaranteed. Query provisioning results to confirm the final state.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| result | u8 | 1 | 0=success, 1=busy, 2=no credentials, 3=cannot start provisioning, 4=session ID mismatch. |
| timeout_sec | u8 | 2 | Association+DHCP timeout in seconds. |
| resv | u8 [6] | 3 | Reserved; set to 0 when sending and ignore when receiving. |
| total_timeout_sec | u8 | 4 | Reserved total provisioning budget; 0 means the app uses its default cloud-confirmation window. |
| uuid | u8 [16] | 5 | Echoed provisioning commit session ID. |

---

<a id="t/cmd/nm/wifi_provision_result_req"></a>

### `/cmd/nm/wifi_provision_result_req` Query the latest provisioning result

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.NmWifiProvisionResultReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/nm/wifi_provision_result_req`

Query the latest result. After a failure and hotspot restoration, reconnect and use the step-1 uuid to query the reason. The result remains queryable after reboot.

- **Response:** [/cmd/nm/wifi_provision_result_rsp](#t/cmd/nm/wifi_provision_result_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| uuid | u8 [16] | 1 | Step-1 uuid. All zeros requests the latest record without validating its uuid. |
| resv | u8 [8] | 2 | Reserved; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/nm/wifi_provision_result_rsp"></a>

### `/cmd/nm/wifi_provision_result_rsp` Latest provisioning result

- **Direction:** Robot → Client (Subscribe)

Latest provisioning result. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.NmWifiProvisionResultRsp`
- **Message key:** `device/cmd/nm/wifi_provision_result_rsp`

Only the latest result is retained. Check uuid_match before using state and fail_reason; provisioning initiated by another client may overwrite the record.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| result | u8 | 1 | 0=result exists and other fields are valid; 1=no result and other fields are 0. |
| uuid_match | u8 | 2 | 1=stored uuid matches, or request uuid is all zeros, so the record can be used. 0=record belongs to a different attempt, possibly overwritten by another app/phone; state/fail_reason are unrelated to this attempt. |
| state | u8 | 3 | 0=no record, 1=credentials submitted but not committed, 2=provisioning, 3=success, 4=failed and hotspot restored. |
| fail_reason | u8 | 4 | Only for state=4: 2=invalid parameters, 3=network interface abnormal, 4=mode switch failed, 5=configuration failed, 6=wireless authentication service failed, 7=association timeout, 8=address allocation timeout, 9=provisioning interrupted, 10=wrong password, 11=AP rejected connection, 12=AP not found, 13=cloud did not confirm connectivity. Otherwise 0. |
| uuid | u8 [16] | 5 | Provisioning uuid stored in the device record. |
| ssid_len | u8 | 6 | Valid SSID byte count. |
| ssid | u8 [32] | 7 | Raw target SSID bytes for this attempt. |
| ipv4 | u8 [4] | 8 | IPv4 address obtained when state=3, in network byte order with ipv4[0] most significant; otherwise all zeros. |
| channel | u8 | 9 | Connected AP channel when state=3; 0=unknown. |
| rssi | i8 | 10 | Connected AP signal strength in dBm when state=3; 0=unavailable. |
| resv | u8 [4] | 11 | Reserved; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/nm/wifi_provision_start_req"></a>

### `/cmd/nm/wifi_provision_start_req` Validate provisioning data and prepare the switch

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.NmWifiProvisionStartReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/nm/wifi_provision_start_req`

Prepare request. The robot validates step-1 credentials and UUID, stores the service endpoint for this attempt, and responds while retaining hotspot and Bluetooth connections. Receive wifi_provision_start_rsp for the same UUID before sending wifi_provision_commit_req.

- **Response:** [/cmd/nm/wifi_provision_start_rsp](#t/cmd/nm/wifi_provision_start_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| trigger_source | u8 | 1 | Source: 0=other, 1=device button, 2=client, 3=production setup. |
| resv | u8 [7] | 2 | Reserved; set to 0 when sending and ignore when receiving. |
| uuid | u8 [16] | 3 | Provisioning UUID, matching step-1 wifi_connect_req.uuid. Retries with the same UUID are idempotent. |
| mqtt_endpoint_len | u8 | 4 | Valid MQTT endpoint hostname byte count: 1..127, without trailing zero. |
| mqtt_endpoint | u8 [128] | 5 | Service endpoint hostname, ASCII/UTF-8 bytes. |
| mqtt_resv | u8 [7] | 6 | Reserved; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/nm/wifi_provision_start_rsp"></a>

### `/cmd/nm/wifi_provision_start_rsp` Provisioning preparation result

- **Direction:** Robot → Client (Subscribe)

Provisioning preparation result. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.NmWifiProvisionStartRsp`
- **Message key:** `device/cmd/nm/wifi_provision_start_rsp`

Prepare response. result=0 means credentials and session ID passed validation; hotspot and Bluetooth remain connected. After a matching successful response, send the commit request.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| result | u8 | 1 | 0=prepared or duplicate request for the same session, 1=already committed and executing, 2=no credentials, 4=session ID mismatch. |
| timeout_sec | u8 | 2 | Actual association+DHCP timeout in seconds, bounded by device limits. Use this value to choose the waiting window rather than hard-coding one. |
| resv | u8 [6] | 3 | Reserved; set to 0 when sending and ignore when receiving. |
| total_timeout_sec | u8 | 4 | Reserved total provisioning budget in seconds, including association, DHCP, and cloud confirmation. 0=not provided (current devices always send 0). The app must add a cloud-confirmation window; timeout_sec covers only association+DHCP and is not the total budget. |
| uuid | u8 [16] | 5 | Echoed provisioning UUID; the app must verify it before accepting the response. |

---

<a id="t/cmd/nm/wifi_scan_req"></a>

### `/cmd/nm/wifi_scan_req` Scan nearby wireless networks

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.NmWifiScanReq`
- **Response timeout:** **15000 ms**
- **Message key:** `app/cmd/nm/wifi_scan_req`

force_scan=1 starts a new Wi-Fi scan, which may take seconds and briefly affect hotspot communication; force_scan=0 returns cached results.

- **Response:** [/cmd/nm/wifi_scan_rsp](#t/cmd/nm/wifi_scan_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| force_scan | u8 | 1 | 1=start a new scan (may take seconds); 0=return cached results. |
| max_count | u8 | 2 | Maximum returned AP count, truncated from strongest to weakest signal. 0 or >64 is treated as 64. |
| resv | u8 [6] | 3 | Reserved; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/nm/wifi_scan_rsp"></a>

### `/cmd/nm/wifi_scan_rsp` Wireless network scan results

- **Direction:** Robot → Client (Subscribe)

Wireless network scan results. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.NmWifiScanRsp`
- **Message key:** `device/cmd/nm/wifi_scan_rsp`

Device → app: APs sorted by RSSI, strongest first, deduplicated by SSID, with hidden SSIDs excluded. total_found > ap_count indicates truncation by max_count.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| result | u8 | 1 | 0=success, 1=scanning, 2=network interface abnormal, 3=wireless control interface abnormal, 4=scan failed. On failure, results may be cached or empty. |
| ap_count | u8 | 2 | Valid AP entries: 0..20. |
| total_found | u8 | 3 | Total deduplicated AP count, capped at 255. A value above ap_count indicates truncation. |
| resv | u8 [5] | 4 | Reserved; set to 0 when sending and ignore when receiving. |
| aps | nm_wifi_ap_info [64] | 5 | Scan results; currently at most 20 entries. |
| recent_ssid_len | u8 | 6 | Length of the most recently provisioned SSID, returned only when visible in this scan; otherwise 0. |
| recent_ssid | u8 [32] | 7 | Most recently provisioned SSID that is visible in the current scan. |
| recent_psk_len | u8 | 8 | Saved password length for recent_ssid; nonzero only when that SSID is visible in this scan. |
| recent_psk | u8 [64] | 9 | Returns only the saved password for the most recently provisioned SSID visible in the current scan. Redact passwords in logs. |

#### Compound field: nm_wifi_ap_info

| Field | Field number | Description |
| --- | --- | --- |
| ssid_len | 1 | Valid network-name byte count: 1–32. |
| ssid | 2 | Raw network-name bytes, usually UTF-8, without trailing terminator. |
| bssid | 3 | 6-byte AP MAC address (BSSID). |
| channel | 4 | Wireless channel: 1–14 for 2.4 GHz, 36 and above for 5 GHz, 0=unknown. |
| rssi | 5 | Received signal strength in dBm; 0=unavailable. |
| security | 6 | 0=open, 1=WEP, 2=WPA, 3=WPA2, 4=WPA3, 5=WPA2/WPA3 mixed. |
| resv | 7 | Reserved; ignore when receiving. |

---

<a id="t/cmd/nm/wifi_switch_to_station_req"></a>

### `/cmd/nm/wifi_switch_to_station_req` Request Wi-Fi station mode

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.NmWifiSwitchToStationReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/nm/wifi_switch_to_station_req`
- **Response:** [/cmd/nm/wifi_switch_to_station_rsp](#t/cmd/nm/wifi_switch_to_station_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| trigger_source | u8 | 1 | Request source: 0=other, 1=device button, 2=client, 3=production setup. |
| watchdog_sec | u8 | 2 | Connectivity-monitoring timeout after switching, in seconds; 0 disables this timeout monitoring. |
| resv | u8 [6] | 3 | Reserved; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/nm/wifi_switch_to_station_result_req"></a>

### `/cmd/nm/wifi_switch_to_station_result_req` Query the network-mode switch result

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.NmWifiSwitchToStationResultReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/nm/wifi_switch_to_station_result_req`

App → device: query the actual result of the most recent asynchronous switch to Wi-Fi station mode.

- **Response:** [/cmd/nm/wifi_switch_to_station_result_rsp](#t/cmd/nm/wifi_switch_to_station_result_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| resv | u8 [8] | 1 | Reserved; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/nm/wifi_switch_to_station_result_rsp"></a>

### `/cmd/nm/wifi_switch_to_station_result_rsp` Network-mode switch progress and result

- **Direction:** Robot → Client (Subscribe)

Network-mode switch progress and result. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.NmWifiSwitchToStationResultRsp`
- **Message key:** `device/cmd/nm/wifi_switch_to_station_result_rsp`

Device → app: status and actual result of the latest asynchronous station-mode switch.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| valid | u8 | 1 | 1=a previous request exists; 0=no request yet. |
| pending | u8 | 2 | 1=switch in progress; 0=result/current_mode are final. |
| result | u8 | 3 | Execution result: 0=success, 1=already in station mode, 3=hotspot stop failed, 4=wireless interface start failed, 5=network monitor start failed. Not complete while pending=1. |
| current_mode | u8 | 4 | Wi-Fi mode: 0=unknown, 1=device hotspot, 2=station ready but disconnected, 3=connected to an upstream network. |
| resv | u8 [4] | 5 | Reserved; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/nm/wifi_switch_to_station_rsp"></a>

### `/cmd/nm/wifi_switch_to_station_rsp` Acceptance response for a network-mode switch

- **Direction:** Robot → Client (Subscribe)

Acceptance response for a network-mode switch. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.NmWifiSwitchToStationRsp`
- **Message key:** `device/cmd/nm/wifi_switch_to_station_rsp`

Device → app: request acknowledgment. result=0 means an asynchronous switch was scheduled. current_mode is the mode when accepted, not confirmation of switch completion.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| result | u8 | 1 | 0=success, 1=already in station mode, 2=Bluetooth stop failed, 3=hotspot stop failed, 4=wireless interface start failed, 5=network monitor start failed. Query switch results for the final outcome. |
| current_mode | u8 | 2 | Wi-Fi mode: 0=unknown, 1=device hotspot, 2=station ready but disconnected, 3=connected to an upstream network. |
| resv | u8 [6] | 3 | Reserved; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/payload/i2c_transfer_req"></a>

### `/cmd/payload/i2c_transfer_req` Top-payload raw I²C read/write passthrough

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.PayloadI2cTransferReq`
- **Response timeout:** **3000 ms**
- **Message key:** `app/cmd/payload/i2c_transfer_req`
- **Response:** [/cmd/payload/i2c_transfer_rsp](#t/cmd/payload/i2c_transfer_rsp)

Perform a raw I²C transfer on the robot's top-payload connector. Before calling this topic, enter SDK I²C mode and enable power (`TEST_ON` or `ON`) via [`/cmd/system/payload_power_req`](#t/cmd/system/payload_power_req); when unpowered, the payload I²C service is offline and requests time out.

- Valid 7-bit slave addresses: `0x08`–`0x77` (all legal addresses).
- Both `write_len` and `read_len` range from `0` to `128` bytes, and at least one must be nonzero.
- **`write_data` must be padded to exactly 128 bytes:** even when `write_len < 128` (including read-only transfers with `write_len=0`), you must encode the full 128 bytes in Protobuf (pad trailing unused bytes with `0`), otherwise the device will not forward or respond to the request. Only the first `write_len` bytes are transmitted on the I²C bus.
- When both `write_len > 0` and `read_len > 0`, the device performs a combined write-then-read using a **repeated START** in a single I²C transaction.
- This operation is non-idempotent; do not auto-retry on timeout or error. After NACK, short transfer, or timeout, the write may already have taken effect—verify state using your peripheral's protocol.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| address | u8 | 1 | 7-bit I²C slave address (`0x08`–`0x77`). |
| write_len | u8 | 2 | Number of bytes to write (`0`–`128`); at least one of `write_len` / `read_len` must be nonzero. |
| read_len | u8 | 3 | Number of bytes to read (`0`–`128`); at least one of `write_len` / `read_len` must be nonzero. |
| write_data | u8 [128] | 4 | Fixed 128-byte write buffer (must be zero-padded to 128 bytes or the request is dropped); only the first `write_len` bytes are valid. |

---

<a id="t/cmd/payload/i2c_transfer_rsp"></a>

### `/cmd/payload/i2c_transfer_rsp` Top-payload raw I²C transfer response

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.cmd.PayloadI2cTransferRsp`
- **Message key:** `device/cmd/payload/i2c_transfer_rsp`

Returns `read_len` valid bytes on success, or 0 bytes on failure. The device does not auto-retry failed transfers.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | Result: `0`=success, `1`=invalid arguments, `2`=address busy, `3`=slave NACK, `4`=bus timeout, `5`=transfer failed. |
| read_len | u8 | 2 | Number of valid bytes read (`0`–`128`); `0` on failure. |
| read_data | u8 [128] | 3 | Read buffer; only the first `read_len` bytes are valid. |

---

<a id="t/status/pmu/osd_status"></a>

### `/status/pmu/osd_status` Power status; bottom charging contacts and Type-C power paths have independent switches.

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.PmuOsdStatus`
- **Message key:** `device/status/pmu/osd_status`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| soc | u8 | 1 | Remaining battery percentage. |
| power_bits | u8 | 2 | bit0=input_present, bit1=is_charging; bit2=bottom charging-contact path switch, bit3=Type-C power-path switch. Bits 2/3 are 1=on, 0=off, can both be on or off, and indicate neither exclusive power sources nor active charging. Bits 4–7 reserved as 0. |
| abnormal_flags | u16 | 3 | Power abnormality flags; 0=no flags asserted. Interpret bits using device power-fault definitions, not as a complete fault code. |

#### Bitfield · bit 0 is the least significant bit

| Field | Bits | Name | Meaning |
| --- | --- | --- | --- |
| power_bits | 0 | input_present | 1=external power input detected. |
| power_bits | 1 | is_charging | 1=charging. |
| power_bits | 2 | charging_contact_mos_enabled | Bottom charging-contact path: 1=on, 0=off. |
| power_bits | 3 | type_c_mos_enabled | Type-C power path: 1=on, 0=off. |
| power_bits | 4–7 | reserved0 | Reserved bits; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/rc/function_key_mode_ack"></a>

### `/cmd/rc/function_key_mode_ack` Remote-controller function-key mode response

- **Direction:** Robot → Client (Subscribe)

Remote-controller function-key mode response. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.RcFunctionKeyModeAck`
- **Message key:** `device/cmd/rc/function_key_mode_ack`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | Key-mode setting result: 0=success, nonzero=failure. |
| reserved | u8 [3] | 2 | Reserved; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/rc/function_key_mode"></a>

### `/cmd/rc/function_key_mode` Remote-controller function-key mode

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.RcFunctionKeyMode`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/rc/function_key_mode`
- **Response:** [/cmd/rc/function_key_mode_ack](#t/cmd/rc/function_key_mode_ack)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| camera_btn_mode | u8 | 1 | Target mode of the remote camera function key. Updated only when update_flag bit0 is set. Interpret using remote-controller mode definitions, not task IDs. |
| update_flag | u8 | 3 | Update selector: always 1, updating only the camera function key. |
| reserved | u16 | 4 | Reserved; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/rc/joystick_calib_req"></a>

### `/cmd/rc/joystick_calib_req` Start or stop remote-controller joystick calibration

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.RcJoystickCalibReq`
- **Message key:** `app/cmd/rc/joystick_calib_req`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| action | u8 | 1 | Calibration: 1=start, 2=stop. Progress is reported through calibration-progress messages. |
| receiver_type | u8 | 2 | Target remote-controller type; UWB remote=7. |
| receiver_index | u8 | 3 | Target address index. The four UWB remote positions use 0x01, 0x11, 0x21, 0x31. Use the actual address. |

---

<a id="t/cmd/rc/vibration_get_req"></a>

### `/cmd/rc/vibration_get_req` Query remote-controller vibration intensity

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.RcVibrationGetReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/rc/vibration_get_req`
- **Response:** [/cmd/rc/vibration_get_rsp](#t/cmd/rc/vibration_get_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| receiver_type | u8 | 1 | Target remote-controller type; UWB remote=7. |
| receiver_index | u8 | 2 | Target address index. The four UWB remote positions use 0x01, 0x11, 0x21, 0x31. Use the actual address. |

---

<a id="t/cmd/rc/vibration_get_rsp"></a>

### `/cmd/rc/vibration_get_rsp` Remote-controller vibration intensity result

- **Direction:** Robot → Client (Subscribe)

Remote-controller vibration intensity result. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.RcVibrationGetRsp`
- **Message key:** `device/cmd/rc/vibration_get_rsp`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| level | u8 | 1 | Vibration intensity: 1=low, 2=medium, 3=high. Setting responses return the device-reported intensity. |

---

<a id="t/cmd/rc/vibration_set_req"></a>

### `/cmd/rc/vibration_set_req` Set remote-controller vibration intensity

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.RcVibrationSetReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/rc/vibration_set_req`
- **Response:** [/cmd/rc/vibration_set_rsp](#t/cmd/rc/vibration_set_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| level | u8 | 1 | Vibration intensity: 1=low, 2=medium, 3=high. Setting responses return the device-reported intensity. |
| receiver_type | u8 | 2 | Target remote-controller type; UWB remote=7. |
| receiver_index | u8 | 3 | Target address index. The four UWB remote positions use 0x01, 0x11, 0x21, 0x31. Use the actual address. |

---

<a id="t/cmd/rc/vibration_set_rsp"></a>

### `/cmd/rc/vibration_set_rsp` Remote-controller vibration setting result

- **Direction:** Robot → Client (Subscribe)

Remote-controller vibration setting result. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.RcVibrationSetRsp`
- **Message key:** `device/cmd/rc/vibration_set_rsp`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| level | u8 | 1 | Vibration intensity: 1=low, 2=medium, 3=high. Setting responses return the device-reported intensity. |

---

<a id="t/status/rc/fsm_state"></a>

### `/status/rc/fsm_state` Remote-controller state-change event

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.RcFsmState`
- **Message key:** `device/status/rc/fsm_state`

Remote-controller state-machine transition telemetry

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| timestamp_ms | u32 | 1 | Remote-controller state-change timestamp in milliseconds. |
| prev_state | u8 | 2 | Main remote-controller state before the change. |
| prev_sub | u8 | 3 | Substate before the change; interpret with the main state. |
| key_id | u8 | 4 | Button ID that triggered the change; 0 for a device-state-triggered change. |
| event_type | u8 | 5 | Button-event type that triggered the change; 0 for a device-state-triggered change. |
| next_state | u8 | 6 | Main remote-controller state after the change. |
| next_sub | u8 | 7 | Substate after the change; interpret with the main state. |

---

<a id="t/status/rc/joystick_calib_progress"></a>

### `/status/rc/joystick_calib_progress` Remote-controller joystick calibration status and progress

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.RcJoystickCalibProgress`
- **Message key:** `device/status/rc/joystick_calib_progress`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| status | u8 | 1 | Calibration status: 0=in progress or complete, 1=failed. Status 0 with progress 100 means complete. |
| progress | u8 | 2 | Calibration progress: 0–100 percent. |

---

<a id="t/status/rc/joystick_raw"></a>

### `/status/rc/joystick_raw` Raw remote-controller joystick position

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.RcJoystickRaw`
- **Message key:** `device/status/rc/joystick_raw`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| x | u8 | 1 | Raw horizontal axis, interpreted as signed 8-bit two's complement; display range −100..100, center 0. |
| y | u8 | 2 | Raw vertical axis, interpreted as signed 8-bit two's complement; display range −100..100, center 0. |

---

<a id="t/cmd/robot/sn_req"></a>

### `/cmd/robot/sn_req` Query robot serial number

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.RobotSnReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/robot/sn_req`
- **Response:** [/cmd/robot/sn_rsp](#t/cmd/robot/sn_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| sn_type | u8 | 1 | Serial-number query type; use 0 for this robot. The response echoes it. |

---

<a id="t/cmd/robot/sn_rsp"></a>

### `/cmd/robot/sn_rsp` Robot serial-number response

- **Direction:** Robot → Client (Subscribe)

Robot serial-number response. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.RobotSnRsp`
- **Message key:** `device/cmd/robot/sn_rsp`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| sn_type | u8 | 1 | Serial-number query type; use 0 for this robot. The response echoes it. |
| lens | u8 | 2 | Valid bytes in sn; 0=no valid serial number obtained. |
| sn | u8 [128] | 3 | Serial-number bytes; use lens valid bytes, excluding trailing padding. |

---

<a id="t/status/robot/status"></a>

### `/status/robot/status` Robot motion status

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.RobotStatus`
- **Message key:** `device/status/robot/status`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| robot_state | u8 | 1 | Body state: 0=released/unpowered, 1=standing up, 2=folding, 3=active. Also check emergency-stop and fault states before motion. |
| velocity | i16 | 2 | Speed in cm/s (m/s = value/100.0). |
| trip_odom_dis | f32 | 3 | Accumulated distance in meters. |
| obstacle_dis | u16 | 4 | Nearest obstacle distance in cm; 0xFFFF=invalid. |
| pitch_angle | i16 | 5 | Body pitch in 0.1 degrees. |
| gimbal_angle | i16 | 6 | Gimbal angle in 0.1 degrees. |
| roll_angle | i16 | 7 | Body roll in 0.1 degrees. |
| height | u8 | 8 | Height in 0.1 cm. |
| sport_mode | u8 | 9 | Current motion-mode ID; interpret using device motion-mode definitions. |
| abnormal_flags | u32 | 10 | Body abnormality flags; 0=no abnormality reported by this field. Obtain complete fault state from HMS. |
| session_odom_dis | f32 | 11 | Distance traveled since standing up, in meters. |

---

<a id="t/data/robot/sdk_sensor_batch"></a>

### `/data/robot/sdk_sensor_batch` 25 Hz batched sensor telemetry (body IMU / gimbal IMU / joint encoders)

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.data.RobotSdkSensorBatch`
- **Message key:** `device/data/robot/sdk_sensor_batch`

25 Hz batched sensor telemetry for SDK clients (one packet every 40 ms). Subscribe directly once the DataChannel Zenoh session is up—no separate enable request is required. Each packet packs time-ordered samples of the body IMU (100 Hz, up to 4 entries), gimbal IMU (50 Hz, up to 2 entries), and 7 joint encoders (50 Hz, up to 2 entries). All `sample_ts_ns` timestamps use the device's `CLOCK_MONOTONIC` clock, sharing the same clock domain as `left_mono_ns` / `right_mono_ns` in the stereo stream's H.265 SEI for SLAM/VIO and relocalization interpolation.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| version | u8 | 1 | Batch telemetry format version; currently always `1`. |
| batch_seq | u32 | 2 | Monotonically increasing batch sequence number (+1 per 25 Hz packet) for packet-loss and ordering checks. |
| body_imu | sdk_imu_sample [4] | 3 | Body IMU samples (100 Hz, ascending time order, up to 4 entries); see compound field below. |
| gimbal_imu | sdk_imu_sample [2] | 4 | Gimbal IMU samples (50 Hz, ascending time order, up to 2 entries); see compound field below. |
| joints | sdk_joint_sample [2] | 5 | 7-joint encoder samples (50 Hz, ascending time order, up to 2 entries; index 6 is the gimbal yaw motor); see compound field below. |

#### Compound field: sdk_imu_sample

| Field | Field number | Description |
| --- | --- | --- |
| sample_ts_ns | 1 | `u64`, device monotonic sample timestamp in ns. |
| gyro | 2 | `f32 [3]`, 3-axis angular velocity `[x, y, z]` in rad/s. |
| accel | 3 | `f32 [3]`, 3-axis linear acceleration `[x, y, z]` in m/s². |
| euler | 4 | `f32 [3]`, attitude Euler angles `[roll, pitch, yaw]` in rad. |
| online | 5 | `u8`, sensor online flag: `0`=offline, `1`=online. |

#### Compound field: sdk_joint_sample

Joint array order is fixed as: `0=L1 (left hip)`, `1=L2 (left knee)`, `2=LW (left wheel)`, `3=R1 (right hip)`, `4=R2 (right knee)`, `5=RW (right wheel)`, `6=GIMY (gimbal yaw motor)`.

| Field | Field number | Description |
| --- | --- | --- |
| sample_ts_ns | 1 | `u64`, device monotonic sample timestamp in ns. |
| position | 2 | `f32 [7]`, 7 joint encoder positions (output shaft angle) in rad; index 6 is the gimbal yaw motor. |
| velocity | 3 | `f32 [7]`, 7 joint output shaft angular velocities in rad/s. |
| online_mask | 4 | `u8`, joint online bitmask (`bit0`–`bit6` correspond to the 7 joints in order: `1`=online, `0`=offline). |

---

<a id="t/status/scheduler/status"></a>

### `/status/scheduler/status` Device task and motion status

- **Direction:** Robot → Client (Subscribe)

[View sub_mode values for each mission_id](#t/cmd/mission/request)

[Camera modes and action requirements](#camera-mode-guide)

- **Protobuf message:** `md.protocol.status.SchedulerStatus`
- **Message key:** `device/status/scheduler/status`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| current_mode | u8 | 1 | Current task ID. This reference exposes 1=Camera, 3=Pet. Do not interpret other values as known tasks. |
| sub_mode | u8 | 2 | Current task submode. Camera: 0=Manual, 1=Spotlight, 2=Follow; interpret other tasks separately. |
| request_source | u8 | 3 | Mode-request source: 0=client, 1=remote controller, 2=voice, 3=autonomous behavior, 4=safety handling. |
| is_locked | u8 | 4 | Posture state: 0=standing, 1=not standing, transitioning, or abnormal. Reports actual state, not the last button intent. |
| is_paused | u8 | 5 | Current emergency-stop state: 0=normal, 1=emergency stop. |
| timestamp_ns | u64 | 6 | Monotonic timestamp in ns when status was generated. |
| sub_mode_state | u8 [14] | 7 | 14-byte task/submode extended state; see the corresponding task-request layout. Byte 14 (index 13) is reserved as 0. |
| ego_active | u8 | 8 | Autonomous interaction active: 0=no, 1=yes; separate from task ID. |
| obstacle_avoidance_flags | u8 | 9 | Obstacle-avoidance flags: bit0=requested, bit1=effective, bit2=user control intervened, bit3=four-state representation supported, bits4–7=0. |

---

<a id="t/cmd/speaker/play_cmd"></a>

### `/cmd/speaker/play_cmd` Play a built-in speaker sound effect

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.SpeakerPlayCmd`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/speaker/play_cmd`
- **Response:** [/cmd/speaker/play_rsp](#t/cmd/speaker/play_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| sound_effect_id | u8 | 1 | Built-in sound ID: 0 or 1; not a file path. |

#### Built-in speaker sound effects

Use sound_effect_id=0 or 1. No need to enable voice communication first. Duplicate requests for the same effect within 1000 ms are deduplicated.

---

<a id="t/cmd/speaker/play_rsp"></a>

### `/cmd/speaker/play_rsp` Acceptance response for sound-effect playback

- **Direction:** Robot → Client (Subscribe)

Acceptance response for sound-effect playback. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.SpeakerPlayRsp`
- **Message key:** `device/cmd/speaker/play_rsp`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | 0=accepted, 1=invalid sound ID, 2=submission failed, 3=duplicate suppressed, 4=invalid format. Acceptance does not mean playback completed. |

ret_code: 0=accepted, 1=invalid sound ID, 2=submission failed, 3=duplicate suppressed, 4=invalid request format. Acceptance does not mean playback completed.

---

<a id="t/status/storage/status"></a>

### `/status/storage/status` Storage status

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.StorageStatus`
- **Message key:** `device/status/storage/status`

Storage status. sd_status exposes only 0..3; USB mass-storage use and formatting are both reported as UNMOUNTED. This alone cannot distinguish a missing card from storage held by a computer. msd_active separately indicates export to a host so consumers such as recording admission can distinguish the cause.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| internal_used_space | u32 | 1 | Used internal-storage capacity in KiB (1024 bytes). |
| internal_total_space | u32 | 2 | Total internal-storage capacity in KiB (1024 bytes). |
| sd_used_space | u32 | 3 | Used SD-card capacity in KiB (1024 bytes). |
| sd_total_space | u32 | 4 | Total SD-card capacity in KiB (1024 bytes). |
| storage_type | u8 | 5 | Selected storage: 0=internal, 1=SD card. |
| sd_status | u8 | 6 | SD status: 0=not inserted, 1=available, 2=unmounted/unavailable locally, 3=abnormal. Check msd_active for computer use. |
| sd_transfer_speed | u8 | 7 | SD speed class: 0=low-speed, 1=high-speed; not real-time throughput. |
| sd_fs_type | u8 | 8 | SD filesystem: 0=unformatted, 1=exFAT, 2=FAT32, 3=unknown. |
| internal_threshold_alert | u16 | 9 | Internal-space warning: 0=normal, 1=space threshold warning. |
| sd_threshold_alert | u16 | 10 | SD-space warning: 0=normal, 1=space threshold warning. |
| internal_exported_to_host | u8 | 11 | Internal storage exported to a computer: 0=no, 1=yes. Do not assume normal local writes during export. |
| msd_active | u8 | 12 | Storage exported as a USB mass-storage device: 1=in use by host; sd_status is then reported as UNMOUNTED. |
| recording_min_free_space_mb | u32 | 13 | Minimum free space for recording start/stop, in MB (1000000 bytes). 0=not provided; existing storage-space limits still apply. |

---

<a id="t/cmd/system/payload_power_req"></a>

### `/cmd/system/payload_power_req` Top-payload power control and SDK I²C mode switch

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.SystemPayloadPowerReq`
- **Response timeout:** **3000 ms**
- **Message key:** `app/cmd/system/payload_power_req`
- **Response:** [/cmd/system/payload_power_rsp](#t/cmd/system/payload_power_rsp)

Controls power and SDK I²C mode for the robot's top-payload connector.

- On boot, the device defaults to `AUTO` (`0`, normal user mode), automatically powering official accessories based on insertion detection. Use official accessory APIs directly without entering SDK I²C mode.
- While an SDK session is active, send `mode=1/2/3` (`OFF` / `TEST_ON` / `ON`) to enter SDK I²C mode: active polling stops and the device only serves raw transfers via [`/cmd/payload/i2c_transfer_req`](#t/cmd/payload/i2c_transfer_req).
- `TEST_ON` (`mode=2`) powers on regardless of insertion detection for `duration_ms=1..60000`, then stops the payload service and powers off automatically; `ON` (`mode=3`) powers on indefinitely; `OFF` (`mode=1`) stops the payload service and powers off; `AUTO` (`mode=0`) explicitly exits SDK I²C mode. Set `duration_ms=0` for all modes other than `TEST_ON` (both fields must be encoded in Protobuf, even when 0).
- The device automatically reverts to `AUTO` when the SDK session disconnects or control expires.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| duration_ms | u32 | 1 | Power-on duration for `TEST_ON` (`1`–`60000` ms); set to `0` for all other modes. |
| mode | system_payload_power_mode | 2 | Target mode: `0`=AUTO, `1`=OFF, `2`=TEST_ON, `3`=ON. |

---

<a id="t/cmd/system/payload_power_rsp"></a>

### `/cmd/system/payload_power_rsp` Top-payload power control response

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.cmd.SystemPayloadPowerRsp`
- **Message key:** `device/cmd/system/payload_power_rsp`

Echoes the request sequence number. `result=0` (`APPLIED`) confirms the power switch was executed; retransmitting with the same sequence number returns the current state without restarting the timer. Use [`/cmd/system/payload_power_status_req`](#t/cmd/system/payload_power_status_req) to inspect payload service status.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| result | system_payload_power_result | 1 | Result: `0`=APPLIED, `1`=INVALID, `2`=BUSY, `3`=IO_ERROR (hardware I/O failure), `4`=STALE, `5`=DENIED (not in SDK control state). |
| mode | system_payload_power_mode | 2 | Current power mode: `0`=AUTO, `1`=OFF, `2`=TEST_ON, `3`=ON. |
| inserted | u8 | 3 | Latest valid insertion reading: `0`=not inserted, `1`=inserted. |
| enabled | u8 | 4 | Power enable readback: `0`=off, `1`=on, `2`=unknown. |

---

<a id="t/cmd/system/payload_power_status_req"></a>

### `/cmd/system/payload_power_status_req` Query top-payload power status

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.SystemPayloadPowerStatusReq`
- **Response timeout:** **3000 ms**
- **Message key:** `app/cmd/system/payload_power_status_req`
- **Response:** [/cmd/system/payload_power_status_rsp](#t/cmd/system/payload_power_status_rsp)

Takes no payload fields; send a nonzero `header.seq_num` with `data` encoded as an empty submessage (`12 00`).

---

<a id="t/cmd/system/payload_power_status_rsp"></a>

### `/cmd/system/payload_power_status_rsp` Top-payload power status response

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.cmd.SystemPayloadPowerStatusRsp`
- **Message key:** `device/cmd/system/payload_power_status_rsp`

Returns the latest top-payload power and payload service state from the device monitor cycle.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| remaining_ms | u32 | 1 | Remaining milliseconds until `TEST_ON` expires (rounded up); `0` for other modes or once expired. |
| mode | system_payload_power_mode | 2 | Current power mode: `0`=AUTO, `1`=OFF, `2`=TEST_ON, `3`=ON. |
| inserted | u8 | 3 | Latest valid insertion reading: `0`=not inserted or detection error, `1`=inserted. |
| enabled | u8 | 4 | Power enable readback: `0`=off, `1`=on, `2`=unknown. |
| daemon_running | u8 | 5 | `1`=latest monitor check confirmed the payload service is running or starting, `0`=not confirmed. |

---

<a id="t/cmd/time/correction_req"></a>

### `/cmd/time/correction_req` Correct the robot system time

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.TimeCorrectionReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/time/correction_req`
- **Response:** [/cmd/time/correction_rsp](#t/cmd/time/correction_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| timestamp | u64 | 1 | Client-provided Unix timestamp in milliseconds. |
| offset | u32 | 2 | Additional correction in milliseconds. Target time=timestamp+offset. This is not a timezone offset; use 0 for no correction. |

---

<a id="t/cmd/time/correction_rsp"></a>

### `/cmd/time/correction_rsp` System-time correction response

- **Direction:** Robot → Client (Subscribe)

System-time correction response. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.TimeCorrectionRsp`
- **Message key:** `device/cmd/time/correction_rsp`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | 0=request handled successfully (possibly rate-limited without another adjustment); nonzero=failure. This does not establish synchronization accuracy. |

---

<a id="t/cmd/tracking/ctrl_cmd"></a>

### `/cmd/tracking/ctrl_cmd` Select or cancel a visual tracking target

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.TrackingCtrlCmd`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/tracking/ctrl_cmd`

Target-tracking control (app → robot, request/response).

- **Response:** [/cmd/tracking/ctrl_cmd_ack](#t/cmd/tracking/ctrl_cmd_ack)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| selected_id | u8 | 1 | Selected target ID. Use an ID from target reports; the response returns the confirmed ID. |
| start_tracking | u8 | 2 | Tracking operation: 1=start, 0=cancel. The response echoes the confirmed operation. |
| selected_pos_x | f32 | 3 | Normalized horizontal target-center coordinate, 0–1, with 0 at the left edge of the actual video image, excluding letterboxing. |
| selected_pos_y | f32 | 4 | Normalized vertical target-center coordinate, 0–1, with 0 at the top. |
| request_mode | u8 | 5 | Selection mode: 0=user selection, 1=assisted Spotlight. User selection also requires correct target coordinates. |
| reserved1 | u8 | 6 | Reserved; set to 0 when sending and ignore when receiving. |
| reserved2 | u8 | 7 | Reserved; set to 0 when sending and ignore when receiving. |
| reserved3 | u8 | 8 | Reserved; set to 0 when sending and ignore when receiving. |

#### User-selected visual tracking

selected_pos_x and selected_pos_y are normalized coordinates within the actual video image (f32, 0–1), with origin at top left and increasing right/down. For a reported bounding box, copy id, center_pos_x, center_pos_y into selected_id, selected_pos_x, selected_pos_y. Coordinates are already normalized; do not divide by resolution again. Exclude letterbox margins before mapping mouse clicks.

request_mode: 0=USER (user selection), 1=ASSIST_SPOTLIGHT (device-assisted Spotlight by tracking ID). User selection uses 0 and matches by position, so correct coordinates are required even with a nonzero ID. Start with start_tracking=1. Cancel with 0, keeping the target ID and zeroing both coordinates. Set reserved1/2/3 to 0 for both operations, as the app does.

---

<a id="t/cmd/tracking/ctrl_cmd_ack"></a>

### `/cmd/tracking/ctrl_cmd_ack` Target-tracking request result

- **Direction:** Robot → Client (Subscribe)

Target-tracking request result. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.TrackingCtrlCmdAck`
- **Message key:** `device/cmd/tracking/ctrl_cmd_ack`

Target-tracking control acknowledgment (robot → app).

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| result | u8 | 1 | 0=success, 1=invalid target ID, 2=target unavailable or lost. |
| selected_id | u8 | 2 | Selected target ID. Use an ID from target reports; the response returns the confirmed ID. |
| start_tracking | u8 | 3 | Tracking operation: 1=start, 0=cancel. The response echoes the confirmed operation. |
| reserved | u8 | 4 | Reserved; set to 0 when sending and ignore when receiving. |

#### Display the device response

Match the original request using header.seq_num. result: 0=success, 1=invalid ID, 2=target unavailable/lost. Display returned selected_id and start_tracking (1=start, 0=cancel). For bounding-box selection, mark accepted only when successful and both ID and operation match. Subscribe to device/status/tracking/status for ongoing tracking status.

---

<a id="t/cmd/tracking/param_query_req"></a>

### `/cmd/tracking/param_query_req` Query target-tracking parameters

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.TrackingParamQueryReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/tracking/param_query_req`
- **Response:** [/cmd/tracking/param_query_rsp](#t/cmd/tracking/param_query_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| param_id | u8 | 1 | Parameter ID: 1=reference human height, 2=user switch for remote-controller positioning in Follow. |
| reserved | u8 [3] | 2 | Reserved; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/tracking/param_query_rsp"></a>

### `/cmd/tracking/param_query_rsp` Target-tracking parameter response

- **Direction:** Robot → Client (Subscribe)

Target-tracking parameter response. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.TrackingParamQueryRsp`
- **Message key:** `device/cmd/tracking/param_query_rsp`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| param_id | u8 | 1 | Parameter ID: 1=reference human height, 2=user switch for remote-controller positioning in Follow. |
| ret_code | u8 | 2 | 0=success, 1=unknown parameter; do not use param_value on failure. |
| reserved | u8 | 3 | Reserved; set to 0 when sending and ignore when receiving. |
| param_value | f32 | 4 | ID1: height in meters, range 0.1–3.0. ID2: only 0.0=off or 1.0=on. ID2 queries return the user's choice; read uwb_follow_enabled for effective state. |

---

<a id="t/cmd/tracking/param_update_req"></a>

### `/cmd/tracking/param_update_req` Set target-tracking parameters

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.TrackingParamUpdateReq`
- **Response timeout:** **5000 ms**
- **Message key:** `app/cmd/tracking/param_update_req`
- **Response:** [/cmd/tracking/param_update_rsp](#t/cmd/tracking/param_update_rsp)

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| param_id | u8 | 1 | Parameter ID: 1=reference human height, 2=user switch for remote-controller positioning in Follow. |
| param_value | f32 | 2 | ID1: height in meters, range 0.1–3.0. ID2: only 0.0=off or 1.0=on. ID2 queries return the user's choice; read uwb_follow_enabled for effective state. |
| reserved | u8 [3] | 3 | Reserved; set to 0 when sending and ignore when receiving. |

---

<a id="t/cmd/tracking/param_update_rsp"></a>

### `/cmd/tracking/param_update_rsp` Target-tracking parameter setting response

- **Direction:** Robot → Client (Subscribe)

Target-tracking parameter setting response. Sent by the robot. Subscribe to this message key and match the response by current session and request sequence. Distinguish acceptance from completed execution as specified in this entry.

- **Protobuf message:** `md.protocol.cmd.TrackingParamUpdateRsp`
- **Message key:** `device/cmd/tracking/param_update_rsp`

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | 0=accepted, 1=unknown parameter, 2=invalid value. Success does not guarantee all settings survive reboot. |
| reserved | u8 [3] | 2 | Reserved; set to 0 when sending and ignore when receiving. |

---

<a id="t/data/tracking/target_data"></a>

### `/data/tracking/target_data` Available visual targets

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.data.TrackingTargetData`
- **Message key:** `device/data/tracking/target_data`

Target detections (robot → app, one-way push).

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| target_count | u8 | 1 | Valid target count, at most 5. |
| targets | tracking_target_info [5] | 2 | Target list; first target_count entries are valid. Each contains ID, center coordinates, bounding-box dimensions, and tracking state; see the compound definition. |

For bounding-box selection, use the target's id and normalized center_pos_x/center_pos_y directly in the tracking request.

#### Compound field: tracking_target_info

| Field | Field number | Description |
| --- | --- | --- |
| id | 1 | Target ID: 0–255. |
| center_pos_x | 2 | Normalized horizontal bounding-box center: 0–1. |
| center_pos_y | 3 | Normalized vertical bounding-box center: 0–1. |
| width | 4 | Bounding-box width in pixels. |
| height | 5 | Bounding-box height in pixels. |
| target_state | 6 | 0=ordinary candidate, 1=user's tracked target, 2=user's tracked target lost. |
| reserved_state_flags | 7 | Reserved status bits; ignore on receipt. |

---

<a id="t/status/tracking/status"></a>

### `/status/tracking/status` Tracking status, distance, and speed

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.TrackingStatus`
- **Message key:** `device/status/tracking/status`

Tracking status (robot → app, one-way push).

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| status | u8 | 1 | Tracking state: 0=idle, 1=tracking, 2=target lost, 255=error. |
| tracking_id | u8 | 2 | Tracked target ID; 255=no target. |
| target_type | u8 | 3 | Target type: 0=person, 1=dog. |
| horizontal_dis | u16 | 4 | Horizontal target distance in cm; 65535=invalid. |
| horizontal_speed | u16 | 5 | Horizontal target speed in cm/s; 65535=invalid. |
| uwb_follow_enabled | u8 | 6 | Effective participation of remote-controller positioning in Follow: 1=participating, 0=not participating. |

---

<a id="t/status/uwb/rc_status_list"></a>

### `/status/uwb/rc_status_list` Online remote-controller battery, charging state, and signal strength

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.status.UwbRcStatusList`
- **Message key:** `device/status/uwb/rc_status_list`

Complete status snapshot of online UWB remotes for accessory/control displays. Sent every 1 second, with no extra send on state changes. Includes only online remotes with valid battery values; offline entries disappear from later snapshots. No snapshot or empty list is sent when there are no valid online entries. Receivers expire old state after a push timeout. Each snapshot replaces the list, at most 4 entries; (receiver_type, receiver_index) pairs must be unique.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| entries | uwb_rc_status_entry [4] | 1 | At most 4 online remotes, with battery, charging state, signal strength, address, and final 6 serial-number characters; see the compound definition. |

#### Compound field: uwb_rc_status_entry

| Field | Field number | Description |
| --- | --- | --- |
| battery | 1 | Battery percentage 0–100; 0 means actual 0%, not unknown. |
| is_charging | 2 | 0=not charging, 1=charging. |
| rssi | 3 | Received signal strength in dBm; 0=no valid value. |
| receiver_type | 4 | Remote-controller type; current UWB remote=7. |
| receiver_index | 5 | Address index; four positions use 0x01, 0x11, 0x21, 0x31. |
| sn_suffix | 6 | Last 6 serial-number characters, for display only, not a unique identifier. |

---

<a id="t/cmd/vt/sdk_preview_req"></a>

### `/cmd/vt/sdk_preview_req` Query or set video preview for the current SDK session (main / stacked stereo)

- **Direction:** Client → Robot (Publish)
- **Protobuf message:** `md.protocol.cmd.VtSdkPreviewReq`
- **Response timeout:** **3000 ms**
- **Message key:** `app/cmd/vt/sdk_preview_req`
- **Response:** [/cmd/vt/sdk_preview_rsp](#t/cmd/vt/sdk_preview_rsp)

Query or toggle video preview streams for the current SDK session without affecting motion control. All streams default to off on a new session; set `session_id` to the first 16 bytes of the 17-byte `beni/session` payload.

- **`version=1` (main camera only):** `operation=0` query or `1` set; `enabled=0` off or `1` on (also disables the stereo stream).
- **`version=2` (dual-track main + stereo):** when `operation=1`, select streams via `stream_mask`: `bit0` (`1`)=main camera, `bit1` (`2`)=stacked stereo pair (`640×960`: top `640×480` left eye + bottom `640×480` right eye). Bits can be combined (e.g., `3` enables both; `0` disables all).

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| version | u8 | 1 | Preview protocol version: `1`=main camera only, `2`=select main/stereo streams via `stream_mask`. |
| operation | u8 | 2 | `0`=query current preview state, `1`=set preview switch. |
| enabled | u8 | 3 | For `version=1` with `operation=1`: `0`=off, `1`=on (use `0` for queries). For `version=2`, must still be `0` or `1` (actual selection follows `stream_mask`). |
| session_id | u8 [16] | 4 | 16-byte control-session ID from the first 16 bytes of `beni/session`. Obtain a fresh ID after reconnecting. |
| stream_mask | u8 | 5 | Optional (default `0`). Target stream bitmask for `version=2`: `bit0`=main camera, `bit1`=stacked stereo (`640×960`); all other bits must be `0`. |

#### Stacked Stereo WebRTC Track & In-Stream SEI Metadata

- **Dual-track SDP negotiation:** To receive the stereo stream, include **2 H.265 receive video tracks** in your WebRTC Offer (offering 1 track negotiates only the main camera). The robot binds the first track to the main camera (stream ID `beni`) and the second track to the stereo pair (stream ID `beni-stereo`, default 30 fps / 1500 kbps, capped at 2 Mbps, lower bitrate priority than the main camera).
- **Per-frame SEI timestamps:** Each stereo H.265 access unit begins with an Annex-B `Prefix SEI` (`payloadType=0x05` `user_data_unregistered`, 50-byte payload): a 16-byte UUID `"MDSTEREOPAIRV1\0\1"` (`4D 44 53 54 45 52 45 4F 50 41 49 52 56 31 00 01`) followed by 34 little-endian bytes:

| Byte offset (after UUID) | Field | Type | Description |
| --- | --- | --- | --- |
| `0` | `version` | `u8` | Always `1`. |
| `1` | `layout` | `u8` | Always `1` (top-bottom stacked: top `640×480` left eye + bottom `640×480` right eye). |
| `2`–`5` | `left_seq` | `u32` | Raw monotonic left-eye sensor frame sequence number. |
| `6`–`9` | `right_seq` | `u32` | Raw monotonic right-eye sensor frame sequence number. |
| `10`–`17` | `left_mono_ns` | `u64` | Left-eye exposure timestamp (`CLOCK_MONOTONIC`, ns), sharing the same clock domain as [`/data/robot/sdk_sensor_batch`](#t/data/robot/sdk_sensor_batch). |
| `18`–`25` | `right_mono_ns` | `u64` | Right-eye exposure timestamp (`CLOCK_MONOTONIC`, ns). |
| `26`–`33` | `unix_ns` | `u64` | Corresponding Unix timestamp (ns). |

---

<a id="t/cmd/vt/sdk_preview_rsp"></a>

### `/cmd/vt/sdk_preview_rsp` Current-session preview switch response

- **Direction:** Robot → Client (Subscribe)
- **Protobuf message:** `md.protocol.cmd.VtSdkPreviewRsp`
- **Message key:** `device/cmd/vt/sdk_preview_rsp`

On success, `enabled` and `stream_mask` report the confirmed stream state. Disabling stops only the corresponding SDK video stream(s); in-flight frames may still arrive and the DataChannel remains open.

| Field | Type / length | `data` tag | Description |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | `0`=success, `1`=session unauthorized/expired or target media track not ready, `2`=invalid parameters, `3`=stereo stream not allowed. |
| enabled | u8 | 2 | Confirmed aggregate preview state: `0`=all streams off, `1`=at least one stream active; valid only on success. |
| stream_mask | u8 | 3 | Optional (default `0`). Active stream bitmask: `bit0`=main camera, `bit1`=stacked stereo (`640×960`). |
