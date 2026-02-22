# ExtendScreen — Hướng dẫn Bảo trì

> Phiên bản: 1.0
> Cập nhật: 2026-02-22

---

## 1. Yêu cầu Môi trường Phát triển

### macOS App

| Yêu cầu | Chi tiết |
|---|---|
| macOS | 14.0 (Sonoma) trở lên |
| Xcode | 15.0 trở lên (hỗ trợ Swift 5.9+) |
| ADB | Android Platform Tools (`brew install android-platform-tools`) |
| Quyền | Screen Recording + Accessibility (trong System Settings > Privacy) |

### Android App

| Yêu cầu | Chi tiết |
|---|---|
| Android SDK | compileSdk 34, minSdk 31 (Android 12) |
| Android Studio | Hedgehog trở lên |
| Java | JDK 17 |
| Thiết bị | Hỗ trợ HEVC hardware decode, USB Debugging bật |

---

## 2. Build & Run

### 2.1 macOS

```bash
# Mở project
open ExtendScreen-macOS/ExtendScreen.xcodeproj

# Hoặc build từ command line
cd ExtendScreen-macOS
xcodebuild -scheme ExtendScreen -configuration Debug build
```

**Lần chạy đầu tiên:**
1. macOS sẽ hỏi quyền **Screen Recording** → cho phép trong System Settings > Privacy & Security > Screen Recording
2. Nếu dùng input injection → cần cho phép **Accessibility** trong System Settings > Privacy & Security > Accessibility
3. App chạy ở chế độ **non-sandboxed** (không có App Sandbox entitlement)

### 2.2 Android

```bash
cd ExtendScreen-Android

# Build debug APK
./gradlew assembleDebug

# Cài trực tiếp lên thiết bị
./gradlew installDebug

# Hoặc cài APK thủ công
adb install app/build/outputs/apk/debug/app-debug.apk
```

### 2.3 Kết nối và sử dụng

```bash
# 1. Kết nối thiết bị Android qua USB
# 2. Bật USB Debugging trên Android
# 3. Chạy app Android → tap "Start Display"
# 4. Chạy app macOS → chọn thiết bị → chọn mode → Start Streaming
```

---

## 3. Cấu trúc File — Tham chiếu nhanh

### macOS (16 file Swift)

| Thư mục | File | Chức năng chính |
|---------|------|----------------|
| App/ | `ExtendScreenApp.swift` | Entry point, WindowGroup + MenuBarExtra |
| UI/ | `ContentView.swift` | Toàn bộ giao diện: device picker, settings, log, stats |
| Pipeline/ | `VideoPipeline.swift` | **Trung tâm điều phối**: capture → encode → send, nhận input |
| Capture/ | `ScreenCaptureManager.swift` | Bọc ScreenCaptureKit, output CVPixelBuffer |
| Encoding/ | `HEVCEncoder.swift` | HEVC hardware encode (VideoToolbox), AVCC→Annex-B |
| Input/ | `InputEventInjector.swift` | Touch/stylus → CGEvent, post vào macOS |
| USB/ | `ADBForwarder.swift` | TCP client qua ADB forward, gửi/nhận packet |
| USB/ | `ADBDeviceManager.swift` | Quét thiết bị ADB, getprop lấy tên |
| VirtualDisplay/ | `VirtualDisplayManager.swift` | Tạo/hủy CGVirtualDisplay (Extend mode) |
| Protocol/ | `ProtocolConstants.swift` | Hằng số: magic, version, PacketType, flags |
| Protocol/ | `PacketHeader.swift` | Serialize/deserialize header 16 byte |
| Protocol/ | `VideoPacket.swift` | VideoFrame (16B+NALU) + VideoConfig (10B+params) |
| Protocol/ | `InputPacket.swift` | Touch (32B) + Stylus (48B) payload |
| Protocol/ | `ControlPacket.swift` | Handshake (16B/20B) + PingPong (16B) |
| Utilities/ | `Logger.swift` | ESLogger (os.Logger) + LogStore (SwiftUI observable) |
| Utilities/ | `RingBuffer.swift` | Bộ đệm vòng generic, thread-safe (dự phòng) |

