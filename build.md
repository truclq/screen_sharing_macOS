Plan đã hoàn thành. Tóm tắt:

**ExtendScreen** - Hệ thống biến Android thành màn hình phụ cho macOS qua USB.

**Kiến trúc chính:**
- macOS: CGVirtualDisplay (private API) → ScreenCaptureKit → VideoToolbox HEVC → USB
- Android: USB → MediaCodec HEVC decode → SurfaceView
- Input: Android touch/stylus → USB → macOS CGEvent injection

**4 giai đoạn triển khai:**
1. **PoC**: ADB transport + HEVC encode/decode + 1080p@30fps + basic touch
2. **Virtual Display + AOA**: CGVirtualDisplay + AOA USB + 60fps + resolution negotiation
3. **Polish**: Adaptive quality + full stylus (pressure/tilt) + latency tracking + UI
4. **High Resolution**: 4K → 120fps → 8K exploration

**Lưu ý thực tế:**
- 4K@60fps với <30ms latency là mục tiêu thực tế và đạt được
- 8K@120fps là mục tiêu tham vọng, bị giới hạn bởi hardware hiện tại (CGVirtualDisplay cap 60Hz, Android thiếu 8K decoder)
- CGVirtualDisplay là private API - không thể lên App Store

-----------------------------------------------------------------------------------------------------------------------------

Project Phase 1 hoàn thành. Đây là tổng kết:

## ExtendScreen - Phase 1 Complete

### macOS App (15 Swift files, builds successfully)
| Module | File | Chức năng |
|--------|------|-----------|
| **Protocol** | `ProtocolConstants.swift` | Magic bytes, packet types, flags |
| | `PacketHeader.swift` | 16-byte binary header + Data serialization |
| | `VideoPacket.swift` | Video frame & config payloads |
| | `InputPacket.swift` | Touch (32B) & Stylus (48B) payloads |
| | `ControlPacket.swift` | Handshake, ping/pong, config |
| **Encoding** | `HEVCEncoder.swift` | VideoToolbox HEVC hardware encoder (low-latency config) |
| **Capture** | `ScreenCaptureManager.swift` | ScreenCaptureKit wrapper (NV12 output) |
| **USB** | `ADBForwarder.swift` | ADB TCP tunnel transport (NWConnection) |
| **Input** | `InputEventInjector.swift` | CGEvent injection (mouse + tablet/stylus pressure) |
| **Pipeline** | `VideoPipeline.swift` | Orchestrator: capture → encode → send + receive input |
| **UI** | `ContentView.swift` | SwiftUI: resolution picker, FPS, start/stop, stats |
| | `ExtendScreenApp.swift` | App entry + menu bar extra |
| **VirtualDisplay** | `VirtualDisplayManager.swift` | Placeholder for Phase 2 CGVirtualDisplay |

### Android App (12 Kotlin files)
| Module | File | Chức năng |
|--------|------|-----------|
| **Protocol** | `ProtocolConstants.kt` | Shared constants (mirror of Swift) |
| | `PacketHeader.kt` | 16-byte header parser (BigEndian ByteBuffer) |
| | `PacketReader.kt` | Stream → packet demux + fragment reassembly |
| | `PacketWriter.kt` | Packet → stream mux (thread-safe) |
| **Video** | `HEVCDecoder.kt` | MediaCodec HEVC hardware decoder (KEY_LOW_LATENCY) |
| **USB** | `ADBSocketManager.kt` | TCP ServerSocket listener for ADB tunnel |
| **Input** | `TouchInputCapture.kt` | MotionEvent → normalized touch/stylus packets |
| **Pipeline** | `ReceivePipeline.kt` | Orchestrator: USB read → decode → render |
| **UI** | `MainActivity.kt` | Compose: setup instructions + start button |
| | `DisplayActivity.kt` | Fullscreen SurfaceView + stats overlay |
| | `Theme.kt` | Material 3 dynamic color theme |

### Cách sử dụng
1. **macOS**: Mở `ExtendScreen-macOS/ExtendScreen.xcodeproj` trong Xcode → Build & Run
2. **Android**: Mở `ExtendScreen-Android/` trong Android Studio → Build & Run trên thiết bị
3. Kết nối Android với Mac qua USB, bật USB Debugging
4. Trên Mac nhấn "Start Streaming", trên Android nhấn "Start Display"
5. Mac chạy `adb forward tcp:38400 tcp:38400` tự động, video stream qua TCP tunnel

### Tiếp theo (Phase 2+)
- CGVirtualDisplay (virtual display thật thay vì capture main display)
- AOA USB transport (lower latency, không cần Developer Mode)
- Adaptive quality + resolution negotiation
- 4K/8K testing