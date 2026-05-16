# WebRTC Functions Reference (rút từ repo `webrtc/samples`)

Tài liệu này tổng hợp các API WebRTC quan trọng được dùng trong repo, kèm vị trí tham khảo trong source. Mục tiêu: làm "cheat-sheet" để dựng một ứng dụng livestream (1 streamer → nhiều viewer hoặc N-N video call).

> Quy ước: mọi đường dẫn đều tương đối tới gốc repo (`D:\samples`).

---

## 1. Lấy media từ thiết bị

### 1.1 `navigator.mediaDevices.getUserMedia(constraints)`
- **Vai trò**: yêu cầu quyền truy cập camera/micro, trả về `MediaStream`.
- **Tham khảo**: `src/content/getusermedia/gum/js/main.js:46`, `src/content/peerconnection/pc1/js/main.js:63`.
- **Constraints quan trọng cho livestream**:
  ```js
  { audio: true, video: { width: 1280, height: 720, frameRate: 30 } }
  ```
- **Lỗi cần xử lý**: `NotAllowedError` (user từ chối), `OverconstrainedError` (thiết bị không đáp ứng), `NotFoundError`.

### 1.2 `navigator.mediaDevices.getDisplayMedia(options)`
- **Vai trò**: lấy stream từ màn hình (share screen) — hữu ích khi streamer muốn share desktop/tab/app.
- **Tham khảo**: `src/content/getusermedia/getdisplaymedia/js/main.js:57`.
- **Phát hiện user dừng share**: lắng nghe sự kiện `ended` trên `videoTrack` (`main.js:31`).

### 1.3 `navigator.mediaDevices.enumerateDevices()`
- **Vai trò**: liệt kê camera/mic/speaker để cho user chọn nguồn input.
- **Tham khảo**: `src/content/devices/input-output/js/main.js`, `src/content/getusermedia/gum/js/main.js:295`.

### 1.4 `MediaStream.getTracks() / getVideoTracks() / getAudioTracks()`
- Mỗi `MediaStream` chứa nhiều `MediaStreamTrack`. Tách track để gắn vào peer connection hoặc tắt riêng audio/video.
- **Tham khảo**: `src/content/peerconnection/pc1/js/main.js:78`, `:98`.

### 1.5 `MediaStreamTrack.stop()` / `track.enabled = false`
- `stop()`: giải phóng thiết bị (camera tắt đèn). Không thể bật lại track này.
- `enabled = false`: mute tạm thời (track vẫn live nhưng gửi frame đen/silent).

---

## 2. RTCPeerConnection — trái tim của WebRTC

### 2.1 `new RTCPeerConnection(configuration)`
- **Vai trò**: khởi tạo kết nối P2P, chịu trách nhiệm gom ICE candidate, mã hoá DTLS-SRTP, truyền media/data.
- **Configuration**:
  ```js
  {
    iceServers: [
      { urls: 'stun:stun.l.google.com:19302' },
      { urls: 'turn:turn.example.com:3478', username: 'u', credential: 'p' }
    ],
    iceTransportPolicy: 'all'  // hoặc 'relay' để bắt buộc qua TURN
  }
  ```
- **Tham khảo**: `src/content/peerconnection/pc1/js/main.js:88`, `src/content/peerconnection/trickle-ice/js/main.js:141`.

### 2.2 `pc.addTrack(track, stream)`
- Gắn từng track của local stream vào peer connection. Trả về một `RTCRtpSender`.
- **Tham khảo**: `pc1/js/main.js:98`.

### 2.3 `pc.addTransceiver(trackOrKind, init)`
- Tạo "ống truyền" có hướng tường minh: `sendonly`, `recvonly`, `sendrecv`, `inactive`.
- Hữu ích cho livestream 1-N: streamer dùng `sendonly`, viewer dùng `recvonly`.
- **Tham khảo**: `src/content/peerconnection/perfect-negotiation/js/peer.js:38-41`.

### 2.4 Cặp Offer/Answer (SDP)
| API | Vai trò | Tham khảo |
|---|---|---|
| `pc.createOffer(options)` | Tạo SDP offer | `pc1/main.js:103` |
| `pc.createAnswer()` | Bên nhận tạo SDP answer | `pc1/main.js:137` |
| `pc.setLocalDescription(desc)` | Áp SDP local, kích hoạt ICE gathering | `pc1/main.js:118` |
| `pc.setRemoteDescription(desc)` | Áp SDP từ peer kia | `pc1/main.js:126` |

