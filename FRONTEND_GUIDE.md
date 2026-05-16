# Frontend Guide — Livestream client với WebRTC + Socket.IO

Tài liệu này hướng dẫn build phần frontend hoàn chỉnh, ghép với NestJS signaling ở [`NESTJS_LIVESTREAM.md`](./NESTJS_LIVESTREAM.md). Có thể serve dưới dạng:

- **HTML thuần** (vanilla JS) — dùng để demo / test với ngrok.
- Hoặc tích hợp vào **React/Vue** sau (logic giống hệt, chỉ chuyển sang component & hook).

Mục tiêu chức năng:
1. Form **Join phòng**: nhập `roomId`, chọn vai trò Host/Viewer.
2. **Xin quyền** camera/mic, có preview trước khi vào phòng.
3. **Chọn thiết bị** (camera/mic input).
4. **Phòng livestream**: hiển thị video local + remote, viewer count, trạng thái kết nối.
5. **Chat** real-time qua socket.io.
6. **Toggle** mic/cam, **Share screen**, **Leave**.
7. **Tự reconnect** khi WS rớt; **ICE restart** khi mạng lag.

---

## 1. Cấu trúc thư mục FE

```
public/                          # NestJS serve qua app.useStaticAssets(__dirname + '/../public')
├── index.html                   # Trang join
├── room.html                    # Trang phòng (host + viewer chung 1 trang, phân vai qua query)
├── css/
│   └── style.css
└── js/
    ├── config.js                # SIGNALING_URL, default ICE servers
    ├── signaling.js             # Wrapper socket.io
    ├── media.js                 # getUserMedia, enumerateDevices, toggles
    ├── peer.js                  # createPeerConnection, offer/answer, ICE restart
    ├── chat.js                  # UI chat
    ├── host.js                  # Logic riêng cho host (1-N peers)
    ├── viewer.js                # Logic riêng cho viewer (1 peer)
    └── main.js                  # Entry: route theo role
```

> Cho NestJS serve FE chung cổng (đỡ phải config CORS ngrok):
> ```ts
> // main.ts
> import { NestExpressApplication } from '@nestjs/platform-express';
> import { join } from 'path';
> const app = await NestFactory.create<NestExpressApplication>(AppModule);
> app.useStaticAssets(join(__dirname, '..', 'public'));
> ```

---

## 2. `index.html` — màn hình Join

```html
<!doctype html>
<html lang="vi">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Live — Join</title>
  <link rel="stylesheet" href="/css/style.css" />
</head>
<body>
  <main class="card">
    <h1>Tham gia phòng livestream</h1>

    <label>Room ID
      <input id="roomId" placeholder="vd: demo" value="demo" />
    </label>

    <fieldset>
      <legend>Vai trò</legend>
      <label><input type="radio" name="role" value="host" checked /> Streamer (Host)</label>
      <label><input type="radio" name="role" value="viewer" /> Người xem (Viewer)</label>
    </fieldset>

    <h2>Kiểm tra thiết bị</h2>
    <video id="preview" autoplay muted playsinline></video>

    <div class="row">
      <label>Camera
        <select id="videoIn"></select>
      </label>
      <label>Mic
        <select id="audioIn"></select>
      </label>
    </div>

    <div class="row">
      <button id="grantBtn">Cấp quyền & xem preview</button>
      <button id="joinBtn" disabled>Vào phòng</button>
    </div>

    <p id="err" class="err"></p>
  </main>

  <script type="module" src="/js/join.js"></script>
</body>
</html>
```

---

## 3. `js/join.js` — xin quyền, chọn thiết bị, vào phòng