### Android (11 file Kotlin)

| Thư mục | File | Chức năng chính |
|---------|------|----------------|
| protocol/ | `ProtocolConstants.kt` | Hằng số giao thức (đồng bộ với macOS) |
| protocol/ | `PacketHeader.kt` | Header 16 byte big-endian |
| protocol/ | `PacketReader.kt` | Đọc packet từ InputStream + reassemble fragment |
| protocol/ | `PacketWriter.kt` | Ghi packet ra OutputStream (thread-safe) |
| usb/ | `ADBSocketManager.kt` | TCP server lắng nghe port 38400 |
| pipeline/ | `ReceivePipeline.kt` | Điều phối nhận → decode → render + input |
| video/ | `HEVCDecoder.kt` | Giải mã HEVC hardware (MediaCodec → Surface) |
| input/ | `TouchInputCapture.kt` | Bắt touch/stylus, chuẩn hóa tọa độ |
| ui/ | `MainActivity.kt` | Màn hình setup/hướng dẫn |
| ui/ | `DisplayActivity.kt` | Fullscreen display + SurfaceView |
| util/ | `LogStore.kt` | Log in-app (StateFlow) |

---

## 4. Cách thêm Tính năng mới

### 4.1 Thêm PacketType mới

**Bước 1: Định nghĩa type**

```
macOS:  ProtocolConstants.swift → enum PacketType → thêm case mới
Android: ProtocolConstants.kt → object PacketType → thêm hằng số mới
```

Đảm bảo giá trị UInt8 **khớp nhau** giữa 2 platform.

**Bước 2: Định nghĩa Payload**

```
macOS:  Tạo struct mới trong Protocol/ → implement serialize() + deserialize()
Android: Tạo data class hoặc thêm parse logic trong PacketReader
```

Tuân thủ:
- Big-endian byte order
- Fixed-size payload nếu có thể (dễ parse hơn)
- Thêm `reserved` bytes cho mở rộng tương lai

**Bước 3: Xử lý trong Pipeline**

```
macOS:  VideoPipeline.handleReceivedPacket() → thêm case trong switch
Android: ReceivePipeline → thêm handler cho packet type mới
```

**Bước 4: Gửi packet**

```
macOS:  ADBForwarder.send(type: .newType, payload: data)
Android: PacketWriter → thêm hàm send mới
```

### 4.2 Thêm Codec mới (ví dụ: H.264, AV1)

1. **ProtocolConstants**: Thêm case vào `CodecType` enum (cả 2 platform)
2. **macOS Encoder**: Tạo file mới trong `Encoding/` (ví dụ: `H264Encoder.swift`)
   - Copy pattern từ `HEVCEncoder.swift`
   - Thay `kCMVideoCodecType_HEVC` → codec tương ứng
   - Điều chỉnh parameter set extraction (H.264 dùng SPS+PPS, không có VPS)
3. **Android Decoder**: Mở rộng `HEVCDecoder.kt` hoặc tạo decoder mới
   - Thay MIME type: `"video/hevc"` → `"video/avc"` (H.264) hoặc `"video/av01"` (AV1)
4. **VideoPipeline**: Chọn encoder dựa trên codec negotiation trong handshake
5. **ContentView**: Thêm codec picker nếu cần

### 4.3 Thêm loại Input mới (ví dụ: Keyboard)

1. **Protocol**: `PacketType.inputKeyboard` đã được reserved (`0x12`)
2. **Tạo payload struct**: `KeyboardInputPayload` trong `InputPacket.swift`
   - Bao gồm: keyCode, modifiers, isDown/isUp
