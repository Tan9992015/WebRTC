# Apply WebRTC vào NestJS — xây chức năng Livestream

Tài liệu này áp dụng các API WebRTC liệt kê trong [`WEBRTC_FUNCTIONS.md`](./WEBRTC_FUNCTIONS.md) vào kiến trúc NestJS để dựng chức năng livestream **1 streamer → N viewer**, dùng:

- **NestJS** + `@nestjs/websockets` + `socket.io` làm **signaling server**.
- **Mesh** cho MVP (≤ 4 viewer), hoặc gắn thêm **mediasoup** cho production.
- Client (browser / React / Flutter Web) dùng các hàm WebRTC.

---

## 1. Tổng quan kiến trúc

```
┌────────────┐      WS signaling       ┌──────────────────┐      WS signaling      ┌────────────┐
│  Streamer  │ <─────────────────────> │  NestJS Gateway  │ <────────────────────> │  Viewer N  │
│ (browser)  │                         │  (signaling)     │                        │ (browser)  │
└─────┬──────┘                         │  + Room state    │                        └─────┬──────┘
      │                                │  + Auth + TURN   │                              │
      │                                │   creds          │                              │
      │                                └──────────────────┘                              │
      │                                                                                  │
      │ ════════════════ SRTP media (P2P qua STUN/TURN) ════════════════════════════════ │
```

NestJS **không xử lý media** — nó chỉ:
1. Auth user (JWT).
2. Quản lý phòng (room): ai là host, ai là viewer.
3. Trung chuyển `offer / answer / ice-candidate` giữa các peer.
4. Cấp credential TURN tạm thời (nếu dùng coturn với REST API).
5. Broadcast event UI: chat, viewer count, host left.

---

## 2. Cấu trúc thư mục NestJS đề xuất

```
src/
├── app.module.ts
├── main.ts
├── auth/
│   ├── auth.module.ts
│   └── ws-jwt.guard.ts
├── livestream/
│   ├── livestream.module.ts
│   ├── livestream.gateway.ts     # WebSocket signaling
│   ├── livestream.service.ts     # Room state, in-memory hoặc Redis
│   ├── dto/
│   │   ├── join-room.dto.ts
│   │   ├── sdp.dto.ts
│   │   └── ice.dto.ts
│   └── types.ts
└── turn/
    └── turn.controller.ts        # Cấp credential TURN tạm thời
```

---

## 3. Cài đặt

```bash
npm i @nestjs/websockets @nestjs/platform-socket.io socket.io
npm i class-validator class-transformer
npm i @nestjs/jwt
# Nếu dùng SFU
npm i mediasoup
```

`main.ts` — bật CORS WS:
```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { cors: true });
  await app.listen(3000);
}
bootstrap();
```

---

## 4. DTO (validate payload từ client)

```ts
// src/livestream/dto/sdp.dto.ts
import { IsString, IsIn, IsNotEmpty } from 'class-validator';

export class SdpDto {
  @IsString() roomId: string;
  @IsString() targetSocketId: string;   // peer cần nhận SDP
  @IsIn(['offer', 'answer']) type: 'offer' | 'answer';
  @IsString() @IsNotEmpty() sdp: string;
}
```

```ts
// src/livestream/dto/ice.dto.ts
import { IsString, IsOptional, IsNumber } from 'class-validator';

export class IceDto {
  @IsString() roomId: string;
  @IsString() targetSocketId: string;
  @IsString() candidate: string;
  @IsOptional() @IsString() sdpMid?: string;
  @IsOptional() @IsNumber() sdpMLineIndex?: number;
}
```

```ts
// src/livestream/dto/join-room.dto.ts
import { IsIn, IsString } from 'class-validator';

export class JoinRoomDto {
  @IsString() roomId: string;
  @IsIn(['host', 'viewer']) role: 'host' | 'viewer';
}
```

---

## 5. Service quản lý room