```js
// public/js/join.js
import { listDevices, requestUserMedia, stopStream } from './media.js';

const $ = (sel) => document.querySelector(sel);
const preview = $('#preview');
const videoSel = $('#videoIn');
const audioSel = $('#audioIn');
const grantBtn = $('#grantBtn');
const joinBtn = $('#joinBtn');
const errEl = $('#err');

let stream = null;

function setError(msg) {
  errEl.textContent = msg ?? '';
}

async function refreshDevices() {
  const { videoInputs, audioInputs } = await listDevices();
  fill(videoSel, videoInputs);
  fill(audioSel, audioInputs);
}

function fill(select, devices) {
  select.innerHTML = '';
  devices.forEach((d, i) => {
    const opt = document.createElement('option');
    opt.value = d.deviceId;
    opt.text = d.label || `Device ${i + 1}`;
    select.appendChild(opt);
  });
}

async function grant() {
  setError('');
  try {
    if (stream) stopStream(stream);
    stream = await requestUserMedia({
      videoDeviceId: videoSel.value || undefined,
      audioDeviceId: audioSel.value || undefined,
    });
    preview.srcObject = stream;
    joinBtn.disabled = false;
    // Sau khi user cấp quyền, label thiết bị mới xuất hiện → refresh lại.
    await refreshDevices();
  } catch (e) {
    setError(mapMediaError(e));
  }
}

function mapMediaError(e) {
  switch (e.name) {
    case 'NotAllowedError': return 'Bạn đã từ chối quyền camera/mic. Vào icon ổ khoá trên thanh URL để cấp lại.';
    case 'NotFoundError': return 'Không tìm thấy camera hoặc mic.';
    case 'NotReadableError': return 'Thiết bị đang bị app khác chiếm dụng (Zoom, Teams...).';
    case 'OverconstrainedError': return 'Camera không hỗ trợ độ phân giải yêu cầu.';
    default: return `Lỗi: ${e.name} - ${e.message}`;
  }
}

grantBtn.addEventListener('click', grant);

videoSel.addEventListener('change', () => stream && grant());
audioSel.addEventListener('change', () => stream && grant());

joinBtn.addEventListener('click', () => {
  const roomId = $('#roomId').value.trim();
  const role = document.querySelector('input[name=role]:checked').value;
  if (!roomId) { setError('Nhập Room ID'); return; }
  // Lưu lựa chọn thiết bị để room.html dùng lại.
  sessionStorage.setItem('joinConfig', JSON.stringify({
    roomId, role,
    videoDeviceId: videoSel.value,
    audioDeviceId: audioSel.value,
  }));
  // Stop preview stream — room.html sẽ tự getUserMedia lại.
  stopStream(stream);
  location.href = `/room.html?room=${encodeURIComponent(roomId)}&role=${role}`;
});

// Liệt kê thiết bị ngay khi load (labels có thể trống cho tới khi cấp quyền).
refreshDevices().catch(e => setError(e.message));
```

---

## 4. `js/media.js` — wrapper getUserMedia / device picker

Áp dụng `src/content/getusermedia/gum/js/main.js` + `src/content/devices/input-output/js/main.js`.

```js
// public/js/media.js
export async function listDevices() {
  const devices = await navigator.mediaDevices.enumerateDevices();
  return {
    videoInputs: devices.filter(d => d.kind === 'videoinput'),
    audioInputs: devices.filter(d => d.kind === 'audioinput'),
    audioOutputs: devices.filter(d => d.kind === 'audiooutput'),
  };
}

export async function requestUserMedia({ videoDeviceId, audioDeviceId } = {}) {
  const constraints = {
    audio: audioDeviceId ? { deviceId: { exact: audioDeviceId } } : true,
    video: videoDeviceId
      ? { deviceId: { exact: videoDeviceId }, width: 1280, height: 720, frameRate: 30 }
      : { width: 1280, height: 720, frameRate: 30 },
  };
  return navigator.mediaDevices.getUserMedia(constraints);
}

export async function requestDisplayMedia() {
  return navigator.mediaDevices.getDisplayMedia({
    video: { frameRate: 30 },
    audio: true,
  });
}

export function stopStream(stream) {
  if (!stream) return;
  stream.getTracks().forEach(t => t.stop());
}

export function toggleTrack(stream, kind, enabled) {
  stream.getTracks()
    .filter(t => t.kind === kind)
    .forEach(t => (t.enabled = enabled));
}
```

---

## 5. `js/signaling.js` — wrapper socket.io

