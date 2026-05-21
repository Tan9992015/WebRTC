# Hướng Dẫn Socket.IO Với NestJS Kết Hợp REST API

## Mục Lục

1. [Cài Đặt](#cài-đặt)
2. [Cấu Trúc Thư Mục](#cấu-trúc-thư-mục)
3. [Gateway - Trung Tâm Socket](#gateway---trung-tâm-socket)
4. [Các Decorator Quan Trọng](#các-decorator-quan-trọng)
5. [Sự Kiện Cơ Bản](#sự-kiện-cơ-bản)
6. [Kết Hợp Service & REST API](#kết-hợp-service--rest-api)
7. [Rooms & Broadcast](#rooms--broadcast)
8. [Ví Dụ: Chat App](#ví-dụ-chat-app)
9. [Ví Dụ: Thông Báo Real-time](#ví-dụ-thông-báo-real-time)
10. [Xác Thực JWT Với Socket](#xác-thực-jwt-với-socket)
11. [Bảng Tóm Tắt](#bảng-tóm-tắt)

---

## Cài Đặt

```bash
# Tạo project NestJS mới
npm i -g @nestjs/cli
nest new my-app

# Cài các package cần thiết
npm install @nestjs/websockets @nestjs/platform-socket.io socket.io
npm install @nestjs/jwt @nestjs/passport passport passport-jwt  # nếu dùng JWT

# Client test
npm install socket.io-client
```

---

## Cấu Trúc Thư Mục

```
src/
├── app.module.ts
├── main.ts
├── chat/
│   ├── chat.module.ts
│   ├── chat.gateway.ts      ← Xử lý Socket.IO
│   ├── chat.service.ts      ← Logic nghiệp vụ
│   ├── chat.controller.ts   ← REST API
│   └── dto/
│       └── send-message.dto.ts
└── notification/
    ├── notification.module.ts
    ├── notification.gateway.ts
    ├── notification.service.ts
    └── notification.controller.ts
```

---

## Gateway - Trung Tâm Socket

Gateway là nơi xử lý tất cả sự kiện Socket.IO trong NestJS.

```ts
// chat/chat.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
  OnGatewayInit,
  OnGatewayConnection,
  OnGatewayDisconnect,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';

@WebSocketGateway({
  cors: { origin: '*' },   // cấu hình CORS
  namespace: '/chat',       // namespace (tùy chọn)
  port: 3001,               // cổng riêng (tùy chọn, mặc định dùng cổng HTTP)
})
export class ChatGateway
  implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect
{
  @WebSocketServer()
  server: Server; // Đối tượng server Socket.IO

  // Gọi sau khi Gateway khởi tạo xong
  afterInit(server: Server) {
    console.log('WebSocket Gateway khởi tạo thành công');
  }

  // Gọi khi có client kết nối
  handleConnection(client: Socket) {
    console.log(`Client kết nối: ${client.id}`);
  }

  // Gọi khi client ngắt kết nối
  handleDisconnect(client: Socket) {
    console.log(`Client ngắt kết nối: ${client.id}`);
  }
}
```

---

## Các Decorator Quan Trọng

| Decorator | Vị trí | Mô tả |
|-----------|--------|-------|
| `@WebSocketGateway()` | Class | Đánh dấu class là Gateway |
| `@WebSocketServer()` | Property | Inject đối tượng `Server` của Socket.IO |
| `@SubscribeMessage('event')` | Method | Lắng nghe sự kiện từ client |
| `@MessageBody()` | Parameter | Lấy dữ liệu từ sự kiện |
| `@ConnectedSocket()` | Parameter | Lấy đối tượng Socket của client gửi |

---

## Sự Kiện Cơ Bản

### Lắng nghe sự kiện từ client

```ts
@SubscribeMessage('gui_tin_nhan')
handleMessage(
  @MessageBody() data: { noi_dung: string },
  @ConnectedSocket() client: Socket,
): void {
  console.log(`Nhận từ ${client.id}:`, data.noi_dung);

  // Gửi lại cho đúng client đó
  client.emit('phan_hoi', { trang_thai: 'Đã nhận' });

  // Gửi cho tất cả (kể cả người gửi)
  this.server.emit('tin_moi', { noi_dung: data.noi_dung });

  // Gửi cho tất cả trừ người gửi
  client.broadcast.emit('tin_moi', { noi_dung: data.noi_dung });
}
```

### Trả về dữ liệu (Acknowledgement)

```ts
@SubscribeMessage('dat_hang')
handleOrder(
  @MessageBody() data: { san_pham: string; so_luong: number },
): { success: boolean; ma_don: string } {
  // Giá trị return sẽ được gửi về như acknowledgement callback
  return { success: true, ma_don: `DH-${Date.now()}` };
}
```

```js
// Client nhận acknowledgement
socket.emit('dat_hang', { san_pham: 'Laptop', so_luong: 1 }, (res) => {
  console.log('Mã đơn:', res.ma_don);
});
```

---

## Kết Hợp Service & REST API

Service dùng chung giữa Gateway (Socket) và Controller (HTTP).

### Service

```ts
// chat/chat.service.ts
import { Injectable } from '@nestjs/common';

export interface TinNhan {
  id: number;
  nguoi_gui: string;
  noi_dung: string;
  thoi_gian: string;
}

@Injectable()
export class ChatService {
  private tinNhanDB: TinNhan[] = [];

  luuTinNhan(nguoi_gui: string, noi_dung: string): TinNhan {
    const tin: TinNhan = {
      id: Date.now(),
      nguoi_gui,
      noi_dung,
      thoi_gian: new Date().toISOString(),
    };
    this.tinNhanDB.push(tin);
    return tin;
  }

  layLichSu(limit = 50): TinNhan[] {
    return this.tinNhanDB.slice(-limit);
  }
}
```

### Gateway dùng Service

```ts
// chat/chat.gateway.ts
import { WebSocketGateway, WebSocketServer, SubscribeMessage, MessageBody, ConnectedSocket } from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { ChatService } from './chat.service';

@WebSocketGateway({ cors: { origin: '*' } })
export class ChatGateway {
  @WebSocketServer() server: Server;

  constructor(private readonly chatService: ChatService) {}

  @SubscribeMessage('gui_tin')
  handleSendMessage(
    @MessageBody() body: { nguoi_gui: string; noi_dung: string },
    @ConnectedSocket() client: Socket,
  ) {
    const tin = this.chatService.luuTinNhan(body.nguoi_gui, body.noi_dung);

    // Phát cho tất cả client
    this.server.emit('tin_moi', tin);

    return { da_gui: true, id: tin.id };
  }
}
```

### REST API Controller

```ts
// chat/chat.controller.ts
import { Controller, Get, Post, Body } from '@nestjs/common';
import { ChatService } from './chat.service';
import { ChatGateway } from './chat.gateway';

@Controller('api/chat')
export class ChatController {
  constructor(
    private readonly chatService: ChatService,
    private readonly chatGateway: ChatGateway,
  ) {}

  // GET /api/chat/lich-su
  @Get('lich-su')
  getLichSu() {
    return { success: true, data: this.chatService.layLichSu() };
  }

  // POST /api/chat/gui  ← Gửi tin qua REST API, phát socket real-time
  @Post('gui')
  guiTinQua(@Body() body: { nguoi_gui: string; noi_dung: string }) {
    const tin = this.chatService.luuTinNhan(body.nguoi_gui, body.noi_dung);

    // Phát socket đến tất cả client đang kết nối
    this.chatGateway.server.emit('tin_moi', tin);

    return { success: true, data: tin };
  }
}
```

### Module

```ts
// chat/chat.module.ts
import { Module } from '@nestjs/common';
import { ChatGateway } from './chat.gateway';
import { ChatService } from './chat.service';
import { ChatController } from './chat.controller';

@Module({
  providers: [ChatGateway, ChatService],
  controllers: [ChatController],
})
export class ChatModule {}
```

```ts
// app.module.ts
import { Module } from '@nestjs/common';
import { ChatModule } from './chat/chat.module';

@Module({
  imports: [ChatModule],
})
export class AppModule {}
```

---

## Rooms & Broadcast

```ts
@WebSocketGateway({ cors: { origin: '*' } })
export class ChatGateway {
  @WebSocketServer() server: Server;

  // Client tham gia phòng
  @SubscribeMessage('tham_gia_phong')
  handleJoinRoom(
    @MessageBody() tenPhong: string,
    @ConnectedSocket() client: Socket,
  ) {
    client.join(tenPhong);

    // Thông báo những người trong phòng
    this.server.to(tenPhong).emit('thong_bao', `${client.id} đã vào phòng`);

    return { da_vao: true, phong: tenPhong };
  }

  // Client rời phòng
  @SubscribeMessage('roi_phong')
  handleLeaveRoom(
    @MessageBody() tenPhong: string,
    @ConnectedSocket() client: Socket,
  ) {
    client.leave(tenPhong);
    this.server.to(tenPhong).emit('thong_bao', `${client.id} đã rời phòng`);
  }

  // Gửi tin nhắn vào phòng
  @SubscribeMessage('tin_nhan_phong')
  handleRoomMessage(
    @MessageBody() body: { phong: string; noi_dung: string },
    @ConnectedSocket() client: Socket,
  ) {
    // Gửi cho cả phòng kể cả người gửi
    this.server.to(body.phong).emit('tin_phong', {
      nguoi_gui: client.id,
      noi_dung: body.noi_dung,
    });

    // Hoặc trừ người gửi:
    // client.to(body.phong).emit('tin_phong', { ... });
  }
}
```

---

## Ví Dụ: Chat App

### DTO

```ts
// chat/dto/send-message.dto.ts
export class SendMessageDto {
  nguoi_gui: string;
  noi_dung: string;
  phong?: string;
}
```

### Gateway hoàn chỉnh

```ts
// chat/chat.gateway.ts
import {
  WebSocketGateway, WebSocketServer, SubscribeMessage,
  MessageBody, ConnectedSocket, OnGatewayConnection, OnGatewayDisconnect,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { ChatService } from './chat.service';
import { SendMessageDto } from './dto/send-message.dto';

@WebSocketGateway({ cors: { origin: '*' } })
export class ChatGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer() server: Server;

  private nguoiDungOnline = new Map<string, string>(); // socketId → tên

  constructor(private readonly chatService: ChatService) {}

  handleConnection(client: Socket) {
    console.log(`Kết nối mới: ${client.id}`);
  }

  handleDisconnect(client: Socket) {
    const ten = this.nguoiDungOnline.get(client.id);
    this.nguoiDungOnline.delete(client.id);

    if (ten) {
      this.server.emit('cap_nhat_online', [...this.nguoiDungOnline.values()]);
      this.server.emit('thong_bao', `${ten} đã rời đi`);
    }
  }

  @SubscribeMessage('dang_nhap')
  handleLogin(
    @MessageBody() ten: string,
    @ConnectedSocket() client: Socket,
  ) {
    this.nguoiDungOnline.set(client.id, ten);
    this.server.emit('cap_nhat_online', [...this.nguoiDungOnline.values()]);
    client.broadcast.emit('thong_bao', `${ten} đã tham gia`);
    return { success: true };
  }

  @SubscribeMessage('gui_tin')
  handleMessage(
    @MessageBody() dto: SendMessageDto,
    @ConnectedSocket() client: Socket,
  ) {
    const ten = this.nguoiDungOnline.get(client.id) ?? 'Ẩn danh';
    const tin = this.chatService.luuTinNhan(ten, dto.noi_dung);

    if (dto.phong) {
      this.server.to(dto.phong).emit('tin_moi', tin);
    } else {
      this.server.emit('tin_moi', tin);
    }

    return { da_gui: true, id: tin.id };
  }

  @SubscribeMessage('dang_nhap_text')
  handleTyping(
    @ConnectedSocket() client: Socket,
  ) {
    const ten = this.nguoiDungOnline.get(client.id);
    client.broadcast.emit('nguoi_dang_nhap', ten);
  }

  @SubscribeMessage('tham_gia_phong')
  handleJoin(
    @MessageBody() phong: string,
    @ConnectedSocket() client: Socket,
  ) {
    client.join(phong);
    this.server.to(phong).emit('thong_bao', `${client.id} đã vào ${phong}`);
    return { da_vao: true };
  }
}
```

---

## Ví Dụ: Thông Báo Real-time

### Notification Service

```ts
// notification/notification.service.ts
import { Injectable } from '@nestjs/common';

export interface ThongBao {
  id: number;
  loai: 'don_hang' | 'tin_nhan' | 'canh_bao';
  noi_dung: string;
  doc_roi: boolean;
  thoi_gian: string;
}

@Injectable()
export class NotificationService {
  private db: Map<string, ThongBao[]> = new Map(); // userId → thongbaos

  taoThongBao(userId: string, loai: ThongBao['loai'], noi_dung: string): ThongBao {
    const tb: ThongBao = {
      id: Date.now(),
      loai,
      noi_dung,
      doc_roi: false,
      thoi_gian: new Date().toISOString(),
    };

    const ds = this.db.get(userId) ?? [];
    ds.push(tb);
    this.db.set(userId, ds);
    return tb;
  }

  layThongBao(userId: string): ThongBao[] {
    return this.db.get(userId) ?? [];
  }

  danhDauDaDoc(userId: string, thongBaoId: number): void {
    const ds = this.db.get(userId) ?? [];
    const tb = ds.find((t) => t.id === thongBaoId);
    if (tb) tb.doc_roi = true;
  }
}
```

### Notification Gateway

```ts
// notification/notification.gateway.ts
import {
  WebSocketGateway, WebSocketServer, SubscribeMessage,
  MessageBody, ConnectedSocket,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { NotificationService } from './notification.service';

@WebSocketGateway({ cors: { origin: '*' }, namespace: '/notification' })
export class NotificationGateway {
  @WebSocketServer() server: Server;

  constructor(private readonly notificationService: NotificationService) {}

  // Client đăng ký nhận thông báo theo userId
  @SubscribeMessage('dang_ky')
  handleSubscribe(
    @MessageBody() userId: string,
    @ConnectedSocket() client: Socket,
  ) {
    client.join(`user_${userId}`);
    const daCoTb = this.notificationService.layThongBao(userId);
    client.emit('thong_bao_cu', daCoTb);
    return { da_dang_ky: true };
  }

  // Client xác nhận đã đọc
  @SubscribeMessage('da_doc')
  handleRead(
    @MessageBody() body: { userId: string; thongBaoId: number },
    @ConnectedSocket() client: Socket,
  ) {
    this.notificationService.danhDauDaDoc(body.userId, body.thongBaoId);
    client.emit('xac_nhan_da_doc', { id: body.thongBaoId });
  }

  // Hàm public để Controller/Service gọi để đẩy thông báo
  guiThongBao(userId: string, loai: string, noi_dung: string) {
    const tb = this.notificationService.taoThongBao(userId, loai as any, noi_dung);
    this.server.to(`user_${userId}`).emit('thong_bao_moi', tb);
    return tb;
  }
}
```

### Notification Controller

```ts
// notification/notification.controller.ts
import { Controller, Post, Body, Get, Param } from '@nestjs/common';
import { NotificationGateway } from './notification.gateway';
import { NotificationService } from './notification.service';

@Controller('api/notification')
export class NotificationController {
  constructor(
    private readonly notificationGateway: NotificationGateway,
    private readonly notificationService: NotificationService,
  ) {}

  // POST /api/notification/gui  ← Hệ thống khác gọi để đẩy thông báo
  @Post('gui')
  guiThongBao(@Body() body: { userId: string; loai: string; noi_dung: string }) {
    const tb = this.notificationGateway.guiThongBao(
      body.userId,
      body.loai,
      body.noi_dung,
    );
    return { success: true, data: tb };
  }

  // GET /api/notification/:userId
  @Get(':userId')
  layThongBao(@Param('userId') userId: string) {
    return { success: true, data: this.notificationService.layThongBao(userId) };
  }
}
```

---

## Xác Thực JWT Với Socket

```ts
// auth/ws-jwt.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { Socket } from 'socket.io';

@Injectable()
export class WsJwtGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  canActivate(context: ExecutionContext): boolean {
    const client: Socket = context.switchToWs().getClient();
    const token =
      client.handshake.auth?.token ||
      client.handshake.headers?.authorization?.replace('Bearer ', '');

    try {
      const payload = this.jwtService.verify(token);
      client.data.user = payload; // Gắn user vào socket để dùng sau
      return true;
    } catch {
      client.disconnect();
      return false;
    }
  }
}
```

```ts
// Dùng guard trong Gateway
import { UseGuards } from '@nestjs/common';
import { WsJwtGuard } from '../auth/ws-jwt.guard';

@WebSocketGateway({ cors: { origin: '*' } })
export class ChatGateway {
  @WebSocketServer() server: Server;

  @UseGuards(WsJwtGuard)
  @SubscribeMessage('gui_tin')
  handleMessage(
    @MessageBody() body: { noi_dung: string },
    @ConnectedSocket() client: Socket,
  ) {
    const user = client.data.user; // payload từ JWT
    console.log(`Tin từ ${user.username}:`, body.noi_dung);
    this.server.emit('tin_moi', { nguoi_gui: user.username, ...body });
  }
}
```

```js
// Client gửi token khi kết nối
const socket = io('http://localhost:3000', {
  auth: { token: 'Bearer your.jwt.token' },
});
```

---

## Bảng Tóm Tắt

### Decorator

| Decorator | Mô tả |
|-----------|-------|
| `@WebSocketGateway(options)` | Khai báo Gateway, cấu hình cors/namespace/port |
| `@WebSocketServer()` | Inject `Server` instance của Socket.IO |
| `@SubscribeMessage('event')` | Đăng ký lắng nghe sự kiện |
| `@MessageBody()` | Lấy payload từ sự kiện |
| `@ConnectedSocket()` | Lấy socket của client gửi |

### Cách gửi sự kiện (Server → Client)

| Cú pháp | Mô tả |
|---------|-------|
| `client.emit(event, data)` | Gửi cho đúng client đó |
| `this.server.emit(event, data)` | Gửi cho tất cả |
| `client.broadcast.emit(event, data)` | Gửi tất cả trừ client đó |
| `this.server.to(room).emit(event, data)` | Gửi cho cả room |
| `client.to(room).emit(event, data)` | Gửi room trừ client đó |
| `return data` trong handler | Acknowledgement về client |

### Lifecycle Hooks

| Interface | Method | Khi nào gọi |
|-----------|--------|-------------|
| `OnGatewayInit` | `afterInit(server)` | Sau khi Gateway khởi tạo |
| `OnGatewayConnection` | `handleConnection(client)` | Client kết nối |
| `OnGatewayDisconnect` | `handleDisconnect(client)` | Client ngắt kết nối |
