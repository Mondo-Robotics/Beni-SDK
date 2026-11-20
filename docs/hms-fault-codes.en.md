# AUTHORIZED TEST USERS ONLY — NOT AVAILABLE IN PUBLIC RELEASES

## MONDO SDK is currently in internal testing. The **app entry point and related firmware capabilities described here are not available in public releases**. Access is limited to users who have been granted testing permission.

## **To express interest in testing, join our [Discord community](https://discord.gg/smcZyv5efa) and wait for further announcements about test access and permissions.**

# HMS Fault Code Reference

Look up robot fault meanings by their full 32-bit fault code (`(module_id << 16) | fault_code`).

[中文版本](<HMS 故障码参考表.md>) ｜ [MONDO SDK User Guide](<MONDO SDK User Guide.en.md>) ｜ [Beni SDK Open Protocol Reference](protocol-reference.en.md)

<a id="sheet-0"></a>

## Fault Code Table

| Fault code | Description |
| --- | --- |
| 0xF0010001 | Insufficient internal-storage space |
| 0xF0010002 | Insufficient SD-card space |
| 0xF0010003 | USB mass-storage mode is using the internal disk |
| 0xF0010004 | Device is charging through a wired connection |
| 0xF0010005 | Robot battery below 25% |
| 0xF0010006 | Robot battery below 10% |
| 0xF0010007 | Available space on the current storage is below 1 GiB |
| 0xF0010008 | SD card removed during recording |
| 0xF0010009 | Device power-off notification |
| 0xF0020001 | Weak video-link signal |
| 0xF0020002 | Severe video-link interference |
| 0xF0020003 | No video-link data |
| 0xF0030101 | No IMU communication feedback for 500 ms |
| 0xF0030102 | Abnormal IMU data values |
| 0xF0030201 | No motor communication feedback for 500 ms |
| 0xF0030202 | Motor temperature exceeds limit |
| 0xF0030203 | Motor stalled continuously for 1.5s |
| 0xF0030204 | Motor reports a hardware error code |
| 0xF0030205 | Sustained joint overload (torque limit exceeded); release torque or brake according to motion state |
| 0xF0030206 | Sustained overall joint-current overload |
| 0xF0030301 | DDS timeout without depth frames |
| 0xF0030401 | Robot tipped over (grav_z exceeds threshold for 2s) |
| 0xF0030402 | Control lost (ctrl_cmd stream interrupted) |
| 0xF0030403 | Flip failed (abnormal attitude or fallen after landing) |
| 0xF0031001 | Gimbal reset-command timeout |
| 0xF0031002 | Joint reset-command timeout |
| 0xF0031003 | Repeated stand-up attempts failed |
| 0xF0031004 | Pre-stand wheel self-check abnormal (missing wheel, mismatched wheel sizes, or detection failure); reported without preventing stand-up |
| 0xF0031005 | Device not activated; further stand-up requests blocked after 5 cumulative successful stand-ups |
| 0xF0031006 | Abnormal device inertia (gimbal sweep during reset exceeds inertia threshold, possibly due to an unauthorized payload); fault reported |
| 0xF0031007 | Stationary-standing verification after joint reset failed; immediately release torque and wait for the user to re-enable reset |
| 0xF0040001 | Battery-pack overvoltage (MD_PMU_ABN_PACK_OVP, bit0) |
| 0xF0040002 | Battery-pack undervoltage (MD_PMU_ABN_PACK_UVP, bit1) |
| 0xF0040003 | Battery-pack overcurrent/abnormal charging or discharging current (MD_PMU_ABN_PACK_OCP, bit2) |
| 0xF0040004 | Battery-pack overtemperature (MD_PMU_ABN_PACK_OTP, bit3) |
| 0xF0040005 | Battery-pack undertemperature (MD_PMU_ABN_PACK_LTP, bit4) |
| 0xF0040006 | Battery-pack short-circuit protection (MD_PMU_ABN_PACK_SCP, bit5) |
| 0xF0040007 | BMS communication error (MD_PMU_ABN_BMS_COMM_ERR, bit6) |
| 0xF0040008 | Charging-IC communication error (MD_PMU_ABN_CHGIC_COMM_ERR, bit7) |
| 0xF0080001 | Actual obstacle-avoidance speed exceeds limit |
| 0xF0080002 | Ambient light too low for obstacle avoidance |
| 0xF0080003 | Obstacle-avoidance software or data abnormal |
| 0xF0080004 | Collision-detection service confirmed a collision |
| 0xF0080005 | Ambient light too low for automatic stair jumping |
| 0xF0080006 | Stereo calibration abnormal; automatic obstacle avoidance and stair jumping unavailable |
| 0xF0080007 | User pitch command exceeds limit; automatic obstacle avoidance restricted |
| 0xF0090001 | Main camera unavailable |
| 0xF00A0001 | Accessory firmware differs from the upgrade baseline; consistency upgrade required |
| 0xF00A0002 | Firmware upgrade in progress; external motion control prohibited during upgrade |
