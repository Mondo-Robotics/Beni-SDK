# 仅限授权内测用户｜公开版本暂不可用

## MONDO SDK 当前处于内部测试阶段。本文介绍的 **App 入口和相关固件能力尚未在公开发布版本中开放**，仅获得测试权限的用户可使用。

## **希望参与测试，请加入我们的 [Discord 社群](https://discord.gg/smcZyv5efa)，等待后续测试开放通知及权限安排。**

# Beni SDK 开放协议手册

> 85 项开放协议 · 标准 Zenoh + Protobuf · 局域网开发参考

[English version](protocol-reference.en.md) ｜ [MONDO SDK 使用简介](<MONDO SDK使用简介.md>) ｜ [HMS 故障码参考表](<HMS 故障码参考表.md>)

---

## 快速接入

1. 将开发机和机器人连到同一局域网。
2. 在 Mondo App 中生成并导出 SDK 凭据 JSON，给机器人配网后打开 SDK 功能（详见 [使用简介](<MONDO SDK使用简介.md>)）。
3. 使用凭据里的 `local_ip`（或机器人当前局域网 IP）连接：**TCP 1024** 仅用于 TLS 1.3 PSK 鉴权和 SDP 交换，完成后关闭；业务消息和 H.265 视频通过 **WebRTC / UDP 1024** 传输。

<details>
<summary>展开查看完整建连、收发与视频订阅流程</summary>

### 1. 配对鉴权与信令（TCP 1024）

1. 确保 App 已开启 SDK 且凭据有效。机器人同一时间**只接受一个 SDK 客户端连接**。
2. 连接机器人当前局域网 IPv4 的 `TCP 1024`，执行 **TLS 1.3 外部 PSK** 握手：
   - 双方用自己的 X25519 私钥和对端公钥计算 32 字节共享秘密；
   - 使用 **HKDF-SHA256**（`salt` = 原始 16 字节 `pair_id`，`info` = ASCII `Beni LAN TLS 1.3 external PSK v1`）派生 32 字节 PSK；
   - PSK 身份标识固定为 `Beni-SDK-v1`，加密套件固定为 `TLS_AES_128_GCM_SHA256`（需启用临时密钥交换，无需 CA 或设备证书）；
   - 客户端须确认握手选用了外部 PSK 模式。TLS 握手超时 3 秒，整个信令阶段总超时 30 秒。

### 2. 建立 WebRTC 连接（UDP 1024）

3. 客户端创建一个 **H.265 接收视频轨道**（若需同时接收双目拼接流，则创建 **2 条 H.265 接收视频轨道**，第一条为主摄 `beni`、第二条为双目 `beni-stereo`），以及一个名为 `beni` 的**可靠、有序、二进制 DataChannel**（不设置消息寿命或最大重传次数）。
4. 等本地 ICE 候选收集完成后，在已鉴权的 TLS 连接内建立 Zenoh 会话：**先订阅 `device/sdk/webrtc/answer`，再向 `app/sdk/webrtc/offer` 发布**完整 UTF-8 SDP offer（`encoding` = `application/octet-stream`，这两条信令无需带业务会话 attachment）。
5. 机器人返回完整 UTF-8 SDP answer，双方按已认证 SDP 的指纹校验 DTLS 证书。SDP 交换完成后机器人关闭 TCP 1024。
6. 机器人只提供局域网 IPv4/UDP 1024 候选地址（不使用公网 STUN/TURN 或 ICE TCP），强制 BUNDLE 与 RTCP-mux。SDP 结束后 **30 秒内**若未打开 DataChannel，机器人将回收本次连接。

### 3. 收发业务消息（DataChannel）

7. DataChannel 打开后，在其中建立业务 Zenoh 会话并订阅 `beni/session`。机器人每 250 ms 推送一次 17 字节消息：**前 16 字节为本次连接 ID，最后 1 字节保留为 0**。后续发往机器人的每条业务 PUT 消息都必须在 `attachment` 中携带这 16 字节连接 ID。
8. 发送键为 `app` + 规范 topic，接收键为 `device` + 规范 topic，业务载荷统一使用 Protobuf（`encoding` = `application/protobuf`）。
9. 带回执的请求填写非零 `header.seq_num`，按当前会话、响应键和原序号匹配回执；单向摇杆消息（`/cmd/rc/joystick`）必须**完全省略 `header`**。
10. 状态上报可能存在采样间隔，请求也可能超时；禁止自动重试的动作不得盲目重发。会话断开后必须重新鉴权、交换 SDP 并获取新的连接 ID，不能沿用旧会话 ID 或旧请求序号。

### 4. 视频预览（WebRTC H.265 轨道）

11. 新会话**默认关闭预览**。先订阅 `device/cmd/vt/sdk_preview_rsp`，再向 `app/cmd/vt/sdk_preview_req` 发送请求：`version=1` 控制主摄（`operation=0` 查询、`1` 设置，`enabled=0` 关、`1` 开），或 `version=2` 配合 `stream_mask` 按位选择主摄（`bit0`）与双目上下拼接 640×960（`bit1`），`session_id` 填当前 16 字节连接 ID。
12. H.265 视频通过 WebRTC 视频轨道（RTP/SRTP）传输，由 RTCP 负责接收质量与关键帧反馈。开启或恢复预览时等待关键帧后再解码。
13. 关闭预览后机器人停止发送新帧（已在途的少量帧可能仍会到达），DataChannel 业务连接不受影响。整个 WebRTC 会话断开后，重连需再次显式开启预览。

### 5. 授权与控制安全

14. 授权在会话期间持续校验。关闭 SDK、撤销或更换凭据都会立即断开当前会话。连接成功不代表每条指令都会执行，仍受设备当前模式、状态和安全条件约束。
15. 摇杆需发送完整 64 位值，超过 **500 ms** 无更新按失联处理。真机运动请务必由现场人员看护操作。

</details>

---

<a id="camera-mode-guide"></a>

## 相机模式下的控制与动作条件