`offerOptions` phổ biến: `{ offerToReceiveAudio: 1, offerToReceiveVideo: 1, iceRestart: true }`.

### 2.5 ICE candidate
| API / Event | Vai trò | Tham khảo |
|---|---|---|
| `pc.onicecandidate` | Phát sinh candidate → gửi cho peer kia qua signaling | `pc1/main.js:90` |
| `pc.addIceCandidate(candidate)` | Áp candidate nhận từ peer | `pc1/main.js:183` |
| `pc.oniceconnectionstatechange` | Theo dõi trạng thái: `new → checking → connected → completed → disconnected → failed` | `pc1/main.js:94` |
| `pc.onicegatheringstatechange` | `new → gathering → complete` | `trickle-ice/main.js:157` |
| `pc.onicecandidateerror` | Bắt lỗi STUN/TURN (auth, không reachable) | `trickle-ice/main.js:282` |
| `pc.restartIce()` / offer với `{iceRestart:true}` | Khôi phục khi mạng đổi | `restart-ice/main.js:89` |

### 2.6 `pc.ontrack`
- Sự kiện nhận remote track. `event.streams[0]` là `MediaStream` để gán vào `<video>.srcObject`.
- **Tham khảo**: `pc1/main.js:96`, `:156`.

### 2.7 `pc.onnegotiationneeded` & Perfect Negotiation
- Trigger khi cần renegotiate (thêm track, đổi codec, restart ICE…).
- **Pattern "Perfect Negotiation"** (chống "glare" — cả 2 đầu cùng offer):
  - Mỗi đầu được gán vai trò `polite` / `impolite`.
  - Đầu `impolite` bỏ qua offer khi đang busy; `polite` luôn nhường.
  - **Tham khảo**: `src/content/peerconnection/perfect-negotiation/js/peer.js:68-128`.

### 2.8 `pc.getSenders()` / `pc.getReceivers()` / `sender.replaceTrack(newTrack)`
- Đổi camera, đổi từ camera sang screenshare mà KHÔNG cần renegotiate.
- **Tham khảo**: `perfect-negotiation/js/peer.js:47-54`.

### 2.9 `pc.getStats()`
- Trả về `RTCStatsReport` (Map) — chứa bitrate, packet loss, jitter, RTT, candidate-pair đang dùng.
- Quan trọng để hiển thị health của stream cho dashboard.
- **Tham khảo**: `restart-ice/main.js:217-249`.

### 2.10 `pc.close()`
- Đóng connection, giải phóng port/track. Sau đó cần set `pc = null`.
- **Tham khảo**: `pc1/main.js:208`.

---

## 3. RTCDataChannel — kênh dữ liệu phụ

### 3.1 `pc.createDataChannel(label, options)`
- Tạo kênh data tin cậy (mặc định) hoặc unreliable (`{ordered:false, maxRetransmits:0}`).
- **Dùng cho livestream**: chat real-time, reactions, file share, viewer count, sync control.
- **Tham khảo**: `src/content/datachannel/basic/js/main.js:39`.

### 3.2 Events trên DataChannel
| Event | Khi nào |
|---|---|
| `onopen` | Channel sẵn sàng gửi |
| `onmessage` | Nhận dữ liệu (string hoặc ArrayBuffer) |
| `onclose` | Đóng |
| `onerror` | Lỗi truyền |
| `pc.ondatachannel` | Bên kia nhận channel được tạo bởi peer | `basic/main.js:54` |

### 3.3 `channel.send(data)`
- Gửi string hoặc binary. Có thể chia chunk với `channel.bufferedAmount` để tránh tràn (xem `datachannel/datatransfer/js/main.js`).

---

## 4. Bandwidth, codec, encoding (quan trọng cho livestream)

### 4.1 `RTCRtpSender.setParameters({ encodings })`
- Điều chỉnh maxBitrate, scaleResolutionDownBy, ưu tiên codec.
- **Tham khảo**: `src/content/peerconnection/bandwidth/js/main.js`.
- Ví dụ giới hạn 500 kbps:
  ```js
  const sender = pc.getSenders().find(s => s.track.kind === 'video');
  const params = sender.getParameters();
  params.encodings[0].maxBitrate = 500_000;
  await sender.setParameters(params);
  ```

### 4.2 Simulcast — gửi nhiều layer
- Một sender, nhiều `encodings` với `rid` khác nhau (`q`, `h`, `f`) → SFU chọn layer phù hợp viewer.
- Cấu hình lúc `addTransceiver`:
  ```js
  pc.addTransceiver(track, {
    sendEncodings: [
      { rid: 'q', maxBitrate: 150_000, scaleResolutionDownBy: 4 },
      { rid: 'h', maxBitrate: 500_000, scaleResolutionDownBy: 2 },
      { rid: 'f', maxBitrate: 1_500_000 }
    ]
  });
  ```