3. **Android**: Bắt KeyEvent trong `DisplayActivity`, serialize và gửi
4. **macOS**: Parse trong `VideoPipeline`, thêm `injectKeyboard()` vào `InputEventInjector`
   - Dùng `CGEvent(keyboardEventSource:virtualKey:keyDown:)` để tạo sự kiện

### 4.4 Thêm Transport mới (ví dụ: Wi-Fi)

1. **Tạo file transport mới**: `WiFiTransport.swift` (macOS), `WiFiSocketManager.kt` (Android)
2. **Implement interface tương tự**: `connect()`, `send()`, `disconnect()`, `onReceive`
3. **Android**: Dùng UDP multicast hoặc mDNS để discovery
4. **macOS**: Dùng `NWBrowser` (Network.framework) để tìm thiết bị
5. **VideoPipeline**: Chọn transport dựa trên loại kết nối (USB vs Wi-Fi)
6. **HandshakeAck**: `transportType` đã hỗ trợ phân biệt

---

## 5. Debugging & Troubleshooting

### 5.1 Hệ thống Log

ExtendScreen có 3 kênh log song song:

| Kênh | Truy cập | Chi tiết |
|------|----------|----------|
| **In-app log** | Giao diện app (log panel) | LogStore → @Published entries, hiển thị real-time |
| **os.Logger** (macOS) | Console.app, filter `com.extendscreen` | ESLogger.capture/encoder/transport/... |
| **Logcat** (Android) | `adb logcat -s ExtendScreen` | Tag: ExtendScreen |

**Categories trong log:**

| Category | Mô tả | Có trong |
|----------|--------|----------|
| `transport` | Kết nối TCP, gửi/nhận packet | macOS + Android |
| `encoder` | Mã hóa HEVC, bitrate, errors | macOS |
| `decoder` | Giải mã HEVC, queue, errors | Android |
| `capture` | Screen capture, frame drops | macOS |
| `pipeline` | Điều phối tổng thể, handshake | macOS + Android |
| `input` | Touch/stylus injection | macOS + Android |
| `display` | Virtual display | macOS |

### 5.2 Các lỗi thường gặp và cách sửa

#### "ADB not found"
```
Nguyên nhân: Không tìm thấy adb executable
Sửa:
  brew install android-platform-tools
  # Hoặc cài Android Studio → SDK Manager → Platform Tools
  # Kiểm tra: which adb
```

#### "Connection timed out" (sau 10 lần thử)
```
Nguyên nhân: App Android chưa chạy hoặc chưa tap "Start Display"
Kiểm tra:
  1. App Android đang mở và đã tap "Start Display"
  2. USB Debugging đã bật: adb devices (phải thấy thiết bị)
  3. Port không bị chiếm: lsof -i :38400
  4. ADB forward hoạt động: adb forward --list
```

#### "Screen Recording permission denied"
```
Nguyên nhân: macOS chưa cấp quyền Screen Recording
Sửa:
  System Settings > Privacy & Security > Screen Recording > bật cho ExtendScreen
  (Cần restart app sau khi cấp quyền)
```

#### "Failed to create virtual display"
```
Nguyên nhân: CGVirtualDisplay API thất bại (private API)
Kiểm tra:
  1. macOS version >= 14.0
  2. App không chạy trong sandbox
  3. Thử restart app
  4. Kiểm tra Console.app cho lỗi CoreGraphics
```

#### "Encode callback error" hoặc frame drops
```
Nguyên nhân: Encoder quá tải (resolution/FPS quá cao cho hardware)
Sửa:
  1. Giảm resolution (thử 1080p thay vì 4K)
  2. Giảm FPS (thử 30 thay vì 60)
  3. Kiểm tra Activity Monitor → CPU/GPU usage
```

#### Thiết bị Android không hiển thị trong dropdown
```
Kiểm tra:
  1. USB Debugging đã bật: Settings > Developer Options > USB Debugging
  2. Cáp USB hỗ trợ data (không phải cáp sạc only)
  3. Đã chấp nhận "Allow USB debugging" dialog trên Android
  4. adb devices -l hiển thị thiết bị status "device"
  5. Thử: adb kill-server && adb start-server
```

