# Allegro Manager — Build Instructions

> Step-by-step guide to scaffold the **Allegro Manager** monorepo from scratch using NestJS, Next.js, Prisma, and Turborepo.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Monorepo Setup (Turborepo + pnpm)](#2-monorepo-setup)
3. [Backend Setup (NestJS)](#3-backend-setup-nestjs)
4. [Prisma Schema & Database](#4-prisma-schema--database)
5. [Frontend Setup (Next.js)](#5-frontend-setup-nextjs)
6. [Shared Packages](#6-shared-packages)
7. [Docker & Local Development](#7-docker--local-development)
8. [Environment Variables](#8-environment-variables)
9. [Development Workflow](#9-development-workflow)
10. [Deployment Notes](#10-deployment-notes)

---

## 1. Prerequisites

### Required Software

| Tool | Version | Purpose |
|---|---|---|
| **Node.js** | 20 LTS or later | Runtime |
| **pnpm** | 9.x+ | Package manager (required by Turborepo) |
| **Docker** & **Docker Compose** | Latest | Local PostgreSQL + Redis |
| **Git** | Latest | Version control |

### Required Accounts

| Service | What You Need | Where to Get It |
|---|---|---|
| **Allegro Developer** | Client ID + Client Secret | [apps.developer.allegro.pl](https://apps.developer.allegro.pl/) |
| **Allegro Sandbox** | Sandbox account for testing | [allegro.pl.allegrosandbox.pl](https://allegro.pl.allegrosandbox.pl/) |
| **Odoo** | Instance URL + API key (Custom plan required) | Your Odoo instance admin panel |

### Install pnpm (if not installed)

```bash
corepack enable
corepack prepare pnpm@latest --activate
```

---

## 2. Monorepo Setup

### 2.1 Create the project

```bash
mkdir allegro-manager && cd allegro-manager
git init
pnpm init
```

### 2.2 Configure pnpm workspace

Create `pnpm-workspace.yaml`:

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

### 2.3 Install Turborepo

```bash
pnpm add -D turbo
```

### 2.4 Create `turbo.json`

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "!.next/cache/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {
      "dependsOn": ["^build"]
    },
    "test": {
      "dependsOn": ["build"]
    },
    "db:generate": {
      "cache": false
    },
    "db:migrate": {
      "cache": false
    }
  }
}
```

### 2.5 Create directory structure

```bash
mkdir -p apps/api apps/web packages/shared packages/eslint-config packages/tsconfig
```

### 2.6 Root `package.json` scripts

Update the root `package.json`:

```json
{
  "name": "allegro-manager",
  "private": true,
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "lint": "turbo lint",
    "test": "turbo test",
    "db:generate": "turbo db:generate",
    "db:migrate": "turbo db:migrate",
    "docker:up": "docker compose up -d",
    "docker:down": "docker compose down"
  },
  "devDependencies": {
    "turbo": "^2"
  },
  "packageManager": "pnpm@9.15.0"
}
```

### 2.7 Shared TypeScript config

Create `packages/tsconfig/base.json`:

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "exclude": ["node_modules"]
}
```

Create `packages/tsconfig/nestjs.json`:

```json
{
  "extends": "./base.json",
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "node",
    "target": "ES2021",
    "outDir": "./dist",
    "declaration": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true
  }
}
```

Create `packages/tsconfig/nextjs.json`:

```json
{
  "extends": "./base.json",
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["DOM", "DOM.Iterable", "ES2022"],
    "module": "ESNext",
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./src/*"] }
  }
}
```

Create `packages/tsconfig/package.json`:

```json
{
  "name": "@allegro-manager/tsconfig",
  "version": "0.0.0",
  "private": true,
  "files": ["base.json", "nestjs.json", "nextjs.json"]
}
```

---

## 3. Backend Setup (NestJS)

### 3.1 Scaffold NestJS inside `apps/api`

```bash
cd apps/api
pnpm init
```

### 3.2 Install NestJS dependencies

```bash
cd apps/api

pnpm add @nestjs/core @nestjs/common @nestjs/platform-express \
  @nestjs/config @nestjs/jwt @nestjs/passport @nestjs/schedule \
  @nestjs/bull @nestjs/terminus @nestjs/swagger \
  @prisma/client \
  passport passport-jwt passport-oauth2 \
  bull ioredis \
  axios class-validator class-transformer \
  reflect-metadata rxjs

pnpm add -D @nestjs/cli @nestjs/schematics @nestjs/testing \
  prisma typescript @types/node @types/express \
  @types/passport-jwt @types/bull \
  ts-node ts-loader \
  jest @types/jest ts-jest \
  @allegro-manager/tsconfig
```

### 3.3 `apps/api/tsconfig.json`

```json
{
  "extends": "@allegro-manager/tsconfig/nestjs.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "test"]
}
```

### 3.4 `apps/api/package.json`

```json
{
  "name": "@allegro-manager/api",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "dev": "nest start --watch",
    "build": "nest build",
    "start": "node dist/main",
    "start:prod": "node dist/main",
    "lint": "eslint \"src/**/*.ts\"",
    "test": "jest",
    "test:e2e": "jest --config ./test/jest-e2e.json",
    "db:generate": "prisma generate",
    "db:migrate": "prisma migrate dev"
  },
  "nest-cli": {
    "sourceRoot": "src",
    "compilerOptions": {
      "deleteOutDir": true
    }
  }
}
```

### 3.5 Create the module structure

```bash
# From apps/api/
mkdir -p src/modules/{auth,allegro,offers,orders,products,odoo,sync,billing,shipping,webhooks,health}
mkdir -p src/common/{decorators,guards,interceptors,filters,pipes}
mkdir -p src/prisma
mkdir -p test
```

### 3.6 Entry point — `apps/api/src/main.ts`

```typescript
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      transform: true,
      forbidNonWhitelisted: true,
    }),
  );

  app.enableCors({
    origin: process.env.FRONTEND_URL || 'http://localhost:3000',
    credentials: true,
  });

  const config = new DocumentBuilder()
    .setTitle('Allegro Manager API')
    .setDescription('Product & sales management platform for Allegro')
    .setVersion('1.0')
    .addBearerAuth()
    .build();
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api/docs', app, document);

  const port = process.env.PORT || 4000;
  await app.listen(port);
  console.log(`API running on http://localhost:${port}`);
  console.log(`Swagger docs at http://localhost:${port}/api/docs`);
}

bootstrap();
```

### 3.7 Root module — `apps/api/src/app.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { ScheduleModule } from '@nestjs/schedule';
import { BullModule } from '@nestjs/bull';
import { AuthModule } from './modules/auth/auth.module';
import { AllegroModule } from './modules/allegro/allegro.module';
import { OffersModule } from './modules/offers/offers.module';
import { OrdersModule } from './modules/orders/orders.module';
import { ProductsModule } from './modules/products/products.module';
import { OdooModule } from './modules/odoo/odoo.module';
import { SyncModule } from './modules/sync/sync.module';
import { BillingModule } from './modules/billing/billing.module';
import { ShippingModule } from './modules/shipping/shipping.module';
import { WebhooksModule } from './modules/webhooks/webhooks.module';
import { HealthModule } from './modules/health/health.module';
import { PrismaModule } from './prisma/prisma.module';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    ScheduleModule.forRoot(),
    BullModule.forRoot({
      redis: {
        host: process.env.REDIS_HOST || 'localhost',
        port: parseInt(process.env.REDIS_PORT || '6379', 10),
      },
    }),
    PrismaModule,
    AuthModule,
    AllegroModule,
    OffersModule,
    OrdersModule,
    ProductsModule,
    OdooModule,
    SyncModule,
    BillingModule,
    ShippingModule,
    WebhooksModule,
    HealthModule,
  ],
})
export class AppModule {}
```

### 3.8 Prisma service — `apps/api/src/prisma/prisma.module.ts` & `prisma.service.ts`

**`prisma.service.ts`:**

```typescript
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

**`prisma.module.ts`:**

```typescript
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

### 3.9 Allegro HTTP client — `apps/api/src/modules/allegro/allegro.service.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios';
import { PrismaService } from '../../prisma/prisma.service';

@Injectable()
export class AllegroService {
  private readonly logger = new Logger(AllegroService.name);

  constructor(
    private config: ConfigService,
    private prisma: PrismaService,
  ) {}

  private createClient(accessToken: string, sandbox = false): AxiosInstance {
    const baseURL = sandbox
      ? 'https://api.allegro.pl.allegrosandbox.pl'
      : 'https://api.allegro.pl';

    return axios.create({
      baseURL,
      headers: {
        Authorization: `Bearer ${accessToken}`,
        Accept: 'application/vnd.allegro.public.v1+json',
        'Content-Type': 'application/vnd.allegro.public.v1+json',
        'Accept-Language': 'pl-PL',
      },
    });
  }

  async getClientForAccount(accountId: string): Promise<AxiosInstance> {
    const account = await this.prisma.allegroAccount.findUniqueOrThrow({
      where: { id: accountId },
    });

    if (new Date() >= account.tokenExpiresAt) {
      const refreshed = await this.refreshToken(account.id);
      return this.createClient(refreshed.accessToken, account.sandboxMode);
    }

    return this.createClient(account.accessToken, account.sandboxMode);
  }

  async refreshToken(accountId: string) {
    const account = await this.prisma.allegroAccount.findUniqueOrThrow({
      where: { id: accountId },
    });

    const authBase = account.sandboxMode
      ? 'https://allegro.pl.allegrosandbox.pl'
      : 'https://allegro.pl';

    const credentials = Buffer.from(
      `${account.clientId}:${account.clientSecret}`,
    ).toString('base64');

    const { data } = await axios.post(
      `${authBase}/auth/oauth/token?grant_type=refresh_token&refresh_token=${account.refreshToken}`,
      null,
      { headers: { Authorization: `Basic ${credentials}` } },
    );

    const updated = await this.prisma.allegroAccount.update({
      where: { id: accountId },
      data: {
        accessToken: data.access_token,
        refreshToken: data.refresh_token,
        tokenExpiresAt: new Date(Date.now() + data.expires_in * 1000),
      },
    });

    this.logger.log(`Refreshed token for account ${accountId}`);
    return updated;
  }

  async request<T>(
    accountId: string,
    config: AxiosRequestConfig,
  ): Promise<T> {
    const client = await this.getClientForAccount(accountId);
    const response = await client.request<T>(config);
    return response.data;
  }
}
```

### 3.10 Odoo client — `apps/api/src/modules/odoo/odoo.service.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import axios from 'axios';

@Injectable()
export class OdooService {
  private readonly logger = new Logger(OdooService.name);
  private readonly baseUrl: string;
  private readonly apiKey: string;
  private readonly db: string;

  constructor(private config: ConfigService) {
    this.baseUrl = this.config.getOrThrow<string>('ODOO_URL');
    this.apiKey = this.config.getOrThrow<string>('ODOO_API_KEY');
    this.db = this.config.getOrThrow<string>('ODOO_DB');
  }

  /**
   * JSON-2 API call (Odoo 19+)
   */
  async call<T>(
    model: string,
    method: string,
    args: any[] = [],
    kwargs: Record<string, any> = {},
  ): Promise<T> {
    const url = `${this.baseUrl}/json/2/${model}/${method}`;
    const { data } = await axios.post(
      url,
      { params: { args, kwargs } },
      {
        headers: {
          Authorization: `Bearer ${this.apiKey}`,
          'Content-Type': 'application/json',
        },
      },
    );

    if (data.error) {
      this.logger.error(`Odoo error: ${JSON.stringify(data.error)}`);
      throw new Error(data.error.message || 'Odoo API error');
    }

    return data.result;
  }

  async searchRead<T>(
    model: string,
    domain: any[][] = [],
    fields: string[] = [],
    limit = 100,
    offset = 0,
  ): Promise<T[]> {
    return this.call<T[]>(model, 'search_read', [], {
      domain,
      fields,
      limit,
      offset,
    });
  }

  async create(model: string, values: Record<string, any>): Promise<number> {
    return this.call<number>(model, 'create', [values]);
  }

  async write(
    model: string,
    ids: number[],
    values: Record<string, any>,
  ): Promise<boolean> {
    return this.call<boolean>(model, 'write', [ids, values]);
  }

  async unlink(model: string, ids: number[]): Promise<boolean> {
    return this.call<boolean>(model, 'unlink', [ids]);
  }
}
```

### 3.11 Example module structure — `apps/api/src/modules/auth/`

```
auth/
├── auth.module.ts
├── auth.controller.ts
├── auth.service.ts
├── strategies/
│   ├── allegro-oauth.strategy.ts
│   └── jwt.strategy.ts
├── guards/
│   └── jwt-auth.guard.ts
└── dto/
    └── auth.dto.ts
```

**`auth.module.ts`:**

```typescript
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { ConfigService } from '@nestjs/config';
import { AuthController } from './auth.controller';
import { AuthService } from './auth.service';
import { JwtStrategy } from './strategies/jwt.strategy';
import { AllegroModule } from '../allegro/allegro.module';

@Module({
  imports: [
    PassportModule,
    JwtModule.registerAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        secret: config.getOrThrow<string>('JWT_SECRET'),
        signOptions: { expiresIn: '24h' },
      }),
    }),
    AllegroModule,
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

**`auth.controller.ts`:**

```typescript
import { Controller, Get, Query, Res } from '@nestjs/common';
import { Response } from 'express';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Get('allegro/login')
  allegroLogin(@Query('sandbox') sandbox: string, @Res() res: Response) {
    const url = this.authService.getAuthorizationUrl(sandbox === 'true');
    res.redirect(url);
  }

  @Get('allegro/callback')
  async allegroCallback(@Query('code') code: string) {
    return this.authService.handleCallback(code);
  }

  @Get('allegro/device')
  async deviceFlow(@Query('sandbox') sandbox: string) {
    return this.authService.initiateDeviceFlow(sandbox === 'true');
  }
}
```

---

## 4. Prisma Schema & Database

### 4.1 Create the schema

Create `apps/api/prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// --- Allegro Account & Auth ---

model AllegroAccount {
  id             String   @id @default(cuid())
  name           String
  allegroUserId  String   @unique
  clientId       String
  clientSecret   String
  accessToken    String
  refreshToken   String
  tokenExpiresAt DateTime
  sandboxMode    Boolean  @default(false)
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt

  offers   Offer[]
  orders   Order[]
  syncJobs SyncJob[]
}

// --- Product Catalog ---

model Product {
  id             String   @id @default(cuid())
  name           String
  sku            String   @unique
  ean            String?
  description    String?
  categoryId     String?
  price          Decimal  @db.Decimal(12, 2)
  currency       String   @default("PLN")
  stockQuantity  Int      @default(0)
  odooProductId  Int?
  odooTemplateId Int?
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt

  offers     Offer[]
  orderItems OrderItem[]

  @@index([sku])
  @@index([ean])
  @@index([odooProductId])
}

// --- Allegro Offers ---

model Offer {
  id             String    @id @default(cuid())
  allegroOfferId String    @unique
  productId      String
  accountId      String
  title          String
  status         String
  price          Decimal   @db.Decimal(12, 2)
  currency       String    @default("PLN")
  stockAvailable Int
  stockSold      Int       @default(0)
  externalId     String?
  lastSyncedAt   DateTime?
  createdAt      DateTime  @default(now())
  updatedAt      DateTime  @updatedAt

  product Product        @relation(fields: [productId], references: [id])
  account AllegroAccount @relation(fields: [accountId], references: [id])

  @@index([accountId])
  @@index([status])
  @@index([externalId])
}

// --- Orders ---

model Order {
  id               String    @id @default(cuid())
  allegroOrderId   String    @unique
  accountId        String
  buyerEmail       String?
  buyerLogin       String?
  totalAmount      Decimal   @db.Decimal(12, 2)
  currency         String    @default("PLN")
  status           String
  paymentStatus    String?
  shippingCarrier  String?
  trackingNumber   String?
  odooSaleOrderId  Int?
  orderDate        DateTime
  lastSyncedAt     DateTime?
  createdAt        DateTime  @default(now())
  updatedAt        DateTime  @updatedAt

  account AllegroAccount @relation(fields: [accountId], references: [id])
  items   OrderItem[]

  @@index([accountId])
  @@index([status])
  @@index([orderDate])
  @@index([odooSaleOrderId])
}

model OrderItem {
  id             String  @id @default(cuid())
  orderId        String
  productId      String?
  allegroOfferId String
  title          String
  quantity       Int
  unitPrice      Decimal @db.Decimal(12, 2)
  currency       String  @default("PLN")

  order   Order    @relation(fields: [orderId], references: [id], onDelete: Cascade)
  product Product? @relation(fields: [productId], references: [id])

  @@index([orderId])
}

// --- Sync Infrastructure ---

model SyncJob {
  id             String    @id @default(cuid())
  accountId      String
  type           String    // ORDERS, OFFERS, STOCK, BILLING
  direction      String    // ALLEGRO_TO_APP, APP_TO_ALLEGRO, APP_TO_ODOO, ODOO_TO_APP
  status         String    // PENDING, RUNNING, COMPLETED, FAILED
  startedAt      DateTime?
  completedAt    DateTime?
  itemsProcessed Int       @default(0)
  errorMessage   String?
  createdAt      DateTime  @default(now())

  account AllegroAccount @relation(fields: [accountId], references: [id])

  @@index([accountId, type])
  @@index([status])
}

model ExternalMapping {
  id         String   @id @default(cuid())
  entityType String   // PRODUCT, ORDER, CUSTOMER
  internalId String
  allegroId  String?
  odooId     String?
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  @@unique([entityType, internalId])
  @@index([entityType, allegroId])
  @@index([entityType, odooId])
}
```

### 4.2 Run initial migration

```bash
# Make sure Docker is running with PostgreSQL (see Section 7)
cd apps/api
pnpm prisma migrate dev --name init
pnpm prisma generate
```

---

## 5. Frontend Setup (Next.js)

### 5.1 Scaffold Next.js

```bash
cd apps/web
pnpm create next-app@latest . --typescript --tailwind --eslint --app --src-dir --no-import-alias
```

When prompted:
- TypeScript: **Yes**
- ESLint: **Yes**
- Tailwind CSS: **Yes**
- `src/` directory: **Yes**
- App Router: **Yes**

### 5.2 Install additional dependencies

```bash
cd apps/web

pnpm add axios zustand @tanstack/react-query @tanstack/react-table \
  date-fns lucide-react clsx tailwind-merge class-variance-authority

pnpm add -D @allegro-manager/tsconfig
```

### 5.3 Install shadcn/ui

```bash
cd apps/web
pnpm dlx shadcn@latest init
```

When prompted, choose:
- Style: **Default**
- Base color: **Slate** (or your preference)
- CSS variables: **Yes**

Then add commonly used components:

```bash
pnpm dlx shadcn@latest add button card input label table \
  dialog dropdown-menu select tabs badge separator \
  toast sheet command popover calendar \
  sidebar navigation-menu avatar skeleton
```

### 5.4 Configure `apps/web/tsconfig.json`

```json
{
  "extends": "@allegro-manager/tsconfig/nextjs.json",
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    },
    "plugins": [{ "name": "next" }]
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### 5.5 Create page structure

```bash
cd apps/web/src

mkdir -p app/\(auth\)/login
mkdir -p app/\(auth\)/callback
mkdir -p app/dashboard
mkdir -p app/offers
mkdir -p app/orders
mkdir -p app/products
mkdir -p app/billing
mkdir -p app/settings
mkdir -p components/shared
mkdir -p lib
```

### 5.6 API client — `apps/web/src/lib/api-client.ts`

```typescript
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:4000',
  withCredentials: true,
});

