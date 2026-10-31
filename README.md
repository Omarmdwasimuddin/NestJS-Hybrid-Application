## NestJS Hybrid Application (REST API + TCP Microservice)

#### Create a NestJS Project
```bash
nest new hybrid-app
```
```bash
cd hybrid-app
```
---

#### Install NestJS Microservices Package
```bash
npm i @nestjs/microservices@^11
```
---

#### Configure Hybrid Application: `src/main.ts`
```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { Transport, MicroserviceOptions } from '@nestjs/microservices';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.TCP,
    options: {
      port: 8889
    }
  })

  await app.startAllMicroservices();
  await app.listen(3000);
  console.log("Hybrid App is running on port on 3000 (REST) and 8889 (TCP)");

}
bootstrap();
```
---


#### Implement REST API and TCP Message Handler: `src/app.controller.ts`
```ts
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';
import { MessagePattern } from '@nestjs/microservices';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get('hello')
  getHello(): string {
    return 'Hello from REST API!';
  }

  @MessagePattern({ cmd: 'get_status'})
  getStatus(data: any): string{
    console.log('Microservice is recieved', data);
    return 'Microservice is Active';
  }
}
```
---

#### Expected Response: OUTPUT
<img width="740" height="146" alt="image" src="https://github.com/user-attachments/assets/6f32f451-b82c-418b-9b7b-c50a70b981d6" />

---