```ts
// src/livestream/livestream.service.ts
import { Injectable, ForbiddenException, NotFoundException } from '@nestjs/common';

type Peer = { socketId: string; userId: string; role: 'host' | 'viewer' };
type Room = { id: string; hostSocketId?: string; peers: Map<string, Peer> };

@Injectable()
export class LivestreamService {
  private rooms = new Map<string, Room>();

  join(roomId: string, peer: Peer): Room {
    let room = this.rooms.get(roomId);
    if (!room) room = { id: roomId, peers: new Map() };

    if (peer.role === 'host') {
      if (room.hostSocketId && room.hostSocketId !== peer.socketId) {
        throw new ForbiddenException('Room already has a host');
      }
      room.hostSocketId = peer.socketId;
    }
    room.peers.set(peer.socketId, peer);
    this.rooms.set(roomId, room);
    return room;
  }

  leave(socketId: string): { roomId: string; wasHost: boolean } | null {
    for (const room of this.rooms.values()) {
      if (room.peers.delete(socketId)) {
        const wasHost = room.hostSocketId === socketId;
        if (wasHost) room.hostSocketId = undefined;
        if (room.peers.size === 0) this.rooms.delete(room.id);
        return { roomId: room.id, wasHost };
      }
    }
    return null;
  }

  getRoom(roomId: string): Room {
    const room = this.rooms.get(roomId);
    if (!room) throw new NotFoundException('Room not found');
    return room;
  }

  getViewers(roomId: string): Peer[] {
    const room = this.getRoom(roomId);
    return Array.from(room.peers.values()).filter(p => p.role === 'viewer');
  }
}
```

> Production: thay `Map` bằng Redis (`ioredis`) để scale nhiều instance NestJS, kết hợp socket.io-redis-adapter.

---

## 6. Gateway — bộ não signaling

```ts
// src/livestream/livestream.gateway.ts
import {
  WebSocketGateway, WebSocketServer, SubscribeMessage,
  OnGatewayConnection, OnGatewayDisconnect, MessageBody, ConnectedSocket,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { UsePipes, ValidationPipe, UseGuards } from '@nestjs/common';
import { LivestreamService } from './livestream.service';
import { JoinRoomDto } from './dto/join-room.dto';
import { SdpDto } from './dto/sdp.dto';
import { IceDto } from './dto/ice.dto';
import { WsJwtGuard } from '../auth/ws-jwt.guard';

@UseGuards(WsJwtGuard)
@UsePipes(new ValidationPipe({ whitelist: true }))
@WebSocketGateway({ namespace: '/live', cors: { origin: '*' } })
export class LivestreamGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer() server: Server;

  constructor(private readonly svc: LivestreamService) {}

  handleConnection(client: Socket) {
    // userId được WsJwtGuard gắn vào client.data.user
    console.log(`[WS] connected ${client.id} user=${client.data.user?.id}`);
  }

  handleDisconnect(client: Socket) {
    const info = this.svc.leave(client.id);
    if (!info) return;
    this.server.to(info.roomId).emit('peer-left', {
      socketId: client.id,
      wasHost: info.wasHost,
    });
  }

  // 1) Join room — phân vai host/viewer
  @SubscribeMessage('join-room')
  onJoin(@ConnectedSocket() client: Socket, @MessageBody() dto: JoinRoomDto) {
    const userId = client.data.user.id;
    this.svc.join(dto.roomId, { socketId: client.id, userId, role: dto.role });
    client.join(dto.roomId);

    const room = this.svc.getRoom(dto.roomId);

    if (dto.role === 'viewer' && room.hostSocketId) {
      // Báo host: có viewer mới → host khởi tạo offer cho viewer này
      this.server.to(room.hostSocketId).emit('viewer-joined', {
        viewerSocketId: client.id,
      });
    }

    // Trả về cho viewer biết hostSocketId (để gửi answer/ice ngược lại)
    client.emit('joined', {
      roomId: dto.roomId,
      hostSocketId: room.hostSocketId ?? null,
      viewerCount: this.svc.getViewers(dto.roomId).length,
    });

    this.server.to(dto.roomId).emit('viewer-count', {
      count: this.svc.getViewers(dto.roomId).length,
    });
  }

  // 2) Relay SDP offer / answer
  @SubscribeMessage('sdp')
  onSdp(@ConnectedSocket() client: Socket, @MessageBody() dto: SdpDto) {
    this.server.to(dto.targetSocketId).emit('sdp', {
      fromSocketId: client.id,
      type: dto.type,
      sdp: dto.sdp,
    });
  }

  // 3) Relay ICE candidate
  @SubscribeMessage('ice')
  onIce(@ConnectedSocket() client: Socket, @MessageBody() dto: IceDto) {
    this.server.to(dto.targetSocketId).emit('ice', {
      fromSocketId: client.id,
      candidate: dto.candidate,
      sdpMid: dto.sdpMid,
      sdpMLineIndex: dto.sdpMLineIndex,
    });
  }

  // 4) Chat (đi qua signaling thay vì RTCDataChannel cho đơn giản)
  @SubscribeMessage('chat')
  onChat(@ConnectedSocket() client: Socket, @MessageBody() body: { roomId: string; text: string }) {
    this.server.to(body.roomId).emit('chat', {
      from: client.data.user.id,
      text: body.text,
      at: Date.now(),
    });
  }
}
```