### 5.3 Debug ADB Connection

```bash
# Kiểm tra thiết bị đã kết nối
adb devices -l

# Kiểm tra port forwarding đang hoạt động
adb forward --list

# Thiết lập thủ công
adb -s <SERIAL> forward tcp:38400 tcp:38400

# Kiểm tra port đang lắng nghe (phía Android)
adb -s <SERIAL> shell ss -tlnp | grep 38400

# Xóa tất cả forwarding
adb forward --remove-all

# Reset ADB
adb kill-server
adb start-server
```

### 5.4 Debug Protocol

Để xem raw packet data, thêm log trong `ADBForwarder.receiveNextPacket()`:

```swift
// Tạm thời: log header info
LogStore.shared.debug("transport",
    "Received: type=\(header.type) flags=\(header.flags) len=\(header.payloadLength) seq=\(header.sequence)")
```

---

## 6. Hiệu năng & Tối ưu

### 6.1 Các thông số quan trọng

| Thông số | File | Giá trị hiện tại | Ảnh hưởng |
|----------|------|-------------------|-----------|
| Fragment size | `ProtocolConstants.swift` | 60 KB | Nhỏ hơn → nhiều packet hơn nhưng ít buffering |
| Queue depth | `ScreenCaptureManager.swift` | 3 | Số frame buffer, ít hơn → ít delay |
| TCP noDelay | `ADBForwarder.swift` | `true` | Tắt Nagle algorithm, giảm latency |
| Bitrate formula | `HEVCEncoder.swift` | 0.1 bit/pixel/frame | Tăng → chất lượng tốt hơn nhưng bandwidth cao hơn |
| Data rate limit | `HEVCEncoder.swift` | 1.5× average | Cho phép burst nhưng ngăn quá lớn |
| Ping interval | `VideoPipeline.swift` | 2 giây | Tần suất đo latency |
| FPS counter | `VideoPipeline.swift` | 1 giây | Cập nhật hiển thị FPS |
| Connection timeout | `ADBForwarder.swift` | 5s × 10 lần | Thời gian chờ kết nối |
| NALU queue size | `HEVCDecoder.kt` | 30 | Buffer giải mã Android |
| Max log entries | `Logger.swift` / `LogStore.kt` | 500 | Giới hạn bộ nhớ log |

### 6.2 Điểm tối ưu Latency

```
Tổng latency = Capture + Encode + Network + Decode + Render

1. Capture latency:
   - queueDepth = 3 (giữ ít buffer)
   - QoS: .userInteractive (ưu tiên cao nhất)

2. Encode latency:
   - RealTime = true
   - AllowFrameReordering = false (không B-frame)
   - MaxFrameDelayCount = 0 (output ngay)
   - LowLatencyRateControl = true

3. Network latency:
   - TCP noDelay = true (tắt Nagle)
   - ADB forward qua USB (không qua mạng)
   - Fragment 60KB (không chờ packet lớn)

4. Decode latency (Android):
   - KEY_LOW_LATENCY = 1
   - Vendor-specific low-latency keys
   - Thread.MAX_PRIORITY cho decoder thread
   - releaseOutputBuffer ngay lập tức (không buffer)
```

### 6.3 Tối ưu Memory

- `LogStore`: Max 500 entries, xoay vòng tự động
- `ScreenCaptureManager`: queueDepth = 3 giới hạn CVPixelBuffer
- `HEVCEncoder`: Không giữ tham chiếu đến frame cũ
- `ADBForwarder`: Packet được gửi ngay, không queue
- `RingBuffer`: Có sẵn nhưng chưa dùng (dự phòng cho future optimization)

---

## 7. Rủi ro Private API

### CGVirtualDisplay

`VirtualDisplayManager.swift` sử dụng các API semi-private của CoreGraphics:

- `CGVirtualDisplay`
- `CGVirtualDisplayDescriptor`
- `CGVirtualDisplayMode`
- `CGVirtualDisplaySettings`

**Rủi ro:**
- Apple có thể thay đổi/xóa API trong các phiên bản macOS tương lai
- API không có tài liệu chính thức
- Không thể dùng trong App Store (sandbox + private API restrictions)

**Giảm thiểu:**
- Extend mode là tính năng tùy chọn; Mirror mode hoạt động không cần private API
- Wrap toàn bộ CGVirtualDisplay code trong `VirtualDisplayManager` → dễ thay thế
- Kiểm tra `@available(macOS 14.0, *)` để đảm bảo tương thích
- Theo dõi macOS release notes mỗi năm

**Thay thế tiềm năng:**
- Apple có thể giới thiệu public API cho virtual display trong tương lai
- Hoặc dùng Sidecar framework (nhưng giới hạn cho iPad)

---

## 8. Checklist khi Release

### Build & Test

- [ ] **macOS**: `xcodebuild -scheme ExtendScreen build` thành công
- [ ] **Android**: `./gradlew assembleRelease` thành công
- [ ] **Mirror mode**: Capture và stream màn hình chính
- [ ] **Extend mode**: Virtual display xuất hiện, capture đúng display
- [ ] **Device discovery**: Danh sách thiết bị hiển thị đúng tên
- [ ] **Input injection**: Touch và stylus hoạt động trên cả 2 mode
- [ ] **Reconnection**: Ngắt USB → kết nối lại → stream lại
- [ ] **Multiple resolutions**: Test 1080p, 1440p, 4K
- [ ] **Multiple FPS**: Test 30, 60 fps
- [ ] **Latency**: Ping/pong hiển thị < 50ms qua USB
- [ ] **Memory**: Chạy 10 phút, kiểm tra không có memory leak (Instruments)
- [ ] **CPU**: Sử dụng CPU ổn định, không tăng liên tục

### Trước khi phân phối

- [ ] Kiểm tra `Info.plist` có mô tả quyền đầy đủ
- [ ] Code signing hoạt động (Developer ID hoặc self-signed)
- [ ] Android APK signed với release key
- [ ] Loại bỏ debug log không cần thiết
- [ ] Kiểm tra trên ít nhất 2 thiết bị Android khác nhau

### Sau khi update macOS

- [ ] Kiểm tra CGVirtualDisplay API vẫn hoạt động
- [ ] Kiểm tra ScreenCaptureKit API không thay đổi
- [ ] Kiểm tra Accessibility permission vẫn hoạt động
- [ ] Build lại và test toàn bộ luồng

---

## 9. Cấu trúc Thư mục Tổng thể

```
vibe_code_extend_screen/
├── ExtendScreen-macOS/          # macOS Xcode project
│   └── ExtendScreen/
│       ├── App/                 # Entry point + config
│       ├── UI/                  # SwiftUI views
│       ├── Pipeline/            # Orchestration
│       ├── Capture/             # Screen capture
│       ├── Encoding/            # Video encoding
│       ├── Input/               # Event injection
│       ├── USB/                 # ADB transport + device mgmt
│       ├── VirtualDisplay/      # Virtual display (Extend mode)
│       ├── Protocol/            # Binary protocol definitions
│       └── Utilities/           # Logger + RingBuffer
├── ExtendScreen-Android/        # Android Gradle project
│   └── app/src/main/java/com/extendscreen/
│       ├── protocol/            # Binary protocol (đồng bộ với macOS)
│       ├── usb/                 # ADB TCP transport
│       ├── pipeline/            # Receive pipeline
│       ├── video/               # HEVC decoding
│       ├── input/               # Touch capture
│       ├── ui/                  # Activities + Compose UI
│       └── util/                # Log store
└── docs/                        # Tài liệu
    ├── DESIGN.md                # Tài liệu thiết kế
    └── MAINTENANCE.md           # Hướng dẫn bảo trì (file này)
```
