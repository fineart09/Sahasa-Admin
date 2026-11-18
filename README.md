# Sahasa Admin

ระบบหลังบ้านจัดการออเดอร์ของบริษัท สหัสสา (เต้าหู้ + น้ำจิ้ม) จากทุกช่องทาง:
Shopee, Lazada, TikTok, Line, Facebook, หน้าร้าน และโทรสั่ง

- เอกสารวิเคราะห์และคำถามค้างตอบ: [_requirements/Database-Design-Notes.md](_requirements/Database-Design-Notes.md) ← เริ่มอ่านที่นี่
- วิเคราะห์ Function Requirement / SRS: [_requirements/Requirements-Analysis.md](_requirements/Requirements-Analysis.md)
- Prototype หน้าจอ: [_requirements/Designs/sahasa-prototype.html](_requirements/Designs/sahasa-prototype.html)

> ไฟล์ Excel ใน `_requirements/Documents/` และ `_requirements/Marketplace Order/` **ไม่อยู่ใน git**
> เพราะมีข้อมูลลูกค้าจริง (ชื่อ ที่อยู่ เบอร์โทร) ถ้าต้องใช้ให้ขอจาก cholcool

## Tech stack

| ส่วน           | ใช้                                                                                                           |
| ------------- | ------------------------------------------------------------------------------------------------------------ |
| Runtime       | Node 22 (`.nvmrc`), pnpm workspaces                                                                          |
| Frontend      | React 19 + Vite 8, TanStack Router (file-based), TanStack Query, Mantine 9 + mantine-datatable, dayjs (พ.ศ.) |
| Backend       | NestJS 12 (ESM), validation ด้วย zod ผ่าน `StandardSchemaValidationPipe` ของ Nest, nestjs-pino                 |
| Database      | MariaDB (Hostinger) + Prisma 7 + `@prisma/adapter-mariadb`                                                   |
| Test          | Vitest (unit + e2e + story test), Storybook 10                                                               |
| Lint / format | Oxlint (`--type-aware`), Prettier                                                                            |

## โครงสร้างโปรเจกต์

### ภาพรวม

```
Sahasa-Admin/
├── _requirements/      เอกสาร requirement, Notes, prototype (ไฟล์ Excel ไม่อยู่ใน git)
├── apps/
│   ├── api/            NestJS: API + ส่งไฟล์หน้าเว็บ (Node app เดียวบน Hostinger)
│   └── web/            React + Vite: build เป็นไฟล์ static
├── packages/
│   └── shared/         zod schema / type / enum ที่ web และ api ใช้ร่วมกัน
├── CLAUDE.md           กติกาสำหรับ Claude ของทีม
└── README.md
```

### ไฟล์ที่ root

| ไฟล์                   | หน้าที่                                                                     |
| --------------------- | ------------------------------------------------------------------------ |
| `package.json`        | script รวมของทั้งโปรเจกต์ (`dev`, `build`, `lint`, `test` …) และเวอร์ชัน pnpm |
| `pnpm-workspace.yaml` | บอก pnpm ว่า package อยู่ใน `apps/*` และ `packages/*`                       |
| `pnpm-lock.yaml`      | ล็อกเวอร์ชัน dependency ทั้งหมด (commit เสมอ ห้ามแก้เอง)                        |
| `tsconfig.base.json`  | ค่า TypeScript กลาง (strict) ที่ package อื่น extend                          |
| `.oxlintrc.json`      | กฎ lint ของทั้งโปรเจกต์ (รวมกฎที่อ่าน type เช่น no-floating-promises)           |
| `.prettierrc.json`    | รูปแบบโค้ด (single quote, ความยาวบรรทัด 100)                                |
| `.prettierignore`     | ไฟล์ที่ Prettier ข้าม                                                        |
| `.gitignore`          | ไฟล์ที่ไม่เข้า git (`.env`, `dist/`, ไฟล์ Excel ของลูกค้า …)                     |
| `.nvmrc`              | Node 22 (`nvm use`)                                                      |
| `CLAUDE.md`           | กติกาการทำงานที่ Claude ของทุกคนอ่านอัตโนมัติ                                    |

### `apps/api` — NestJS