### Mapping API WebRTC ↔ event Gateway

| Bên client gọi | Gateway nhận | Gateway phát ngược |
|---|---|---|
| `pc.createOffer` → `setLocalDescription` → emit `sdp` | `@SubscribeMessage('sdp')` | `sdp` đến `targetSocketId` |
| `pc.onicecandidate` → emit `ice` | `@SubscribeMessage('ice')` | `ice` đến peer kia |
| Disconnect | `handleDisconnect` | `peer-left` cho cả phòng |
| Viewer mới vào | `join-room` | `viewer-joined` → host khởi tạo offer |

---

## 7. WS JWT Guard (auth)

```ts
// src/auth/ws-jwt.guard.ts
import { CanActivate, ExecutionContext, Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { Socket } from 'socket.io';

@Injectable()
export class WsJwtGuard implements CanActivate {
  constructor(private jwt: JwtService) {}

  canActivate(ctx: ExecutionContext): boolean {
    const client: Socket = ctx.switchToWs().getClient();
    const token = client.handshake.auth?.token ?? client.handshake.headers.authorization?.split(' ')[1];
    if (!token) throw new UnauthorizedException('No token');
    try {
      const payload = this.jwt.verify(token);
      client.data.user = payload;     // gắn vào socket để các handler dùng
      return true;
    } catch {
      throw new UnauthorizedException('Invalid token');
    }
  }
}
```

---

## 8. Endpoint cấp credential TURN tạm thời

Coturn hỗ trợ "TURN REST API": HMAC-SHA1(secret, `expiry:username`). Trả về username/credential ngắn hạn để client embed vào `iceServers`.

```ts
// src/turn/turn.controller.ts
import { Controller, Get, UseGuards } from '@nestjs/common';
import { createHmac } from 'crypto';
import { AuthGuard } from '@nestjs/passport';

@Controller('turn')
export class TurnController {
  @UseGuards(AuthGuard('jwt'))
  @Get('credentials')
  getCreds() {
    const secret = process.env.TURN_SECRET!;
    const ttl = 3600; // 1h
    const username = `${Math.floor(Date.now() / 1000) + ttl}:livestream`;
    const credential = createHmac('sha1', secret).update(username).digest('base64');
    return {
      iceServers: [
        { urls: 'stun:stun.l.google.com:19302' },
        {
          urls: [
            `turn:${process.env.TURN_HOST}:3478?transport=udp`,
            `turn:${process.env.TURN_HOST}:3478?transport=tcp`,
          ],
          username,
          credential,
        },
      ],
      ttl,
    };
  }
}
```

---

## 9. Client (Browser) — ráp với các hàm WebRTC từ samples

Đây là phần "port" từ `src/content/peerconnection/pc1/js/main.js` & `perfect-negotiation/js/peer.js` sang flow socket.io.

### 9.1 Host (streamer)