apiClient.interceptors.request.use((config) => {
  const token =
    typeof window !== 'undefined' ? localStorage.getItem('token') : null;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401 && typeof window !== 'undefined') {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  },
);

export default apiClient;
```

### 5.7 React Query provider — `apps/web/src/lib/query-provider.tsx`

```tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useState, type ReactNode } from 'react';

export function QueryProvider({ children }: { children: ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 30 * 1000,
            retry: 1,
          },
        },
      }),
  );

  return (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
}
```

### 5.8 Update root layout — `apps/web/src/app/layout.tsx`

```tsx
import type { Metadata } from 'next';
import { Inter } from 'next/font/google';
import { QueryProvider } from '@/lib/query-provider';
import './globals.css';

const inter = Inter({ subsets: ['latin', 'latin-ext'] });

export const metadata: Metadata = {
  title: 'Allegro Manager',
  description: 'Product & sales management platform for Allegro',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="pl">
      <body className={inter.className}>
        <QueryProvider>{children}</QueryProvider>
      </body>
    </html>
  );
}
```

### 5.9 `apps/web/package.json` scripts

```json
{
  "name": "@allegro-manager/web",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "dev": "next dev --port 3000",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  }
}
```

---

## 6. Shared Packages

### 6.1 `packages/shared/package.json`

```json
{
  "name": "@allegro-manager/shared",
  "version": "0.0.1",
  "private": true,
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "scripts": {
    "build": "tsc",
    "lint": "eslint \"src/**/*.ts\""
  },
  "devDependencies": {
    "@allegro-manager/tsconfig": "workspace:*",
    "typescript": "^5"
  }
}
```

### 6.2 `packages/shared/tsconfig.json`

```json
{
  "extends": "@allegro-manager/tsconfig/base.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist"
  },
  "include": ["src"]
}
```

### 6.3 Allegro types — `packages/shared/src/types/allegro.ts`

```typescript
export interface AllegroPrice {
  amount: string;
  currency: string;
}

