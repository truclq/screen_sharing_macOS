# ExtendScreen — Tài liệu Thiết kế Hệ thống

> Phiên bản: 1.0
> Cập nhật: 2026-02-22
> Giao thức: ESProtocol v1

---

## 1. Tổng quan

**ExtendScreen** là hệ thống cho phép sử dụng thiết bị Android làm màn hình phụ cho máy Mac, kết nối qua cáp USB.

### Tính năng chính

- **Mirror mode**: Chiếu nội dung màn hình chính Mac sang Android
- **Extend mode**: Tạo màn hình ảo mới trên Mac và hiển thị trên Android (như màn hình phụ thật)
- **Truyền video thời gian thực**: Mã hóa HEVC phần cứng, độ trễ thấp
- **Input ngược**: Điều khiển Mac từ màn hình cảm ứng Android (touch + stylus)
- **Hỗ trợ độ phân giải cao**: 1080p, 1440p, 4K, 8K tại 30/60/120 FPS

### Yêu cầu

| Thành phần | Yêu cầu tối thiểu |
|---|---|
| macOS | 14.0 (Sonoma) trở lên |
| Android | 12 (API 31) trở lên |
| Kết nối | Cáp USB với USB Debugging đã bật |
| ADB | Android Platform Tools đã cài đặt |

---

## 2. Kiến trúc tổng thể

```
┌─────────────────────────────────────────────────────────────────────┐
│                           macOS App                                 │
│                                                                     │
│  ┌──────────────────┐    ┌─────────────┐    ┌──────────────────┐   │
│  │ ScreenCapture    │───▶│ HEVCEncoder │───▶│ VideoPipeline    │   │
│  │ Manager          │    │ (VideoToolbox)   │ (Orchestrator)   │   │
│  │ (ScreenCaptureKit)    └─────────────┘    │                  │   │
│  └──────────────────┘                       │  ┌────────────┐  │   │
│                                             │  │ ADBForwarder│  │   │
│  ┌──────────────────┐                       │  │ (TCP/ADB)  │  │   │
│  │ VirtualDisplay   │◀──────────────────────│  └─────┬──────┘  │   │
│  │ Manager          │                       │        │         │   │
│  │ (CGVirtualDisplay)                       │        │ USB     │   │
│  └──────────────────┘                       │        │         │   │
│                                             │  ┌─────▼──────┐  │   │
│  ┌──────────────────┐                       │  │ InputEvent │  │   │
│  │ ADBDevice        │                       │  │ Injector   │  │   │
│  │ Manager          │                       │  │ (CGEvent)  │  │   │
│  └──────────────────┘                       └──┴────────────┴──┘   │
│                                                                     │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ ADB Forward (TCP port 38400)
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                          Android App                                │
│                                                                     │
│  ┌──────────────────┐    ┌─────────────┐    ┌──────────────────┐   │
│  │ ADBSocket        │───▶│ PacketReader│───▶│ ReceivePipeline  │   │
│  │ Manager          │    │             │    │ (Orchestrator)   │   │
│  │ (TCP Listener)   │    └─────────────┘    │                  │   │
│  └──────────────────┘                       │  ┌────────────┐  │   │
│                                             │  │ HEVCDecoder│  │   │
│  ┌──────────────────┐                       │  │ (MediaCodec)│  │   │
│  │ TouchInput       │──────────────────────▶│  └─────┬──────┘  │   │
│  │ Capture          │                       │        │         │   │
│  │ (MotionEvent)    │                       │        ▼         │   │
│  └──────────────────┘                       │  ┌────────────┐  │   │
│                                             │  │ SurfaceView│  │   │
│  ┌──────────────────┐                       │  │ (Render)   │  │   │
│  │ DisplayActivity  │◀─────────────────────▶│  └────────────┘  │   │
│  │ (Fullscreen UI)  │                       └──────────────────┘   │
│  └──────────────────┘                                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Luồng dữ liệu chính

```
Video (Mac → Android):
  ScreenCaptureKit → CVPixelBuffer → HEVCEncoder → Annex-B NALU
  → PacketHeader + VideoFramePayload → TCP → PacketReader
  → HEVCDecoder (MediaCodec) → SurfaceView

