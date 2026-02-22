Context
Build a system that turns an Android phone/tablet (Android 12+) into a secondary display for macOS, connected via USB 3 or Thunderbolt. Target: up to 8K 120fps, <50ms latency. Supports touch + stylus/pen input from Android back to macOS.
Tech Stack

macOS: Swift + SwiftUI, VideoToolbox (HEVC encoding), CGVirtualDisplay (private API), ScreenCaptureKit
Android: Kotlin + Jetpack Compose, MediaCodec (HEVC decoding), USB Accessory API
USB: AOA (primary) + ADB TCP forwarding (fallback)

Architecture Overview
macOS Pipeline:
CGVirtualDisplay → ScreenCaptureKit → VideoToolbox HEVC → USB → Android

Android Pipeline:
USB → FrameAssembler → MediaCodec HEVC → SurfaceView

Input Pipeline:
Android MotionEvent → serialize → USB → macOS CGEvent injection

Key Realistic Expectations
Scenario     | Resolution | FPS | Latency  | USB
------------------------------------------------
Casual       | 1920x1080  | 60  | 15-25ms  | 2.0
Productivity | 2560x1440  | 60  | 20-30ms  | 2.0
High quality | 3840x2160  | 60  | 25-40ms  | 3.0
Maximum      | 3840x2160  | 120 | 30-50ms  | 3.0
Aspirational | 7680x4320  | 30  | 50-100ms | 3.0
------------------------------------------------

Note: CGVirtualDisplay caps at 60Hz refresh rate. 120fps requires ScreenCaptureKit high-rate sampling. 8K@120fps is not achievable on current hardware but architecture supports future scaling.

Project Structure
macOS (ExtendScreen-macOS/)
ExtendScreen/
├── App/                    # SwiftUI app entry, AppDelegate
├── UI/                     # MainWindow, StatusBar, DeviceList, Settings, Diagnostics
├── VirtualDisplay/         # CGVirtualDisplay manager + helper subprocess
├── Capture/                # ScreenCaptureKit SCStream wrapper
├── Encoding/               # VideoToolbox HEVC encoder (low-latency config)
├── USB/                    # AOAManager (libusb), ADBForwarder, DeviceDiscovery
├── Input/                  # CGEvent injection (touch, stylus pressure/tilt)
├── Protocol/               # Binary packet types (header, video, input, control)
├── Pipeline/               # Video pipeline orchestrator, adaptive quality, latency tracker
└── Utilities/              # Ring buffer, atomic counter, logger
Android (ExtendScreen-Android/)
app/src/main/java/com/extendscreen/
├── ui/                     # MainActivity, DisplayActivity, Compose screens
├── usb/                    # AOAAccessoryManager, ADBSocketManager, DeviceMonitor
├── video/                  # MediaCodec HEVC decoder, SurfaceView renderer
├── input/                  # Touch + Stylus capture, coordinate mapper, serializer
├── protocol/               # Binary packet types (mirror of macOS protocol)
├── pipeline/               # Receive pipeline, input pipeline, latency tracker
└── util/                   # Ring buffer, logger
Binary Protocol (custom, over USB bulk transfer)
Packet Header (16 bytes)
[4B magic "EXSD"] [1B version] [1B type] [2B flags] [4B payloadLen] [4B sequence]
Packet Types

0x01/0x02 HANDSHAKE_REQ/ACK - Connection + capability negotiation
0x03 VIDEO_FRAME - Encoded HEVC NALUs (fragmented if >60KB)
0x04 VIDEO_CONFIG - SPS/PPS/VPS parameter sets
0x10 INPUT_TOUCH - Touch events (28 bytes: action, x, y, pressure)
0x11 INPUT_STYLUS - Stylus events (48 bytes: + tilt, orientation, buttons)
0x20/0x21 PING/PONG - Latency measurement
0x22 CONFIG - Dynamic display config changes
0x24 IDR_REQ - Request keyframe on corruption

Implementation Plan (4 Phases)
Phase 1: Proof of Concept

ADB TCP transport - Socket-based communication over USB via adb forward
Binary protocol - PacketHeader, VideoPacket, ControlPacket (handshake)
HEVC encoder (macOS) - VTCompressionSession with low-latency config (no B-frames, realtime, CBR)
HEVC decoder (Android) - MediaCodec with KEY_LOW_LATENCY rendering to SurfaceView
Screen capture - ScreenCaptureKit capturing primary display (1080p@30fps to start)
Basic touch input - Single-touch MotionEvent → CGEvent mouse injection

Phase 2: Virtual Display + AOA

CGVirtualDisplay - Private API virtual display via helper subprocess
Resolution negotiation - Handshake protocol, Android reports device capabilities
Frame fragmentation - Multi-packet video frames for large keyframes
AOA transport - libusb-based AOA on macOS + USB Accessory API on Android
Push to 60fps - Tune encoder, optimize pipeline

Phase 3: Polish + Full Input

Adaptive quality - Dynamic bitrate/resolution based on USB throughput + decoder stats
Latency measurement - PING/PONG protocol with UI display
Full stylus support - Pressure, tilt, barrel button, eraser via CGEvent tablet events
IDR request - Android requests keyframe on corruption detection
SwiftUI settings UI - Device list, resolution picker, quality controls, diagnostics
Compose UI - Connection status, settings, latency display
Edge cases - USB reconnect, display sleep/wake, app backgrounding

Phase 4: High Resolution

4K support - Validate encoder/decoder at 3840x2160
120fps exploration - ScreenCaptureKit high-rate capture
8K exploration - Test VideoToolbox at 7680x4320

HEVC Encoder Config (Low Latency)
RealTime: true
AllowFrameReordering: false          // No B-frames
MaxKeyFrameInterval: 120             // 1 IDR/sec at 120fps
EnableLowLatencyRateControl: true
MaxFrameDelayCount: 0                // Encode immediately
ExpectedFrameRate: <target>
PixelFormat: NV12 (420YpCbCr8BiPlanar)
Known Limitations

CGVirtualDisplay is a private API - cannot go on Mac App Store, may break in macOS updates
60Hz cap on virtual display refresh rate (CGVirtualDisplay limitation)
No native multi-touch on macOS external displays - single pointer only
USB 2.0 bandwidth (~40 MB/s) limits max to ~4K@60fps
8K decode not supported on most Android hardware
Accessibility permission required for CGEvent injection
AOA support varies by Android device/OEM

Verification

Build macOS app, verify CGVirtualDisplay creates a virtual screen visible in System Settings > Displays
Build Android app, connect via USB, verify handshake completes
Verify video stream at 1080p@30fps, then 1080p@60fps, then 4K@60fps
Measure latency with PING/PONG protocol - target <50ms
Test touch input: tap on Android triggers click on macOS virtual display
Test stylus: pressure-sensitive drawing in a macOS drawing app
Test USB reconnect recovery
Test adaptive quality by throttling bandwidth