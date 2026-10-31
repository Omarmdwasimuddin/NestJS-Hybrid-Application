# NestJS `@MessagePattern()` — TCP Microservice-এ Request গ্রহণ

> আগের Hybrid Application-এ আমরা Port `8889`-এ TCP Server চালু করেছিলাম। এখন দেখবো সেই TCP Server-এ আসা **Message** কীভাবে Handle করতে হয়।

---

## 📌 সংক্ষেপে (TL;DR)

| Decorator | Client Method | ধরন |
|---|---|---|
| `@MessagePattern()` | `client.send()` | **Request–Response** (উত্তর ফেরত আসে) |
| `@EventPattern()` | `client.emit()` | **Event-based** (উত্তরের অপেক্ষা নেই) |

---

## 📑 সূচিপত্র

1. [পুরো কোড](#1-পুরো-কোড)
2. [MessagePattern Import](#2-messagepattern-import)
3. [@MessagePattern() কী?](#3-messagepattern-কী)
4. [Method ও Payload](#4-method-ও-payload)
5. [console.log()](#5-consolelog)
6. [return](#6-return)
7. [পুরো Communication Flow](#7-পুরো-communication-flow)
8. [REST API vs MessagePattern](#8-rest-api-vs-messagepattern)
9. [send() এবং emit()](#9-send-এবং-emit)
10. [মনে রাখার সহজ নিয়ম](#10-মনে-রাখার-সহজ-নিয়ম)

---

## 1. পুরো কোড

```ts
import { Controller, Get } from '@nestjs/common';
import { MessagePattern } from '@nestjs/microservices';

@Controller()
export class AppController {

  // HTTP (REST API)
  @Get('hello')
  getHello(): string {
    return 'Hello from REST API!';
  }

  // TCP Microservice
  @MessagePattern({ cmd: 'get_status' })
  getStatus(data: any): string {
    console.log('Microservice received:', data);
    return 'Microservice is Active';
  }
}
```

একই Controller-এ **HTTP** এবং **TCP** দুই ধরনের communication handle হচ্ছে। এবার অংশগুলো আলাদা করে বুঝি 👇

---

## 2. MessagePattern Import

```ts
import { MessagePattern } from '@nestjs/microservices';
```

`MessagePattern` হলো NestJS Microservices-এর একটি **Decorator**।

**কাজ:** কোন Pattern-এর Message এলে কোন Method চলবে, সেটা Microservice-কে জানানো।

**উদাহরণ:** অন্য একটি Microservice এই Message পাঠালো:

```ts
{ cmd: 'get_status' }
```

NestJS তখন খুঁজবে — কোন Method-এর `@MessagePattern()`-এ এই Pattern দেওয়া আছে। **Match করলে সেই Method Execute হবে।**

---

## 3. `@MessagePattern()` কী?

```ts
@MessagePattern({ cmd: 'get_status' })
```

এখানে `{ cmd: 'get_status' }` হলো **Message Pattern** (Message Identifier)।

কোনো TCP Client এই Pattern-এর Message পাঠালে নিচের `getStatus()` Method চলবে।

### 🧭 এটাকে Routing System-এর মতো ভাবো

REST API-তে:

```ts
@Get('hello')   // URL match করলে getHello() চলে
```

Microservice-এ:

```ts
@MessagePattern({ cmd: 'get_status' })   // Pattern match করলে getStatus() চলে
```

| | `@Get()` | `@MessagePattern()` |
|---|---|---|
| কী দিয়ে match হয় | URL | Message Pattern |
| কোন request handle করে | HTTP | Microservice Message |

---

## 4. Method ও Payload

```ts
getStatus(data: any): string {
```

| অংশ | মানে |
|---|---|
| `getStatus` | Method-এর নাম |
| `data: any` | Client থেকে আসা **Payload** গ্রহণ করে |
| `: string` | Method একটি `string` return করবে |

### Pattern আর Payload আলাদা জিনিস

Client `send()`-এ **দুটো জিনিস** পাঠায়:

```ts
client.send(
  { cmd: 'get_status' },            // 1️⃣ Pattern  → কোন method চলবে
  { message: 'Hello from Client' }  // 2️⃣ Payload  → method-এর data
);
```

তাহলে `getStatus(data)`-এর `data`-তে আসবে শুধু **Payload**:

```ts
{ message: 'Hello from Client' }
```

- `cmd` → Pattern-এর অংশ (routing-এর জন্য)
- `message` → Payload (আসল data)

> ⚠️ **Production tip:** `data: any` শুধু শেখার জন্য ঠিক আছে। Production-এ নির্দিষ্ট **Type বা DTO** ব্যবহার করা ভালো।

---

## 5. `console.log()`

```ts
console.log('Microservice received:', data);
```

Client থেকে আসা data Terminal-এ print করে। এটি **Debug**-এর জন্য — Microservice request পেয়েছে কিনা এবং কী data পেয়েছে সেটা দেখা যায়।

**Console output:**

```
Microservice received: { message: 'Hello from Client' }
```

---

## 6. `return`

```ts
return 'Microservice is Active';
```

এই লাইন **Client-কে Response পাঠায়**।

Client `get_status` Pattern-এর Message পাঠালে উত্তর পাবে:

```
Microservice is Active
```

> 💡 `@MessagePattern()` মূলত **Request–Response** communication-এর জন্য।

---

## 7. পুরো Communication Flow

```
┌──────────────────────────────┐
│  TCP Client                  │
│  (অন্য NestJS Application)   │
└──────────────┬───────────────┘
               │  send({ cmd: 'get_status' }, payload)
               ▼
        TCP  →  Port 8889
               │
               ▼
┌──────────────────────────────┐
│  NestJS Microservice         │
│                              │
│  AppController               │
│    │                         │
│    ▼                         │
│  MessagePattern Matching     │
│  { cmd: 'get_status' } ✔     │
│    │                         │
│    ▼                         │
│  getStatus(data)             │
│    ├─ console.log(data)      │
│    └─ return 'Microservice   │
│         is Active'           │
└──────────────┬───────────────┘
               │  TCP Response
               ▼
┌──────────────────────────────┐
│  TCP Client                  │
│  Response: "Microservice     │
│  is Active"                  │
└──────────────────────────────┘
```

---

## 8. REST API vs MessagePattern

একই Controller-এ দুটো পাশাপাশি:

```ts
// HTTP Request handle করে
@Get('hello')
getHello(): string {
  return 'Hello from REST API!';
}
```

```ts
// TCP Microservice Message handle করে
@MessagePattern({ cmd: 'get_status' })
getStatus(data: any): string {
  console.log('Microservice received:', data);
  return 'Microservice is Active';
}
```

| বিষয় | `@Get()` | `@MessagePattern()` |
|---|---|---|
| Communication | HTTP | Microservice Transport |
| Request | HTTP GET | Message |
| Routing | URL | Message Pattern |
| Response | HTTP Response | Microservice Response |
| Client | Browser / Frontend | Microservice Client |

---

## 9. `send()` এবং `emit()`

Client থেকে Message পাঠানোর দুটি Method:

| Method | কাজ | Decorator |
|---|---|---|
| `send()` | **Request–Response** | `@MessagePattern()` |
| `emit()` | **Event-based** (response লাগে না) | `@EventPattern()` |

### `send()` উদাহরণ

```ts
import { firstValueFrom } from 'rxjs';

const response = await firstValueFrom(
  client.send(
    { cmd: 'get_status' },
    { message: 'Hello from Client' }
  )
);

console.log(response);
```

**Output:**

```
Microservice is Active
```

**কী ঘটছে:**

1. `client.send()` → Pattern + Payload পাঠায়
2. Microservice-এর `getStatus()` চলে
3. Return করা string Client-এর কাছে ফিরে আসে

---

## 10. মনে রাখার সহজ নিয়ম

```
@MessagePattern()  +  send()   =  Request–Response   (উত্তর চাই)
@EventPattern()    +  emit()   =  Event-based        (শুধু জানিয়ে দাও)
```

| মনে রাখো | মানে |
|---|---|
| Pattern | কোন method চলবে সেটা ঠিক করে |
| Payload | method-এর ভেতরে আসা আসল data |
| `return` | Client-এর কাছে Response হয়ে যায় |

এই দুটো ভালোভাবে বুঝলে NestJS Microservices-এর **TCP Communication**-এর মূল ভিত্তি পরিষ্কার হয়ে যাবে। 🚀