```
apps/api/
├── package.json              dependency + script ของ api และ "imports" ที่ทำให้ใช้ #src/ ได้
├── nest-cli.json             ค่าของ Nest CLI (build ไป dist/)
├── tsconfig.json             TypeScript ของ api (รวม test/) ใช้ตอน typecheck
├── tsconfig.build.json       TypeScript ตอน build (เฉพาะ src/ ไม่รวม *.spec.ts)
├── vitest.config.ts          unit test: src/**/*.spec.ts
├── vitest.config.e2e.ts      e2e test: test/**/*.e2e-spec.ts
├── prisma.config.ts          บอก Prisma CLI ว่า schema / migrations อยู่ไหน และอ่าน DATABASE_URL
├── .env.example              ตัวอย่าง environment variable (คัดลอกเป็น .env)
├── AGENTS.md                 คำแนะนำของ NestJS 12 สำหรับ AI (ESM, Vitest, validation)
├── prisma/
│   ├── schema.prisma         นิยามตารางทั้งหมด (ยังว่าง รอออกแบบ DB)
│   └── migrations/           SQL migration ที่ Prisma สร้าง (ยังไม่มี)
├── src/
│   ├── main.ts               จุดเริ่มแอป: ตั้ง prefix /api, validation pipe, logger แล้ว listen PORT
│   ├── app.module.ts         module หลัก: รวมทุก module + ตั้ง Config, Logger, ServeStatic (ส่ง apps/web/dist)
│   ├── config/
│   │   └── env.ts            zod schema ของ .env — แอปไม่ start ถ้าค่าขาดหรือผิดรูปแบบ
│   ├── prisma/
│   │   ├── prisma.module.ts  ประกาศ PrismaService เป็น global (inject ได้ทุก module ไม่ต้อง import)
│   │   └── prisma.service.ts PrismaClient ที่ต่อ MariaDB ผ่าน @prisma/adapter-mariadb
│   ├── common/               ของที่ใช้หลาย module
│   │   └── guards/
│   │       └── cron-secret.guard.ts  ให้เฉพาะ request ที่มี header x-cron-secret ถูกต้องผ่าน
│   ├── modules/              1 โฟลเดอร์ = 1 โมดูลธุรกิจ
│   │   ├── health/
│   │   │   ├── health.module.ts      ประกาศ controller ของโมดูล
│   │   │   └── health.controller.ts  GET /api/health — เช็กว่าแอปและ DB ทำงาน
│   │   ├── orders/
│   │   │   ├── orders.module.ts          ประกาศ controller + service ของโมดูล
│   │   │   ├── orders.controller.ts      GET /api/orders — รับ request, ตรวจ query ด้วย schema, เรียก service
│   │   │   ├── orders.service.ts         logic ธุรกิจ + query DB (ตอนนี้คืนค่าว่าง)
│   │   │   └── orders.controller.spec.ts unit test ของ controller
│   │   └── jobs/
│   │       ├── jobs.module.ts
│   │       └── jobs.controller.ts    POST /api/jobs/run — hPanel cron เรียกทุก 5 นาที (ป้องกันด้วย CronSecretGuard)
│   └── generated/prisma/     Prisma Client ที่ `prisma generate` สร้าง (ไม่ commit ห้ามแก้)
├── test/
│   └── app.e2e-spec.ts       e2e test: เปิดแอปจริงแล้วยิง HTTP ด้วย supertest
└── dist/                     ผล build (ไม่ commit) — Hostinger รัน dist/main.js
```

**ไฟล์ในหนึ่งโมดูล (`modules/<module>/`)** ทุกโมดูลใช้รูปแบบเดียวกัน:

| ไฟล์                           | หน้าที่                                                                                                                    | ห้ามทำ                      |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------- | -------------------------- |
| `<module>.module.ts`          | รวม controller + service ของโมดูล และ export service ที่โมดูลอื่นต้องใช้                                                        | ใส่ logic                   |
| `<module>.controller.ts`      | กำหนด route (`@Get`, `@Post`), ตรวจ input ด้วย `@Query({ schema })` / `@Body({ schema })`, ตรวจสิทธิ์, เรียก service แล้วคืนผล | query DB หรือคำนวณเงินตรงนี้   |
| `<module>.service.ts`         | logic ธุรกิจทั้งหมด: query/บันทึกผ่าน `PrismaService`, คำนวณยอด, เปลี่ยนสถานะ, transaction                                       | อ่าน `req` / `res` ของ HTTP |
| `<module>.service.spec.ts`    | unit test ของ service (ส่วนที่สำคัญที่สุดที่ต้อง test)                                                                            |                            |
| `<module>.controller.spec.ts` | unit test ของ controller (ถ้ามี logic การตรวจ input หรือสิทธิ์)                                                               |                            |