Input (Android → Mac):
  MotionEvent → TouchInputCapture → normalize(0.0-1.0)
  → PacketWriter → TCP → ADBForwarder.onReceive
  → VideoPipeline → InputEventInjector → CGEvent → macOS
```

---

## 3. Cấu trúc dự án macOS

```
ExtendScreen-macOS/ExtendScreen/
├── App/
│   ├── ExtendScreenApp.swift      # Entry point, tạo WindowGroup + MenuBarExtra
│   ├── Info.plist                  # Bundle config, mô tả quyền
│   └── ExtendScreen.entitlements   # Entitlements (hiện tại rỗng)
├── UI/
│   └── ContentView.swift           # Giao diện chính: device picker, settings, log viewer
├── Pipeline/
│   └── VideoPipeline.swift         # Điều phối toàn bộ pipeline: capture → encode → send
├── Capture/
│   └── ScreenCaptureManager.swift  # Chụp màn hình bằng ScreenCaptureKit
├── Encoding/
│   └── HEVCEncoder.swift           # Mã hóa HEVC phần cứng bằng VideoToolbox
├── Input/
│   └── InputEventInjector.swift    # Tiêm sự kiện touch/stylus vào macOS qua CGEvent
├── USB/
│   ├── ADBForwarder.swift          # Truyền tải TCP qua ADB port forwarding
│   └── ADBDeviceManager.swift      # Quét và quản lý thiết bị Android qua ADB
├── VirtualDisplay/
│   └── VirtualDisplayManager.swift # Tạo/hủy virtual display (Extend mode)
├── Protocol/
│   ├── ProtocolConstants.swift     # Hằng số giao thức, enum PacketType, flags
│   ├── PacketHeader.swift          # Header 16 byte + Data extension (big-endian)
│   ├── VideoPacket.swift           # VideoFramePayload + VideoConfigPayload
│   ├── InputPacket.swift           # TouchInputPayload (32B) + StylusInputPayload (48B)
│   └── ControlPacket.swift         # HandshakeRequest/Ack + PingPongPayload
└── Utilities/
    ├── Logger.swift                # Hệ thống log: ESLogger + LogStore (observable)
    └── RingBuffer.swift            # Bộ đệm vòng thread-safe (dự phòng)
```

### Vai trò chi tiết từng file

| File | Lớp/Struct chính | Vai trò |
|------|-------------------|---------|
| `ExtendScreenApp.swift` | `ExtendScreenApp` | Entry point, tạo `VideoPipeline` singleton |
| `ContentView.swift` | `ContentView`, `MenuBarView`, `StatView`, `LogEntryRow` | UI chính, chọn thiết bị, cài đặt, log |
| `VideoPipeline.swift` | `VideoPipeline` (@MainActor, ObservableObject) | Điều phối toàn bộ: capture → encode → transport → input |
| `ScreenCaptureManager.swift` | `ScreenCaptureManager` (SCStreamOutput) | Bọc ScreenCaptureKit, output CVPixelBuffer |
| `HEVCEncoder.swift` | `HEVCEncoder`, `EncoderConfiguration` | Mã hóa HEVC real-time, chuyển AVCC → Annex-B |
| `InputEventInjector.swift` | `InputEventInjector` | Chuyển touch/stylus thành CGEvent, post vào hệ thống |
| `ADBForwarder.swift` | `ADBForwarder` | TCP client qua ADB forward, gửi/nhận packet |
| `ADBDeviceManager.swift` | `ADBDeviceManager`, `ADBDevice` | Quét thiết bị ADB, lấy tên qua getprop |
| `VirtualDisplayManager.swift` | `VirtualDisplayManager`, `DisplayMode` | Tạo CGVirtualDisplay cho Extend mode |
| `ProtocolConstants.swift` | `ESProtocol`, `PacketType`, `PacketFlags`, ... | Hằng số và enum giao thức |
| `PacketHeader.swift` | `PacketHeader`, Data extensions | Serialize/deserialize header 16 byte |
| `VideoPacket.swift` | `VideoFramePayload`, `VideoConfigPayload` | Payload video frame và cấu hình codec |
| `InputPacket.swift` | `TouchInputPayload`, `StylusInputPayload` | Payload sự kiện touch (32B) và stylus (48B) |
| `ControlPacket.swift` | `HandshakeRequest`, `HandshakeAck`, `PingPongPayload` | Payload bắt tay và đo độ trễ |
| `Logger.swift` | `ESLogger`, `LogStore`, `LogEntry` | Log song song: in-app (SwiftUI) + os.Logger |
| `RingBuffer.swift` | `RingBuffer<T>` | Bộ đệm vòng generic, thread-safe bằng NSLock |

---

## 4. Cấu trúc dự án Android

```
ExtendScreen-Android/app/src/main/java/com/extendscreen/
├── protocol/
│   ├── ProtocolConstants.kt    # Hằng số giao thức (đồng bộ với macOS)
│   ├── PacketHeader.kt         # Header 16 byte
│   ├── PacketReader.kt         # Đọc packet từ InputStream, reassemble fragment
│   └── PacketWriter.kt         # Ghi packet ra OutputStream (thread-safe)
├── usb/
│   └── ADBSocketManager.kt     # TCP server lắng nghe trên port 38400
├── pipeline/
│   └── ReceivePipeline.kt      # Điều phối: nhận packet → decode → render + input
├── video/
│   └── HEVCDecoder.kt          # Giải mã HEVC phần cứng bằng MediaCodec
├── input/
│   └── TouchInputCapture.kt    # Bắt sự kiện touch/stylus, chuẩn hóa tọa độ
├── ui/
│   ├── MainActivity.kt         # Màn hình setup/hướng dẫn
│   ├── DisplayActivity.kt      # Màn hình hiển thị fullscreen + SurfaceView
│   ├── LogViewer.kt            # Component hiển thị log (Compose)
│   └── theme/Theme.kt          # Material 3 theme
└── util/
    └── LogStore.kt             # Lưu trữ log in-app (StateFlow)