export interface AllegroOffer {
  id: string;
  name: string;
  category: { id: string };
  sellingMode: {
    format: 'BUY_NOW' | 'AUCTION' | 'ADVERTISEMENT';
    price: AllegroPrice;
    minimalPrice?: AllegroPrice;
    startingPrice?: AllegroPrice;
  };
  stock: {
    available: number;
    unit: string;
  };
  publication: {
    status: 'INACTIVE' | 'ACTIVE' | 'ACTIVATING' | 'ENDED';
    duration?: string;
    startingAt?: string;
    endingAt?: string;
  };
  external?: { id: string };
  images: string[];
  description: {
    sections: Array<{
      items: Array<{ type: string }>;
    }>;
  };
  createdAt: string;
  updatedAt: string;
}

export interface AllegroOrder {
  id: string;
  buyer: {
    email: string;
    login: string;
    firstName?: string;
    lastName?: string;
    address?: AllegroAddress;
  };
  lineItems: AllegroLineItem[];
  payment: {
    id: string;
    type: string;
    paidAmount: AllegroPrice;
  };
  status: string;
  fulfillment: {
    status: string;
    shipmentSummary: {
      lineItemsSent: string;
    };
  };
  delivery: {
    address: AllegroAddress;
    method: { id: string; name: string };
    pickupPoint?: { id: string; name: string };
    cost: AllegroPrice;
    time?: {
      from: string;
      to: string;
    };
  };
  invoice: {
    required: boolean;
    address?: AllegroAddress;
  };
  boughtAt: string;
  updatedAt: string;
}