```ts
import { io, Socket } from 'socket.io-client';

const socket: Socket = io('http://localhost:3000/live', {
  auth: { token: localStorage.getItem('jwt') },
});

const peers = new Map<string, RTCPeerConnection>();   // viewerSocketId → pc
let localStream: MediaStream;
let iceServers: RTCIceServer[] = [];

async function startHosting(roomId: string) {
  // 1. Lấy media — tương đương getusermedia/gum/js/main.js:46
  localStream = await navigator.mediaDevices.getUserMedia({
    audio: true,
    video: { width: 1280, height: 720, frameRate: 30 },
  });
  document.querySelector<HTMLVideoElement>('#preview')!.srcObject = localStream;

  // 2. Lấy ICE servers từ NestJS
  const res = await fetch('/turn/credentials', {
    headers: { Authorization: `Bearer ${localStorage.getItem('jwt')}` },
  });
  iceServers = (await res.json()).iceServers;

  // 3. Join room
  socket.emit('join-room', { roomId, role: 'host' });
}

// 4. Mỗi khi có viewer mới → host tạo PC riêng cho viewer đó (mesh)
socket.on('viewer-joined', async ({ viewerSocketId }) => {
  const pc = createPeerForViewer(viewerSocketId);
  peers.set(viewerSocketId, pc);

  // sendonly transceiver — viewer chỉ nhận
  localStream.getTracks().forEach(t => pc.addTrack(t, localStream));

  const offer = await pc.createOffer();
  await pc.setLocalDescription(offer);
  socket.emit('sdp', {
    roomId: currentRoomId,
    targetSocketId: viewerSocketId,
    type: 'offer',
    sdp: offer.sdp!,
  });
});

socket.on('sdp', async ({ fromSocketId, type, sdp }) => {
  const pc = peers.get(fromSocketId);
  if (!pc) return;
  if (type === 'answer') {
    await pc.setRemoteDescription({ type: 'answer', sdp });
  }
});

socket.on('ice', async ({ fromSocketId, candidate, sdpMid, sdpMLineIndex }) => {
  const pc = peers.get(fromSocketId);
  if (!pc || !candidate) return;
  await pc.addIceCandidate({ candidate, sdpMid, sdpMLineIndex });
});

socket.on('peer-left', ({ socketId }) => {
  peers.get(socketId)?.close();
  peers.delete(socketId);
});

function createPeerForViewer(viewerSocketId: string): RTCPeerConnection {
  const pc = new RTCPeerConnection({ iceServers });

  // Tương đương pc1/main.js:90
  pc.onicecandidate = (e) => {
    if (e.candidate) {
      socket.emit('ice', {
        roomId: currentRoomId,
        targetSocketId: viewerSocketId,
        candidate: e.candidate.candidate,
        sdpMid: e.candidate.sdpMid,
        sdpMLineIndex: e.candidate.sdpMLineIndex,
      });
    }
  };

  // Tương đương restart-ice/main.js:116 — tự khôi phục khi mạng đổi
  pc.oniceconnectionstatechange = () => {
    if (pc.iceConnectionState === 'failed') pc.restartIce();
  };

  return pc;
}
```

### 9.2 Viewer

```ts
const socket = io('http://localhost:3000/live', {
  auth: { token: localStorage.getItem('jwt') },
});

let pc: RTCPeerConnection;
let hostSocketId: string;
const remoteVideo = document.querySelector<HTMLVideoElement>('#remote')!;

async function joinAsViewer(roomId: string) {
  const res = await fetch('/turn/credentials', {
    headers: { Authorization: `Bearer ${localStorage.getItem('jwt')}` },
  });
  const { iceServers } = await res.json();

  pc = new RTCPeerConnection({ iceServers });

  // recvonly — viewer chỉ nhận
  pc.addTransceiver('video', { direction: 'recvonly' });
  pc.addTransceiver('audio', { direction: 'recvonly' });

  // Tương đương pc1/main.js:156
  pc.ontrack = (e) => {
    if (remoteVideo.srcObject !== e.streams[0]) {
      remoteVideo.srcObject = e.streams[0];
    }
  };

  pc.onicecandidate = (e) => {
    if (e.candidate && hostSocketId) {
      socket.emit('ice', {
        roomId,
        targetSocketId: hostSocketId,
        candidate: e.candidate.candidate,
        sdpMid: e.candidate.sdpMid,
        sdpMLineIndex: e.candidate.sdpMLineIndex,
      });
    }
  };

  socket.emit('join-room', { roomId, role: 'viewer' });
}

socket.on('joined', ({ hostSocketId: hid }) => {
  hostSocketId = hid;   // sẽ nhận offer từ host
});

socket.on('sdp', async ({ fromSocketId, type, sdp }) => {
  if (type !== 'offer') return;
  hostSocketId = fromSocketId;
  await pc.setRemoteDescription({ type: 'offer', sdp });
  const answer = await pc.createAnswer();
  await pc.setLocalDescription(answer);
  socket.emit('sdp', {
    roomId: currentRoomId,
    targetSocketId: hostSocketId,
    type: 'answer',
    sdp: answer.sdp!,
  });
});

socket.on('ice', async ({ candidate, sdpMid, sdpMLineIndex }) => {
  if (!candidate) return;
  await pc.addIceCandidate({ candidate, sdpMid, sdpMLineIndex });
});
```