```js
// public/js/signaling.js
import { SIGNALING_URL } from './config.js';

// socket.io-client từ CDN, đính qua <script> trong room.html
//   <script src="https://cdn.socket.io/4.7.5/socket.io.min.js"></script>
// hoặc bundle qua Vite.

export function connectSignaling(token) {
  const socket = io(`${SIGNALING_URL}/live`, {
    auth: { token },
    transports: ['websocket'],     // bỏ polling để latency thấp
    reconnection: true,
    reconnectionAttempts: Infinity,
    reconnectionDelay: 1000,
    reconnectionDelayMax: 5000,
  });

  socket.on('connect', () => console.log('[WS] connected', socket.id));
  socket.on('disconnect', (reason) => console.warn('[WS] disconnect', reason));
  socket.on('connect_error', (err) => console.error('[WS] connect_error', err.message));

  return socket;
}
```

```js
// public/js/config.js
export const SIGNALING_URL = `${location.protocol}//${location.host}`;
// Khi serve FE qua chính NestJS/ngrok → tự khớp domain.
```

---

## 6. `js/peer.js` — wrapper RTCPeerConnection

Tổng hợp từ `pc1/main.js` + `restart-ice/main.js`.

```js
// public/js/peer.js
export async function fetchIceServers(token) {
  const res = await fetch('/turn/credentials', {
    headers: { Authorization: `Bearer ${token}` },
  });
  if (!res.ok) {
    // Fallback chỉ STUN nếu BE chưa cấp TURN.
    return [{ urls: 'stun:stun.l.google.com:19302' }];
  }
  const { iceServers } = await res.json();
  return iceServers;
}

export function createPeer({ iceServers, onIceCandidate, onTrack, onStateChange }) {
  const pc = new RTCPeerConnection({ iceServers });

  pc.onicecandidate = (e) => onIceCandidate(e.candidate);
  pc.ontrack = (e) => onTrack(e);

  pc.oniceconnectionstatechange = () => {
    onStateChange?.(pc.iceConnectionState);
    if (pc.iceConnectionState === 'failed') {
      console.warn('[ICE] failed → restartIce()');
      pc.restartIce();
    }
  };

  return pc;
}

export async function makeOffer(pc) {
  const offer = await pc.createOffer();
  await pc.setLocalDescription(offer);
  return offer;
}

export async function makeAnswer(pc, remoteOffer) {
  await pc.setRemoteDescription({ type: 'offer', sdp: remoteOffer });
  const answer = await pc.createAnswer();
  await pc.setLocalDescription(answer);
  return answer;
}
```

---

## 7. `room.html` — UI phòng

```html
<!doctype html>
<html lang="vi">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Live — Phòng</title>
  <link rel="stylesheet" href="/css/style.css" />
</head>
<body class="room">
  <header>
    <span>Phòng: <b id="roomLabel"></b></span>
    <span>Vai trò: <b id="roleLabel"></b></span>
    <span>Viewer: <b id="viewerCount">0</b></span>
    <span>Kết nối: <b id="connState">…</b></span>
    <button id="leaveBtn">Rời phòng</button>
  </header>

  <section class="videos">
    <div class="video-wrap">
      <video id="localVideo" autoplay muted playsinline></video>
      <span class="tag">Bạn</span>
    </div>
    <!-- Viewer chỉ thấy 1 remote -->
    <div class="video-wrap" id="remoteWrap" hidden>
      <video id="remoteVideo" autoplay playsinline></video>
      <span class="tag">Streamer</span>
    </div>
  </section>

  <section class="controls">
    <button id="toggleMic">Tắt mic</button>
    <button id="toggleCam">Tắt cam</button>
    <button id="shareScreen" data-host-only>Share màn hình</button>
  </section>

  <aside class="chat">
    <ul id="chatLog"></ul>
    <form id="chatForm">
      <input id="chatInput" placeholder="Nhập tin nhắn…" autocomplete="off" />
      <button>Gửi</button>
    </form>
  </aside>

  <script src="https://cdn.socket.io/4.7.5/socket.io.min.js"></script>
  <script type="module" src="/js/main.js"></script>