export interface AllegroLineItem {
  id: string;
  offer: {
    id: string;
    name: string;
    external?: { id: string };
  };
  quantity: number;
  originalPrice: AllegroPrice;
  price: AllegroPrice;
  boughtAt: string;
}

export interface AllegroAddress {
  firstName?: string;
  lastName?: string;
  street: string;
  city: string;
  postCode: string;
  countryCode: string;
  companyName?: string;
}

export interface AllegroOrderEvent {
  id: string;
  type: string;
  order: { id: string };
  occurredAt: string;
}

export interface AllegroCategory {
  id: string;
  name: string;
  parent?: { id: string };
  leaf: boolean;
}

export interface AllegroProduct {
  id: string;
  name: string;
  category: { id: string };
  parameters: AllegroParameter[];
  images: Array<{ url: string }>;
}

export interface AllegroParameter {
  id: string;
  name: string;
  values?: string[];
  valuesIds?: string[];
  rangeValue?: { from: string; to: string };
}
```

### 6.4 Odoo types — `packages/shared/src/types/odoo.ts`

```typescript
export interface OdooProduct {
  id: number;
  name: string;
  default_code: string; // Internal reference / SKU
  barcode?: string;
  list_price: number;
  qty_available: number;
  type: string;
  categ_id: [number, string];
}