---

## 10. Chat & reactions qua DataChannel (tuỳ chọn)

Nếu muốn chat đi P2P (không qua server) — tham khảo `datachannel/basic/main.js:39`:

```ts
// Host phía streamer khi tạo pc cho viewer:
const chat = pc.createDataChannel('chat');
chat.onopen = () => console.log('chat ready');
chat.onmessage = e => appendChatMessage(JSON.parse(e.data));

// Viewer:
pc.ondatachannel = (e) => {
  const chat = e.channel;
  chat.onmessage = ev => appendChatMessage(JSON.parse(ev.data));
};
```

> Thực tế cho livestream nhiều viewer, **chat nên qua server** (như event `chat` ở mục 6) để server lưu lịch sử, kiểm duyệt, anti-spam. Dùng DataChannel chỉ khi cần low-latency 1-1 (call, game control).

---

## 11. Health & monitoring với `getStats()`

Định kỳ 5s gửi metrics lên NestJS để dashboard:

```ts
setInterval(async () => {
  for (const [viewerId, pc] of peers) {
    const stats = await pc.getStats();
    let outbound: any;
    stats.forEach(r => { if (r.type === 'outbound-rtp' && r.kind === 'video') outbound = r; });
    socket.emit('stats', {
      viewerId,
      bytesSent: outbound?.bytesSent,
      packetsLost: outbound?.packetsLost,
      framesPerSecond: outbound?.framesPerSecond,
    });
  }
}, 5000);
```

Thêm `@SubscribeMessage('stats')` vào gateway để lưu vào DB / Prometheus.

---

## 12. Khi nào nâng cấp Mesh → SFU (mediasoup)

Mesh chỉ hoạt động khi uplink streamer ≥ N × bitrate. Với 720p ~1.5 Mbps × 10 viewer = 15 Mbps upload — quá tải hầu hết mạng nhà.

**Chuyển sang SFU `mediasoup`** khi:
- Viewer > 5.
- Cần record server-side.
- Cần simulcast tự động.

Trong kiến trúc đó:
- Streamer chỉ tạo **1 PC duy nhất** đến mediasoup worker (do NestJS quản lý).
- Mỗi viewer tạo **1 PC đến mediasoup** (NestJS làm signaling như cũ).
- Mediasoup forward RTP — không decode.

Mediasoup chạy ngay trong process NestJS:
```ts
import * as mediasoup from 'mediasoup';
const worker = await mediasoup.createWorker();
const router = await worker.createRouter({ mediaCodecs: [...] });
// Mỗi user → mediasoup.Transport (WebRtcTransport)
// Streamer.produce() → router → Viewer.consume()
```

Tham khảo: https://mediasoup.org/documentation/v3/mediasoup/api/

---

## 13. Checklist triển khai

- [ ] HTTPS bắt buộc trên production (camera/mic chỉ hoạt động trên secure context).
- [ ] Triển khai coturn (UDP 3478 + TCP 443 fallback cho mạng chặn UDP).
- [ ] JWT verify cho cả HTTP lẫn WebSocket.
- [ ] Rate-limit event `sdp` / `ice` (chống spam).
- [ ] Validate `targetSocketId` thực sự cùng phòng với `client.id` trước khi relay (chống injection).
- [ ] Redis adapter cho socket.io khi scale > 1 instance.
- [ ] Log + metrics (`getStats`, ICE state) để debug "viewer thấy màn đen".
- [ ] UI: hiển thị `iceConnectionState` để user biết "đang kết nối / đã mất kết nối".
- [ ] Nút "tắt mic / tắt cam" — dùng `track.enabled = false`, không stop track.
- [ ] Reconnect: viewer auto rejoin khi WS rớt; host gọi `pc.restartIce()` khi `iceConnectionState === 'failed'`.