</body>
</html>
```

---

## 8. `js/main.js` — entry point phân vai

```js
// public/js/main.js
import { runHost } from './host.js';
import { runViewer } from './viewer.js';

const params = new URLSearchParams(location.search);
const roomId = params.get('room');
const role = params.get('role');

if (!roomId || !['host', 'viewer'].includes(role)) {
  location.href = '/index.html';
  throw new Error('Missing room/role');
}

document.getElementById('roomLabel').textContent = roomId;
document.getElementById('roleLabel').textContent = role;

// Ẩn nút host-only nếu là viewer
if (role !== 'host') {
  document.querySelectorAll('[data-host-only]').forEach(el => el.hidden = true);
}

const token = localStorage.getItem('jwt') ?? 'demo-token';   // dev: BE cho phép token dummy
const config = JSON.parse(sessionStorage.getItem('joinConfig') ?? '{}');

if (role === 'host') {
  runHost({ roomId, token, deviceConfig: config });
} else {
  runViewer({ roomId, token });
}
```

---

## 9. `js/host.js` — Streamer logic (1 PC mỗi viewer)

```js
// public/js/host.js
import { connectSignaling } from './signaling.js';
import { requestUserMedia, requestDisplayMedia, stopStream, toggleTrack } from './media.js';
import { fetchIceServers, createPeer, makeOffer } from './peer.js';
import { setupChatUI } from './chat.js';

export async function runHost({ roomId, token, deviceConfig }) {
  const socket = connectSignaling(token);
  const iceServers = await fetchIceServers(token);

  const localVideo = document.getElementById('localVideo');
  const stateEl = document.getElementById('connState');
  const viewerCountEl = document.getElementById('viewerCount');

  let localStream = await requestUserMedia({
    videoDeviceId: deviceConfig.videoDeviceId,
    audioDeviceId: deviceConfig.audioDeviceId,
  });
  localVideo.srcObject = localStream;

  /** @type {Map<string, RTCPeerConnection>} viewerSocketId → pc */
  const peers = new Map();

  socket.emit('join-room', { roomId, role: 'host' });

  socket.on('joined', (info) => {
    stateEl.textContent = 'Đang chờ viewer';
    viewerCountEl.textContent = info.viewerCount ?? 0;
  });

  socket.on('viewer-joined', async ({ viewerSocketId }) => {
    const pc = createPeer({
      iceServers,
      onIceCandidate: (cand) => {
        if (!cand) return;
        socket.emit('ice', {
          roomId, targetSocketId: viewerSocketId,
          candidate: cand.candidate, sdpMid: cand.sdpMid, sdpMLineIndex: cand.sdpMLineIndex,
        });
      },
      onTrack: () => {},
      onStateChange: (s) => {
        stateEl.textContent = `ICE: ${s}`;
      },
    });
    peers.set(viewerSocketId, pc);
    localStream.getTracks().forEach(t => pc.addTrack(t, localStream));

    const offer = await makeOffer(pc);
    socket.emit('sdp', { roomId, targetSocketId: viewerSocketId, type: 'offer', sdp: offer.sdp });
  });

  socket.on('sdp', async ({ fromSocketId, type, sdp }) => {
    const pc = peers.get(fromSocketId);
    if (!pc || type !== 'answer') return;
    await pc.setRemoteDescription({ type: 'answer', sdp });
  });

  socket.on('ice', async ({ fromSocketId, candidate, sdpMid, sdpMLineIndex }) => {
    const pc = peers.get(fromSocketId);
    if (!pc || !candidate) return;
    try { await pc.addIceCandidate({ candidate, sdpMid, sdpMLineIndex }); }
    catch (e) { console.warn('addIceCandidate err', e); }
  });

  socket.on('peer-left', ({ socketId }) => {
    peers.get(socketId)?.close();
    peers.delete(socketId);
  });

  socket.on('viewer-count', ({ count }) => viewerCountEl.textContent = count);

  // === Controls ===
  const micBtn = document.getElementById('toggleMic');
  const camBtn = document.getElementById('toggleCam');
  let micOn = true, camOn = true;
  micBtn.onclick = () => {
    micOn = !micOn;
    toggleTrack(localStream, 'audio', micOn);
    micBtn.textContent = micOn ? 'Tắt mic' : 'Bật mic';
  };
  camBtn.onclick = () => {
    camOn = !camOn;
    toggleTrack(localStream, 'video', camOn);
    camBtn.textContent = camOn ? 'Tắt cam' : 'Bật cam';
  };

  document.getElementById('shareScreen').onclick = async () => {
    const display = await requestDisplayMedia();
    const newVideoTrack = display.getVideoTracks()[0];
    // replaceTrack trên TẤT CẢ sender của các viewer — không cần renegotiate.
    for (const pc of peers.values()) {
      const sender = pc.getSenders().find(s => s.track?.kind === 'video');
      sender?.replaceTrack(newVideoTrack);
    }
    // Khi user dừng share trên thanh browser → quay về camera.
    newVideoTrack.onended = async () => {
      const cam = await requestUserMedia({
        videoDeviceId: deviceConfig.videoDeviceId,
        audioDeviceId: deviceConfig.audioDeviceId,
      });
      const camTrack = cam.getVideoTracks()[0];
      for (const pc of peers.values()) {
        const sender = pc.getSenders().find(s => s.track?.kind === 'video');
        sender?.replaceTrack(camTrack);
      }
      // Cập nhật preview & localStream
      stopStream(localStream);
      localStream = cam;
      localVideo.srcObject = cam;
    };
    localVideo.srcObject = display;
  };

  document.getElementById('leaveBtn').onclick = () => {
    for (const pc of peers.values()) pc.close();
    stopStream(localStream);
    socket.disconnect();
    location.href = '/index.html';
  };

  setupChatUI(socket, roomId);
}
```

---

## 10. `js/viewer.js` — Người xem (1 PC duy nhất)

```js
// public/js/viewer.js
import { connectSignaling } from './signaling.js';
import { fetchIceServers, createPeer, makeAnswer } from './peer.js';
import { setupChatUI } from './chat.js';