export interface OdooSaleOrder {
  id: number;
  name: string; // SO001, SO002, etc.
  partner_id: [number, string];
  state: 'draft' | 'sent' | 'sale' | 'done' | 'cancel';
  amount_total: number;
  date_order: string;
  order_line: number[];
}

export interface OdooPartner {
  id: number;
  name: string;
  email?: string;
  street?: string;
  city?: string;
  zip?: string;
  country_id?: [number, string];
  is_company: boolean;
}

export interface OdooStockQuant {
  id: number;
  product_id: [number, string];
  location_id: [number, string];
  quantity: number;
  reserved_quantity: number;
}
```

### 6.5 Constants — `packages/shared/src/constants/allegro-endpoints.ts`

```typescript
export const ALLEGRO_BASE_URL = 'https://api.allegro.pl';
export const ALLEGRO_SANDBOX_URL = 'https://api.allegro.pl.allegrosandbox.pl';
export const ALLEGRO_AUTH_URL = 'https://allegro.pl/auth/oauth';
export const ALLEGRO_SANDBOX_AUTH_URL =
  'https://allegro.pl.allegrosandbox.pl/auth/oauth';

export const ALLEGRO_CONTENT_TYPE =
  'application/vnd.allegro.public.v1+json';

export const ALLEGRO_RATE_LIMIT = 9000; // per minute per Client ID
```

### 6.6 Barrel export — `packages/shared/src/index.ts`

```typescript
export * from './types/allegro';
export * from './types/odoo';
export * from './constants/allegro-endpoints';
```

---

## 7. Docker & Local Development

### 7.1 `docker-compose.yml` (project root)

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: allegro-manager-db
    restart: unless-stopped
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: allegro
      POSTGRES_PASSWORD: allegro_secret
      POSTGRES_DB: allegro_manager
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U allegro"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: allegro-manager-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
  redisdata:
```