---

## 14. Bảng đối chiếu nhanh "samples → NestJS livestream"

| Sample trong repo | Phần áp dụng vào livestream NestJS |
|---|---|
| `peerconnection/pc1` | Toàn bộ flow offer/answer/ICE — backbone của host & viewer |
| `peerconnection/perfect-negotiation` | Áp dụng khi host hỗ trợ đổi camera/screen-share giữa chừng |
| `peerconnection/restart-ice` | Logic reconnect khi mạng đổi (`oniceconnectionstatechange` → `restartIce`) |
| `peerconnection/trickle-ice` | Kiểm tra cấu hình STUN/TURN của bạn trước khi deploy |
| `peerconnection/bandwidth` | Giới hạn bitrate viewer-side hoặc adaptive theo network |
| `peerconnection/change-codecs` | Ưu tiên VP9/AV1 nếu hỗ trợ, fallback H264 |
| `getusermedia/gum` | `startHosting()` của streamer |
| `getusermedia/getdisplaymedia` | Nút "Share screen" của streamer |
| `devices/input-output` | Dropdown chọn camera/mic |
| `datachannel/basic` | Chat low-latency 1-1 (optional) |
| `datachannel/filetransfer` | Gửi file trong stream (optional) |

---

Đọc kèm với [`WEBRTC_FUNCTIONS.md`](./WEBRTC_FUNCTIONS.md) để hiểu chi tiết từng API mà mã NestJS/Client ở trên đang gọi. Phần code FE/HTML đầy đủ xem [`FRONTEND_GUIDE.md`](./FRONTEND_GUIDE.md).

---

## 15. Test trên máy local — kết nối 2 máy qua ngrok

**Có, hoàn toàn test được giữa 2 máy** (cùng/khác mạng) bằng ngrok. Đây là setup tối thiểu nhất.

### 15.1 Vì sao cần ngrok

- WebRTC API (`getUserMedia`, `getDisplayMedia`) **chỉ hoạt động trên `https://` hoặc `http://localhost`**. Mở `http://192.168.x.x:3000` từ máy khác → browser block camera/mic.
- Ngrok tạo tunnel `https://xxxx.ngrok-free.app` → forward về `localhost:3000`, vừa có HTTPS, vừa expose ra Internet để máy bạn bè vào được.
- WebSocket cũng tunnel được qua ngrok (socket.io vẫn hoạt động bình thường).

### 15.2 Setup

**Bước 1 — Cài ngrok:** https://ngrok.com/download → đăng ký lấy authtoken.
```bash
ngrok config add-authtoken <YOUR_TOKEN>
```

**Bước 2 — Chạy NestJS:**
```bash
npm run start:dev    # http://localhost:3000
```

**Bước 3 — Expose:** Mở terminal thứ 2:
```bash
ngrok http 3000
```
Output:
```
Forwarding   https://abcd-1234.ngrok-free.app -> http://localhost:3000
```

**Bước 4 — Cấu hình CORS cho ngrok** (trong `main.ts`):
```ts
const app = await NestFactory.create(AppModule, {
  cors: {
    origin: [
      'http://localhost:5173',
      /\.ngrok-free\.app$/,
      /\.ngrok\.io$/,
    ],
    credentials: true,
  },
});
```

Và trong gateway:
```ts
@WebSocketGateway({
  namespace: '/live',
  cors: { origin: true, credentials: true },   // true = allow same origin
})
```

**Bước 5 — Sửa FE để trỏ tới ngrok URL:**
```ts
// src/config.ts
export const SIGNALING_URL = import.meta.env.VITE_SIGNALING_URL
  ?? `${window.location.protocol}//${window.location.host}`;
