# Mening Fermam — backend

Fermerlar uchun "Mening Fermam" ilovasining backend API'si.

**Stack:** NestJS 12 · Prisma 7 (PostgreSQL 16) · Redis 7 · S3 (developmentda
MinIO) · TypeScript 6 (ESM) · Zod · Vitest · oxlint · Docker.

---

## Mundarija

1. [Hozirgacha nimalar qilingan](#hozirgacha-nimalar-qilingan)
2. [Loyiha tuzilmasi](#loyiha-tuzilmasi)
3. [API endpoint'lar](#api-endpointlar)
4. [Ma’lumotlar bazasi](#malumotlar-bazasi)
5. [Environment o‘zgaruvchilari](#environment-ozgaruvchilari)
6. [Tez start (development)](#tez-start-development)
7. [Production](#production)
8. [Hali qilinmagan ishlar](#hali-qilinmagan-ishlar)

---

## Hozirgacha nimalar qilingan

### 1. Loyiha poydevori

- NestJS 12 loyihasi **ESM** (`"type": "module"`) rejimida, Node.js 22+ uchun.
- Barcha endpoint'lar `api/v1` prefiksi ostida (`src/main.ts`).
- Global `ValidationPipe`: `whitelist`, `forbidNonWhitelisted`, `transform` —
  DTO'da yo‘q maydon yuborilsa so‘rov rad etiladi.
- `enableShutdownHooks()` — container to‘xtaganda DB, Redis ulanishlari toza
  yopiladi.
- Server `0.0.0.0` da tinglaydi (container ichidan tashqariga chiqishi uchun).

### 2. Konfiguratsiya va env validatsiyasi

- `src/config/env.ts` — barcha env o‘zgaruvchilar **Zod** sxemasi bilan
  tekshiriladi. Xato bo‘lsa ilova ishga tushmaydi va aniq xabar beradi.
- `.env` dagi bo‘sh qiymatlar "berilmagan" deb hisoblanadi.
- Xavfsizlik qoidalari:
  - `JWT_SECRET` kamida 32 belgi bo‘lishi shart;
  - `NODE_ENV=production` da `SMS_PROVIDER=mock` bo‘lsa ilova ishga tushmaydi.
- `ConfigModule` global va keshlangan; `ConfigService<Env, true>` orqali
  tiplangan holda o‘qiladi.

### 3. Ma’lumotlar bazasi (Prisma 7 + PostgreSQL)

- Prisma 7 yangi `prisma-client` generatori: ESM, `.js` import kengaytmasi,
  client `src/generated/prisma` ga generatsiya qilinadi.
- `@prisma/adapter-pg` driver adapter orqali ulanish.
- `prisma.config.ts` — schema, migratsiyalar yo‘li va `DATABASE_URL`.
- `PrismaService` / `PrismaModule` — global, modul yopilganda `$disconnect`.
- Birinchi migratsiya `20261015000000_init`: `UserRole` enum va `users`
  jadvali.

### 4. Autentifikatsiya

- **Login telefon raqam + parol bilan** (`POST /auth/login`).
  - Telefon formati: `+998XXXXXXXXX` (DTO darajasida tekshiriladi).
  - Parol kamida 8 belgi.
- **Parol hashlash** — `src/auth/password.ts`:
  - Node.js `scrypt` (tashqi kutubxonasiz), 16 baytli tasodifiy salt,
    64 baytli kalit;
  - saqlash formati: `scrypt$<salt hex>$<hash hex>`;
  - solishtirish `timingSafeEqual` bilan (timing attack'dan himoya).
- **JWT** — `@nestjs/jwt`, payload: `{ sub: userId, role }`, muddati
  `JWT_EXPIRES_IN` (default `1d`).
- **`JwtAuthGuard`** — `Authorization: Bearer <token>` sarlavhasini tekshiradi
  va `request.user` ga payload'ni yozadi.
- `GET /auth/me` — joriy foydalanuvchi ma’lumotlari.
- Javoblarda `passwordHash` hech qachon qaytarilmaydi (`toPublic`).
- Noto‘g‘ri telefon yoki parolda bir xil xabar qaytadi — qaysi biri
  xatoligi oshkor qilinmaydi.

### 5. Redis

- `RedisModule` (global) — `ioredis` client, `REDIS` token orqali inject
  qilinadi, `maxRetriesPerRequest: 3`, modul yopilganda `quit()`.
- Hozircha faqat health check'da ishlatiladi (kesh, OTP, rate limit uchun
  tayyor).

### 6. Fayl saqlash (S3 / MinIO)

- `StorageModule` (global) — `S3Client` provider.
- Developmentda MinIO (`S3_ENDPOINT`, `S3_FORCE_PATH_STYLE=true`).
- Production'da `S3_ENDPOINT` va kalitlar bo‘sh qoldirilsa, AWS SDK IAM
  role'dan foydalanadi.
- `meningfermam-private` bucket'i dev'da avtomatik yaratiladi va **public
  access o‘chiriladi** (`mc anonymous set none`).

### 7. Health check

- `GET /health` — uchta tizimni tekshiradi:
  - `database` — `SELECT 1`;
  - `redis` — `PING`;
  - `storage` — S3 `HeadBucket`.
- Hammasi ishlasa `200 { status: "ok", checks }`, biri ishlamasa
  `503 { status: "error", checks }`.
- Production Docker image'ning `HEALTHCHECK` i shu endpoint'dan foydalanadi.

### 8. API hujjatlari (Swagger)

- `@nestjs/swagger` bilan `/docs` manzilida, Bearer auth qo‘llab-quvvatlanadi.
- Faqat `SWAGGER_ENABLED=true` bo‘lsa yoqiladi (production'da default o‘chiq).

### 9. Monitoring (ixtiyoriy)

- `@nestjs/observe` telemetriyasi ulangan, lekin faqat `OBSERVE_APP_KEY` va
  `OBSERVE_APP_SECRET` berilganda yoqiladi.

### 10. Demo seed

- `prisma/seed.ts` — ikkita demo akkaunt (ADMIN va FARMER).
- **Idempotent** (`upsert`): qayta ishga tushganda dublikat yaratmaydi,
  mavjud akkauntni o‘zgartirmaydi.
- `NODE_ENV=production` da ishlashdan bosh tortadi.

### 11. Docker — development muhiti

- `Dockerfile.dev` + `compose.dev.yaml`: `api`, `postgres`, `redis`, `minio`,
  `minio-init` servislari.
- Barcha servislarda healthcheck; API faqat ular tayyor bo‘lgach ishga tushadi.
- `docker/dev-entrypoint.sh`:
  1. `package.json`/`package-lock.json` o‘zgargan bo‘lsa `npm install`
     (sha256 hash bo‘yicha);
  2. `prisma generate`;
  3. PostgreSQL tayyor bo‘lishini kutish;
  4. `prisma migrate deploy`;
  5. demo seed;
  6. `nest start --watch` (hot reload).
- Kod bind mount qilingan, `node_modules` alohida volume'da.
- Barcha portlar faqat `127.0.0.1` ga bog‘langan; port band bo‘lsa `.env` orqali
  o‘zgartiriladi.

### 12. Docker — production

- Multi-stage `Dockerfile` (`dependencies` → `builder` → `runner`).
- Yakuniy image: faqat `dist`, production `node_modules`, `prisma`, `docker/`;
  root bo‘lmagan `appuser` bilan ishlaydi.
- `compose.prod.yaml` qattiqlashtirilgan:
  - `read_only: true` fayl tizimi, faqat `/tmp` tmpfs;
  - `cap_drop: ALL`, `no-new-privileges`;
  - port default `127.0.0.1` ga (reverse proxy ortida);
  - secret'lar faylda emas — serverdagi env fayldan (AWS Secrets Manager).
- `docker/prod-entrypoint.sh`:
  - `migrate` rejimi — deploy'dan oldin bir marta migratsiya (release command);
  - `serve` rejimi — server; ixtiyoriy `RUN_MIGRATIONS_ON_START=true`.

### 13. Sifat vositalari

- **Vitest** — unit (`*.spec.ts`) va e2e (`vitest.config.e2e.ts`) testlar,
  coverage (`@vitest/coverage-v8`).
- **oxlint** — linter (`oxlint.json`).
- **Prettier** — formatlash (`.prettierrc`).

---

## Loyiha tuzilmasi

```
.
├── src/
│   ├── main.ts                 # bootstrap: prefix, ValidationPipe, Swagger
│   ├── app.module.ts           # root modul, Observe ulanishi
│   ├── config/env.ts           # Zod env sxemasi
│   ├── auth/
│   │   ├── auth.controller.ts  # /auth/login, /auth/me
│   │   ├── auth.service.ts     # login, JWT, me
│   │   ├── jwt-auth.guard.ts   # Bearer token guard
│   │   ├── password.ts         # scrypt hash/verify
│   │   └── dto/login.dto.ts
│   ├── health/health.controller.ts
│   ├── prisma/                 # PrismaService (global)
│   ├── redis/redis.module.ts   # ioredis (global)
│   ├── storage/storage.module.ts # S3Client (global)
│   └── generated/prisma/       # Prisma tomonidan generatsiya qilinadi
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed.ts
├── docker/
│   ├── dev-entrypoint.sh
│   └── prod-entrypoint.sh
├── test/                       # e2e testlar
├── Dockerfile                  # production
├── Dockerfile.dev              # development
├── compose.dev.yaml
├── compose.prod.yaml
└── prisma.config.ts
```

---

## API endpoint'lar

Bazaviy manzil: `http://localhost:3001/api/v1`

| Metod | Yo‘l          | Auth   | Tavsif                                  |
| ----- | ------------- | ------ | --------------------------------------- |
| GET   | `/`           | —      | Oddiy javob (`Hello World!`)            |
| GET   | `/health`     | —      | DB, Redis, S3 holati                    |
| POST  | `/auth/login` | —      | Telefon + parol → `accessToken` + user  |
| GET   | `/auth/me`    | Bearer | Joriy foydalanuvchi                     |

**Login so‘rovi:**

```bash
curl -X POST http://localhost:3001/api/v1/auth/login \
  -H 'content-type: application/json' \
  -d '{"phone":"+998901111111","password":"Demo12345!"}'
```

**Javob:**

```json
{
  "accessToken": "eyJhbGciOi...",
  "user": {
    "id": "uuid",
    "phone": "+998901111111",
    "fullName": "Demo Admin",
    "role": "ADMIN"
  }
}
```

**Joriy foydalanuvchi:**

```bash
curl http://localhost:3001/api/v1/auth/me \
  -H 'authorization: Bearer <accessToken>'
```

**Health javobi:**

```json
{ "status": "ok", "checks": { "database": "up", "redis": "up", "storage": "up" } }
```

---

## Ma’lumotlar bazasi

### `users` jadvali

| Ustun           | Tip         | Izoh                          |
| --------------- | ----------- | ----------------------------- |
| `id`            | UUID        | Primary key                   |
| `phone`         | TEXT        | Unique                        |
| `full_name`     | TEXT        |                               |
| `password_hash` | TEXT        | `scrypt$salt$hash`            |
| `role`          | `UserRole`  | `ADMIN` yoki `FARMER` (default) |
| `created_at`    | TIMESTAMP   | default `now()`               |
| `updated_at`    | TIMESTAMP   | avtomatik yangilanadi         |

---

## Environment o‘zgaruvchilari

To‘liq ro‘yxat va dev qiymatlari `.env.example` da.

| O‘zgaruvchi                 | Majburiy | Default       | Izoh                                  |
| --------------------------- | -------- | ------------- | ------------------------------------- |
| `NODE_ENV`                  | —        | `development` | `development` / `test` / `production` |
| `PORT`                      | —        | `3001`        |                                       |
| `DATABASE_URL`              | ha       |               | PostgreSQL URL                        |
| `REDIS_URL`                 | ha       |               | Redis URL                             |
| `S3_ENDPOINT`               | —        |               | AWS'da bo‘sh, dev'da MinIO            |
| `S3_REGION`                 | —        | `us-east-1`   |                                       |
| `S3_BUCKET`                 | ha       |               |                                       |
| `S3_ACCESS_KEY_ID`          | —        |               | Bo‘sh bo‘lsa IAM role                 |
| `S3_SECRET_ACCESS_KEY`      | —        |               |                                       |
| `S3_FORCE_PATH_STYLE`       | —        | `false`       | MinIO uchun `true`                    |
| `JWT_SECRET`                | ha       |               | Kamida 32 belgi                       |
| `JWT_EXPIRES_IN`            | —        | `1d`          |                                       |
| `SWAGGER_ENABLED`           | —        | `false`       |                                       |
| `SMS_PROVIDER`              | —        | `mock`        | Production'da `mock` taqiqlangan      |
| `OBSERVE_APP_KEY` / `_SECRET` | —      |               | Ikkalasi bo‘lsa telemetriya yoqiladi  |
| `DEMO_PASSWORD`             | —        | `Demo12345!`  | Faqat seed uchun                      |
| `RUN_MIGRATIONS_ON_START`   | —        | `false`       | Faqat production entrypoint           |

---

## Tez start (development)

Talab: Docker Desktop (yoki Docker Engine + Compose v2) va Node.js 22+.

```bash
git clone <repository>
cd my-farm
cp .env.example .env
npm install
npm run start:dev
```

`npm run start:dev` image'ni build qiladi va `api`, `postgres`, `redis`, `minio`,
`minio-init` servislarini ko‘taradi. API container o‘zi PostgreSQL'ni kutadi,
migratsiyalarni bajaradi, demo seed'ni yuritadi va `npm run start:watch` bilan
NestJS'ni ishga tushiradi.

| Servis      | Manzil                                               |
| ----------- | ---------------------------------------------------- |
| API         | http://localhost:3001/api/v1                         |
| Health      | http://localhost:3001/api/v1/health                  |
| Swagger     | http://localhost:3001/docs                           |
| MinIO API   | http://localhost:9000                                |
| MinIO panel | http://localhost:9001 (`minioadmin` / `minioadmin`)  |
| PostgreSQL  | localhost:5432 (`meningfermam` / `meningfermam_dev`) |
| Redis       | localhost:6379                                       |

Barcha portlar faqat `127.0.0.1` ga bog‘langan.

### Demo akkauntlar

| Rol    | Telefon       | Parol        |
| ------ | ------------- | ------------ |
| ADMIN  | +998901111111 | `Demo12345!` |
| FARMER | +998902222222 | `Demo12345!` |

Parol `.env` dagi `DEMO_PASSWORD` dan olinadi. Seed idempotent: mavjud
akkauntlarga tegmaydi, dublikat yaratmaydi.

### Komandalar

| Komanda               | Vazifasi                                                 |
| --------------------- | -------------------------------------------------------- |
| `npm run start:dev`   | Butun dev stack'ni build qilib ko‘tarish                 |
| `npm run dev:logs`    | API loglarini kuzatish                                   |
| `npm run dev:restart` | API container'ni qayta ishga tushirish                   |
| `npm run dev:down`    | Stack'ni to‘xtatish (ma’lumotlar saqlanadi)              |
| `npm run dev:reset`   | Stack'ni to‘xtatish va **barcha volume'larni o‘chirish** |
| `npm run start:watch` | Container ichida ishlatiladigan `nest start --watch`     |
| `npm run db:migrate`  | Yangi migratsiya yaratish (`prisma migrate dev`)         |
| `npm run db:deploy`   | Migratsiyalarni qo‘llash (`prisma migrate deploy`)       |
| `npm run db:seed`     | Demo seed                                                |
| `npm run db:generate` | Prisma client generatsiyasi                              |
| `npm run build`       | Production build (`dist/`)                               |
| `npm test`            | Unit testlar                                             |
| `npm run test:e2e`    | E2E testlar                                              |
| `npm run test:cov`    | Coverage                                                 |
| `npm run lint`        | oxlint                                                   |
| `npm run format`      | Prettier                                                 |

> `npm run start:dev` host'da chaqiriladi. Container ichida u chaqirilmaydi
> (rekursiya bo‘ladi) — container `npm run start:watch` ni ishlatadi.

### Qanday ishlaydi

- **Hot reload:** loyiha papkasi `/app` ga bind mount qilingan. Kod o‘zgarsa,
  NestJS container ichida o‘zi qayta kompilyatsiya qiladi — image qayta build
  qilinmaydi.
- **node_modules:** `api_node_modules` nomli volume'da turadi (macOS
  binary'lari Linux container'ga aralashmasligi uchun). `package.json` yoki
  `package-lock.json` o‘zgarsa, entrypoint buni sezadi va `npm install`
  qiladi.
- **Container ichidagi manzillar:** container ichida `localhost` — bu API
  container'ning o‘zi. Shuning uchun `DATABASE_URL`, `REDIS_URL`,
  `S3_ENDPOINT` compose servis nomlari (`postgres`, `redis`, `minio`) bilan
  yoziladi; `compose.dev.yaml` ularni har doim shu qiymatlarga o‘rnatadi.

### Port band bo‘lsa

Kompyuteringizda masalan lokal PostgreSQL 5432 da ishlayotgan bo‘lsa, `.env`
ga boshqa host portini yozing:

```env
POSTGRES_PORT=5433
```

Xuddi shunday: `API_PORT`, `REDIS_PORT`, `MINIO_PORT`, `MINIO_CONSOLE_PORT`.
Container ichidagi manzillar o‘zgarmaydi.

### Host'dan Prisma komandalari

Host'dan ishlatilganda `DATABASE_URL` `localhost` ga qaratilgan bo‘lishi kerak:

```bash
DATABASE_URL=postgresql://meningfermam:meningfermam_dev@localhost:5432/meningfermam npm run db:migrate
```

Yoki container ichida:
`docker compose -f compose.dev.yaml exec api npx prisma migrate dev`.

---

## Production

`Dockerfile` — multi-stage build. Yakuniy image:

- `appuser` (root emas) bilan ishlaydi;
- faqat `dist`, production `node_modules`, `prisma` va `docker/` ni o‘z ichiga
  oladi — source code va dev tool'lar yo‘q;
- seed bajarmaydi, demo akkaunt yaratmaydi (seed `NODE_ENV=production` da
  ishlashdan bosh tortadi);
- `NODE_ENV=production` va `SMS_PROVIDER=mock` bo‘lsa ishga tushmaydi;
- Swagger faqat `SWAGGER_ENABLED=true` bo‘lsa yoqiladi (default: o‘chiq);
- `/api/v1/health` orqali Docker `HEALTHCHECK`.

PostgreSQL, Redis va S3 production'da managed servislar (RDS, ElastiCache, S3).
S3 uchun `S3_ENDPOINT` va kalitlarni bo‘sh qoldirsangiz, AWS SDK IAM role'dan
foydalanadi.

### Secret'lar

Secret'lar image'ga, git'ga va `compose.prod.yaml` ga yozilmaydi. Ular serverda
AWS Secrets Manager'dan env faylga chiqariladi (default yo‘l
`/etc/meningfermam/api.env`, `API_ENV_FILE` bilan o‘zgartiriladi):

```bash
aws secretsmanager get-secret-value \
  --secret-id meningfermam/api/production \
  --query SecretString --output text \
  | jq -r 'to_entries[] | "\(.key)=\(.value)"' \
  | sudo install -m 600 /dev/stdin /etc/meningfermam/api.env
```

### Deploy

```bash
export API_IMAGE=<registry>/meningfermam-api:<git-sha>

# 1. Release command — yangi API ishga tushishidan oldin, bir marta
docker compose -f compose.prod.yaml run --rm migrate

# 2. API
docker compose -f compose.prod.yaml up -d api
```

Bitta instance'li sodda setup'da migratsiyani start paytida bajarish ham mumkin:
`RUN_MIGRATIONS_ON_START=true`.

---

## Hali qilinmagan ishlar

Kod hozirgi holatida quyidagilar hali yo‘q — keyingi bosqichlar uchun:

- **Ro‘yxatdan o‘tish / SMS OTP** — `SMS_PROVIDER` sozlamasi bor, lekin SMS
  moduli va register endpoint'i hali yozilmagan.
- **Rol bo‘yicha ruxsat** — JWT'da `role` bor, lekin `RolesGuard` /
  `@Roles()` dekoratori yo‘q.
- **Refresh token va logout** — hozir faqat bitta access token.
- **Login uchun rate limiting** — brute-force himoyasi yo‘q (Redis tayyor).
- **Fayl yuklash endpoint'lari** — `S3Client` sozlangan, lekin upload /
  presigned URL API'si yo‘q.
- **Ferma domeni** — ferma, hayvon/ekin va boshqa biznes modellari hali
  qo‘shilmagan.
- **Testlar** — hozircha faqat shablon `AppController` testlari bor; auth va
  health uchun testlar kerak.
- **Lock fayllar** — repoda ham `package-lock.json`, ham `pnpm-lock.yaml` bor;
  Docker `npm` ishlatadi, shuning uchun `pnpm-lock.yaml` ni olib tashlash
  mumkin.