ไฟล์ที่จะเพิ่มเมื่อโมดูลใหญ่ขึ้น (สร้างเฉพาะตอนจำเป็น):

- `<module>.guard.ts` สิทธิ์เฉพาะโมดูล
- `<sub>.service.ts` เมื่อ service ยาวเกินไป ให้แยกเป็นหลาย service เช่น `platforms/shopee.service.ts`, `platforms/lazada.service.ts`
- schema ของ input/output **ไม่ต้องสร้าง DTO class** ให้เขียนเป็น zod ใน `packages/shared/src/<module>.ts` แทน

### `apps/web` — React + Vite

```
apps/web/
├── package.json              dependency + script ของ web
├── index.html                หน้า HTML เดียวของแอป (lang="th") — Vite ใส่ script ให้ตอน build
├── vite.config.ts            plugin TanStack Router + React, alias @/, proxy /api ไป :3000, ตั้งค่า Vitest
├── postcss.config.cjs        PostCSS ของ Mantine (breakpoint, mixin ใน CSS Module)
├── tsconfig.json             รวม tsconfig.app.json + tsconfig.node.json
├── tsconfig.app.json         TypeScript ของโค้ดหน้าเว็บ + paths @/*
├── tsconfig.node.json        TypeScript ของ vite.config.ts
├── vitest.shims.d.ts         type ของ Vitest browser mode (สำหรับ story test)
├── .storybook/
│   ├── main.ts               ตั้งค่า Storybook: หา *.stories.tsx, addon docs / a11y / vitest
│   └── preview.tsx           ห่อทุก story ด้วย MantineProvider + theme + ปุ่มสลับ light/dark
├── public/
│   └── favicon.svg           ไฟล์ static ที่ copy ไปตรงๆ
└── src/
    ├── main.tsx              จุดเริ่ม: โหลดฟอนต์ไทย + CSS ของ Mantine แล้ว render <RouterProvider>
    ├── routeTree.gen.ts      แผนที่ route ที่ plugin สร้างจาก routes/ (commit แต่ห้ามแก้)
    ├── app/
    │   ├── providers.tsx     ห่อแอปด้วย Mantine, Dates (locale th), React Query, Modals, Notifications
    │   ├── router.ts         สร้าง router จาก routeTree + ส่ง queryClient เป็น context
    │   └── query-client.ts   ค่าเริ่มต้นของ TanStack Query (cache 30 วินาที, retry 1 ครั้ง)
    ├── theme/
    │   ├── theme.ts          Mantine theme: สีแดง (brand) / ทอง (gold) และฟอนต์จาก prototype
    │   └── global.css        สีพื้นหลังและสี sidebar ของโหมด light / dark
    ├── lib/                  ฟังก์ชันกลางที่ไม่ใช่ UI
    │   ├── api-client.ts     fetch ไป /api พร้อม cookie, แปลง error เป็น ApiError, สร้าง query string
    │   ├── api-client.test.ts
    │   └── dayjs.ts          ตั้ง dayjs เป็นภาษาไทย / ปี พ.ศ. / เวลาไทย + formatThaiDate()
    ├── components/           UI ที่ใช้หลายโมดูล
    │   └── layout/
    │       ├── AppLayout.tsx        AppShell: sidebar + header (desktop), bottom nav (mobile)
    │       ├── AppLayout.module.css สีและ style ของ sidebar / bottom nav
    │       └── nav-items.ts         รายการเมนู (ชื่อ, path, icon, แสดงบนมือถือไหม)
    ├── features/             1 โฟลเดอร์ = 1 โมดูลธุรกิจ (ชื่อเดียวกับ apps/api/src/modules)
    │   └── orders/
    │       ├── api.ts        query options ของ TanStack Query ที่เรียก /api/orders
    │       └── components/
    │           ├── OrderStatusBadge.tsx          ป้ายสถานะออเดอร์ (สีตามสถานะ)
    │           └── OrderStatusBadge.stories.tsx  story + test ของป้ายสถานะ
    └── routes/               หน้าจอ — ชื่อไฟล์ = URL (file-based routing)
        ├── __root.tsx        root ของทุกหน้า (รวม login) ประกาศ context ของ router
        ├── login.tsx         /login — หน้าเข้าสู่ระบบ (ไม่มี sidebar)
        ├── _app.tsx          layout ของทุกหน้าหลัง login (ไม่เพิ่ม path ใน URL)
        └── _app/
            ├── index.tsx         /           — แดชบอร์ด
            └── orders/
                ├── index.tsx     /orders     — รายการออเดอร์ + filter ใน URL
                └── $orderId.tsx  /orders/:id — รายละเอียดออเดอร์
```