```
Khi serve FE qua chính NestJS (`app.useStaticAssets`) → URL tự match ngrok. Khi serve FE riêng (Vite dev), set `VITE_SIGNALING_URL=https://abcd-1234.ngrok-free.app`.

### 15.3 Quy trình test 2 máy

| Vai | Máy | Thao tác |
|---|---|---|
| Host | Máy A (chạy NestJS) | Mở `https://abcd-1234.ngrok-free.app` → nhập roomId `demo` → bấm **Start hosting** → cấp quyền camera/mic |
| Viewer | Máy B (bất kỳ) | Mở chính URL ngrok đó → nhập roomId `demo` → bấm **Join as viewer** |

Sau ~2-3s viewer thấy video host. Mở DevTools → console để xem log `iceConnectionState`.

### 15.4 Cảnh báo về ngrok

- Plan free đôi khi cần click "Visit Site" trên trang interstitial — fix bằng `--header="ngrok-skip-browser-warning:true"` hoặc plan trả phí.
- Ngrok ổn cho signaling (WS) nhưng **media (SRTP)** thì **không** đi qua ngrok — đi P2P trực tiếp qua STUN/TURN. Nếu 1 máy ở mạng đối xứng NAT, bạn vẫn cần TURN server thật (không thể tunnel qua ngrok).
- Nếu cả 2 máy cùng LAN → STUN Google là đủ; khác mạng → cần TURN (coturn) hoặc dịch vụ như Twilio NAT Traversal, Metered TURN.

### 15.5 Không có TURN public — fallback dùng coturn cục bộ

```bash
docker run -d --network host coturn/coturn -n \
  --log-file=stdout --min-port=49160 --max-port=49200 \
  --lt-cred-mech --realm=live.local \
  --user=test:test123 \
  --external-ip=$(curl -s ifconfig.me)
```
Sau đó FE/BE dùng:
```js
iceServers: [
  { urls: 'stun:stun.l.google.com:19302' },
  { urls: 'turn:<YOUR_PUBLIC_IP>:3478', username: 'test', credential: 'test123' },
]
```

> Lưu ý: máy phải có IP public hoặc port-forward router 3478 UDP. Trên ngrok không tunnel được TURN (UDP arbitrary port).

---

## 16. Socket.IO + Redis adapter — khi nào, vì sao, code

### 16.1 Khi nào CẦN Redis

| Tình huống | Cần Redis? |
|---|---|
| 1 instance NestJS, ≤ vài chục phòng | **Không** |
| Local dev / ngrok test 2 máy | **Không** |
| Chạy 2+ replica NestJS sau load balancer | **Có** — host ở pod A, viewer ở pod B → không thấy nhau nếu không pub/sub |
| Cần restart server mà không mất room state | **Có** (lưu state Redis thay vì Map in-memory) |
| Cần presence (đếm viewer toàn cluster) | **Có** |

→ Cho MVP/ngrok demo: **bỏ qua Redis**. Chỉ thêm khi scale ngang.

### 16.2 Cài đặt

```bash
npm i @socket.io/redis-adapter ioredis
```

### 16.3 Mount adapter

Tạo `IoAdapter` tuỳ biến:

```ts
// src/livestream/redis-io.adapter.ts
import { IoAdapter } from '@nestjs/platform-socket.io';
import { ServerOptions } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';
import { INestApplicationContext } from '@nestjs/common';

export class RedisIoAdapter extends IoAdapter {
  private adapterConstructor: ReturnType<typeof createAdapter>;

  constructor(app: INestApplicationContext) {
    super(app);
  }

  async connectToRedis(url = process.env.REDIS_URL ?? 'redis://localhost:6379') {
    const pubClient = createClient({ url });
    const subClient = pubClient.duplicate();
    await Promise.all([pubClient.connect(), subClient.connect()]);
    this.adapterConstructor = createAdapter(pubClient, subClient);
  }

  createIOServer(port: number, options?: ServerOptions) {
    const server = super.createIOServer(port, options);
    server.adapter(this.adapterConstructor);
    return server;
  }
}
```

Mount trong `main.ts`:
```ts
const app = await NestFactory.create(AppModule);
const redisAdapter = new RedisIoAdapter(app);
await redisAdapter.connectToRedis();
app.useWebSocketAdapter(redisAdapter);
await app.listen(3000);
```