```

---

## 5. Giao thức nhị phân (ESProtocol)

### 5.1 Packet Header (16 byte)

Mọi gói tin đều bắt đầu bằng header 16 byte, big-endian:

```
Offset  Size  Type      Field           Mô tả
──────  ────  ────      ─────           ─────
0       4     UInt32    magic           0x45585344 ("EXSD") - magic number xác thực
4       1     UInt8     version         Phiên bản giao thức (hiện tại: 1)
5       1     UInt8     type            Loại gói tin (PacketType)
6       2     UInt16    flags           Cờ (PacketFlags, OptionSet)
8       4     UInt32    payloadLength   Độ dài payload tính bằng byte
12      4     UInt32    sequence        Số thứ tự gói tin (tăng dần)
```

### 5.2 Packet Types

| Mã | Tên | Hướng | Payload |
|----|------|-------|---------|
| `0x01` | HANDSHAKE_REQ | Mac → Android | HandshakeRequest (16B) |
| `0x02` | HANDSHAKE_ACK | Android → Mac | HandshakeAck (20B) |
| `0x03` | VIDEO_FRAME | Mac → Android | VideoFramePayload (16B + NALU data) |
| `0x04` | VIDEO_CONFIG | Mac → Android | VideoConfigPayload (10B + param sets) |
| `0x10` | INPUT_TOUCH | Android → Mac | TouchInputPayload (32B) |
| `0x11` | INPUT_STYLUS | Android → Mac | StylusInputPayload (48B) |
| `0x12` | INPUT_KEYBOARD | Android → Mac | (dự phòng) |
| `0x20` | CONTROL_PING | Mac → Android | PingPongPayload (16B) |
| `0x21` | CONTROL_PONG | Android → Mac | PingPongPayload (16B) |
| `0x22` | CONTROL_CONFIG | Cả hai | (dự phòng) |
| `0x23` | CONTROL_STATS | Cả hai | (dự phòng) |
| `0x24` | CONTROL_IDR_REQ | Android → Mac | (yêu cầu keyframe) |
| `0xFF` | DISCONNECT | Cả hai | (ngắt kết nối) |

### 5.3 Packet Flags (OptionSet, UInt16)

| Bit | Tên | Mô tả |
|-----|------|-------|
| 0 | `fragmentStart` | Đánh dấu mảnh đầu tiên |
| 1 | `fragmentEnd` | Đánh dấu mảnh cuối cùng |
| 2 | `keyframe` | Gói tin chứa keyframe (IDR) |
| 3 | `priority` | Gói tin ưu tiên cao |

### 5.4 Wire Format chi tiết

#### HandshakeRequest (16 byte) — Mac gửi Android

```
Offset  Size  Type     Field
0       2     UInt16   protocolVersion
2       2     UInt16   macOSVersion
4       2     UInt16   desiredWidth
6       2     UInt16   desiredHeight
8       1     UInt8    desiredFPS
9       1     UInt8    codecType (0=HEVC, 1=H264, 2=AV1)
10      4     UInt32   maxBitrate (kbps)
14      2     UInt16   reserved
```

#### HandshakeAck (20 byte) — Android trả Mac

```
Offset  Size  Type     Field
0       2     UInt16   protocolVersion
2       2     UInt16   androidVersion (SDK level)
4       2     UInt16   screenWidth
6       2     UInt16   screenHeight
8       2     UInt16   screenDPI
10      1     UInt8    maxFPS
11      1     UInt8    codecSupport (bit0=HEVC, bit1=H264, bit2=AV1)
12      1     UInt8    hasLowLatency (0/1)
13      1     UInt8    hasStylusSupport (0/1)
14      2     UInt16   maxWidth
16      2     UInt16   maxHeight
18      1     UInt8    transportType (0=AOA, 1=ADB)
19      1     UInt8    reserved
```

#### VideoFramePayload (16 byte header + NALU data)

```
Offset  Size  Type     Field
0       8     UInt64   timestamp (nanoseconds)
8       4     UInt32   frameNumber
12      1     UInt8    nalUnitType (19=IDR keyframe, 1=P-frame)
13      3     UInt8×3  reserved
16      var   Data     naluData (Annex-B format với start code 0x00000001)
```

#### VideoConfigPayload (10 byte header + parameter sets)

```
Offset  Size  Type     Field
0       1     UInt8    codec
1       1     UInt8    reserved
2       2     UInt16   width
4       2     UInt16   height
6       1     UInt8    fps
7       3     UInt8×3  reserved
10      var   Data     parameterSets (VPS + SPS + PPS, Annex-B format)
```

#### TouchInputPayload (32 byte)

```
Offset  Size  Type     Field
0       8     UInt64   timestamp (nanoseconds)
8       1     UInt8    action (0=down, 1=move, 2=up, 3=cancel)
9       1     UInt8    pointerCount
10      1     UInt8    pointerIndex
11      1     UInt8    toolType (0=finger, 1=stylus, 2=eraser)
12      4     Float    x (0.0–1.0, chuẩn hóa)
16      4     Float    y (0.0–1.0, chuẩn hóa)
20      4     Float    pressure (0.0–1.0)
24      4     Float    size
28      4     UInt32   reserved
```

#### StylusInputPayload (48 byte)

```
Offset  Size  Type     Field
0       32    —        touchBase (giống TouchInputPayload)
32      4     Float    tiltX (radian)
36      4     Float    tiltY (radian)
40      4     Float    orientation (radian)
44      2     UInt16   buttonState (bitmask: bit1 = barrel button)
46      2     UInt16   distance (khoảng cách lơ lửng)
```

#### PingPongPayload (16 byte)

```
Offset  Size  Type     Field
0       8     UInt64   sendTimestamp (nanoseconds)
8       4     UInt32   sequence
12      4     UInt32   reserved
```

### 5.5 Cơ chế phân mảnh (Fragmentation)

Khi payload vượt quá **60 KB** (`ESProtocol.fragmentSize`):

1. Chia payload thành các mảnh ≤ 60 KB
2. Mảnh đầu: flags = `[.fragmentStart]`
3. Mảnh giữa: flags = `[]` (không có flag)
4. Mảnh cuối: flags = `[.fragmentEnd]`
5. Android reassemble theo thứ tự nhận

Nếu payload ≤ 60 KB → gửi nguyên vẹn với flags = `[.fragmentStart, .fragmentEnd]`

---

## 6. Pipeline xử lý Video

### 6.1 macOS: Capture → Encode → Send

```
1. ScreenCaptureManager.start(displayID, width, height, fps)
   │
   ├─ Tạo SCContentFilter cho display cụ thể
   ├─ Cấu hình: NV12 pixel format, showsCursor=true, queueDepth=3
   └─ SCStream bắt đầu → callback: onFrame(CVPixelBuffer, CMTime)