先订阅 [调度状态](#t/status/scheduler/status)：当 `current_mode=1` 时表示处于相机任务，再看 `sub_mode` 区分当前子模式：

| 子模式 | 说明与可用动作 |
| --- | --- |
| `0` · 手动（Manual） | 摇杆控制行驶，`jump_cmd` / `flip_cmd` 可触发跳跃和空翻。注意：如果机器人正处于自主互动接管或回放回退状态，手动动作可能被拦截，不能只凭 `sub_mode=0` 断定动作一定会执行。 |
| `1` · 聚焦（Spotlight） | 云台锁定目标，支持摇杆驾驶、跳跃和空翻（同样受设备状态和动作冷却限制）。选中目标的响应仅表示请求已受理，是否切入聚焦请以调度状态为准。 |
| `2` · 跟随（Follow） | 运动由跟随策略接管；具备自动跳跃能力的策略会自行触发跳跃，不接受手动/聚焦模式下的通用手动跳跃和空翻指令。 |

**发送运动指令前建议检查**：
- 机身状态为 `Active` 且 `abnormal_flags=0`；
- 调度状态显示已起身（`is_locked=0`）、未急停（`is_paused=0`），且处于相机任务的手动或聚焦子模式。
- 摇杆指令为单向发布，跳跃和空翻没有单独的执行回执，请通过后续状态上报确认结果；静止状态下也可以请求普通跳跃。按键发帧节奏和冷却时间见 [`/cmd/rc/joystick`](#t/cmd/rc/joystick)。

**低电量限制**：
当机器人触发低于 25% 或低于 10% 的电量告警时，会禁止新的跳跃和前/后/左/右四向空翻。请通过电源状态和 HMS 故障状态确认告警状态，充电并确认告警解除后再发送跳跃或空翻指令。

---

## Protobuf 封装约定

所有业务消息的外层均为两层结构：`header`（tag 1）与 `data`（tag 2）。下文各协议表格中的字段编号均指 **`data` 内部的 Protobuf 字段编号**（仅列出对外传输字段）。无参数请求的 `data` 须编码为空子消息（`12 00`）；`/cmd/rc/joystick` 必须完全省略 `header`。[查看编码示例](#t/cmd/rc/joystick)。视频画面通过 WebRTC H.265 媒体轨道传输，不走 Protobuf。

---

<a id="t/cmd/rc/joystick"></a>

### `/cmd/rc/joystick` 遥控摇杆指令

- **方向**：客户端 → 机器人（发布）

[查看 Camera 模式与动作适用条件](#camera-mode-guide)

- **Protobuf 消息**：`md.protocol.cmd.RcJoystick`

**编码示例（仅用于校验序列化）**：`data.joystick = 1000`，整个 Protobuf `header` 必须省略，空 header 也会被固件拒绝。

```text
12 03 08 E8 07
```

以上字节是完整 Protobuf 载荷。Zenoh key、encoding 和 16 字节会话 attachment 另行设置。

- **消息键**：`app/cmd/rc/joystick`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| joystick | u64 | 1 | 64 位位域，具体分配见下表。 |

#### 位域 · bit 0 为最低位

| 字段 | 位 | 名称 | 含义 |
| --- | --- | --- | --- |
| joystick | 0–10 | channel0 | 左摇杆上下->前进后退速度 |
| joystick | 11–21 | channel1 | 左摇杆左右->转向速度 |
| joystick | 22–32 | channel2 | 屏幕左右->云台偏航 |
| joystick | 33–38 | channel3 | 俯仰输入，0..62 中值31；由 bit60 区分角度参考或瞬时杆量 |
| joystick | 39–44 | roll | 横滚调整 |
| joystick | 45–49 | height | 高度调整 |
| joystick | 50 | jump_cmd | 跳跃指令 |
| joystick | 51–53 | flip_cmd | 滑动空翻 |
| joystick | 54–55 | max_speed | 挡位 最大速度 |
| joystick | 56–57 | nav_flag | 导航等级：0关闭，1避障，2自动跳跃 |
| joystick | 58 | ispause | 暂停 |
| joystick | 59 | islock | 0 请求起身，1 请求收纳 |
| joystick | 60 | pitch_input_is_stick | 0=旧版角度参考，1=瞬时俯仰杆量；确认支持版本2后，手动模式用于俯仰角速度控制，聚焦和跟随模式用于调整构图。 |
| joystick | 61 | third_person_mode | 第三人称模式 |
| joystick | 62 | jump_mode | 跳跃模式 |
| joystick | 63 | no_liveview | 图传开关(反转) |

#### 快捷键触发跳跃与空翻（相机手动／聚焦）

flip_cmd（bit 51–53）的字段值：0 无动作，1 前空翻，2 后空翻，3 左空翻，4 右空翻。空翻单独触发，jump_cmd 和 jump_mode 均填 0。普通跳将 jump_cmd（bit 50）置 1，jump_mode（bit 62）填 0；立定跳远在同一帧把两位置 1。立定跳远要求期望及实际平移速率均小于 0.1 m/s，否则按普通跳处理。

建议以 50 Hz 发送控制消息，每次按下触发一次，忽略键盘长按重复事件。跳跃连续 10 帧置位，第 11 帧清零；空翻连续 9 帧保持方向，第 10 帧清零。只改动作位，保留其余摇杆状态；设备可能在收到前几帧时就执行动作。相机任务接受手动跳跃后冷却约 0.42 秒，空翻后约 2.5 秒；冷却期间的新动作不会排队。

---

<a id="t/cmd/audio/op_req"></a>

### `/cmd/audio/op_req` 开启或关闭实时通话

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.AudioOpReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/audio/op_req`
- **对应回执**：[/cmd/audio/op_rsp](#t/cmd/audio/op_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| operation_type | u8 | 1 | 通话操作：0=开启，1=关闭。本手册不开放其他操作值。 |

#### 通话开关

电脑向机器人讲话时，先发送 operation_type=0（START_RECORD），等 device/cmd/audio/op_rsp 的 ret_code=0 再发送音频。结束时发 1（STOP_RECORD）。正常通话仅使用 0 和 1。

---

<a id="t/cmd/audio/op_rsp"></a>

### `/cmd/audio/op_rsp` 实时通话操作响应

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.AudioOpRsp`
- **消息键**：`device/cmd/audio/op_rsp`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | 通话操作结果：0=成功；非零表示操作失败，不应开始发送音频。 |

---

<a id="t/data/audio/playback_data"></a>

### `/data/audio/playback_data` 向机器人发送实时音频，由机器人扬声器播放；需先开启通话。

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.data.AudioPlaybackData`
- **消息键**：`app/data/audio/playback_data`

向机器人发送实时音频，由机器人扬声器播放；需先开启通话。SDK 向 app/data/audio/playback_data 发布消息。先发送 /cmd/audio/op_req（operation_type=0），收到成功回执后再连续发送音频帧。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| encoding_type | u8 | 1 | 音频编码类型：0=Opus。 |
| sample_rate | u32 | 2 | 音频采样率，单位 Hz；发送端使用 48000，接收端按消息值解码。 |
| channels | u8 | 3 | 声道数；当前发送格式为 1（单声道）。 |
| frame_duration_ms | u16 | 4 | 每帧音频时长，单位毫秒；当前发送格式为 20。 |
| timestamp | u32 | 5 | 音频帧时间戳，单位毫秒；用于同一音频流的时序，不直接视为日期时间。 |
| seq_num_audio | u32 | 6 | 音频帧序号，每帧递增，用于识别连续性。 |
| data_size | u32 | 7 | 有效编码字节数 (<=960) |
| data | u8 [960] | 8 | 一帧编码后的音频字节；只使用 data_size 指定的有效长度。 |

#### 电脑麦克风 → 机器人扬声器

通话开启后，每 20 ms 发送一条 Protobuf 消息。encoding_type=0（Opus），sample_rate=48000，channels=1，frame_duration_ms=20。一帧对应 960 个单声道 PCM 输入采样；data 放一帧编码后的 Opus 有效载荷，长度 1–960 字节，data_size 必须等于实际长度，不补齐。timestamp 用毫秒，seq_num_audio 每帧递增。该流为单向发布；通话未开启或格式不符时设备会丢弃音频。

---

<a id="t/status/audio/status"></a>

### `/status/audio/status` 音频采集状态与录像响应能力

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.AudioStatus`
- **消息键**：`device/status/audio/status`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| record_enabled | u8 | 1 | 当前音频采集或音频录制是否启用：0=未启用，1=已启用；不等同于录像音轨偏好，也不能单独证明视频正在录像。 |
| record_executor_version | u8 | 2 | 2 表示录像动作执行与终态反馈。 |

---

<a id="t/cmd/camera/capture_controls_set_req"></a>

### `/cmd/camera/capture_controls_set_req` 按 update_mask 局部设置相机参数，数值使用原始单位，不使用档位索引

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.CaptureControlsSetReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/camera/capture_controls_set_req`

按 update_mask 局部设置相机参数，数值使用原始单位，不使用档位索引。 设备以当前配置与请求参数的最终组合进行校验。 空 mask、未知 bit、未知设置模式或非法组合在校验阶段整条拒绝，不执行修改。 所选快门、ISO、WB 为手动模式时对应数值必须非零且满足设备范围和步进；自动模式时忽略数值。 超范围、不支持的设置不得静默替换为另一档位；实际生效值以状态上报为准。 整体模式实际切换时，快门、ISO、WB 默认恢复自动，EV 归零；切入专业模式后再应用本请求中的参数。 重复设置相同整体模式不触发重置；最终为自动模式时不允许请求手动参数。 快门与 ISO 均手动时 EV 禁用并归零；任一恢复自动时 EV 重新启用且初始为零，可同请求指定新 EV。 最终禁用 EV 却显式选中 EV 更新时拒绝整条请求；ev_mode 不允许直接写入。 请求关联复用标准头序号，响应回显请求序号，不新增 request_id；多个参数可能仅部分设置成功。

- **对应回执**：[/cmd/camera/capture_controls_set_rsp](#t/cmd/camera/capture_controls_set_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| update_mask | u32 | 1 | 更新位：bit0=整体专业模式，bit1=快门，bit2=ISO，bit3=白平衡，bit4=曝光补偿；可按位或组合，其余位为 0。 |
| is_pro_mode | bool | 2 | 整体专业模式：true=专业、false=自动；仅当 update_mask 的 bit0 置位时修改。 |
| shutter_mode | camera_parameter_mode | 3 | 快门模式：1=自动、2=手动；设置时不能用 0（未知）。 |
| shutter_ns | u64 | 4 | 手动曝光时间，ns；快门为自动时忽略。 |
| iso_mode | camera_parameter_mode | 5 | ISO 模式：1=自动、2=手动；设置时不能用 0（未知）。 |
| iso | u32 | 6 | 手动 ISO；ISO为自动时忽略。 |
| wb_mode | camera_parameter_mode | 7 | 白平衡模式：1=自动、2=手动；设置时不能用 0（未知）。 |
| wb_kelvin | u32 | 8 | 手动色温，K；白平衡为自动时忽略。 |
| ev_compensation | f32 | 9 | 曝光补偿，EV。拒绝 NaN/Inf；当前范围 -3..+3、步进 1/3 EV，距最近合法步进不超过 0.000001 EV 才接受。 |

---

<a id="t/cmd/camera/capture_controls_set_rsp"></a>

### `/cmd/camera/capture_controls_set_rsp` 相机参数提交响应，标准头序号回显请求序号

- **方向**：机器人 → 客户端（订阅）

相机参数提交响应，回显请求序号。result=0表示所选参数组均已接受，不代表下一帧画面参数已稳定。applied_mask与failed_mask分别表示成功和失败的参数组，互不重叠，按位或等于请求的update_mask。参数无效或忙时所有选中组均标为失败；执行中部分成功返回5。设置失败不保证回滚，实际配置以状态上报为准。

- **Protobuf 消息**：`md.protocol.cmd.CaptureControlsSetRsp`
- **消息键**：`device/cmd/camera/capture_controls_set_rsp`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| result | capture_controls_set_result | 1 | 提交结果：0=成功，1=参数无效，2=不支持，3=忙，4=设置失败，5=部分成功；成功不代表下一帧图像参数已稳定。 |
| applied_mask | u32 | 2 | 成功设置的参数组。更新位：bit0=整体专业模式，bit1=快门，bit2=ISO，bit3=白平衡，bit4=曝光补偿；可按位或组合，其余位为 0。 |
| failed_mask | u32 | 3 | 未成功设置的参数组。更新位：bit0=整体专业模式，bit1=快门，bit2=ISO，bit3=白平衡，bit4=曝光补偿；可按位或组合，其余位为 0。 |

---

<a id="t/cmd/camera/record_req"></a>

### `/cmd/camera/record_req` 录制请求

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.CameraRecordReq`
- **回执超时**：**80000 ms**
- **消息键**：`app/cmd/camera/record_req`

录制请求。以 camera_status.record_response_version 为准：版本 1 仅校验准入，通过后由调用方发送旧录制命令；版本 2 执行录像动作，并通过 camera_record_rsp 返回执行终态。版本 2 调用方不得追加 unified_cmd。超时表示结果未知，不可自动重发或反向操作。

- **对应回执**：[/cmd/camera/record_rsp](#t/cmd/camera/record_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| action | u8 | 1 | 录像动作：1=开始，2=停止。 |
| request_id | u64 | 2 | App 连接会话内的请求身份；响应回显该值供 App 关联本次请求 |

---

<a id="t/cmd/camera/record_rsp"></a>

### `/cmd/camera/record_rsp` 请求序号与 request_id 回显请求身份

- **方向**：机器人 → 客户端（订阅）

请求序号与request_id回显请求身份。版本1的ret_code=0仅表示准入通过；版本2表示实际动作完成：开始要求state=2，停止要求state=0。启动中、停止中和排队均不构成执行成功；失败或取消返回非0错误码，state是响应时的实际状态。录像概览通过/status/camera/status上报。

- **Protobuf 消息**：`md.protocol.cmd.CameraRecordRsp`
- **消息键**：`device/cmd/camera/record_rsp`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| action | u8 | 1 | 回显请求动作：1=开始，2=停止。 |
| request_id | u64 | 2 | 回显请求身份 |
| ret_code | u8 | 3 | 结果：0=成功，1=请求无效，2=状态冲突，3=存储忙，4=存储不可用，5=开始失败，6=停止失败，7=服务不可用，8=开始提示音失败，9=操作取消，10=空间不足，11=主摄异常，12=存储正被电脑占用。成功含义须结合录像响应版本判断。 |
| state | u8 | 4 | 响应时的实际录像状态：0=空闲，1=正在启动，2=正在录像，3=正在停止。版本 2 开始成功要求状态为 2，停止成功要求为 0。 |

---

<a id="t/cmd/camera/unified_cmd"></a>

### `/cmd/camera/unified_cmd` 相机控制：拍照、变焦、防抖、旧版拍摄参数、高光标记及录像收音

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.CameraUnifiedCmd`
- **消息键**：`app/cmd/camera/unified_cmd`

SDK 向 app/cmd/camera/unified_cmd 发布相机控制命令。cmd_type 选择操作，对应字段填写目标值。本消息没有独立的成功响应；发送成功只表示消息已发出，设置是否生效应结合 device/status/camera/status、device/status/camera/capture_controls 或相关业务状态确认。

为兼容不同设备，每条消息只选择一个 cmd_type，不将多个操作按位合并。data 中 16 个字段都按协议编码；未使用字段可按下方示例初始化。拍照、RAW 采集及高光标记属于一次性动作，不应持续循环发送，也不能因未收到独立响应而自动补发。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| cmd_type | u16 | 1 | 操作选择值，见下表。每次只填写一个已列出的命令值；本手册未开放的位保持 0。 |
| trigger_flags | u8 | 2 | 仅用于触发动作：拍照时为 1（bit0）；旧版开始录像为 2（bit1），停止录像为 0。其他操作填 0。 |
| zoom_data | u8 | 3 | 变焦倍率乘以 10：10=1.0 倍、20=2.0 倍。通常范围 10–20；部分录像规格只接受 10 或 20，支持范围及录像中的切换限制以设备为准。 |
| eis_enable | u8 | 4 | 电子防抖开关请求：0=关闭、1=开启。部分设备采用常开防抖，仅支持切换 eis_level，不能通过此字段关闭。 |
| eis_level | u8 | 5 | 防抖等级：0=基础、1=标准、2=增强；实际支持等级以设备为准。仅在 cmd_type=0x0008 时使用。 |
| horizon_lock | u8 | 6 | 当前不作为对外可用设置，填 0。 |
| storage_type | u8 | 7 | 本命令不用于切换存储位置，填 0。 |
| raw_record_enable | u8 | 8 | cmd_type=0x0040 时，1=请求单帧 RAW 采集，0=不执行。不是持续录像开关；仅设备支持 RAW 采集且满足执行条件时生效。 |
| record_profile_id | u16 | 9 | 本手册不开放此字段对应的设置，填 0。 |
| vt_config_index | u8 | 10 | 本手册不开放此字段对应的设置，填 0。 |
| pro_mode_enable | u8 | 11 | cmd_type=0x0080 时，0=自动模式、1=专业模式；同时提交快门、ISO、EV、白平衡四项字段，不能把未填写项理解为保持原值。 |
| exp_time_index | u8 | 12 | 旧版快门档位：0=自动；1–25 的映射见下方。它是索引，不能直接填写曝光纳秒值。仅随 0x0080 使用。 |
| iso_index | u8 | 13 | 旧版 ISO 档位：0=自动；非零索引按目标设备的档位定义解释，不可将 ISO 数值直接填入，也不可跨设备套用索引。需要实际 ISO 数值设置时，在支持设备上使用 capture_controls_set_req。 |
| ev_index | u8 | 14 | 旧版曝光补偿档位：0–18，9=0 EV；目标补偿约为 (index−9)/3 EV，范围 −3 至 +3 EV。实际量化值以状态为准。仅随 0x0080 使用，未使用时建议填 9。 |
| wb_index | u8 | 15 | 旧版白平衡档位：0=自动；1–81 对应 2000–10000 K，色温=(index−1)×100+2000。仅随 0x0080 使用。 |
| reserved | u8 | 16 | 保留字段；发送时置0，接收时忽略。 |

#### 操作与填写方式

| cmd_type | 用途 | 填写方式与结果确认 |
| --- | --- | --- |
| 0x0001（1） | 拍照 | trigger_flags=1，发起一次拍照请求；发送完成不代表照片已保存。 |
| 0x0002（2） | 旧版录像启停 | trigger_flags=2 开始、0 停止；必须先遵守下方录像版本规则，不能直接绕过录像请求的准入检查。 |
| 0x0004（4） | 变焦 | 填写 zoom_data；从相机状态的 zoom_data 确认设备上报倍率。 |
| 0x0008（8） | 电子防抖 | 填写 eis_enable 和 eis_level；从相机状态的 eis_status 确认。 |
| 0x0040（64） | 单帧 RAW 采集 | raw_record_enable=1；仅在设备支持且允许采集时执行，不作为普通照片或持续录像接口。 |
| 0x0080（128） | 旧版专业模式与拍摄参数 | 填写 pro_mode_enable、exp_time_index、iso_index、ev_index、wb_index 的完整组合；支持新协议时，优先使用 capture_controls_set_req。 |
| 0x0100（256） | 录像高光标记 | 请求在当前录像中标记高光；应在已确认正在录像时发送，非录像期间不保证生效。 |
| 0x0800（2048） | 录像收音开关 | reserved=0 静音、1 录音；下一次录像生效，可查看 device/status/mic/status 的 record_audio_enabled。 |

本手册未列出的操作不作为开放能力使用。水平锁定目前没有可用执行实现；存储切换不能通过本命令完成。录像规格及图传模式设置未在本手册开放，不通过其他命令位调用。

#### 录像启停的版本规则

先读取相机状态的 record_response_version：版本 1 时，先发送 /cmd/camera/record_req，只有收到匹配请求的准入成功响应后，才发送本命令的 0x0002；版本 2 时，record_req 已负责执行并返回终态，不得再追加 0x0002。未声明版本的设备属于旧版兼容场景，不能据此认为支持新录像响应协议。请求超时表示结果未知，不自动重发或反向操作。

#### 旧版拍摄参数

0x0080 一次提交整组拍摄设置。恢复自动模式可填写 pro_mode_enable=0、exp_time_index=0、iso_index=0、ev_index=9、wb_index=0。手动设置时须使用目标设备支持的组合；快门与 ISO 独立自动/手动组合及曝光补偿的适用条件可能因设备而异。各项可能部分生效，不能把发送成功理解为整组设置成功。

快门索引 0=自动；索引 1–25 依次对应：1/8000、1/6400、1/5000、1/4000、1/3200、1/2500、1/2000、1/1600、1/1250、1/1000、1/800、1/640、1/500、1/400、1/320、1/250、1/200、1/160、1/125、1/100、1/80、1/60、1/50、1/40、1/30 秒。实际曝光受当前帧率及设备能力限制。

新拍摄参数协议 capture_controls_set_req 使用实际快门时长、ISO、色温和 EV，并提供响应；它仅覆盖拍摄参数，不替代本命令中的拍照、变焦等操作。同一次参数修改只选一种协议发送，避免设置互相覆盖。

#### 示例：设置 1.0 倍变焦

下面是 Protobuf 编码前的字段取值示例，不是直接发送 JSON。按 CameraUnifiedCmd 编码后，向 app/cmd/camera/unified_cmd 发布；encoding=application/protobuf，attachment 携带当前会话的 16 字节连接 ID。

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

### `/status/camera/capture_controls` 相机参数权威完整快照

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.CaptureControlsStatus`
- **消息键**：`device/status/camera/capture_controls`

相机参数权威完整快照。配置变化后主动发布，实际值随实际测量更新；支持首次订阅及重连恢复。 控制模式与数值独立：模式反映设备已接受的控制状态；实际值可能延迟更新。 不上报用户目标值或档位索引。快门、ISO、WB 无有效测量时数值为 0，禁止回退填入设置值； 实际测量暂不可用不改变已知模式，只有控制模式本身不可确定时才上报未知。 EV 为当前生效的曝光补偿，不是场景测光 EV；禁用时上报禁用且曝光补偿为0。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| is_pro_mode | bool | 1 | 当前是否处于专业模式：true=是、false=否。 |
| shutter_mode | camera_parameter_mode | 2 | 快门模式：0=未知，1=自动，2=手动。 |
| iso_mode | camera_parameter_mode | 3 | ISO 模式：0=未知，1=自动，2=手动。 |
| wb_mode | camera_parameter_mode | 4 | 白平衡模式：0=未知，1=自动，2=手动。 |
| ev_mode | camera_ev_mode | 5 | 曝光补偿状态：0=未知，1=启用，2=禁用。快门与 ISO 均为手动时禁用。 |
| shutter_ns | u64 | 6 | 当前实际曝光时间，单位纳秒；0 表示暂不可用。 |
| iso | u32 | 7 | 当前实际 ISO 数值；0 表示暂不可用。 |
| wb_kelvin | u32 | 8 | 当前实际色温，单位 K；0 表示暂不可用。 |
| ev_compensation | f32 | 9 | 当前生效的曝光补偿，EV；禁用时为 0。 |

---

<a id="t/status/camera/status"></a>

### `/status/camera/status` 相机状态上报：录像状态、时长、分辨率、帧率及变焦等信息

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.CameraStatus`
- **消息键**：`device/status/camera/status`

机器人主动上报的相机状态。SDK 订阅 device/status/camera/status，用于显示录像状态、录像时长、拍照与录像规格、图传规格、电子防抖、变焦和环境亮度等信息。本消息为状态通知，不是控制命令，也不是某次请求的成功回执；发送设置后应结合对应响应及后续状态确认结果。支持新拍摄参数协议时，快门、ISO、白平衡和曝光补偿的实际值读取 device/status/camera/capture_controls。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| rec_state | u8 | 1 | 录像标志：0=未正式录像，1=正在正式录像；不能单独用于判断预录状态。 |
| cur_rec_width | u16 | 2 | 当前录像画面宽度，单位像素。 |
| cur_rec_height | u16 | 3 | 当前录像画面高度，单位像素。 |
| cur_rec_bitrate | u8 | 4 | 设备上报的录像码率，单位 Mbps。 |
| cur_snap_width | u16 | 5 | 当前拍照画面宽度，单位像素。 |
| cur_snap_height | u16 | 6 | 当前拍照画面高度，单位像素。 |
| camera_width | u16 | 7 | 相机采集画幅宽度，单位像素；不等同于录像或图传输出宽度。 |
| camera_height | u16 | 8 | 相机采集画幅高度，单位像素；不等同于录像或图传输出高度。 |
| rec_frame_rate | u16 | 9 | 录像帧率，单位 fps。 |
| vt_width | u16 | 10 | 图传画面宽度，单位像素。 |
| vt_height | u16 | 11 | 图传画面高度，单位像素。 |
| vt_frame_rate | u16 | 12 | 图传帧率，单位 fps。 |
| vt_bitrate | u16 | 13 | 设备上报的图传码率，单位 kbps；不代表客户端实时测得的网络吞吐量。 |
| eis_status | u8 | 14 | 电子防抖状态或等级；不同设备版本可能采用不同取值定义，不应直接当作统一的开关布尔值。 |
| pro_shutter_idx | u8 | 15 | 旧版快门档位索引，不是曝光时长。需要实际时长时读取 capture_controls 的 shutter_ns。 |
| pro_iso_idx | u8 | 16 | 旧版 ISO 档位索引，不是 ISO 数值。需要实际数值时读取 capture_controls 的 iso。 |
| pro_ev_idx | u8 | 17 | 旧版曝光补偿档位索引。需要实际 EV 值时读取 capture_controls 的 ev_compensation。 |
| pro_wb_idx | u8 | 18 | 旧版白平衡档位索引，不是色温值。需要实际色温时读取 capture_controls 的 wb_kelvin。 |
| rec_duration_s | u16 | 19 | 时长，单位秒。recording_state=2时为当前预录视频窗口时长，=1时为正式录像经过时长（不含预录历史），=0时为0。正式起录后重新计时；无有效读数时为0，最大65535。 |
| zoom_data | u8 | 20 | 变焦倍率的 10 倍整数值，例如 10=1.0 倍、20=2.0 倍；支持范围取决于设备，0 表示未知。 |
| current_mode | u8 | 21 | 兼容模式字段，含义取决于设备版本；255 表示无效。不要用它判断是否正在录像，任务模式应读取调度状态。 |
| current_preset | u8 | 22 | 兼容预设编号，含义取决于设备版本；255 表示无效，不应推断为录像规格档位。 |
| cur_lux | u32 | 23 | 环境照度估计，单位 lux；0 表示无有效估计，不应解释为全黑。 |
| lux_level | u8 | 24 | 环境明暗等级，越大越亮：1=极暗（1–<5 lux），2=很暗（5–<20），3=暗（20–<50），4=偏暗（50–<100），5=正常（100–<300），6=明亮（300–<750），7=很亮（750–<2000），8=强光（2000–<10000），9=极强光（≥10000）；255=无效。 |
| current_profile_id | u16 | 25 | 当前录像规格的档位标识；未提供或为 0 时不能确认档位。查看规格请结合录像宽度、高度及帧率，不能把编号直接当作分辨率。 |
| recording_state | camera_recording_state | 26 | 录像状态：0=空闲，1=正式录像，2=预录或起录准备中。空闲不保证文件收尾完成。字段缺失表示设备未声明此状态，不能用缺省值覆盖 rec_state。 |
| record_response_version | u8 | 27 | 相机响应版本：0=未声明；1=请求准入校验；2=录像动作执行与终态反馈。 |

---

<a id="t/status/capability/voice_language"></a>

### `/status/capability/voice_language` 语音语言与资源可用状态

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.VoiceLanguageStatus`
- **消息键**：`device/status/capability/voice_language`

设备语言与播报资源状态，初始化、语言或资源变化后上报，并每秒更新。当前格式版本为1；超过3秒未更新、时间戳异常或乱序时，不应沿用旧状态。重连后重新获取。current_language决定播报及本地语音指令识别的语言；supported_mask只表示播报资源是否可用。未保存语言选择时默认美式英文；无法读取选择时当前语言及资源位均为0，不能把未知解释成已选择中文或英文。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| version | u16 | 1 | 状态格式版本，当前为 1；不表示语音功能已经就绪。 |
| source_monotonic_ns | u64 | 2 | 状态生成时的设备单调时钟时间，单位 ns；不能与客户端时钟直接比较。 |
| supported_mask | u8 | 3 | 可用播报资源位：bit0=简体中文，bit1=美式英文；其余位为 0，整体为 0 表示没有可用播报资源包。此字段不表示语音识别模型是否可用。 |
| current_language | u8 | 4 | 当前语言设置：0=未知，1=简体中文，2=美式英文；其他值按未知处理。设置用于播报语言和本地语音指令识别语言，不是对用户当前说话语言的检测结果。 |

---

<a id="t/cmd/common/get_version_req"></a>

### `/cmd/common/get_version_req` 查询设备版本信息

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.CommonGetVersionReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/common/get_version_req`
- **对应回执**：[/cmd/common/get_version_rsp](#t/cmd/common/get_version_rsp)

---

<a id="t/cmd/common/get_version_rsp"></a>

### `/cmd/common/get_version_rsp` 设备版本信息响应

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.CommonGetVersionRsp`
- **消息键**：`device/cmd/common/get_version_rsp`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | 版本查询结果：0=成功；非零表示失败。 |
| major_version | u8 | 2 | 兼容版本编码字节；低4位为次版本，高4位为主版本，见位定义。 |
| minor_version | u8 | 3 | 独立次版本字节；与打包版本字节分别保留，不应将两者混为同一字段。 |
| hardware_version | u8 [16] | 4 | 设备硬件版本标识，最大16字节；不是芯片型号说明。 |
| loader_version | u8 [4] | 5 | 启动程序版本编码，4字节；未约定格式时保留原始字节。 |
| firmware_version | u8 [4] | 6 | 固件版本编码，4字节；未约定格式时保留原始字节。 |
| is_mass_product | u8 | 7 | 量产标志：bit0=1 表示量产版本；其余位保留。 |

#### 位域 · bit 0 为最低位

| 字段 | 位 | 名称 | 含义 |
| --- | --- | --- | --- |
| major_version | 0–3 | minor_version | 打包版本号中的次版本位。 |
| major_version | 4–7 | major_version | 打包版本号中的主版本位。 |
| is_mass_product | 0 | is_mass_product | 1=量产版本，0=非量产版本。 |
| is_mass_product | 1–31 | reserved | 保留位，发送时置0，接收时忽略。 |

---

<a id="t/cmd/common/ping_req"></a>

### `/cmd/common/ping_req` 通信连通性探测请求

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.CommonPingReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/common/ping_req`
- **对应回执**：[/cmd/common/ping_rsp](#t/cmd/common/ping_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| ping_data | u8 [16] | 1 | 16字节探测数据；请求由客户端填写，响应原样回显，用于匹配探测及确认通信可达。 |

---

<a id="t/cmd/common/ping_rsp"></a>

### `/cmd/common/ping_rsp` 通信探测响应与数据回显

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.CommonPingRsp`
- **消息键**：`device/cmd/common/ping_rsp`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | 探测结果：0=成功；非零表示失败。 |
| ping_data | u8 [16] | 2 | 16字节探测数据；请求由客户端填写，响应原样回显，用于匹配探测及确认通信可达。 |

---

<a id="t/status/hms/active_snapshot"></a>

### `/status/hms/active_snapshot` 当前有效故障的完整快照

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.HmsActiveSnapshot`
- **消息键**：`device/status/hms/active_snapshot`

HMS 活跃故障快照响应

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| at_offset | u64 | 1 | 生成快照时对应的故障事件序号，用于与事件流衔接。 |
| count | u32 | 2 | 有效故障数 (<=256) |
| module_ids | u32 [256] | 3 | 故障所属模块编号数组；前 count 项有效。 |
| fault_codes | u32 [256] | 4 | 完整32位故障码数组，与其他故障数组按相同下标对应。 |
| instance_ids | u32 [256] | 5 | 故障实例编号数组；同一故障码可能有多个实例。 |
| severities | u32 [256] | 6 | 故障严重程度数组：0=信息，1=警告，2=错误，3=致命；与其他故障数组按相同下标对应。 |
| first_assert_ts | u64 [256] | 7 | 各故障实例首次出现的时间计数；保留原始值，不自行推断时间原点。 |
| last_update_ts | u64 [256] | 8 | 各故障实例最近更新时间计数；与 first_assert_ts 使用同一时间基准。 |
| daemon_instance_id | u64 | 9 | 故障状态会话标识；变化表示原状态会话失效，应重新获取完整快照。 |
| state_revision | u64 | 10 | 当前故障状态修订号，用于拒绝旧状态和关联后续事件。 |
| recovery_state | hms_recovery_state | 11 | 状态有效性：0=无效，1=恢复中，2=就绪；只有有效且就绪的完整快照才能用于清除旧故障状态。 |
| ret_code | u32 | 12 | 0=成功，1=筛选条件无效，2=超出容量；失败时不能当作空故障列表。 |
| request_id | u64 | 13 | 对应 hms_query_active.request_id；APP 仅接受当前查询代际。 |

先订阅故障快照和事件，再查询当前快照。以同一故障会话中有效且就绪的快照建立完整状态；事件只表示变化。重连、会话标识改变、事件缺口或数据异常时重新查询，不能把缺失消息解释为故障已解除。

---

<a id="t/status/hms/event_broadcast"></a>

### `/status/hms/event_broadcast` 故障事件通知

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.HmsEventBroadcast`
- **消息键**：`device/status/hms/event_broadcast`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| offset | u64 | 1 | 故障事件序号，用于检测事件缺口及与完整快照衔接。 |
| module_id | u16 | 2 | 故障模块编号；完整故障码=(module_id<<16)\|fault_code。 |
| fault_code | u16 | 3 | 故障码低16位，须与 module_id 合并后查故障码参考表。 |
| instance_id | u16 | 4 | 发生变化的故障实例编号。 |
| kind | u8 | 5 | 故障事件类型：1=出现，2=解除，3=更新。事件可能遗漏，不能代替完整故障快照。 |
| severity | u8 | 6 | 严重程度：0=信息，1=警告，2=错误，3=致命。 |
| timestamp_ms | u64 | 7 | 事件时间计数，单位毫秒；不自行推断时间原点。 |
| daemon_instance_id | u64 | 8 | 故障状态会话标识，与完整快照的同名字段匹配。 |
| state_revision | u64 | 9 | 事件对应的故障状态修订号；出现缺口或回退时重新查询快照。 |
| origin | u8 | 10 | 事件来源：0=无效，1=实时变化，2=状态重同步，3=来源失联后处理。 |
| recovery_state | u8 | 11 | 状态有效性：0=无效，1=恢复中，2=就绪；恢复期间不能仅凭事件清除旧约束。 |

先订阅故障快照和事件，再查询当前快照。以同一故障会话中有效且就绪的快照建立完整状态；事件只表示变化。重连、会话标识改变、事件缺口或数据异常时重新查询，不能把缺失消息解释为故障已解除。

---

<a id="t/status/hms/query_active"></a>

### `/status/hms/query_active` 查询全部或指定模块的当前故障

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.status.HmsQueryActive`
- **消息键**：`app/status/hms/query_active`

HMS 查询活跃故障请求

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| filter_kind | u8 | 1 | 查询范围：0=全部故障，1=指定模块；其他值无效。 |
| _pad0 | u8 | 2 | 保留字段，固定为0。 |
| module_id | u16 | 3 | filter_kind=1 时指定的故障模块编号；查询全部时填0。 |
| request_id | u64 | 4 | 查询标识；active_snapshot 必须原样回显。 |

先订阅故障快照和事件，再查询当前快照。以同一故障会话中有效且就绪的快照建立完整状态；事件只表示变化。重连、会话标识改变、事件缺口或数据异常时重新查询，不能把缺失消息解释为故障已解除。

---

<a id="t/cmd/led/ctrl_oneway"></a>

### `/cmd/led/ctrl_oneway` 直接设置14颗灯的颜色

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.LedCtrlOneway`
- **消息键**：`app/cmd/led/ctrl_oneway`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| seq | u32 | 1 | 灯光帧递增序号，用于识别帧连续性。 |
| timestamp_us | u64 | 2 | 发送时间戳，单位微秒，用于时序及过期判断。 |
| rgbw_data | u32 [14] | 3 | 14颗灯的颜色值数组；每项32位，编码为 (R<<24)\|(G<<16)\|(B<<8)\|W，各颜色分量0–255。 |

---

<a id="t/status/media/catalog_changed"></a>

### `/status/media/catalog_changed` 媒体列表变化通知

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.MediaCatalogChanged`
- **消息键**：`device/status/media/catalog_changed`

媒体列表变化时发送通知，客户端据此刷新列表。storage_kind表示存储位置：0=内置存储、1=SD卡、255=未知；change_reason表示变化原因：1=录像停止、2=重新扫描完成、3=拍照完成、4=存储恢复、5=存储离线。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| revision | u64 | 1 | 媒体列表修订号；变化后客户端应重新查询媒体列表。 |
| storage_kind | u8 | 2 | 发生变化的存储位置：0=内置存储，1=SD卡，255=未知。 |
| change_reason | u8 | 3 | 变化原因：1=录像停止，2=重新扫描完成，3=拍照完成，4=存储恢复可用，5=存储离线。 |
| reserved | u16 | 4 | 保留字段；发送时置0，接收时忽略。 |
| mount_generation | u64 | 5 | 存储连接代次，用于识别更换或重新挂载后的新存储会话。 |

---

<a id="t/cmd/mic/saved_list_req"></a>

### `/cmd/mic/saved_list_req` 请求重新上报已保存的麦克风列表

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.MicSavedListReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/mic/saved_list_req`

请求重新上报已保存的麦克风列表。响应仅表示受理，不包含列表数据；列表从 /status/mic/saved_list 获取。首次订阅后发送本请求获取当前列表，后续变化由设备主动推送。

- **对应回执**：[/cmd/mic/saved_list_rsp](#t/cmd/mic/saved_list_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| reserved0 | u8 | 1 | 保留字段；发送时置0，接收时忽略。 |

---

<a id="t/cmd/mic/saved_list_rsp"></a>

### `/cmd/mic/saved_list_rsp` 请求重新上报麦克风列表的受理结果

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.MicSavedListRsp`
- **消息键**：`device/cmd/mic/saved_list_rsp`

已保存列表请求的受理回执，不包含列表数据。收到请求后，设备通过 /status/mic/saved_list 上报当前列表。首次订阅不会收到历史列表，应主动请求一次；后续变化由设备推送。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | 请求处理结果：0=成功受理；非零表示未受理或处理失败。音源切换的最终结果仍读取麦克风状态。 |

---

<a id="t/cmd/mic/scan_req"></a>

### `/cmd/mic/scan_req` 开始或停止搜索附近的蓝牙麦克风

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.MicScanReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/mic/scan_req`

开始或停止搜索附近的蓝牙麦克风。搜索结果走 mic_scan_result。设备只上报当前机型与固件已验证支持的型号，判断不依赖设备名或品牌。

- **对应回执**：[/cmd/mic/scan_rsp](#t/cmd/mic/scan_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| action | u8 | 1 | 0=停止搜索，1=开始搜索 |
| duration_s | u8 | 2 | 搜索时长秒；0=持续搜索直到显式停止（搜索半弹窗用 0） |

---

<a id="t/cmd/mic/scan_rsp"></a>

### `/cmd/mic/scan_rsp` 麦克风搜索请求的受理结果

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.MicScanRsp`
- **消息键**：`device/cmd/mic/scan_rsp`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | 请求处理结果：0=成功受理；非零表示未受理或处理失败。音源切换的最终结果仍读取麦克风状态。 |

---

<a id="t/cmd/mic/source_select_req"></a>

### `/cmd/mic/source_select_req` 选择视频收音音源并联动录像收音设置

- **方向**：客户端 → 机器人（发布）

选择视频收音来源并联动录像收音设置。kind=0关闭收音并取消尚未完成的切换，保留已有蓝牙连接；kind=1或2切换成功后开启录像收音，失败恢复原设置。音轨设置从下一次录像生效。响应仅表示受理，最终结果读取/status/mic/status；蓝牙连接及恢复原蓝牙音源各最多等待15秒。

- **Protobuf 消息**：`md.protocol.cmd.MicSourceSelectReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/mic/source_select_req`
- **对应回执**：[/cmd/mic/source_select_rsp](#t/cmd/mic/source_select_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| kind | u8 | 1 | 0=关闭全部收音，1=启用内置麦克风，2=启用指定蓝牙麦克风；取值见 mic_source_kind |
| mac | u8 [6] | 2 | kind=2时填写目标蓝牙麦克风的6字节地址；kind=0或1时全部填0。 |

---

<a id="t/cmd/mic/source_select_rsp"></a>

### `/cmd/mic/source_select_rsp` 收音来源切换请求的受理结果

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.MicSourceSelectRsp`
- **消息键**：`device/cmd/mic/source_select_rsp`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | 请求处理结果：0=成功受理；非零表示未受理或处理失败。音源切换的最终结果仍读取麦克风状态。 |

---

<a id="t/status/mic/saved_list"></a>

### `/status/mic/saved_list` 已保存麦克风列表的主动推送，是列表数据的唯一来源：电量变化、可用性变化、新增与移除都推这条，展示端不轮询

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.MicSavedList`
- **消息键**：`device/status/mic/saved_list`

已保存麦克风列表的主动推送，是列表数据的唯一来源：电量变化、可用性变化、新增与移除都推这条，展示端不轮询。entries 按首次成功建立可用音频连接并保存的顺序排列，先保存的在前；重连、切源及状态更新不改位置，新设备和移除后重新配对的设备追加到末尾。已有记录保持原有顺序。mic_saved_list_req 只触发一次重推，其响应 mic_saved_list_rsp 仅含 ret_code、不带数据。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| entries | mic_saved_device [32] | 1 | 已保存麦克风列表，最多32项；每项包含蓝牙地址、名称、连接状态、电量有效标志、电量及连接记录，见复合字段说明。 |

#### 复合字段说明：mic_saved_device

| 字段 | 字段编号 | 中文说明 |
| --- | --- | --- |
| mac | 1 | 6字节蓝牙地址，作为设备标识。 |
| name | 2 | UTF-8设备名，不包含字符串终止符。 |
| state | 3 | 0=未知，1=当前不可用，2=本轮搜索可见，3=连接中，4=已连接。当前不可用不等于已关机。 |
| battery_valid | 4 | 0=电量不可用，1=电量有效。 |
| battery_pct | 5 | 电量百分比0–100；需先检查battery_valid。 |
| last_success_seq | 6 | 最近一次连接成功的序号，不用于列表排序。 |
| last_success_ms | 7 | 最近连接成功的Unix时间，单位毫秒；0表示时间不可用。 |
| supported | 8 | 当前设备软件是否支持此型号：0=不支持，1=支持。 |

---

<a id="t/status/mic/scan_result"></a>

### `/status/mic/scan_result` 搜索结果全量快照

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.MicScanResult`
- **消息键**：`device/status/mic/scan_result`

搜索结果全量快照；收到新快照时替换上一份搜索结果。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| scanning | u8 | 1 | 当前是否正在搜索：0=否，1=是。 |
| entries | mic_scan_device [32] | 2 | 本次搜索结果完整列表，最多32项；每项包含蓝牙地址、名称、信号强度、型号标识和是否已保存。 |

#### 复合字段说明：mic_scan_device

| 字段 | 字段编号 | 中文说明 |
| --- | --- | --- |
| mac | 1 | 6字节蓝牙地址。 |
| name | 2 | UTF-8设备名，不包含字符串终止符。 |
| rssi | 3 | 接收信号强度，单位dBm。 |
| model_id | 4 | 设备型号标识，用于区分搜索结果中的型号。 |
| saved | 5 | 0=未保存，1=已在保存列表中。 |

---

<a id="t/status/mic/status"></a>

### `/status/mic/status` 视频收音的权威状态：实际生效音源、连接中目标、最近一次可感知事件、输入电平

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.MicStatus`
- **消息键**：`device/status/mic/status`

视频收音状态：实际生效音源、连接中目标、最近一次事件、输入电平。音源切换的最终结果以本状态为准，请求响应只表示受理。录像音轨开关为只读状态；需要修改时使用 camera_unified_cmd 的 cmd_type bit11 与 reserved 字段，修改从下一次录像生效。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| active_kind | u8 | 1 | 当前实际收音来源：0=无，1=内置麦克风，2=蓝牙麦克风；需结合 record_audio_enabled 判断是否启用收音。 |
| active_mac | u8 [6] | 2 | active_kind=2时为当前蓝牙麦克风地址，否则全0。 |
| switching | u8 | 3 | 1=正在为某个目标建立连接，展示端显示等待指示 |
| target_mac | u8 [6] | 4 | switching=1 时表示正在连接的目标；switching=0 时表示最近事件关联的设备。全局关闭收音事件中为全0。 |
| last_event_kind | u8 | 5 | 最近事件：0=无，1=切换成功，2=切换失败或超时，3=切换取消，4=意外断开，5=关闭收音，6=启用内置收音。 |
| last_event_reason | u8 | 6 | 失败原因，仅事件2时解释：0=无，1=设备未知，2=蓝牙音频服务不可用，3=认证失败，4=忙，5=无响应，6=音频格式未就绪，7=无法打开音频，8=设备已被移除，9=录像收音设置提交失败。 |
| result_seq | u16 | 7 | 事件序号，每次新事件递增，16位循环计数。客户端用它避免重复提示；首次收到的快照用于初始化，不应重复弹出历史事件。 |
| input_valid | u8 | 8 | 当前能否取得音频输入。0 表示取不到，是「当前无声音」的判据，不等于设备故障。**不能用 input_level==0 代替**：那既可能是取不到、也可能是环境静音 |
| input_level | u8 | 9 | 声音强度刻度0–100，input_valid=0时无效。按分贝映射：−60 dBFS对应0，0 dBFS对应100；表示两次上报间的峰值，不是线性振幅。 |
| record_audio_enabled | u8 | 10 | 录像音轨开关，只读。1=录制实际声音，0=静音（默认）。通过 camera_unified_cmd 的 cmd_type bit11 与 reserved 字段设置，命令发送成功不等于设置已生效；修改从下一次录像生效。 |

---

<a id="t/cmd/mission/request"></a>

### `/cmd/mission/request` 任务请求命令

- **方向**：客户端 → 机器人（发布）

<a id="mission-submodes"></a>

#### 任务模式选择

向 `app/cmd/mission/request` 发布请求。先选 mission_id，再按该任务解释sub_mode。请求使用非零 header.seq_num、version=1 和 9 字节 mode_params。

| mission_id | sub_mode 取值 | 使用说明 |
| --- | --- | --- |
| 1 · 相机 | 0 手动；1 聚焦；2 跟随 | 切换尾随／平行跟随时保持 sub_mode=2，改 mode_params[0]。 |
| 3 · 宠物 | 0 待机；2 回基站对接 | 以调度状态回报的子模式确认切换。 |

#### 手动模式参数（mission_id=1，sub_mode=0）

字节下标从 0 开始。`mode_params[4]` 是第 5 个字节，表示控制方向，**不是保留位**。

| 字节下标 | 字段 | 取值与说明 |
| --- | --- | --- |
| 0 | drive_mode | 0=底盘转向、云台跟随；1=云台控制朝向、底盘跟随云台。 |
| 1 | 保留位 | 填 0。 |
| 2 | speed_radius_priority | 0=关闭；1=启用转弯减速约束。 |
| 3 | 保留位 | 填 0。 |
| 4 | control_orientation | 0=未指定控制方向；1=竖屏；2=横屏。 |
| 5–8 | 保留位 | 填 0。 |

**竖屏（control_orientation=1）会强制使用 drive_mode=0，即使请求中 drive_mode=1。需要底盘随云台运动时，显式设置 drive_mode=1、control_orientation=2。** 物理遥控器输入始终按底盘转向模式处理。

底盘随云台运动的完整请求参数：`version=1`、`mission_id=1`、`sub_mode=0`，`mode_params` 九个原始字节为：

```text
01 00 00 00 02 00 00 00 00
```

#### Camera 请求示例

进入第一人称聚焦：mission_id=1，sub_mode=1，mode_params 全 0（无固定环绕）。选择跟踪目标时，先确认跟踪响应，再请求切换模式。

普通尾随和普通平行跟随均发 mission_id=1、sub_mode=2。mode_params[0] 分别为 0、1；[1–4] 是距离（米，f32 小端），[5–6] 是角度（度，i16 小端），[7] 为环绕方向，[8] 为路由。例：平行跟随、距离 2 米、角度 0、默认路由的九字节为 `01 00 00 00 40 00 00 00 00`。退出跟随回手动，发 mission_id=1、sub_mode=0，参数重新按手动模式布局填写。

订阅 [请求回执](#t/cmd/mission/request_ack)并按 header.seq_num 匹配；result_code=0 后核对 current_mission_id 和 current_sub_mode。持续显示以 [调度状态](#t/status/scheduler/status)的 current_mode、sub_mode 为准。跟随模式中区分尾随和平行跟随看 sub_mode_state[0]。

[查看 Camera 模式与动作适用条件](#camera-mode-guide)

- **Protobuf 消息**：`md.protocol.cmd.MissionRequest`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/mission/request`

任务请求命令，用于启动、切换或停止任务。本手册开放的任务编号：1=相机、3=宠物。

- **对应回执**：[/cmd/mission/request_ack](#t/cmd/mission/request_ack)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| version | u8 | 1 | 协议版本 当前=1 |
| mission_id | u8 | 2 | 本手册开放的任务编号：1=相机，3=宠物。 |
| sub_mode | u8 | 3 | 按 mission_id 查本条目开头的取值表；相机任务：0手动、1聚焦、2跟随。 |
| mode_params | u8 [9] | 4 | 按任务编号、子模式和协议版本解释，具体填写方式见本条目的模式说明。 |

---

<a id="t/cmd/mission/request_ack"></a>

### `/cmd/mission/request_ack` 任务请求应答，仅表示任务生命周期的处理结果，不表示具体动作已执行完成。

- **方向**：机器人 → 客户端（订阅）

[查看各 mission_id 对应的 sub_mode 取值表](#t/cmd/mission/request)

- **Protobuf 消息**：`md.protocol.cmd.MissionRequestAck`
- **消息键**：`device/cmd/mission/request_ack`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| result_code | u8 | 1 | 0=成功，1=任务切换失败，2=子模式切换失败，3=参数设置失败。 |
| current_mission_id | u8 | 2 | 当前生效的 mission_id |
| current_sub_mode | u8 | 3 | 当前生效的 sub_mode |
| error_code_lo | u8 | 4 | error_code 低字节 |
| error_code_hi | u8 | 5 | error_code 高字节 |
| pitch_handoff | pitch_handoff | 6 | 成功切换到相机手动模式时可选返回的俯仰保持目标。只接受当前连接且与请求序号匹配的值，恢复旧版绝对俯仰控制前应用一次；缺失不表示归零。内部 held_pitch_rad 为弧度单位的绝对俯仰目标。 |

#### 复合字段说明：pitch_handoff

| 字段 | 字段编号 | 中文说明 |
| --- | --- | --- |
| held_pitch_rad | 1 | 本次切换锁定的绝对俯仰控制目标，单位弧度；只用于匹配的成功请求，不是延迟到达的姿态测量。 |

---

<a id="t/cmd/nm/wifi_connect_req"></a>

### `/cmd/nm/wifi_connect_req` 配网第 1 步：提交 Wi-Fi 凭据与本次配网标识

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.NmWifiConnectReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/nm/wifi_connect_req`

配网第 1 步：提交凭据（ssid/psk/security）与本次配网的 uuid，设备确认接收后返回响应。此时不会切换 Wi-Fi 模式，设备热点保持可用。随后发送 wifi_provision_start_req，收到响应后再发送 wifi_provision_commit_req 执行联网。配网结束后用同一 uuid 查询 wifi_provision_result_req。

- **对应回执**：[/cmd/nm/wifi_connect_rsp](#t/cmd/nm/wifi_connect_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| ssid_len | u8 | 1 | ssid 有效字节数, 1..32 |
| ssid | u8 [32] | 2 | 目标 SSID 原始字节 (不含结尾 0, 通常为 UTF-8) |
| psk_len | u8 | 3 | psk 有效字节数; 开放网络填 0 |
| psk | u8 [64] | 4 | Wi-Fi 密码：WPA-PSK passphrase 为 8..63 字节，或 64 字节 hex PSK；开放网络留空。 |
| security | u8 | 5 | 网络加密方式：0=开放网络，1=WEP，2=WPA，3=WPA2，4=WPA3，5=WPA2/WPA3混合。 |
| timeout_sec | u8 | 6 | 第 2 步设备等待关联+DHCP 的总超时 (秒); 0 = 用设备默认 15s, >60 按 60 处理 |
| uuid | u8 [16] | 7 | 本次配网的来源 uuid (App 每次配网新生成的 16 字节随机 session id); 设备连同配网结果一起存下, 供 /cmd/nm/wifi_provision_result_req 按 uuid 查询 |
| resv | u8 [4] | 8 | 保留字段；发送时置0，接收时忽略。 |

---

<a id="t/cmd/nm/wifi_connect_rsp"></a>

### `/cmd/nm/wifi_connect_rsp` 配网凭据接收确认

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.NmWifiConnectRsp`
- **消息键**：`device/cmd/nm/wifi_connect_rsp`

配网第 1 步的受理响应。state=PENDING 表示凭据与 uuid 已接收，响应回显 ssid 供客户端核对。此时设备热点保持可用，尚未开始连接目标网络；连接结果通过 /cmd/nm/wifi_provision_result_rsp 获取。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| result | u8 | 1 | 0=凭据已接收，1=忙且未覆盖原凭据，2=凭据参数无效。 |
| ssid_len | u8 | 2 | 回显的 ssid 有效字节数; result != 0 时为 0 |
| ssid | u8 [32] | 3 | 设备实际存下的 SSID 原始字节, 供 App 核对下发是否被正确接收 |
| resv | u8 [6] | 4 | 保留字段；发送时置0，接收时忽略。 |

---

<a id="t/cmd/nm/wifi_provision_commit_req"></a>

### `/cmd/nm/wifi_provision_commit_req` 确认执行配网并切换网络

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.NmWifiProvisionCommitReq`
- **回执超时**：**1000 ms**
- **消息键**：`app/cmd/nm/wifi_provision_commit_req`

客户端确认执行配网。必须先收到同一会话标识的配网准备成功响应；提交后设备会关闭热点和蓝牙连接，并开始连接目标网络。当前连接可能随即断开，因此不能以是否收到提交响应作为进入等待结果状态的条件。

- **对应回执**：[/cmd/nm/wifi_provision_commit_rsp](#t/cmd/nm/wifi_provision_commit_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| trigger_source | u8 | 1 | 来源：0=其他，1=设备按键，2=客户端，3=生产设置。 |
| resv | u8 [7] | 2 | 保留字段；发送时置0，接收时忽略。 |
| uuid | u8 [16] | 3 | 必须与已准备成功的配网会话标识一致；同 UUID 重试幂等 |

---

<a id="t/cmd/nm/wifi_provision_commit_rsp"></a>

### `/cmd/nm/wifi_provision_commit_rsp` 配网执行提交响应

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.NmWifiProvisionCommitRsp`
- **消息键**：`device/cmd/nm/wifi_provision_commit_rsp`

配网执行提交的诊断响应，不能保证在网络切换前送达；客户端按配网结果查询确认最终状态。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| result | u8 | 1 | 0=成功，1=忙，2=没有凭据，3=无法启动配网，4=会话标识不匹配。 |
| timeout_sec | u8 | 2 | 设备关联+DHCP 超时（秒） |
| resv | u8 [6] | 3 | 保留字段；发送时置0，接收时忽略。 |
| total_timeout_sec | u8 | 4 | 预留的配网总预算；0 表示 App 使用默认云端确认窗口 |
| uuid | u8 [16] | 5 | 回显配网执行提交的会话标识 |

---

<a id="t/cmd/nm/wifi_provision_result_req"></a>

### `/cmd/nm/wifi_provision_result_req` 查询最近一次配网结果

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.NmWifiProvisionResultReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/nm/wifi_provision_result_req`

查询最近一次配网结果。配网失败并恢复设备热点后，客户端可重新连接并使用第 1 步的 uuid 查询失败原因；结果记录在设备重启后仍可查询。

- **对应回执**：[/cmd/nm/wifi_provision_result_rsp](#t/cmd/nm/wifi_provision_result_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| uuid | u8 [16] | 1 | 第 1 步下发的来源 uuid; 全 0 表示「不校验 uuid, 直接把设备最后一条记录给我」 |
| resv | u8 [8] | 2 | 保留字段；发送时置0，接收时忽略。 |

---

<a id="t/cmd/nm/wifi_provision_result_rsp"></a>

### `/cmd/nm/wifi_provision_result_rsp` 最近一次配网结果

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.NmWifiProvisionResultRsp`
- **消息键**：`device/cmd/nm/wifi_provision_result_rsp`

最近一次配网结果。设备仅保留最后一次记录，客户端必须先校验uuid_match，再使用state和fail_reason；另一客户端发起的新配网可能覆盖原记录。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| result | u8 | 1 | 0=有结果记录，其他字段有效；1=无结果记录，其他字段为0。 |
| uuid_match | u8 | 2 | 1 = 记录的 uuid 与请求一致 (或请求 uuid 全 0 表示不校验), 可以采信; 0 = 设备记录属于另一次配网 (被别的 App/手机覆盖了), state/fail_reason 与本次无关 |
| state | u8 | 3 | 0=无记录，1=凭据已提交但未执行，2=配网中，3=成功，4=失败且已恢复热点。 |
| fail_reason | u8 | 4 | 仅state=4时解释：2=参数无效，3=网络接口异常，4=模式切换失败，5=配置失败，6=无线认证服务失败，7=关联超时，8=地址分配超时，9=配网中断，10=密码错误，11=热点拒绝连接，12=未找到热点，13=云端未确认联网。其他状态为0。 |
| uuid | u8 [16] | 5 | 设备记录里的来源 uuid |
| ssid_len | u8 | 6 | ssid 有效字节数 |
| ssid | u8 [32] | 7 | 本次配网的目标 SSID 原始字节 |
| ipv4 | u8 [4] | 8 | state=3时取得的IPv4地址，网络字节序，ipv4[0]为最高位；其他情况全0。 |
| channel | u8 | 9 | state=3时连接热点的信道号；未知为0。 |
| rssi | i8 | 10 | state=3时连接热点的接收信号强度，单位dBm；未取得时为0。 |
| resv | u8 [4] | 11 | 保留字段；发送时置0，接收时忽略。 |

---

<a id="t/cmd/nm/wifi_provision_start_req"></a>

### `/cmd/nm/wifi_provision_start_req` 校验配网信息并准备切换

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.NmWifiProvisionStartReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/nm/wifi_provision_start_req`

配网准备请求。设备校验第 1 步的凭据与 UUID，并保存本次使用的服务接入点，然后返回响应。此时保持 Wi-Fi 热点和蓝牙连接。客户端必须收到同一 UUID 的 wifi_provision_start_rsp 后，才可发送 wifi_provision_commit_req 执行联网。

- **对应回执**：[/cmd/nm/wifi_provision_start_rsp](#t/cmd/nm/wifi_provision_start_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| trigger_source | u8 | 1 | 来源：0=其他，1=设备按键，2=客户端，3=生产设置。 |
| resv | u8 [7] | 2 | 保留字段；发送时置0，接收时忽略。 |
| uuid | u8 [16] | 3 | 本次配网会话 UUID，必须与第 1 步 wifi_connect_req.uuid 一致；同一 UUID 重试由设备幂等处理 |
| mqtt_endpoint_len | u8 | 4 | MQTT 接入点域名有效字节数, 1..127, 不含结尾 0 |
| mqtt_endpoint | u8 [128] | 5 | 服务接入点域名，ASCII/UTF-8 字节。 |
| mqtt_resv | u8 [7] | 6 | 保留字段；发送时置0，接收时忽略。 |

---

<a id="t/cmd/nm/wifi_provision_start_rsp"></a>

### `/cmd/nm/wifi_provision_start_rsp` 配网准备结果

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.NmWifiProvisionStartRsp`
- **消息键**：`device/cmd/nm/wifi_provision_start_rsp`

配网准备响应。result=0表示凭据与会话标识校验成功，此时热点和蓝牙连接仍保持。客户端收到匹配的成功响应后，再发送配网执行提交请求。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| result | u8 | 1 | 0=准备成功或同一会话的重复请求，1=配网已提交正在执行，2=没有凭据，4=会话标识不匹配。 |
| timeout_sec | u8 | 2 | 设备本次实际采用的关联+DHCP 总超时 (秒), 已按设备上下限收敛; App 用它决定等待窗口长度, 而不是自己硬编码 |
| resv | u8 [6] | 3 | 保留字段；发送时置0，接收时忽略。 |
| total_timeout_sec | u8 | 4 | 预留: 设备本次配网的总预算 (秒), 含关联+DHCP+云端联网确认窗口, 即「App 最晚什么时候能等到结果」。0 = 设备未提供 (当前设备固定填 0), App 必须自己兜底补上云端确认窗口, 不能直接把 timeout_sec 当总预算 —— 后者只覆盖关联+DHCP |
| uuid | u8 [16] | 5 | 回显请求的配网会话 UUID，App 必须校验后才能接受该响应 |

---

<a id="t/cmd/nm/wifi_scan_req"></a>

### `/cmd/nm/wifi_scan_req` 搜索周边无线网络

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.NmWifiScanReq`
- **回执超时**：**15000 ms**
- **消息键**：`app/cmd/nm/wifi_scan_req`

扫描周边 Wi-Fi。force_scan=1 发起新扫描，可能耗时数秒并短暂影响热点通信；force_scan=0 返回已有扫描结果。

- **对应回执**：[/cmd/nm/wifi_scan_rsp](#t/cmd/nm/wifi_scan_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| force_scan | u8 | 1 | 1=发起新扫描（可能耗时数秒）；0=返回已有扫描结果。 |
| max_count | u8 | 2 | 最多返回几个 AP (按信号从强到弱截断); 0 或 >64 按 64 处理 |
| resv | u8 [6] | 3 | 保留字段；发送时置0，接收时忽略。 |

---

<a id="t/cmd/nm/wifi_scan_rsp"></a>

### `/cmd/nm/wifi_scan_rsp` 无线网络搜索结果

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.NmWifiScanRsp`
- **消息键**：`device/cmd/nm/wifi_scan_rsp`

设备 -> App: Wi-Fi 扫描结果。aps 按 RSSI 从强到弱排序, 同 SSID 已去重, 隐藏 SSID 已过滤。total_found > ap_count 表示结果被 max_count 截断。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| result | u8 | 1 | 0=成功，1=正在搜索，2=网络接口异常，3=无线控制接口异常，4=搜索失败；失败时结果可能来自已有记录或为空。 |
| ap_count | u8 | 2 | aps 有效项数, 0..20 |
| total_found | u8 | 3 | 去重后实际扫描到的 AP 总数 (可能 > ap_count, 表示被截断); 上限 255 |
| resv | u8 [5] | 4 | 保留字段；发送时置0，接收时忽略。 |
| aps | nm_wifi_ap_info [64] | 5 | 扫描结果列表；当前最多返回 20 项。 |
| recent_ssid_len | u8 | 6 | 最近一次成功配网的 SSID 长度；仅当该 SSID 在本次扫描结果中出现时返回，否则为 0 |
| recent_ssid | u8 [32] | 7 | 最近一次成功配网且当前扫描可见的 SSID |
| recent_psk_len | u8 | 8 | 与 recent_ssid 对应的已保存密码长度；仅当 SSID 在本次扫描中可见时非 0 |
| recent_psk | u8 [64] | 9 | 仅返回与当前扫描可见的最近配网 SSID 对应的已保存密码；日志必须脱敏 |

#### 复合字段说明：nm_wifi_ap_info

| 字段 | 字段编号 | 中文说明 |
| --- | --- | --- |
| ssid_len | 1 | 网络名称的有效字节数，1–32。 |
| ssid | 2 | 网络名称原始字节，通常为UTF-8，不含末尾终止符。 |
| bssid | 3 | 热点的6字节MAC地址（BSSID）。 |
| channel | 4 | 无线信道号；1–14为2.4GHz，36及以上为5GHz，0为未知。 |
| rssi | 5 | 接收信号强度，单位dBm；0表示未取得。 |
| security | 6 | 0=开放网络，1=WEP，2=WPA，3=WPA2，4=WPA3，5=WPA2/WPA3混合。 |
| resv | 7 | 保留字段，接收时忽略。 |

---

<a id="t/cmd/nm/wifi_switch_to_station_req"></a>

### `/cmd/nm/wifi_switch_to_station_req` 请求设备切换为无线网络客户端模式

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.NmWifiSwitchToStationReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/nm/wifi_switch_to_station_req`
- **对应回执**：[/cmd/nm/wifi_switch_to_station_rsp](#t/cmd/nm/wifi_switch_to_station_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| trigger_source | u8 | 1 | 请求来源：0=其他，1=设备按键，2=客户端，3=生产设置。 |
| watchdog_sec | u8 | 2 | 切换后的联网监测超时，单位秒；0=禁用该超时监测。 |
| resv | u8 [6] | 3 | 保留字段；发送时置0，接收时忽略。 |

---

<a id="t/cmd/nm/wifi_switch_to_station_result_req"></a>

### `/cmd/nm/wifi_switch_to_station_result_req` 查询网络模式切换结果

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.NmWifiSwitchToStationResultReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/nm/wifi_switch_to_station_result_req`

App -> 设备: 查询最近一次异步切换为无线网络客户端模式 的实际执行结果。

- **对应回执**：[/cmd/nm/wifi_switch_to_station_result_rsp](#t/cmd/nm/wifi_switch_to_station_result_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| resv | u8 [8] | 1 | 保留字段；发送时置0，接收时忽略。 |

---

<a id="t/cmd/nm/wifi_switch_to_station_result_rsp"></a>

### `/cmd/nm/wifi_switch_to_station_result_rsp` 网络模式切换的当前进度和结果

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.NmWifiSwitchToStationResultRsp`
- **消息键**：`device/cmd/nm/wifi_switch_to_station_result_rsp`

设备 -> App: 最近一次异步切换为无线网络客户端模式 的状态和实际结果。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| valid | u8 | 1 | 1=存在最近一次请求; 0=尚无请求 |
| pending | u8 | 2 | 1=切换仍在执行; 0=result/current_mode 为最终值 |
| result | u8 | 3 | 实际执行结果：0=成功，1=已处于客户端模式，3=热点停止失败，4=无线接口启动失败，5=联网监测启动失败。pending=1时尚未完成。 |
| current_mode | u8 | 4 | Wi-Fi工作模式：0=未知，1=设备热点，2=客户端模式就绪但未连接，3=已连接上游无线网络。 |
| resv | u8 [4] | 5 | 保留字段；发送时置0，接收时忽略。 |

---

<a id="t/cmd/nm/wifi_switch_to_station_rsp"></a>

### `/cmd/nm/wifi_switch_to_station_rsp` 网络模式切换请求的受理结果

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.NmWifiSwitchToStationRsp`
- **消息键**：`device/cmd/nm/wifi_switch_to_station_rsp`

设备 -> App: 切换请求的接受确认。result=0 表示已排定异步切换；current_mode 是接受请求时的模式，不表示耗时切换已完成。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| result | u8 | 1 | 0=成功，1=已经处于客户端模式，2=蓝牙停止失败，3=热点停止失败，4=无线接口启动失败，5=联网监测启动失败；实际结果以切换结果查询为准。 |
| current_mode | u8 | 2 | Wi-Fi工作模式：0=未知，1=设备热点，2=客户端模式就绪但未连接，3=已连接上游无线网络。 |
| resv | u8 [6] | 3 | 保留字段；发送时置0，接收时忽略。 |

---

<a id="t/cmd/payload/i2c_transfer_req"></a>

### `/cmd/payload/i2c_transfer_req` 头顶配件原始 I²C 读写透传

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.PayloadI2cTransferReq`
- **回执超时**：**3000 ms**
- **消息键**：`app/cmd/payload/i2c_transfer_req`
- **对应回执**：[/cmd/payload/i2c_transfer_rsp](#t/cmd/payload/i2c_transfer_rsp)

向机器人头顶配件接口发起原始 I²C 读写。使用前须先通过 [`/cmd/system/payload_power_req`](#t/cmd/system/payload_power_req) 切入 SDK I²C 模式并完成供电（`TEST_ON` 或 `ON`）；未供电时配件 I²C 服务未启动，请求会超时。

- 7 位从机地址范围 `0x08`–`0x77`（支持全部合法地址）。
- `write_len` 和 `read_len` 均支持 `0`–`128` 字节，且两者至少一个非零。
- **`write_data` 必须固定填满 128 字节**：即使 `write_len < 128`（包括纯读操作 `write_len=0`），Protobuf 中也必须编码完整 128 字节（未使用的尾部字节补 `0`），否则设备不会转发或响应本次请求；实际发往 I²C 总线的仅前 `write_len` 字节。
- 当 `write_len > 0` 且 `read_len > 0` 时，设备在单次 I²C 事务中使用 **repeated START** 执行组合写读。
- 本操作非幂等，超时或失败后不得自动重发；收到 NACK、短传输或超时后写入可能已产生副作用，请按外设协议核实。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| address | u8 | 1 | 7 位 I²C 从机地址，合法范围 `0x08`–`0x77`。 |
| write_len | u8 | 2 | 写入字节数（`0`–`128`），与 `read_len` 至少一个非零。 |
| read_len | u8 | 3 | 读取字节数（`0`–`128`），与 `write_len` 至少一个非零。 |
| write_data | u8 [128] | 4 | 固定 128 字节写入缓冲（不足 128 字节须补 `0` 填满，否则请求会被丢弃）；仅前 `write_len` 字节有效。 |

---

<a id="t/cmd/payload/i2c_transfer_rsp"></a>

### `/cmd/payload/i2c_transfer_rsp` 头顶配件原始 I²C 读写响应

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.PayloadI2cTransferRsp`
- **消息键**：`device/cmd/payload/i2c_transfer_rsp`

成功时返回 `read_len` 个有效读取字节；失败时返回 0 字节，不自动重试。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | 执行结果：`0`=成功，`1`=参数无效，`2`=地址占用，`3`=从机未应答（NACK），`4`=总线超时，`5`=传输失败。 |
| read_len | u8 | 2 | 实际有效读取字节数（`0`–`128`）；失败时为 `0`。 |
| read_data | u8 [128] | 3 | 读取数据缓冲，仅前 `read_len` 字节有效。 |

---

<a id="t/status/pmu/osd_status"></a>

### `/status/pmu/osd_status` 电源状态；底部充电触点与 Type-C 供电通路的开关状态相互独立。

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.PmuOsdStatus`
- **消息键**：`device/status/pmu/osd_status`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| soc | u8 | 1 | 电池剩余电量百分比。 |
| power_bits | u8 | 2 | bit0=input_present，bit1=is_charging；bit2=底部充电触点通路开关，bit3=Type-C 供电通路开关。bit2、bit3 均为 1=开启、0=关闭，可同时开启或关闭，不表示互斥电源来源，也不表示正在充电；bit4–7 保留为 0。 |
| abnormal_flags | u16 | 3 | 电源异常标志位，0表示没有已置位的异常；具体位按设备电源故障定义解释，不能直接作为完整故障码。 |

#### 位域 · bit 0 为最低位

| 字段 | 位 | 名称 | 含义 |
| --- | --- | --- | --- |
| power_bits | 0 | input_present | 1=检测到外部供电输入。 |
| power_bits | 1 | is_charging | 1=正在充电。 |
| power_bits | 2 | charging_contact_mos_enabled | 底部充电触点通路开关：1=开启，0=关闭。 |
| power_bits | 3 | type_c_mos_enabled | Type-C供电通路开关：1=开启，0=关闭。 |
| power_bits | 4–7 | reserved0 | 保留位，发送时置0，接收时忽略。 |

---

<a id="t/cmd/rc/function_key_mode_ack"></a>

### `/cmd/rc/function_key_mode_ack` 遥控器功能按键模式响应

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.RcFunctionKeyModeAck`
- **消息键**：`device/cmd/rc/function_key_mode_ack`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | 按键模式设置结果：0=成功，非零=失败。 |
| reserved | u8 [3] | 2 | 保留字段；发送时置0，接收时忽略。 |

---

<a id="t/cmd/rc/function_key_mode"></a>

### `/cmd/rc/function_key_mode` 遥控器功能按键模式

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.RcFunctionKeyMode`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/rc/function_key_mode`
- **对应回执**：[/cmd/rc/function_key_mode_ack](#t/cmd/rc/function_key_mode_ack)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| camera_btn_mode | u8 | 1 | 遥控器相机功能键的目标模式编号，仅在 update_flag 的 bit0 置位时更新；按遥控器模式定义解释，不直接套用任务编号。 |
| update_flag | u8 | 3 | 更新选择值：固定为 1，仅更新相机功能键。 |
| reserved | u16 | 4 | 保留字段；发送时置0，接收时忽略。 |

---

<a id="t/cmd/rc/joystick_calib_req"></a>

### `/cmd/rc/joystick_calib_req` 开始或停止遥控器摇杆校准

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.RcJoystickCalibReq`
- **消息键**：`app/cmd/rc/joystick_calib_req`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| action | u8 | 1 | 校准操作：1=开始，2=停止；进度通过遥控器校准进度消息上报。 |
| receiver_type | u8 | 2 | 目标遥控器设备类型；UWB遥控器为7。 |
| receiver_index | u8 | 3 | 目标遥控器地址索引；四个UWB遥控器位置对应0x01、0x11、0x21、0x31，应使用实际设备地址。 |

---

<a id="t/cmd/rc/vibration_get_req"></a>

### `/cmd/rc/vibration_get_req` 查询遥控器震动强度

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.RcVibrationGetReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/rc/vibration_get_req`
- **对应回执**：[/cmd/rc/vibration_get_rsp](#t/cmd/rc/vibration_get_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| receiver_type | u8 | 1 | 目标遥控器设备类型；UWB遥控器为7。 |
| receiver_index | u8 | 2 | 目标遥控器地址索引；四个UWB遥控器位置对应0x01、0x11、0x21、0x31，应使用实际设备地址。 |

---

<a id="t/cmd/rc/vibration_get_rsp"></a>

### `/cmd/rc/vibration_get_rsp` 遥控器震动强度查询结果

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.RcVibrationGetRsp`
- **消息键**：`device/cmd/rc/vibration_get_rsp`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| level | u8 | 1 | 遥控器震动强度：1=低，2=中，3=高；设置响应返回设备报告的强度。 |

---

<a id="t/cmd/rc/vibration_set_req"></a>

### `/cmd/rc/vibration_set_req` 设置遥控器震动强度

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.RcVibrationSetReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/rc/vibration_set_req`
- **对应回执**：[/cmd/rc/vibration_set_rsp](#t/cmd/rc/vibration_set_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| level | u8 | 1 | 遥控器震动强度：1=低，2=中，3=高；设置响应返回设备报告的强度。 |
| receiver_type | u8 | 2 | 目标遥控器设备类型；UWB遥控器为7。 |
| receiver_index | u8 | 3 | 目标遥控器地址索引；四个UWB遥控器位置对应0x01、0x11、0x21、0x31，应使用实际设备地址。 |

---

<a id="t/cmd/rc/vibration_set_rsp"></a>

### `/cmd/rc/vibration_set_rsp` 遥控器震动强度设置结果

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.RcVibrationSetRsp`
- **消息键**：`device/cmd/rc/vibration_set_rsp`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| level | u8 | 1 | 遥控器震动强度：1=低，2=中，3=高；设置响应返回设备报告的强度。 |

---

<a id="t/status/rc/fsm_state"></a>

### `/status/rc/fsm_state` 遥控器状态变化事件

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.RcFsmState`
- **消息键**：`device/status/rc/fsm_state`

RC 状态机转移事件遥测

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| timestamp_ms | u32 | 1 | 遥控器状态变化时间戳，单位毫秒。 |
| prev_state | u8 | 2 | 变化前的遥控器主状态编号。 |
| prev_sub | u8 | 3 | 变化前的子状态编码，需结合主状态解释。 |
| key_id | u8 | 4 | 触发变化的按键编号；由设备状态变化触发时为0。 |
| event_type | u8 | 5 | 触发变化的按键事件类型；由设备状态变化触发时为0。 |
| next_state | u8 | 6 | 变化后的遥控器主状态编号。 |
| next_sub | u8 | 7 | 变化后的子状态编码，需结合主状态解释。 |

---

<a id="t/status/rc/joystick_calib_progress"></a>

### `/status/rc/joystick_calib_progress` 遥控器摇杆校准状态与进度

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.RcJoystickCalibProgress`
- **消息键**：`device/status/rc/joystick_calib_progress`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| status | u8 | 1 | 校准状态：0=正常进行或已完成，1=失败；状态0且进度100表示完成。 |
| progress | u8 | 2 | 校准进度百分比，0–100。 |

---

<a id="t/status/rc/joystick_raw"></a>

### `/status/rc/joystick_raw` 遥控器摇杆原始位置

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.RcJoystickRaw`
- **消息键**：`device/status/rc/joystick_raw`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| x | u8 | 1 | 摇杆横轴原始值；按有符号8位补码解释，显示范围−100至100，0为中位。 |
| y | u8 | 2 | 摇杆纵轴原始值；按有符号8位补码解释，显示范围−100至100，0为中位。 |

---

<a id="t/cmd/robot/sn_req"></a>

### `/cmd/robot/sn_req` 查询机器人序列号

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.RobotSnReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/robot/sn_req`
- **对应回执**：[/cmd/robot/sn_rsp](#t/cmd/robot/sn_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| sn_type | u8 | 1 | 序列号查询类型；查询本机器人时填0，响应回显请求值。 |

---

<a id="t/cmd/robot/sn_rsp"></a>

### `/cmd/robot/sn_rsp` 机器人序列号查询响应

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.RobotSnRsp`
- **消息键**：`device/cmd/robot/sn_rsp`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| sn_type | u8 | 1 | 序列号查询类型；查询本机器人时填0，响应回显请求值。 |
| lens | u8 | 2 | sn 中有效字节数；0表示未取得有效序列号。 |
| sn | u8 [128] | 3 | 设备序列号字节，按 lens 取有效内容，不包含末尾填充。 |

---

<a id="t/status/robot/status"></a>

### `/status/robot/status` 机器人运动状态

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.RobotStatus`
- **消息键**：`device/status/robot/status`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| robot_state | u8 | 1 | 机身状态：0=释放/未驱动，1=正在起身，2=正在收纳，3=活动状态；仍须结合暂停及故障状态判断能否运动。 |
| velocity | i16 | 2 | 速度 cm/s (m/s = /100.0) |
| trip_odom_dis | f32 | 3 | 累计里程 m |
| obstacle_dis | u16 | 4 | 最近障碍距离 cm (0xFFFF 无效) |
| pitch_angle | i16 | 5 | 机身俯仰角，单位0.1度。 |
| gimbal_angle | i16 | 6 | 云台角 0.1度 |
| roll_angle | i16 | 7 | 机身横滚角，单位0.1度。 |
| height | u8 | 8 | 高度 0.1cm |
| sport_mode | u8 | 9 | 当前运动模式编号，按设备运动模式定义解释。 |
| abnormal_flags | u32 | 10 | 机身异常标志位；0表示本字段未报告异常，完整故障状态仍通过故障接口获取。 |
| session_odom_dis | f32 | 11 | 起身里程，单位 m。 |

---

<a id="t/data/robot/sdk_sensor_batch"></a>

### `/data/robot/sdk_sensor_batch` 25 Hz 批量传感器遥测（机身 IMU / 云台 IMU / 关节编码器）

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.data.RobotSdkSensorBatch`
- **消息键**：`device/data/robot/sdk_sensor_batch`

面向 SDK 的 25 Hz 批量传感器遥测（每 40 ms 推送一包）。建立 DataChannel Zenoh 会话后直接订阅即可，无需额外开启命令。单包按时间升序合并机身 IMU（100 Hz，最多 4 条）、云台 IMU（50 Hz，最多 2 条）与 7 关节编码器状态（50 Hz，最多 2 条）。所有样本的 `sample_ts_ns` 均使用设备单调时钟（`CLOCK_MONOTONIC`），与双目 SDK 视频流 SEI 中的 `left_mono_ns` / `right_mono_ns` 同源，可用于 SLAM/VIO 与重定位帧间插值对齐。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| version | u8 | 1 | 批量遥测格式版本，当前固定为 `1`。 |
| batch_seq | u32 | 2 | 递增批次序号（每 25 Hz 批次加 1），用于检测丢包与乱序。 |
| body_imu | sdk_imu_sample [4] | 3 | 机身 IMU 样本序列（100 Hz，时间升序，最多 4 条），见下方复合字段说明。 |
| gimbal_imu | sdk_imu_sample [2] | 4 | 云台 IMU 样本序列（50 Hz，时间升序，最多 2 条），见下方复合字段说明。 |
| joints | sdk_joint_sample [2] | 5 | 7 关节编码器样本序列（50 Hz，时间升序，最多 2 条；下标 6 为云台 yaw 电机），见下方复合字段说明。 |

#### 复合字段说明：sdk_imu_sample

| 字段 | 字段编号 | 说明 |
| --- | --- | --- |
| sample_ts_ns | 1 | `u64`，设备单调时钟采样时间戳，单位 ns。 |
| gyro | 2 | `f32 [3]`，三轴角速度 `[x, y, z]`，单位 rad/s。 |
| accel | 3 | `f32 [3]`，三轴线加速度 `[x, y, z]`，单位 m/s²。 |
| euler | 4 | `f32 [3]`，姿态欧拉角 `[roll, pitch, yaw]`，单位 rad。 |
| online | 5 | `u8`，传感器在线状态：`0`=离线，`1`=在线。 |

#### 复合字段说明：sdk_joint_sample

7 个关节的数组顺序固定为：`0=L1（左髋）`、`1=L2（左膝）`、`2=LW（左轮）`、`3=R1（右髋）`、`4=R2（右膝）`、`5=RW（右轮）`、`6=GIMY（云台 yaw 电机）`。

| 字段 | 字段编号 | 说明 |
| --- | --- | --- |
| sample_ts_ns | 1 | `u64`，设备单调时钟采样时间戳，单位 ns。 |
| position | 2 | `f32 [7]`，7 关节编码器位置（输出轴角），单位 rad；下标 6 为云台 yaw 电机编码器角。 |
| velocity | 3 | `f32 [7]`，7 关节输出轴角速度，单位 rad/s。 |
| online_mask | 4 | `u8`，关节在线掩码，`bit0`–`bit6` 依次对应 7 个关节：`1`=在线，`0`=离线。 |

---

<a id="t/status/scheduler/status"></a>

### `/status/scheduler/status` 设备任务与运动状态

- **方向**：机器人 → 客户端（订阅）

[查看各 mission_id 对应的 sub_mode 取值表](#t/cmd/mission/request)

[查看 Camera 模式与动作适用条件](#camera-mode-guide)

- **Protobuf 消息**：`md.protocol.status.SchedulerStatus`
- **消息键**：`device/status/scheduler/status`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| current_mode | u8 | 1 | 当前生效的任务编号；本手册中的1=相机、3=宠物。未列出的值不应按已知任务推断。 |
| sub_mode | u8 | 2 | 当前任务的子模式。相机任务：0=手动，1=聚焦，2=跟随；其他任务按各自定义解释。 |
| request_source | u8 | 3 | 模式请求来源：0=客户端，1=遥控器，2=语音，3=设备自主行为，4=安全处理。 |
| is_locked | u8 | 4 | 姿态锁定状态：0=已起身，1=未起身、过渡中或异常；表示实际状态，不是上一次按键意图。 |
| is_paused | u8 | 5 | 当前暂停状态：0=正常，1=暂停。 |
| timestamp_ns | u64 | 6 | 状态生成的单调时间戳，单位纳秒。 |
| sub_mode_state | u8 [14] | 7 | 当前任务与子模式的14字节扩展状态，具体布局见任务请求中的对应说明；第14字节（下标13）保留为0。 |
| ego_active | u8 | 8 | 当前是否处于设备自主互动状态：0=否，1=是；不等同于任务编号。 |
| obstacle_avoidance_flags | u8 | 9 | 避障状态位：bit0=请求避障，bit1=避障实际生效，bit2=用户控制已介入，bit3=支持四状态表示，bit4–7保留为0。 |

---

<a id="t/cmd/speaker/play_cmd"></a>

### `/cmd/speaker/play_cmd` 播放设备内置喇叭音效

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.SpeakerPlayCmd`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/speaker/play_cmd`
- **对应回执**：[/cmd/speaker/play_rsp](#t/cmd/speaker/play_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| sound_effect_id | u8 | 1 | 内置喇叭音效编号：0或1；不能填写文件路径。 |

#### 设备自带喇叭音效

sound_effect_id=0 或 1 播放对应音效，可直接请求，不需要先开启通话。同一音效在 1000 ms 内重复请求会去重。

---

<a id="t/cmd/speaker/play_rsp"></a>

### `/cmd/speaker/play_rsp` 内置音效播放请求的受理结果

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.SpeakerPlayRsp`
- **消息键**：`device/cmd/speaker/play_rsp`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | 0=已接受，1=音效编号无效，2=提交失败，3=重复请求已去重，4=格式异常；已接受不表示播放完成。 |

ret_code：0 已接受，1 音效 ID 无效，2 请求提交失败，3 重复请求已去重，4 请求格式异常。已接受不表示播放完成。

---

<a id="t/status/storage/status"></a>

### `/status/storage/status` 存储状态

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.StorageStatus`
- **消息键**：`device/status/storage/status`

存储状态。sd_status 是既有公开契约，只暴露 0..3，MSD 占用与格式化在线上都 折叠为 UNMOUNTED，消费方据此无法区分「没插卡」与「插着但被电脑占用」。 msd_active 单独携带 MSD 是否正把存储导出给主机，供需要给出可行动提示的 消费方（相机录制准入）区分两种成因。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| internal_used_space | u32 | 1 | 内置存储已用容量，单位KiB（1024字节）。 |
| internal_total_space | u32 | 2 | 内置存储总容量，单位KiB（1024字节）。 |
| sd_used_space | u32 | 3 | SD卡已用容量，单位KiB（1024字节）。 |
| sd_total_space | u32 | 4 | SD卡总容量，单位KiB（1024字节）。 |
| storage_type | u8 | 5 | 当前存储选择：0=内置存储，1=SD卡。 |
| sd_status | u8 | 6 | SD卡状态：0=未插入，1=可用，2=未挂载/不可用于本机访问，3=异常；需结合 msd_active 区分电脑占用。 |
| sd_transfer_speed | u8 | 7 | SD卡速度等级：0=低速，1=高速；不是实时吞吐量数值。 |
| sd_fs_type | u8 | 8 | SD卡文件系统类型：0=未格式化，1=exFAT，2=FAT32，3=未知。 |
| internal_threshold_alert | u16 | 9 | 内置存储空间提示：0=正常，1=空间阈值告警。 |
| sd_threshold_alert | u16 | 10 | SD卡空间提示：0=正常，1=空间阈值告警。 |
| internal_exported_to_host | u8 | 11 | 内置存储是否正导出给电脑：0=否，1=是；导出期间不能当作本机可正常写入。 |
| msd_active | u8 | 12 | 存储是否正作为 USB 大容量设备导出给主机；1=被主机占用，此时 sd_status 折叠为 UNMOUNTED |
| recording_min_free_space_mb | u32 | 13 | 起录和停录的剩余可用容量下限；单位MB=1000000字节，0 表示设备未提供该容量下限，客户端仍需遵守现有存储空间限制 |

---

<a id="t/cmd/system/payload_power_req"></a>

### `/cmd/system/payload_power_req` 头顶配件供电控制与 SDK I²C 模式切换

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.SystemPayloadPowerReq`
- **回执超时**：**3000 ms**
- **消息键**：`app/cmd/system/payload_power_req`
- **对应回执**：[/cmd/system/payload_power_rsp](#t/cmd/system/payload_power_rsp)

控制机器人头顶配件接口供电并切换 SDK I²C 模式。

- 开机默认为 `AUTO`（`0`，正常用户模式），按在位信号自动供电以服务官方配件。使用官方配件时请直接走业务协议，不要切入 SDK I²C 模式。
- 仅在 SDK 会话生效期间允许下发 `mode=1/2/3`（`OFF` / `TEST_ON` / `ON`）进入 SDK I²C 模式：设备停止主动轮询，仅按需响应 [`/cmd/payload/i2c_transfer_req`](#t/cmd/payload/i2c_transfer_req)。
- `TEST_ON`（`mode=2`）不等待在位信号，需填写 `duration_ms=1..60000`，到期后自动停止配件服务并断电；`ON`（`mode=3`）持续供电；`OFF`（`mode=1`）停止配件服务并断电；`AUTO`（`mode=0`）显式恢复正常用户模式。除 `TEST_ON` 外，`duration_ms` 均填 `0`（Protobuf 中两个字段均须编码，包括值为 0 时）。
- SDK 会话断开或控制权失效时，设备会自动恢复 `AUTO` 模式。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| duration_ms | u32 | 1 | `TEST_ON` 供电时长：`1`–`60000` ms；其他模式填 `0`。 |
| mode | system_payload_power_mode | 2 | 目标模式：`0`=AUTO，`1`=OFF，`2`=TEST_ON，`3`=ON。 |

---

<a id="t/cmd/system/payload_power_rsp"></a>

### `/cmd/system/payload_power_rsp` 头顶配件供电控制响应

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.SystemPayloadPowerRsp`
- **消息键**：`device/cmd/system/payload_power_rsp`

回显请求序号。`result=0`（APPLIED）表示设备已执行供电切换；重发同序号请求时回报当前状态且不重启计时。可通过 [`/cmd/system/payload_power_status_req`](#t/cmd/system/payload_power_status_req) 查询配件 I²C 服务运行状态。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| result | system_payload_power_result | 1 | 执行结果：`0`=已应用（APPLIED），`1`=参数无效（INVALID），`2`=正忙（BUSY），`3`=硬件控制失败（IO_ERROR），`4`=请求过期（STALE），`5`=拒绝/非 SDK 控制态（DENIED）。 |
| mode | system_payload_power_mode | 2 | 当前供电模式：`0`=AUTO，`1`=OFF，`2`=TEST_ON，`3`=ON。 |
| inserted | u8 | 3 | 最近一次有效在位读数：`0`=未接入，`1`=已接入。 |
| enabled | u8 | 4 | 电源使能状态回读：`0`=关闭，`1`=开启，`2`=无法确认。 |

---

<a id="t/cmd/system/payload_power_status_req"></a>

### `/cmd/system/payload_power_status_req` 查询头顶配件供电状态

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.SystemPayloadPowerStatusReq`
- **回执超时**：**3000 ms**
- **消息键**：`app/cmd/system/payload_power_status_req`
- **对应回执**：[/cmd/system/payload_power_status_rsp](#t/cmd/system/payload_power_status_rsp)

无业务字段，发送时填写非零 `header.seq_num` 并将 `data` 编码为空子消息（`12 00`）。

---

<a id="t/cmd/system/payload_power_status_rsp"></a>

### `/cmd/system/payload_power_status_rsp` 头顶配件供电状态响应

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.SystemPayloadPowerStatusRsp`
- **消息键**：`device/cmd/system/payload_power_status_rsp`

返回设备最近一次监控周期的头顶配件供电与配件服务运行状态。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| remaining_ms | u32 | 1 | 距 `TEST_ON` 到期的剩余毫秒数（向上取整）；其他模式或已到期时为 `0`。 |
| mode | system_payload_power_mode | 2 | 当前供电模式：`0`=AUTO，`1`=OFF，`2`=TEST_ON，`3`=ON。 |
| inserted | u8 | 3 | 最近一次有效在位读数：`0`=未在位或检测异常，`1`=已在位。 |
| enabled | u8 | 4 | 电源使能状态回读：`0`=关闭，`1`=开启，`2`=无法确认。 |
| daemon_running | u8 | 5 | `1`=最近一次检查确认配件服务已运行或正在启动，`0`=未确认。 |

---

<a id="t/cmd/time/correction_req"></a>

### `/cmd/time/correction_req` 校正机器人系统时间

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.TimeCorrectionReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/time/correction_req`
- **对应回执**：[/cmd/time/correction_rsp](#t/cmd/time/correction_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| timestamp | u64 | 1 | 客户端提供的Unix时间戳，单位毫秒。 |
| offset | u32 | 2 | 附加时间补偿，单位毫秒，设备按 timestamp+offset 计算目标时间；不是时区偏移，无补偿时填0。 |

---

<a id="t/cmd/time/correction_rsp"></a>

### `/cmd/time/correction_rsp` 系统时间校正响应

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.TimeCorrectionRsp`
- **消息键**：`device/cmd/time/correction_rsp`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | 0=校时请求处理成功（也可能因频率限制未重复调整），非零=失败；不能据此推断时钟同步精度。 |

---

<a id="t/cmd/tracking/ctrl_cmd"></a>

### `/cmd/tracking/ctrl_cmd` 选择或取消视觉跟踪目标

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.TrackingCtrlCmd`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/tracking/ctrl_cmd`

目标跟踪控制命令 (APP端 → 车端，双向命令)

- **对应回执**：[/cmd/tracking/ctrl_cmd_ack](#t/cmd/tracking/ctrl_cmd_ack)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| selected_id | u8 | 1 | 客户端选中的目标编号；请求填写目标上报中的 id，响应返回设备确认的目标编号。 |
| start_tracking | u8 | 2 | 跟踪操作：1=开始，0=取消；响应回显确认的操作。 |
| selected_pos_x | f32 | 3 | 目标中心的归一化横坐标，0–1，左侧为0；坐标以有效视频画面为准，不包含显示黑边。 |
| selected_pos_y | f32 | 4 | 目标中心的归一化纵坐标，0–1，上侧为0。 |
| request_mode | u8 | 5 | 选择方式：0=用户点选，1=辅助聚焦。用户点选需同时提供正确目标坐标。 |
| reserved1 | u8 | 6 | 保留字段；发送时置0，接收时忽略。 |
| reserved2 | u8 | 7 | 保留字段；发送时置0，接收时忽略。 |
| reserved3 | u8 | 8 | 保留字段；发送时置0，接收时忽略。 |

#### 用户点选跟踪

selected_pos_x、selected_pos_y 为图传有效画面内的归一化坐标（f32，0–1），原点在左上，向右、向下递增。选择已上报的目标框时，把同一目标的 id、center_pos_x、center_pos_y 分别填入 selected_id、selected_pos_x、selected_pos_y。坐标已归一化，无需再次除以分辨率。用鼠标点选时应先剔除视频黑边。

request_mode 枚举：0 = USER（用户点选），1 = ASSIST_SPOTLIGHT（设备辅助聚焦，按指定跟踪 ID 选目标）。用户点选使用 0，视觉跟踪会按位置匹配目标，即使已填写非零 ID 也应提供正确坐标。开始填 start_tracking=1；取消填 0，沿用目标 ID，两坐标清零。两种操作的 reserved1/2/3 均按 App 填 0。

---

<a id="t/cmd/tracking/ctrl_cmd_ack"></a>

### `/cmd/tracking/ctrl_cmd_ack` 目标跟踪请求的处理结果

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.TrackingCtrlCmdAck`
- **消息键**：`device/cmd/tracking/ctrl_cmd_ack`

目标跟踪控制命令ACK (车端 → APP端)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| result | u8 | 1 | 0=成功，1=目标编号无效，2=目标不可用或已丢失。 |
| selected_id | u8 | 2 | 客户端选中的目标编号；请求填写目标上报中的 id，响应返回设备确认的目标编号。 |
| start_tracking | u8 | 3 | 跟踪操作：1=开始，0=取消；响应回显确认的操作。 |
| reserved | u8 | 4 | 保留字段；发送时置0，接收时忽略。 |

#### 显示设备回执

按 header.seq_num 关联原请求。result=0 成功，1 目标 ID 无效，2 目标不可用或已丢失。显示设备返回的 selected_id 和 start_tracking（1 开始、0 取消）。按目标框点选时，只有成功且目标 ID 和操作均与请求一致，才将该目标标为已接受。持续跟踪状态订阅 device/status/tracking/status。

---

<a id="t/cmd/tracking/param_query_req"></a>

### `/cmd/tracking/param_query_req` 查询目标跟踪参数

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.TrackingParamQueryReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/tracking/param_query_req`
- **对应回执**：[/cmd/tracking/param_query_rsp](#t/cmd/tracking/param_query_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| param_id | u8 | 1 | 跟踪参数编号：1=参考人体身高，2=遥控器定位参与跟随的用户开关。 |
| reserved | u8 [3] | 2 | 保留字段；发送时置0，接收时忽略。 |

---

<a id="t/cmd/tracking/param_query_rsp"></a>

### `/cmd/tracking/param_query_rsp` 目标跟踪参数查询响应

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.TrackingParamQueryRsp`
- **消息键**：`device/cmd/tracking/param_query_rsp`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| param_id | u8 | 1 | 跟踪参数编号：1=参考人体身高，2=遥控器定位参与跟随的用户开关。 |
| ret_code | u8 | 2 | 0=成功，1=未知参数；失败时不使用 param_value。 |
| reserved | u8 | 3 | 保留字段；发送时置0，接收时忽略。 |
| param_value | f32 | 4 | 编号1：身高，单位米，范围0.1–3.0；编号2：只允许0.0关闭或1.0开启。编号2查询返回用户选择，实际生效状态读取 uwb_follow_enabled。 |

---

<a id="t/cmd/tracking/param_update_req"></a>

### `/cmd/tracking/param_update_req` 设置目标跟踪参数

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.TrackingParamUpdateReq`
- **回执超时**：**5000 ms**
- **消息键**：`app/cmd/tracking/param_update_req`
- **对应回执**：[/cmd/tracking/param_update_rsp](#t/cmd/tracking/param_update_rsp)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| param_id | u8 | 1 | 跟踪参数编号：1=参考人体身高，2=遥控器定位参与跟随的用户开关。 |
| param_value | f32 | 2 | 编号1：身高，单位米，范围0.1–3.0；编号2：只允许0.0关闭或1.0开启。编号2查询返回用户选择，实际生效状态读取 uwb_follow_enabled。 |
| reserved | u8 [3] | 3 | 保留字段；发送时置0，接收时忽略。 |

---

<a id="t/cmd/tracking/param_update_rsp"></a>

### `/cmd/tracking/param_update_rsp` 目标跟踪参数设置响应

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.TrackingParamUpdateRsp`
- **消息键**：`device/cmd/tracking/param_update_rsp`

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | 0=设置已接受，1=未知参数，2=数值无效；成功不保证所有设置在重启后保留。 |
| reserved | u8 [3] | 2 | 保留字段；发送时置0，接收时忽略。 |

---

<a id="t/data/tracking/target_data"></a>

### `/data/tracking/target_data` 可选视觉目标列表

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.data.TrackingTargetData`
- **消息键**：`device/data/tracking/target_data`

目标检测数据上报 (车端 → APP端，单向推送)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| target_count | u8 | 1 | 有效目标数量，最多5个。 |
| targets | tracking_target_info [5] | 2 | 目标信息列表，前 target_count 项有效；每项含编号、中心坐标、目标框尺寸和跟踪状态，见复合字段说明。 |

用户选择目标框时，直接将该目标的 id 和归一化框中心 center_pos_x、center_pos_y 用于跟踪请求。

#### 复合字段说明：tracking_target_info

| 字段 | 字段编号 | 中文说明 |
| --- | --- | --- |
| id | 1 | 目标编号，0–255。 |
| center_pos_x | 2 | 目标框中心横坐标，归一化到0–1。 |
| center_pos_y | 3 | 目标框中心纵坐标，归一化到0–1。 |
| width | 4 | 目标框宽度，单位像素。 |
| height | 5 | 目标框高度，单位像素。 |
| target_state | 6 | 0=普通候选，1=用户正在跟踪，2=用户跟踪目标已丢失。 |
| reserved_state_flags | 7 | 保留状态位，接收时忽略。 |

---

<a id="t/status/tracking/status"></a>

### `/status/tracking/status` 目标跟踪状态、距离与速度

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.TrackingStatus`
- **消息键**：`device/status/tracking/status`

跟踪状态上报 (车端 → APP端，单向推送)

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| status | u8 | 1 | 跟踪状态：0=空闲，1=跟踪中，2=目标丢失，255=错误。 |
| tracking_id | u8 | 2 | 正在跟踪的目标编号；255表示没有跟踪目标。 |
| target_type | u8 | 3 | 目标类型：0=人，1=狗。 |
| horizontal_dis | u16 | 4 | 目标水平距离，单位厘米；65535表示无效。 |
| horizontal_speed | u16 | 5 | 目标水平速度，单位厘米/秒；65535表示无效。 |
| uwb_follow_enabled | u8 | 6 | 遥控器定位参与跟随的实际生效状态：1=参与，0=不参与。 |

---

<a id="t/status/uwb/rc_status_list"></a>

### `/status/uwb/rc_status_list` 在线遥控器电量、充电状态与信号强度

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.status.UwbRcStatusList`
- **消息键**：`device/status/uwb/rc_status_list`

在线 UWB 遥控器实时状态的完整快照，供配件页面和驾驶界面使用电量、充电状态与信号强度。固定每 1 秒发送一次，状态变化不触发额外发送；只包含在线且已获取有效电量的遥控器，离线条目从后续快照移除；无有效在线条目时不发送，不发送空列表。接收端通过推送超时清除旧状态。每条快照替换整个列表，最多 4 个条目，(receiver_type, receiver_index) 不得重复。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| entries | uwb_rc_status_entry [4] | 1 | 最多4个在线遥控器状态条目，包含电量、充电状态、信号强度、地址和序列号末6位，见复合字段说明。 |

#### 复合字段说明：uwb_rc_status_entry

| 字段 | 字段编号 | 中文说明 |
| --- | --- | --- |
| battery | 1 | 电量百分比0–100；0表示真实0%，不表示未知。 |
| is_charging | 2 | 0=未充电，1=正在充电。 |
| rssi | 3 | 接收信号强度，单位dBm；0表示未取得有效值。 |
| receiver_type | 4 | 遥控器设备类型，当前UWB遥控器为7。 |
| receiver_index | 5 | 遥控器地址索引，四个位置对应0x01、0x11、0x21、0x31。 |
| sn_suffix | 6 | 序列号末6个字符，仅用于显示，不能作为唯一设备标识。 |

---

<a id="t/cmd/vt/sdk_preview_req"></a>

### `/cmd/vt/sdk_preview_req` 查询或设置当前 SDK 会话的视频预览开关（主摄 / 双目拼接）

- **方向**：客户端 → 机器人（发布）
- **Protobuf 消息**：`md.protocol.cmd.VtSdkPreviewReq`
- **回执超时**：**3000 ms**
- **消息键**：`app/cmd/vt/sdk_preview_req`
- **对应回执**：[/cmd/vt/sdk_preview_rsp](#t/cmd/vt/sdk_preview_rsp)

查询或设置当前 SDK 会话的视频预览开关，不改变运动控制。新会话默认关闭全部视频流；`session_id` 取 `beni/session` 17 字节载荷的前 16 字节。

- **`version=1`（仅主摄）**：`operation=0` 查询、`1` 设置；`enabled=0` 关闭、`1` 开启主摄（开启或关闭时会自动关闭双目流）。
- **`version=2`（主摄 + 双目拼接双轨）**：`operation=1` 时按 `stream_mask` 控制各路开关：`bit0`（`1`）=主摄，`bit1`（`2`）=双目上下拼接（`640×960`，上左 `640×480` + 下右 `640×480`），可按位组合（如 `3` 同时开启两路，`0` 全部关闭）。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| version | u8 | 1 | 预览控制协议版本：`1`=仅主摄，`2`=按 `stream_mask` 控制主摄与双目流。 |
| operation | u8 | 2 | `0`=查询当前状态，`1`=设置开关。 |
| enabled | u8 | 3 | `version=1` 且 `operation=1` 时：`0`=关闭，`1`=开启；查询时填 `0`；`version=2` 时取值仍须为 `0` 或 `1`（实际开关以 `stream_mask` 为准）。 |
| session_id | u8 [16] | 4 | 当前控制连接的 16 字节会话 ID，取 `beni/session` 载荷前 16 字节；重连后须重新获取。 |
| stream_mask | u8 | 5 | 可选（默认 `0`）。`version=2` 目标流掩码：`bit0`=主摄，`bit1`=双目上下拼接（640×960）；其余位必须为 `0`。 |

#### 双目拼接 WebRTC 轨道与帧内 SEI 元数据

- **双轨协商**：需要接收双目流时，客户端在 WebRTC Offer 中添加 **2 条 H.265 接收视频轨道**（仅添加 1 条时只协商主摄）。机器人会在 Answer 中将第一条绑为主摄（stream ID `beni`），第二条绑为双目（stream ID `beni-stereo`，默认 30 fps / 1500 kbps，上限 2 Mbps，码率优先级低于主摄）。
- **帧内 SEI 时间戳**：每帧双目 H.265 AU 开头附带一条 Annex-B `Prefix SEI`（`payloadType=0x05` `user_data_unregistered`，总载荷 50 字节），前 16 字节为固定 UUID `"MDSTEREOPAIRV1\0\1"`（`4D 44 53 54 45 52 45 4F 50 41 49 52 56 31 00 01`），后 34 字节按小端序排列：

| 字节偏移（UUID 后） | 字段 | 类型 | 说明 |
| --- | --- | --- | --- |
| `0` | `version` | `u8` | 固定为 `1`。 |
| `1` | `layout` | `u8` | 固定为 `1`（上下拼接：上左 640×480 + 下右 640×480）。 |
| `2`–`5` | `left_seq` | `u32` | 左眼传感器原始递增帧号。 |
| `6`–`9` | `right_seq` | `u32` | 右眼传感器原始递增帧号。 |
| `10`–`17` | `left_mono_ns` | `u64` | 左眼曝光单调时间戳（`CLOCK_MONOTONIC`，ns），与 [`/data/robot/sdk_sensor_batch`](#t/data/robot/sdk_sensor_batch) 同源。 |
| `18`–`25` | `right_mono_ns` | `u64` | 右眼曝光单调时间戳（`CLOCK_MONOTONIC`，ns）。 |
| `26`–`33` | `unix_ns` | `u64` | 对应 Unix 时间戳（ns）。 |

---

<a id="t/cmd/vt/sdk_preview_rsp"></a>

### `/cmd/vt/sdk_preview_rsp` 当前会话的预览开关响应

- **方向**：机器人 → 客户端（订阅）
- **Protobuf 消息**：`md.protocol.cmd.VtSdkPreviewRsp`
- **消息键**：`device/cmd/vt/sdk_preview_rsp`

成功时 `enabled` 和 `stream_mask` 返回设备确认后的当前流状态；关闭仅停止当前 SDK 的对应视频流，已在途的数据仍可能到达，DataChannel 控制连接保持可用。

| 字段 | 类型 / 长度 | `data` tag | 说明 |
| --- | --- | --- | --- |
| ret_code | u8 | 1 | `0`=成功，`1`=会话未授权/失效或对应媒体轨未就绪，`2`=参数无效，`3`=双目流未开放。 |
| enabled | u8 | 2 | 设备确认的汇总预览状态：`0`=全部关闭，`1`=至少一路已开启；仅成功响应有效。 |
| stream_mask | u8 | 3 | 可选（默认 `0`）。当前生效流掩码：`bit0`=主摄，`bit1`=双目上下拼接（640×960）。 |