Sau bước này, `server.to(roomId).emit(...)` tự broadcast tới socket ở **mọi instance** qua Redis pub/sub.

### 16.4 Lưu room state vào Redis (thay Map in-memory)

Map in-memory sẽ mất khi process restart hoặc không share giữa 2 pod. Refactor `LivestreamService` dùng Redis:

```ts
// src/livestream/livestream.redis.service.ts
import { Injectable, OnModuleInit } from '@nestjs/common';
import Redis from 'ioredis';

@Injectable()
export class LivestreamRedisService implements OnModuleInit {
  private redis: Redis;
  onModuleInit() {
    this.redis = new Redis(process.env.REDIS_URL ?? 'redis://localhost:6379');
  }

  // Key design:
  //   room:{id}:peers  → HSET socketId → JSON({userId, role})
  //   room:{id}:host   → STRING socketId

  async join(roomId: string, peer: { socketId: string; userId: string; role: 'host' | 'viewer' }) {
    if (peer.role === 'host') {
      const exist = await this.redis.get(`room:${roomId}:host`);
      if (exist && exist !== peer.socketId) throw new Error('Room already has a host');
      await this.redis.set(`room:${roomId}:host`, peer.socketId, 'EX', 60 * 60 * 6);
    }
    await this.redis.hset(`room:${roomId}:peers`, peer.socketId, JSON.stringify(peer));
    await this.redis.expire(`room:${roomId}:peers`, 60 * 60 * 6);
    return this.getRoom(roomId);
  }

  async leave(socketId: string) {
    // Quét toàn bộ phòng (chỉ vài chục → vài trăm, OK).
    const keys = await this.redis.keys('room:*:peers');
    for (const key of keys) {
      const removed = await this.redis.hdel(key, socketId);
      if (removed) {
        const roomId = key.split(':')[1];
        const hostKey = `room:${roomId}:host`;
        const host = await this.redis.get(hostKey);
        const wasHost = host === socketId;
        if (wasHost) await this.redis.del(hostKey);
        const left = await this.redis.hlen(key);
        if (left === 0) await this.redis.del(key);
        return { roomId, wasHost };
      }
    }
    return null;
  }

  async getRoom(roomId: string) {
    const peersHash = await this.redis.hgetall(`room:${roomId}:peers`);
    const hostSocketId = (await this.redis.get(`room:${roomId}:host`)) ?? undefined;
    const peers = new Map(Object.entries(peersHash).map(([k, v]) => [k, JSON.parse(v)]));
    return { id: roomId, hostSocketId, peers };
  }

  async getViewers(roomId: string) {
    const room = await this.getRoom(roomId);
    return [...room.peers.values()].filter((p: any) => p.role === 'viewer');
  }
}
```

Gateway chỉ cần đổi inject `LivestreamRedisService` thay cho `LivestreamService`, các handler giữ nguyên (chỉ thêm `await`).

### 16.5 Chạy Redis local

```bash
docker run -d --name livestream-redis -p 6379:6379 redis:7-alpine
```

`.env`:
```
REDIS_URL=redis://localhost:6379
```

### 16.6 Khuyến nghị

- **MVP / ngrok 2 máy:** giữ `LivestreamService` in-memory. Đơn giản, ít dependency.
- **Trước khi deploy production:** chuyển sang `RedisIoAdapter` + `LivestreamRedisService`. Không cần đổi code FE.
- Đừng dùng Redis như "lớp đệm cho mọi event" — pub/sub Redis chỉ cần thiết khi có > 1 instance NestJS. Single-node: in-memory nhanh hơn.

---

## 17. Tham khảo nhanh các file đi kèm

- [`WEBRTC_FUNCTIONS.md`](./WEBRTC_FUNCTIONS.md) — chi tiết từng API WebRTC trong repo `webrtc/samples`.
- [`FRONTEND_GUIDE.md`](./FRONTEND_GUIDE.md) — HTML + JS frontend hoàn chỉnh: form join, chọn thiết bị, xin quyền, chat, reconnect, indicator trạng thái.