2. HEVCEncoder.encode(pixelBuffer, presentationTime)
   │
   ├─ VTCompressionSessionEncodeFrame() → callback
   ├─ Trích xuất parameter sets (VPS/SPS/PPS) khi gặp keyframe
   ├─ Chuyển AVCC → Annex-B (thay length prefix bằng start code)
   └─ Gọi onEncodedFrame(naluData, isKeyframe, timestampNanos)

3. VideoPipeline.handleEncodedFrame()
   │
   ├─ Nếu keyframe → gửi VIDEO_CONFIG (codec + parameter sets)
   ├─ Tạo VideoFramePayload (timestamp, frameNumber, nalType, data)
   ├─ Nếu payload > 60KB → phân mảnh
   └─ ADBForwarder.send(type: .videoFrame, payload)

4. ADBForwarder.send()
   │
   ├─ Tạo PacketHeader (magic, version, type, flags, length, sequence)
   ├─ Serialize header + payload
   └─ NWConnection.send() → TCP → adb forward → Android
```

### 6.2 Android: Receive → Decode → Render

```
1. ADBSocketManager (TCP server trên port 38400)
   │
   ├─ ServerSocket.accept() → nhận kết nối từ Mac
   └─ Truyền InputStream/OutputStream cho PacketReader/Writer