export async function runViewer({ roomId, token }) {
  const socket = connectSignaling(token);
  const iceServers = await fetchIceServers(token);

  const remoteWrap = document.getElementById('remoteWrap');
  const remoteVideo = document.getElementById('remoteVideo');
  const localVideo = document.getElementById('localVideo');
  localVideo.parentElement.hidden = true;     // viewer không cần local
  const stateEl = document.getElementById('connState');

  let hostSocketId = null;

  const pc = createPeer({
    iceServers,
    onIceCandidate: (cand) => {
      if (!cand || !hostSocketId) return;
      socket.emit('ice', {
        roomId, targetSocketId: hostSocketId,
        candidate: cand.candidate, sdpMid: cand.sdpMid, sdpMLineIndex: cand.sdpMLineIndex,
      });
    },
    onTrack: (e) => {
      if (remoteVideo.srcObject !== e.streams[0]) {
        remoteVideo.srcObject = e.streams[0];
        remoteWrap.hidden = false;
      }
    },
    onStateChange: (s) => { stateEl.textContent = `ICE: ${s}`; },
  });

  // recvonly — báo SDP rằng viewer chỉ nhận
  pc.addTransceiver('video', { direction: 'recvonly' });
  pc.addTransceiver('audio', { direction: 'recvonly' });

  socket.emit('join-room', { roomId, role: 'viewer' });

  socket.on('joined', ({ hostSocketId: hid }) => {
    hostSocketId = hid;
    stateEl.textContent = hid ? 'Đang đợi offer từ host' : 'Host chưa vào phòng';
  });

  socket.on('sdp', async ({ fromSocketId, type, sdp }) => {
    if (type !== 'offer') return;
    hostSocketId = fromSocketId;
    const answer = await makeAnswer(pc, sdp);
    socket.emit('sdp', { roomId, targetSocketId: hostSocketId, type: 'answer', sdp: answer.sdp });
  });

  socket.on('ice', async ({ candidate, sdpMid, sdpMLineIndex }) => {
    if (!candidate) return;
    try { await pc.addIceCandidate({ candidate, sdpMid, sdpMLineIndex }); }
    catch (e) { console.warn('addIceCandidate err', e); }
  });

  socket.on('peer-left', ({ wasHost }) => {
    if (wasHost) {
      stateEl.textContent = 'Host đã rời phòng';
      remoteWrap.hidden = true;
      remoteVideo.srcObject = null;
    }
  });

  document.getElementById('toggleMic').hidden = true;   // viewer không có mic stream
  document.getElementById('toggleCam').hidden = true;

  document.getElementById('leaveBtn').onclick = () => {
    pc.close();
    socket.disconnect();
    location.href = '/index.html';
  };

  setupChatUI(socket, roomId);
}
```

---

## 11. `js/chat.js`

```js
// public/js/chat.js
export function setupChatUI(socket, roomId) {
  const form = document.getElementById('chatForm');
  const input = document.getElementById('chatInput');
  const log = document.getElementById('chatLog');

  form.addEventListener('submit', (e) => {
    e.preventDefault();
    const text = input.value.trim();
    if (!text) return;
    socket.emit('chat', { roomId, text });
    input.value = '';
  });

  socket.on('chat', ({ from, text, at }) => {
    const li = document.createElement('li');
    const time = new Date(at).toLocaleTimeString();
    li.innerHTML = `<span class="from">${escape(from)}</span>
                    <span class="time">${time}</span>
                    <span class="text">${escape(text)}</span>`;
    log.appendChild(li);
    log.scrollTop = log.scrollHeight;
  });
}