**ไฟล์ในหนึ่งโมดูล (`features/<module>/`)**:

| ไฟล์                             | หน้าที่                                                                          |
| ------------------------------- | ----------------------------------------------------------------------------- |
| `api.ts`                        | `queryOptions()` สำหรับอ่านข้อมูล และ `useMutation` hook สำหรับสร้าง/แก้/ลบ ของโมดูลนี้ |
| `components/<Name>.tsx`         | component ของโมดูล เช่น ตาราง, ฟอร์ม, modal, ป้ายสถานะ (ชื่อไฟล์ PascalCase)         |
| `components/<Name>.stories.tsx` | story ของ component ที่ใช้ซ้ำ (รันเป็น test อัตโนมัติด้วย)                              |
| `components/<Name>.module.css`  | style เพิ่มเติม ถ้า prop ของ Mantine ไม่พอ                                         |
| `hooks/use<Name>.ts`            | hook เฉพาะโมดูล (สร้างเมื่อจำเป็น)                                                 |
| `<name>.test.ts`                | unit test ของ logic ที่ไม่ใช่ UI                                                  |

**ไฟล์ใน `routes/`** เขียนให้บางที่สุด: ประกาศ `createFileRoute` (+ `validateSearch` ถ้ามี filter) แล้วประกอบ component จาก `features/` ถ้าหน้าไหนยาวเกิน ~100 บรรทัด ให้ย้ายส่วนนั้นไปเป็น component ใน `features/`

| สัญลักษณ์ในชื่อไฟล์          | ความหมาย                  | ตัวอย่าง                         |
| ---------------------- | ------------------------- | ------------------------------ |
| `__root.tsx`           | root ของทั้งแอป             |                                |
| `_name.tsx` + `_name/` | layout ที่ไม่เพิ่ม path ใน URL | `_app.tsx`                     |
| `index.tsx`            | หน้าหลักของโฟลเดอร์          | `orders/index.tsx` → `/orders` |
| `$name.tsx`            | ค่าที่เปลี่ยนได้ใน URL (param)  | `$orderId.tsx` → `/orders/123` |

### `packages/shared`

```
packages/shared/
├── package.json              export จาก dist/ (ต้อง build ก่อน api / web ใช้ได้)
├── tsconfig.json             TypeScript (NodeNext ESM) — import ต้องลงท้าย .js
├── tsconfig.build.json       build ไป dist/ (ไม่รวม *.spec.ts)
├── vitest.config.ts
└── src/
    ├── index.ts              export ทุกอย่างของ package (เพิ่มไฟล์ใหม่ต้อง export ที่นี่)
    ├── channel.ts            ช่องทางขาย 7 ช่องทาง (CHANNELS + channelSchema)
    ├── order-status.ts       สถานะออเดอร์ + ชื่อภาษาไทย (ร่าง รอยืนยัน)
    ├── order-list-query.ts   filter ของหน้ารายการออเดอร์ (ใช้ทั้ง URL และ API)
    └── order-list-query.spec.ts
```

ใส่ใน `shared` เฉพาะสิ่งที่**ทั้งสองฝั่ง**ต้องรู้ตรงกัน: zod schema ของ input/output ของ API, enum (สถานะ, ช่องทาง, role), ค่าคงที่ทางธุรกิจ ห้ามใส่โค้ดที่ใช้ได้ฝั่งเดียว (React, NestJS, Prisma)

### เพิ่มโมดูลใหม่ต้องสร้างไฟล์อะไร (ตัวอย่าง `customers`)

1. `packages/shared/src/customer.ts` — zod schema ของ input/output แล้ว export ใน `index.ts` (ต้องถามก่อนตาม CLAUDE.md)
2. `apps/api/src/modules/customers/` — `customers.module.ts`, `customers.controller.ts`, `customers.service.ts`, `customers.service.spec.ts` แล้วเพิ่ม `CustomersModule` ใน `app.module.ts`
3. `apps/web/src/features/customers/` — `api.ts` + `components/`
4. `apps/web/src/routes/_app/customers/index.tsx` — หน้าจอ และเพิ่มเมนูใน `components/layout/nav-items.ts`

### หลักการแบ่งโค้ด

