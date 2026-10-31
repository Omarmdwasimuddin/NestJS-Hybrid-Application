# NestJS Hybrid Application (HTTP + TCP Microservice)

> একটি NestJS application থেকে **REST API (HTTP)** এবং **TCP Microservice** — দুটোই একসাথে চালানোর গাইড।

---

## 📌 সংক্ষেপে (TL;DR)

| বিষয় | HTTP | TCP |
|---|---|---|
| Protocol | HTTP | TCP |
| Port | `3000` | `8889` |
| কাজ | REST API | Microservice Communication |
| চালু করার method | `app.listen()` | `startAllMicroservices()` |

---

## 📑 সূচিপত্র

1. [পুরো কোড](#1-পুরো-কোড)
2. [Import গুলো কী?](#2-import-গুলো-কী)
3. [Application তৈরি করা](#3-application-তৈরি-করা)
4. [connectMicroservice()](#4-connectmicroservice)
5. [startAllMicroservices()](#5-startallmicroservices)
6. [app.listen()](#6-applisten)
7. [পুরো flow একনজরে](#7-পুরো-flow-একনজরে)
8. [Hybrid App কেন ব্যবহার করব?](#8-hybrid-app-কেন-ব্যবহার-করব)
9. [মনে রাখার সহজ নিয়ম](#9-মনে-রাখার-সহজ-নিয়ম)
10. [পরবর্তী ধাপ](#10-পরবর্তী-ধাপ)

---

## 1. পুরো কোড

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { Transport, MicroserviceOptions } from '@nestjs/microservices';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // TCP Microservice Configuration
  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.TCP,
    options: {
      port: 8889,
    },
  });

  // Start TCP Microservice
  await app.startAllMicroservices();

  // Start HTTP Server
  await app.listen(3000);

  console.log('Hybrid App is running on port 3000 (REST) and 8889 (TCP)');
}

bootstrap();
```

এখন এক এক করে প্রতিটি অংশ বুঝি 👇

---

## 2. Import গুলো কী?

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { Transport, MicroserviceOptions } from '@nestjs/microservices';
```

### `NestFactory`
NestJS application **তৈরি** করার জন্য ব্যবহার হয়।

```ts
const app = await NestFactory.create(AppModule);
```

### `Transport`
Microservice কোন **Transport Protocol** ব্যবহার করবে সেটি ঠিক করে।

| Transport | ব্যবহার |
|---|---|
| `Transport.TCP` | ✅ আমাদের কোডে এটাই ব্যবহার হয়েছে |
| `Transport.REDIS` | Redis pub/sub |
| `Transport.RMQ` | RabbitMQ |
| `Transport.KAFKA` | Kafka |
| `Transport.GRPC` | gRPC |
| `Transport.NATS` | NATS |

### `MicroserviceOptions`
এটি একটি **TypeScript Interface**, যা Microservice-এর configuration কেমন হবে তা define করে। এখানে **Generic Type** হিসেবে ব্যবহার হয়েছে:

```ts
app.connectMicroservice<MicroserviceOptions>({ ... })
```

এতে TypeScript বুঝতে পারে — "এখানে Microservice-এর config দেওয়া হচ্ছে" — এবং ভুল হলে error দেখায়।

---

## 3. Application তৈরি করা

```ts
const app = await NestFactory.create(AppModule);
```

| অংশ | মানে |
|---|---|
| `NestFactory` | Application তৈরির Factory |
| `AppModule` | Application-এর **Root Module** |
| `app` | তৈরি হওয়া Application-এর **Instance** |

এই `app` দিয়ে REST API চালানো যায়। যেমন:

```ts
@Get('users')
getUsers() {
  return ['Wasim', 'Rahim'];
}
```

```http
GET http://localhost:3000/users
```

> ⚠️ **খেয়াল রাখো:** এই পর্যায়ে এখনো TCP Microservice চালু হয়নি।

---

## 4. `connectMicroservice()`

```ts
app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.TCP,
  options: {
    port: 8889,
  },
});
```

এটি Hybrid Application-এর **সবচেয়ে গুরুত্বপূর্ণ অংশ**। বিদ্যমান HTTP application-এর সঙ্গে একটি Microservice **যুক্ত** করে।

### `transport: Transport.TCP`
NestJS-কে বলছি — Microservice communication-এ **TCP protocol** ব্যবহার করো। ফলে অন্য NestJS application TCP দিয়ে আমাদের service-এর সঙ্গে কথা বলতে পারবে।

### `options.port: 8889`
TCP Microservice কোন port-এ **listen** করবে।

```
TCP Microservice → localhost:8889
```

> ⚠️ **মনে রাখো:** `connectMicroservice()` শুধু **configuration যুক্ত** করে। এটি Microservice **চালু করে না**।

---

## 5. `startAllMicroservices()`

```ts
await app.startAllMicroservices();
```

আগে configuration দেওয়া হয়েছে। এখন এই line সেই Microservice-গুলোকে **start** করে।

```
connectMicroservice()
        │
        ▼
Microservice Configuration (শুধু সেটআপ)
        │
        ▼
startAllMicroservices()
        │
        ▼
TCP Server Started
        │
        ▼
Listening on Port 8889 ✅
```

এই line execute হওয়ার পর TCP Microservice request নেওয়ার জন্য প্রস্তুত।

---

## 6. `app.listen()`

```ts
await app.listen(3000);
```

এটি **HTTP Server** চালু করে। এখন application port `3000`-এ HTTP request নেবে।

**উদাহরণ:**

```ts
@Controller('users')
export class UserController {
  @Get()
  getUsers() {
    return ['Wasim', 'Rahim'];
  }
}
```

**Request:**
```http
GET http://localhost:3000/users
```

**Response:**
```json
["Wasim", "Rahim"]
```

---

## 7. পুরো flow একনজরে

```
              ┌─────────────────────────────────┐
              │   একটি NestJS Application (app) │
              └───────────────┬─────────────────┘
                              │
            ┌─────────────────┴─────────────────┐
            ▼                                   ▼
   ┌─────────────────┐                 ┌─────────────────┐
   │  HTTP Server    │                 │  TCP Server     │
   │  Port: 3000     │                 │  Port: 8889     │
   │  REST API       │                 │  Microservice   │
   └────────▲────────┘                 └────────▲────────┘
            │                                   │
      app.listen()                  startAllMicroservices()
```

দুটি server **একই NestJS application**-এর সঙ্গে যুক্ত।

---

## 8. Hybrid App কেন ব্যবহার করব?

ধরো, তুমি একটা **E-commerce application** বানাচ্ছো, যেখানে আছে:

- User Service
- Order Service
- Payment Service
- Inventory Service

```
   ┌────────────────────┐
   │ Client / Frontend  │
   └─────────┬──────────┘
             │  HTTP Request
             ▼
   ┌──────────────────────────────┐
   │   NestJS Hybrid Application  │
   │                              │
   │   REST API        : 3000     │
   │   TCP Microservice: 8889     │
   └─────────┬────────────────────┘
             │  TCP Communication
             ▼
   ┌──────────────────────────────┐
   │   Other Microservices        │
   │   Order / Payment / Inventory│
   └──────────────────────────────┘
```

- **Frontend** → HTTP দিয়ে request পাঠায়
- **Hybrid App** → অন্য Microservice-এর সঙ্গে TCP দিয়ে communicate করে

একটি app-ই দুই কাজ করতে পারে — আলাদা করে দুটো project বানাতে হয় না।

---

## 9. মনে রাখার সহজ নিয়ম

| Method / Option | কাজ |
|---|---|
| `NestFactory.create()` | NestJS HTTP Application তৈরি করে |
| `connectMicroservice()` | HTTP App-এর সঙ্গে Microservice **যুক্ত** করে |
| `Transport.TCP` | TCP Protocol বেছে নেয় |
| `options.port` | Microservice-এর Port ঠিক করে |
| `startAllMicroservices()` | Microservice **চালু** করে |
| `app.listen()` | HTTP Server **চালু** করে |

### 🧠 এক লাইনের মুখস্থ সূত্র

> **Create → Connect → Start Microservice → Listen HTTP**

---

## 10. পরবর্তী ধাপ

শুধু TCP Microservice চালু করলেই হবে না। TCP দিয়ে **request কীভাবে গ্রহণ করবে** এবং **response কীভাবে পাঠাবে** — সেটাও জানতে হবে।

👉 এরপর শেখো: **`@MessagePattern()`**

সংক্ষিপ্ত ধারণা:

```ts
@MessagePattern({ cmd: 'get_users' })
getUsers() {
  return ['Wasim', 'Rahim'];
}
```

এখানে অন্য কোনো service `{ cmd: 'get_users' }` pattern সহ message পাঠালে এই method সেটি handle করবে।