### 4.3 Đổi codec ưu tiên
- `RTCRtpSender.getCapabilities('video').codecs` → lọc → `transceiver.setCodecPreferences(preferred)`.
- **Tham khảo**: `src/content/peerconnection/change-codecs/js/main.js`.

---

## 5. Quy trình signaling (rút từ pattern pc1 + perfect-negotiation)

```
Streamer (A)                  Signaling server                  Viewer (B)
   |  getUserMedia                                                |
   |  new RTCPeerConnection                                       |
   |  addTrack / addTransceiver                                   |
   |  createOffer → setLocalDescription                           |
   |  ----- offer (SDP) ------>                                   |
   |                                ----- offer ----->            |
   |                                                              |  setRemoteDescription
   |                                                              |  createAnswer → setLocalDescription
   |                                <----- answer ----            |
   |  <----- answer ----            |                             |
   |  setRemoteDescription                                        |
   |                                                              |
   |  onicecandidate                                              |
   |  ----- ICE ----->                                            |
   |                                ----- ICE ----->              |
   |                                                              |  addIceCandidate
   |  <--- ICE từ B ---             |                             |
   |  addIceCandidate                                             |
   |                                                              |
   |  ontrack (media) <===========================================|  addTrack thì viewer chỉ recv
   |  iceConnectionState='connected' → stream chạy                |
```

Signaling **không nằm trong WebRTC spec** — bạn tự chọn (WebSocket, SSE, HTTP long-poll…). NestJS ở phần kế tiếp sẽ đảm nhiệm vai trò này.

---

## 6. Kiến trúc thường dùng cho livestream

| Mô hình | Ưu | Nhược | Khi nào dùng |
|---|---|---|---|
| **Mesh** (N peer kết với N-1 peer) | Đơn giản, không server media | Băng thông uplink streamer = O(N) | Demo, ≤4 viewer |
| **SFU** (Selective Forwarding Unit) | Streamer chỉ upload 1 lần, SFU forward | Cần media server (mediasoup, Janus, LiveKit, Pion) | Livestream phổ biến nhất |
| **MCU** (mix server-side) | Viewer chỉ nhận 1 stream pha trộn | Tốn CPU server, độ trễ cao hơn | Conference cũ |

Với NestJS, bạn thường dùng **SFU** (ví dụ `mediasoup`) — NestJS đóng vai trò signaling + orchestration; mediasoup làm việc media.

---

## 7. Checklist các hàm cần "port" sang client của bạn

- [ ] `getUserMedia` / `getDisplayMedia`
- [ ] `new RTCPeerConnection({ iceServers })`
- [ ] `addTransceiver` với `direction` phù hợp role (streamer=sendonly, viewer=recvonly)
- [ ] `createOffer` / `createAnswer` / `setLocal` / `setRemote`
- [ ] `onicecandidate` → gửi qua signaling
- [ ] `addIceCandidate` khi nhận từ signaling
- [ ] `ontrack` → gán vào `<video>`
- [ ] `oniceconnectionstatechange` → reconnect logic (`restartIce`)
- [ ] `createDataChannel('chat')` cho chat/reaction
- [ ] `getStats()` định kỳ → đẩy lên server cho dashboard QoE
- [ ] `close()` khi user rời

---

## 8. Tham khảo nhanh trong repo

| Tính năng | File |
|---|---|
| Hello-world 1-1 call | `src/content/peerconnection/pc1/js/main.js` |
| Perfect negotiation (chống glare) | `src/content/peerconnection/perfect-negotiation/js/peer.js` |
| ICE restart khi mạng đổi | `src/content/peerconnection/restart-ice/js/main.js` |
| Kiểm tra STUN/TURN | `src/content/peerconnection/trickle-ice/js/main.js` |
| Giới hạn bandwidth | `src/content/peerconnection/bandwidth/js/main.js` |
| Đổi codec | `src/content/peerconnection/change-codecs/js/main.js` |
| Data channel cơ bản | `src/content/datachannel/basic/js/main.js` |
| File transfer qua DC | `src/content/datachannel/filetransfer/js/main.js` |
| Share screen | `src/content/getusermedia/getdisplaymedia/js/main.js` |
| Chọn thiết bị | `src/content/devices/input-output/js/main.js` |