2. PacketReader.readPacket()
   │
   ├─ Đọc 16-byte header → parse PacketHeader
   ├─ Đọc payload theo payloadLength
   ├─ Nếu fragmented → reassemble vào buffer
   └─ Dispatch theo type → ReceivePipeline

3. ReceivePipeline
   │
   ├─ VIDEO_CONFIG → cấu hình MediaCodec (codec, width, height, fps)
   ├─ VIDEO_FRAME → đẩy NALU vào HEVCDecoder queue
   ├─ CONTROL_PING → phản hồi PONG
   └─ HANDSHAKE_REQ → gửi HANDSHAKE_ACK

4. HEVCDecoder
   │
   ├─ MediaCodec.configure(surface, format) → low-latency mode
   ├─ Dequeue input buffer → copy NALU data → queue
   ├─ Dequeue output buffer → releaseOutputBuffer(render=true)
   └─ Hiển thị trực tiếp lên SurfaceView
```

### 6.3 Cấu hình HEVC Encoder (macOS)

| Thuộc tính | Giá trị | Mục đích |
|---|---|---|
| Codec | HEVC (H.265) | Nén hiệu quả |
| Profile | Main AutoLevel | Tương thích rộng |
| Real-time | `true` | Ưu tiên tốc độ |
| B-frames | Disabled (AllowFrameReordering=false) | Giảm latency |
| MaxFrameDelayCount | 0 | Output ngay lập tức |
| KeyFrameInterval | 60 frames (1s ở 60fps) | Cân bằng quality/size |
| DataRateLimits | 1.5× average bitrate | Ngăn burst quá lớn |
| Bitrate mặc định | ~0.1 bit/pixel/frame | Tự tính theo resolution |

---

## 7. Hệ thống Input

### 7.1 Luồng Input (Android → macOS)

```
Android:
  MotionEvent → TouchInputCapture
  ├─ Lấy action: ACTION_DOWN/MOVE/UP/CANCEL
  ├─ Chuẩn hóa tọa độ: x/width, y/height → 0.0–1.0
  ├─ Lấy pressure, size, toolType
  ├─ Nếu stylus: thêm tiltX, tiltY, orientation, buttonState
  └─ PacketWriter.sendTouchInput() / sendStylusInput()
      → Serialize payload → TCP → Mac