function escape(s) {
  return String(s).replace(/[&<>"']/g, c => ({
    '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'
  })[c]);
}
```

---

## 12. `css/style.css` (gợi ý tối thiểu)

```css
* { box-sizing: border-box; }
body { font-family: system-ui, sans-serif; margin: 0; background: #111; color: #eee; }

.card { max-width: 540px; margin: 40px auto; padding: 24px; background: #1c1c1f; border-radius: 12px; }
.card label, .card fieldset { display: block; margin: 10px 0; }
.card input, .card select, .card button { width: 100%; padding: 8px; margin-top: 4px;
  background: #2a2a2e; color: #eee; border: 1px solid #444; border-radius: 6px; }
.card button { cursor: pointer; }
.card button:disabled { opacity: 0.5; cursor: not-allowed; }
.row { display: flex; gap: 12px; }
.row > * { flex: 1; }
video#preview { width: 100%; aspect-ratio: 16/9; background: black; border-radius: 8px; }
.err { color: #ff7a7a; }

body.room { display: grid; grid-template-rows: auto 1fr auto; height: 100vh; }
body.room header { display: flex; gap: 16px; align-items: center; padding: 8px 16px; background: #000; }
body.room .videos { display: flex; gap: 16px; padding: 16px; overflow: auto; }
.video-wrap { position: relative; flex: 1; min-width: 320px; }
.video-wrap video { width: 100%; background: #000; aspect-ratio: 16/9; border-radius: 8px; }
.video-wrap .tag { position: absolute; left: 8px; bottom: 8px; background: rgba(0,0,0,.6); padding: 2px 8px; border-radius: 4px; font-size: 12px; }
.controls { display: flex; gap: 8px; padding: 8px 16px; background: #000; }
.chat { position: fixed; right: 16px; top: 64px; bottom: 80px; width: 320px;
  background: #1c1c1f; border-radius: 12px; display: flex; flex-direction: column; }
.chat ul { flex: 1; overflow: auto; list-style: none; margin: 0; padding: 12px; }
.chat ul li { margin: 6px 0; font-size: 13px; }
.chat .from { color: #82c4ff; margin-right: 6px; }
.chat .time { color: #888; font-size: 11px; margin-right: 6px; }
.chat form { display: flex; padding: 8px; gap: 6px; }
.chat input { flex: 1; }
```

---

## 13. Luồng UX đầu-cuối (sau khi BE đã chạy + ngrok)

1. **Mở** `https://abcd-1234.ngrok-free.app/` trên Máy A và Máy B.
2. Cả 2 máy → nhập `roomId = demo`.
3. **Máy A**: chọn **Host** → bấm "Cấp quyền & xem preview" → browser xin quyền camera/mic → chọn camera/mic → "Vào phòng".
4. **Máy B**: chọn **Viewer** → "Cấp quyền" (chỉ để có icon micro/cam ở thanh URL trong tương lai; viewer không stream lên) → "Vào phòng".
   - *(Có thể bỏ bước cấp quyền cho viewer — nhưng để UI nhất quán thì giữ.)*
5. Trên Máy B, sau ~1-3 giây thấy video Máy A. Trạng thái `ICE: connected`.
6. Gõ chat → cả 2 đều thấy.
7. Bấm "Share màn hình" trên Máy A → Máy B thấy desktop. Dừng share → quay lại camera.
8. Tắt mic/cam → Máy B thấy track im lặng / màn đen.
9. Rớt WiFi tạm → state đổi sang `disconnected` → trở lại WiFi → `restartIce` tự gọi → quay về `connected`.

---

## 14. Edge case cần xử lý trên FE

| Tình huống | Hành xử FE |
|---|---|
| User bấm "Block" quyền camera | Show hướng dẫn click icon ổ khoá để bật lại; không retry vô tận |
| Browser cũ không hỗ trợ `RTCPeerConnection` | `if (!window.RTCPeerConnection) showFallbackMsg()` |
| `getUserMedia` chỉ chạy trên `https` hoặc `localhost` | Cảnh báo khi `location.protocol === 'http:'` và host khác `localhost` |
| Viewer vào trước khi host có mặt | `joined` event trả `hostSocketId = null` → hiển thị "Chờ streamer…", không error |
| Host đổi camera giữa chừng | Gọi `getUserMedia` mới → `replaceTrack` trên mọi sender (xem ví dụ share screen) |
| Tab bị throttle khi background | `iceConnectionState` đôi khi báo `disconnected` → đợi 5-10s rồi mới `restartIce` (tránh restart liên tục) |
| Mobile Safari | `autoplay` cần `playsinline`; user phải tương tác trước khi audio chạy (đã add `playsinline` ở `<video>`) |

---

## 15. Test nhanh không cần BE (sanity check)

Trước khi ráp BE, mở 2 tab Chrome cùng máy:

1. Tab 1 → `index.html` → role host.
2. Tab 2 → `index.html` → role viewer.

Cả 2 dùng chung `BroadcastChannel('signaling')` thay cho socket.io (chỉ vài chục dòng) — không thực tế dùng prod, nhưng giúp verify offer/answer/ICE pipeline. Sau khi flow ổn, switch sang `connectSignaling` thật. (Không bắt buộc; nếu BE đã sẵn sàng thì test thẳng qua ngrok.)

---

## 16. Checklist FE production

- [ ] HTTPS bắt buộc (localhost OK khi dev).
- [ ] Cảnh báo khi `navigator.mediaDevices === undefined` (insecure context).
- [ ] Cleanup: `stopStream`, `pc.close()`, `socket.disconnect()` khi unmount / `beforeunload`.
- [ ] Throttle nút "Vào phòng" để tránh tạo nhiều PC.
- [ ] Hiển thị trạng thái `iceConnectionState` rõ ràng để user biết khi nào đang reconnect.
- [ ] Tắt `console.log` SDP ở production (SDP có thể chứa IP).
- [ ] Lazy-load `socket.io-client` nếu bundle to.
- [ ] A11y: `<button>` có `aria-label` cho icon-only buttons.
- [ ] Mobile: kiểm tra `playsinline`, không autoplay audio.

---

## 17. Tham chiếu chéo

- API WebRTC chi tiết: [`WEBRTC_FUNCTIONS.md`](./WEBRTC_FUNCTIONS.md)
- Server signaling (NestJS) + ngrok + Redis: [`NESTJS_LIVESTREAM.md`](./NESTJS_LIVESTREAM.md)
- Các sample gốc đọc thêm: `src/content/peerconnection/pc1/`, `src/content/peerconnection/perfect-negotiation/`, `src/content/getusermedia/gum/`, `src/content/devices/input-output/`.