### 7.2 Start local infrastructure

```bash
# From project root
docker compose up -d

# Verify services are running
docker compose ps
```

---

## 8. Environment Variables

### 8.1 Create `.env.example` (project root)

```bash
# ──── Database ────
DATABASE_URL="postgresql://allegro:allegro_secret@localhost:5432/allegro_manager?schema=public"

# ──── Redis ────
REDIS_HOST=localhost
REDIS_PORT=6379

# ──── NestJS API ────
PORT=4000
NODE_ENV=development
JWT_SECRET=your-jwt-secret-change-in-production

# ──── Frontend ────
NEXT_PUBLIC_API_URL=http://localhost:4000
FRONTEND_URL=http://localhost:3000

# ──── Allegro API (Production) ────
ALLEGRO_CLIENT_ID=your-allegro-client-id
ALLEGRO_CLIENT_SECRET=your-allegro-client-secret
ALLEGRO_REDIRECT_URI=http://localhost:4000/auth/allegro/callback

# ──── Allegro API (Sandbox) ────
ALLEGRO_SANDBOX_CLIENT_ID=your-sandbox-client-id
ALLEGRO_SANDBOX_CLIENT_SECRET=your-sandbox-client-secret
ALLEGRO_SANDBOX_REDIRECT_URI=http://localhost:4000/auth/allegro/callback

# ──── Odoo ────
ODOO_URL=https://your-company.odoo.com
ODOO_DB=your-odoo-database
ODOO_API_KEY=your-odoo-api-key
```