macOS:
  ADBForwarder.onReceive → VideoPipeline.handleReceivedPacket()
  ├─ Parse: TouchInputPayload.deserialize() hoặc StylusInputPayload.deserialize()
  └─ InputEventInjector.injectTouch() / injectStylus()
      ├─ Chuyển tọa độ chuẩn hóa → tọa độ tuyệt đối:
      │   absoluteX = displayBounds.origin.x + normalized.x × displayBounds.width
      │   absoluteY = displayBounds.origin.y + normalized.y × displayBounds.height
      ├─ Map action → CGEventType:
      │   down → .leftMouseDown
      │   move + pressure > 0 → .leftMouseDragged
      │   move + pressure == 0 → .mouseMoved
      │   up/cancel → .leftMouseUp
      ├─ Nếu stylus: set tablet subtype + pressure + tilt
      ├─ Nếu barrel button (bit 0x02): → .rightMouseDown
      └─ CGEvent.post(tap: .cghidEventTap) → macOS xử lý
```

### 7.2 Yêu cầu quyền

- **Accessibility permission**: Bắt buộc để post CGEvent
- Kiểm tra: `AXIsProcessTrusted()`
- Yêu cầu: `AXIsProcessTrustedWithOptions()` hiển thị dialog

---

## 8. Virtual Display (Extend mode)

### 8.1 Cơ chế hoạt động

Extend mode sử dụng **CGVirtualDisplay** (semi-private API, CoreGraphics, macOS 14+):

```
1. VirtualDisplayManager.createDisplay(width, height, refreshRate)
   │
   ├─ Tạo CGVirtualDisplayDescriptor:
   │   ├─ name = "ExtendScreen"
   │   ├─ maxPixelsWide/High = resolution
   │   ├─ sizeInMillimeters = (600, 340) → mô phỏng màn hình 27"
   │   └─ productID = 0x1234, vendorID = 0x5678
   │
   ├─ Tạo CGVirtualDisplay(descriptor)
   │
   ├─ Tạo CGVirtualDisplayMode(width, height, refreshRate)
   ├─ Tạo CGVirtualDisplaySettings(modes: [mode])
   ├─ vDisplay.apply(settings) → macOS đăng ký display mới
   │
   └─ displayID = vDisplay.displayID → truyền cho capture + input

2. macOS nhận diện display mới:
   ├─ Xuất hiện trong System Settings > Displays
   ├─ ScreenCaptureKit phát hiện qua SCShareableContent
   └─ User có thể kéo cửa sổ sang virtual display

3. VirtualDisplayManager.destroyDisplay()
   ├─ virtualDisplay = nil → ARC release
   └─ macOS tự động xóa display
