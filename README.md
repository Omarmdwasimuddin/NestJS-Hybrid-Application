## NestJS-Hybrid-Application

####
```bash
nest new hybrid-app
```
```bash
cd hybrid-app
```
---

####
```bash
npm i @nestjs/microservices@^11
```
---

#### `src/main.ts`
```bash
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