### 8.2 Copy to `.env`

```bash
cp .env.example .env
# Then fill in your actual values
```

### 8.3 Add `.env` to `.gitignore`

```bash
echo ".env" >> .gitignore
echo "node_modules" >> .gitignore
echo ".next" >> .gitignore
echo "dist" >> .gitignore
echo ".turbo" >> .gitignore
```

---

## 9. Development Workflow

### 9.1 Install all dependencies

```bash
# From project root
pnpm install
```

### 9.2 Start infrastructure + generate Prisma client + run migrations

```bash
pnpm docker:up
cd apps/api
pnpm prisma generate
pnpm prisma migrate dev --name init
cd ../..
```

### 9.3 Run the full stack in dev mode

```bash
# Starts both api (port 4000) and web (port 3000) concurrently
pnpm dev
```

Or run them individually:

```bash
# Terminal 1 — Backend
cd apps/api && pnpm dev

# Terminal 2 — Frontend
cd apps/web && pnpm dev
```

### 9.4 Access points

| Service | URL |
|---|---|
| **Next.js Dashboard** | [http://localhost:3000](http://localhost:3000) |
| **NestJS API** | [http://localhost:4000](http://localhost:4000) |
| **Swagger API Docs** | [http://localhost:4000/api/docs](http://localhost:4000/api/docs) |
| **PostgreSQL** | `localhost:5432` (user: `allegro`, db: `allegro_manager`) |
| **Redis** | `localhost:6379` |

### 9.5 Useful commands

```bash
# Add a new Prisma migration
cd apps/api && pnpm prisma migrate dev --name describe_your_change

# Open Prisma Studio (visual DB browser)
cd apps/api && pnpm prisma studio

# Build everything
pnpm build

# Run tests
pnpm test

# Lint
pnpm lint
```

---

## 10. Deployment Notes

### 10.1 Environment-Specific Config

| Environment | DATABASE_URL | ALLEGRO base | Notes |
|---|---|---|---|
| **Development** | Local Docker PostgreSQL | Sandbox (`allegrosandbox.pl`) | Use sandbox credentials |
| **Staging** | Managed PostgreSQL (e.g., Supabase, Neon) | Sandbox | Test with real-like data |
| **Production** | Managed PostgreSQL | Production (`allegro.pl`) | Real Allegro credentials |

### 10.2 Build for Production

```bash
# Build all apps
pnpm build

# Run the API
cd apps/api && pnpm start:prod

# Run the frontend (or deploy to Vercel)
cd apps/web && pnpm start
```

### 10.3 Prisma Migrations in Production

```bash
# Deploy pending migrations (non-interactive, safe for CI/CD)
cd apps/api && pnpm prisma migrate deploy
```

Never run `prisma migrate dev` in production — it can reset data. Always use `prisma migrate deploy`.

### 10.4 Recommended Hosting

| Component | Recommended | Alternative |
|---|---|---|
| **NestJS API** | Railway, Render, Fly.io | AWS ECS, DigitalOcean App Platform |
| **Next.js Frontend** | Vercel | Netlify, Cloudflare Pages |
| **PostgreSQL** | Supabase, Neon, Railway | AWS RDS, DigitalOcean Managed DB |
| **Redis** | Upstash, Railway | AWS ElastiCache, Redis Cloud |

### 10.5 CI/CD Pipeline (GitHub Actions outline)

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  build-and-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: allegro
          POSTGRES_PASSWORD: allegro_secret
          POSTGRES_DB: allegro_manager_test
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm install
      - run: pnpm prisma generate
        working-directory: apps/api
      - run: pnpm prisma migrate deploy
        working-directory: apps/api
        env:
          DATABASE_URL: postgresql://allegro:allegro_secret@localhost:5432/allegro_manager_test
      - run: pnpm build
      - run: pnpm test
      - run: pnpm lint
```