```

### 8.2 Discovery Loop

Sau khi tạo virtual display, ScreenCaptureKit cần thời gian để phát hiện:

```swift
// Trong VideoPipeline.start()
for attempt in 1...10 {
    let content = try await SCShareableContent.excludingDesktopWindows(false, onScreenWindowsOnly: false)
    if content.displays.contains(where: { $0.displayID == virtualDisplayID }) {
        break // Đã tìm thấy
    }
    try await Task.sleep(nanoseconds: 300_000_000) // Chờ 300ms
}
```

### 8.3 Các mode hỗ trợ

| Mode | Độ phân giải | Refresh Rate |
|------|-------------|--------------|
| 1080p | 1920 × 1080 | 60 Hz |
| 1440p | 2560 × 1440 | 60 Hz |
| 4K | 3840 × 2160 | 60 Hz / 120 Hz |
| 8K | 7680 × 4320 | 60 Hz |

---

## 9. Mô hình đa luồng (Concurrency)

| Component | Thread | QoS | Mô tả |
|---|---|---|---|
| `VideoPipeline` | Main (@MainActor) | — | Điều phối, cập nhật @Published |
| `ContentView` | Main (SwiftUI) | — | Render giao diện |
| `LogStore` | Main (@MainActor) | — | Singleton, cập nhật log entries |
| `ADBDeviceManager` | Main (@MainActor) | — | Quét thiết bị, cập nhật danh sách |
| `VirtualDisplayManager` | Main (@MainActor) | — | Quản lý virtual display |
| `ScreenCaptureManager` | captureQueue | .userInteractive | Nhận CVPixelBuffer từ SCStream |
| `ADBForwarder` | adb queue | .userInteractive | Gửi/nhận TCP data |
| `HEVCEncoder` | VideoToolbox internal | — | Mã hóa phần cứng |
| `InputEventInjector` | Main | — | Post CGEvent |

### Giao tiếp giữa luồng

- Callback từ background → `Task { @MainActor in ... }` để dispatch về main
- `[weak self]` trong closure để tránh retain cycle
- `activeConnectionID: UUID` để bỏ qua callback từ kết nối cũ

---

## 10. Luồng kết nối (Connection Flow)

```
┌──────────┐                           ┌──────────┐
│  macOS   │                           │ Android  │
└────┬─────┘                           └────┬─────┘
     │                                      │
     │ 1. adb forward tcp:38400 tcp:38400   │
     │─────────────────────────────────────▶│
     │                                      │
     │ 2. TCP connect to localhost:38400    │
     │─────────────────────────────────────▶│
     │                                      │ (ADBSocketManager.accept)
     │                                      │
     │ 3. HANDSHAKE_REQ                    │
     │  (version, width, height, fps,      │
     │   codec, bitrate)                   │
     │─────────────────────────────────────▶│
     │                                      │
     │ 4. HANDSHAKE_ACK                    │
     │  (version, screen info, DPI,        │
     │   codec support, stylus support)    │
     │◀─────────────────────────────────────│
     │                                      │
     │ 5. VIDEO_CONFIG (VPS+SPS+PPS)       │
     │─────────────────────────────────────▶│ (MediaCodec.configure)
     │                                      │
     │ 6. VIDEO_FRAME (keyframe)           │
     │─────────────────────────────────────▶│ (decode + render)
     │                                      │
     │ 7. VIDEO_FRAME (P-frame, lặp lại)  │
     │─────────────────────────────────────▶│
     │                                      │
     │ 8. INPUT_TOUCH (khi user chạm)      │
     │◀─────────────────────────────────────│
     │                                      │
     │ 9. CONTROL_PING (mỗi 2 giây)       │
     │─────────────────────────────────────▶│
     │                                      │
     │ 10. CONTROL_PONG                    │
     │◀─────────────────────────────────────│
     │                                      │
```

### Retry Logic

- **Kết nối TCP**: Tối đa 10 lần thử, mỗi lần chờ 5s timeout, nghỉ 1s giữa các lần
- **ADB forward**: Thiết lập lại trước mỗi lần thử lại (xóa cũ → tạo mới → chờ 300ms)
- **Sau khi kết nối**: Nếu mất kết nối → gọi `onDisconnect` callback → UI hiển thị lỗi

---

## 11. Xử lý lỗi

### Các loại lỗi

| Enum | Cases | File |
|------|-------|------|
| `CaptureError` | `.displayNotFound`, `.permissionDenied` | ScreenCaptureManager.swift |
| `EncoderError` | `.createFailed(OSStatus)`, `.propertyFailed(OSStatus)` | HEVCEncoder.swift |
| `TransportError` | `.connectionTimeout`, `.adbForwardFailed(String)`, `.adbNotFound`, `.noDeviceConnected`, `.notConnected` | ADBForwarder.swift |
| `VirtualDisplayError` | `.creationFailed`, `.settingsApplyFailed`, `.displayNotFoundInSCK` | VirtualDisplayManager.swift |

### Chiến lược xử lý

1. **Startup failure**: Pipeline catch lỗi → set `connectionStatus = .error` → cleanup tất cả resource → log lỗi
2. **Runtime disconnection**: `onDisconnect` callback → stop pipeline → UI thông báo
3. **Encoder error**: Log warning/error → tiếp tục (encoder tự phục hồi nếu có thể)
4. **Frame drop**: Đếm `droppedFrames` → log khi dừng capture

### Hệ thống Logging

Dual logging cho mọi component:

```
LogStore.shared.info("category", "message")
  ├─ 1. Thêm LogEntry vào @Published entries → UI hiển thị
  └─ 2. os.Logger.info() → Console.app / system log

Categories: transport, encoder, capture, pipeline, input, display, general
Levels: info (🔵), warning (🟡), error (🔴), debug (⚪)
Max entries: 500 (xoay vòng)
```