1. **แบ่งตามโมดูลธุรกิจ ไม่ใช่ตามชนิดไฟล์** ทั้ง `apps/api/src/modules/<x>` และ `apps/web/src/features/<x>` ใช้ชื่อเดียวกัน ตาม 10 โมดูลใน SRS:

   | โมดูล               | api `modules/`                                  | web `features/` + `routes/`   |
   | ------------------ | ----------------------------------------------- | ----------------------------- |
   | Auth / ผู้ใช้ / สิทธิ์   | `auth`, `users`                                 | `/login`, `/users`            |
   | Dashboard          | `dashboard`                                     | `/`                           |
   | Order              | `orders`, `customers`                           | `/orders`, `/orders/$orderId` |
   | โรงงาน             | `factory`, `shipments`                          | `/factory`                    |
   | บัญชี                | `accounting`                                    | `/accounting`                 |
   | รายงาน             | `reports`                                       | `/reports`                    |
   | เชื่อมต่อ Platform    | `platforms` (shopee, lazada, tiktok), `imports` | `/platforms`                  |
   | ข้อมูลหลัก            | `products`, `master`                            | `/master`                     |
   | สต็อก (รอยืนยันข้อ 24) | `stock`                                         | `/stock`                      |
   | งานเบื้องหลัง         | `jobs` (cron endpoint)                          | —                             |

   ตอนนี้มีจริงแค่ `health`, `orders`, `jobs` ที่เหลือเพิ่มตอนเริ่มทำโมดูลนั้น (`pnpm --filter @sahasa/api exec nest g resource modules/<ชื่อ>`)

2. **schema กลางอยู่ที่ `packages/shared`** เช่น `orderListQuerySchema` ถูกใช้ 2 ที่:
   - หน้าเว็บ: `validateSearch` ของ route `/orders` → filter อยู่ใน URL แบบ type-safe
   - API: `@Query({ schema: orderListQuerySchema })` → ตรวจ query ด้วยกฎเดียวกัน
3. **`routes/` บาง `features/` หนา** หน้าจอแค่ประกอบ component และ query จาก `features/`
   - ไม่ import ด้วย `../../` ให้ใช้ alias แทน:
     - `apps/web`: `@/` = `src/` เช่น `import { formatThaiDate } from '@/lib/dayjs'`
     - `apps/api`: `#src/` = `src/` เช่น `import { PrismaService } from '#src/prisma/prisma.service.js'`
4. **Controller บาง Service หนา** logic ธุรกิจอยู่ใน service เพื่อให้ test ได้ง่าย
5. **เงินใช้ Decimal / เวลาเก็บ UTC** แสดงผลเป็นเวลาไทยและปี พ.ศ. ด้วย `formatThaiDate()`

## เริ่มใช้งาน

```bash
nvm use                                  # Node 22
pnpm install
cp apps/api/.env.example apps/api/.env   # ใส่ DATABASE_URL ของ MariaDB
pnpm --filter @sahasa/shared build
pnpm dev                                 # api :3000 + web :5173 (proxy /api)
```

| คำสั่ง                                 | ทำอะไร                                          |
| ------------------------------------ | ----------------------------------------------- |
| `pnpm dev`                           | รัน api + web พร้อมกัน                             |
| `pnpm build`                         | build shared → web → api                        |
| `pnpm start`                         | รัน production (api ส่งหน้าเว็บจาก `apps/web/dist`) |
| `pnpm test`                          | test ทุก package (รวม story test ใน Chromium)    |
| `pnpm --filter @sahasa/api test:e2e` | e2e test ของ API                                |
| `pnpm lint`                          | Oxlint แบบ type-aware                           |
| `pnpm format`                        | Prettier                                        |
| `pnpm storybook`                     | Storybook ที่ :6006                               |

## Deploy บน Hostinger (ร่าง)

1. build บนเครื่องหรือ GitHub Actions (`pnpm build`) ไม่ build บน Hostinger เพราะ RAM ใช้ร่วมกัน
2. อัปโหลด `apps/api/dist`, `apps/web/dist`, `packages/shared/dist` และ package.json ที่เกี่ยวข้อง
3. hPanel → Node.js: entry `apps/api/dist/main.js`, Node 22, ตั้ง environment variable ตาม `.env.example`
4. hPanel → Cron: ทุก 5 นาที
   `curl -s -X POST -H "x-cron-secret: <CRON_SECRET>" https://<domain>/api/jobs/run`

## สิ่งที่ยังไม่ได้ทำ

- ตาราง database ทั้งหมด (รอลูกค้าตอบคำถาม 🔴)
- Better Auth + session ใน MariaDB, RBAC ด้วย @casl/ability
- Playwright E2E ของทั้งระบบ, Sentry
## ทดสอบ Sync