# Plan de Desarrollo — PollStream

Plataforma de encuestas públicas/privadas con votación segura y resultados en tiempo real (SSE/WS). Stack: **SvelteKit 2, Svelte 5, TypeScript, Prisma, PostgreSQL, Docker y Tailwind CSS 4**.

> Este plan es un **checklist paso a paso** desde cero hasta producción. Úsalo como guía operativa.

---

## 0) Requisitos previos (local)
- [x] Node 20+ y PNPM/NPM
- [ ] Docker + Docker Compose
- [x] Git
- [x] Editor con ESLint/Prettier (VSCode recomendado)

---

## 1) Bootstrap del repo
- [ ] `git init && git branch -m main`
- [ ] Crear proyecto: `npm create svelte@latest pollstream` → elegir **SvelteKit + TS**.
- [ ] Instalar dependencias base:
  - [ ] `npm i -D eslint prettier @typescript-eslint/{eslint-plugin,parser} zod`
  - [ ] `npm i @prisma/client prisma`
  - [ ] `npm i @sveltejs/kit@latest`
- [ ] Tailwind 4: `npm i -D tailwindcss postcss autoprefixer` y configurar según docs.
- [ ] Añadir scripts a `package.json`:
  - [ ] `dev`, `build`, `preview`, `lint`, `format`, `test`, `e2e`, `seed`, `dev:docker`

---

## 2) Docker & Base de datos
- [ ] Crear `docker-compose.yml` con servicios **db (Postgres 16)** y **app**.
- [ ] Variables compuestas:
  - [ ] `DATABASE_URL=postgresql://postgres:postgres@db:5432/pollstream`
  - [ ] `TZ=America/Argentina/Buenos_Aires`
- [ ] `docker compose up -d db` para levantar Postgres.
- [ ] Crear `.env` y `.env.example` con:
  - [ ] `DATABASE_URL`, `ADMIN_TOKEN` o `ADMIN_PASSWORD`, `AUTH_SECRET`
  - [ ] `VOTER_SALT`, `RATE_LIMIT_*`
  - [ ] `RESULTS_PUSH=SSE|WS`, `PUBLIC_BASE_URL`, `CAPTCHA_*`, `TZ`

---

## 3) Prisma: modelos y migraciones
- [ ] `npx prisma init`
- [ ] Definir modelos en `prisma/schema.prisma`:
  - [ ] **User, Survey, Question, Option, Vote, VoteOption, AccessToken, AuditLog**
- [ ] Índices y constraints:
  - [ ] `@@unique([questionId, tokenId])` (1 voto por token)
  - [ ] `@@unique([questionId, voterHash])` (1 voto por dispositivo si aplica)
- [ ] `npx prisma migrate dev -n init`
- [ ] Crear `prisma/seed.ts`:
  - [ ] Usuario admin
  - [ ] Encuesta demo (3 preguntas: SINGLE/MULTIPLE/YESNO)
  - [ ] 50 votos ficticios + 20 tokens
- [ ] `npm run seed`

---

## 4) Estructura del proyecto
```
src/
  hooks.server.ts
  lib/
    db/index.ts
    auth/{admin.ts,captcha.ts,csrf.ts}
    realtime/{broker.ts,sse.ts,ws.ts}
    validation/{survey.ts,question.ts,vote.ts}
    utils/{hash.ts,rateLimit.ts,results.ts}
    components/{ChartBars.svelte,ChartLines.svelte,ResultTotal.svelte,Turnstile.svelte}
  routes/
    api/...      # endpoints server
    admin/...    # UI protegida
    [slug]/+page.svelte  # UI pública de encuesta
```
- [ ] Crear cada carpeta/archivo base con `TODO:` inicial para guiar implementación.

---

## 5) Hooks y auth admin
- [ ] `hooks.server.ts`: inyectar `locals` con prisma y broker realtime.
- [ ] Auth admin:
  - [ ] Opción A: **ADMIN_TOKEN** (header/cookie firmada)
  - [ ] Opción B: **ADMIN_PASSWORD** + cookie firmada (`HttpOnly`, `Secure`, `SameSite=Lax`)
- [ ] Guardia en `src/routes/admin/+layout.server.ts`:
  - [ ] redirige a `/admin/login` si no hay sesión válida

---

## 6) Validación (Zod)
- [ ] `voteInput`:
  - [ ] `questionId: string`
  - [ ] `optionIds: string[]`
  - [ ] reglas: `SINGLE` (exactamente 1), `YESNO` (exactamente 1 y válida), `MULTIPLE` (1..=maxChoices, únicas)
- [ ] `surveyInput`, `questionInput`, `optionInput` para CRUD admin

---

## 7) Anti‑abuso y privacidad
- [ ] Rate‑limit IP/UA por ruta (ej. 60/min) en `utils/rateLimit.ts`
- [ ] CAPTCHA opcional:
  - [ ] Turnstile/hCaptcha integrado en `/api/votes` si `CAPTCHA_REQUIRED=true`
- [ ] CSRF:
  - [ ] Admin con `csrf` token y `SameSite=Lax`
- [ ] Sanitización:
  - [ ] Entradas validadas con Zod
  - [ ] Escapar HTML en renderizados dinámicos
- [ ] Hash de dispositivo:
  - [ ] `voterHash = sha256(VOTER_SALT + ip + ua + stableDeviceIdCookie)` si `ONE_VOTE_PER_DEVICE=true`
- [ ] `AuditLog` para acciones admin y votos (sin PII)

---

## 8) API — endpoints mínimos
- [ ] `GET /api/health` y `GET /api/version`
- [ ] `POST /api/surveys` (admin) — crear encuesta
- [ ] `GET /api/surveys` — listar públicas (metadatos y reglas de visibilidad de resultados)
- [ ] `GET /api/surveys/[id]` — detalle público (sin exponer conteos si no corresponde)
- [ ] `PUT /api/surveys/[id]` (admin) — editar
- [ ] `DELETE /api/surveys/[id]` (admin) — eliminar
- [ ] `POST /api/votes` — registrar voto público o privado
  - [ ] verifica `status` y `closeAt`
  - [ ] aplica idempotencia por `tokenId` y/o `voterHash`
  - [ ] guarda `Vote` + `VoteOption[]` en transacción
  - [ ] `broadcast(surveyId, results)`
- [ ] `POST /api/tokens` (admin) — generar lote
- [ ] `GET /api/tokens/[surveyId].csv` (admin) — descargar CSV
- [ ] `GET /api/surveys/[id]/export.csv` — resultados por pregunta/opción
- [ ] `GET /api/surveys/[id]/results.sse` — stream SSE
- [ ] `GET /api/surveys/[id]/results.ws` — handler WS (si `RESULTS_PUSH=WS`)

---

## 9) Realtime (SSE/WS)
- [ ] `lib/realtime/broker.ts` interfaz `RealtimeBroker`
- [ ] `lib/realtime/sse.ts`:
  - [ ] `Map<surveyId, Set<controller>>`
  - [ ] `subscribe()` y `broadcast()`
- [ ] `lib/realtime/ws.ts`:
  - [ ] `Map<surveyId, Set<WebSocket>>`
  - [ ] `wsHandler()` y `broadcast()`
- [ ] `brokerFromEnv(RESULTS_PUSH)` en `hooks.server.ts`

---

## 10) Reglas de visibilidad de resultados
- [ ] ALWAYS: el endpoint incluye agregados desde el inicio
- [ ] AFTER_VOTE: setear cookie `voted:<surveyId>=true`; exigirla para resultados
- [ ] AFTER_CLOSE: exponer solo si `status=CLOSED` o `closeAt<=now()`

---

## 11) UI pública (encuesta)
- [ ] Ruta `/[slug]/+page.svelte`:
  - [ ] Mostrar título, descripción, estado y `closeAt`
  - [ ] Renderizar preguntas según `type` y `maxChoices`
  - [ ] Validación client-side (límite de opciones)
  - [ ] Envío de voto → feedback (ok/idempotente/errores)
  - [ ] Conectar SSE/WS para actualizar:
    - [ ] `ChartBars` y/o `ChartLines`
    - [ ] Totales por opción y total general
  - [ ] Respetar reglas de visibilidad

---

## 12) UI admin
- [ ] `/admin/login` — formulario
- [ ] `/admin` — dashboard: encuestas, métricas básicas
- [ ] `/admin/surveys` — listado CRUD
- [ ] `/admin/surveys/new` — crear
- [ ] `/admin/surveys/[id]` — editar, **duplicar**, exportar CSV, generar tokens

---

## 13) Pruebas
- [ ] **Vitest** (unidad):
  - [ ] `validateVoteSelection(question, optionIds)`
  - [ ] `oneVotePerDevice` respeta constraints
  - [ ] Token privado: idempotencia (no duplica votos)
- [ ] **Playwright** (E2E):
  - [ ] Crear encuesta → votar público → ver resultados según modo
  - [ ] SSE/WS recibe updates en vivo
  - [ ] Export CSV entrega datos correctos

---

## 14) Observabilidad y errores
- [ ] Middleware de **manejo centralizado de errores** (logging + respuesta consistente)
- [ ] Endpoint `/health` (DB ok, broker ok) y `/version` (tag/commit)
- [ ] Logs en `AuditLog` + consola (nivel INFO/WARN/ERROR)

---

## 15) Rendimiento y DB
- [ ] Índices: `surveyId`, `questionId`, `optionId` en tablas clave
- [ ] Queries agregadas para resultados (usar `count` por `optionId`)
- [ ] Evitar N+1 (prisma `include/select` adecuados)
- [ ] Paginar listados admin

---

## 16) CI/CD
- [ ] GitHub Actions:
  - [ ] `lint`, `test`, `e2e` (headless)
  - [ ] build y artefactos
- [ ] Publicar imagen Docker (opc.)
- [ ] Gate de calidad: no merge si fallan tests

---

## 17) Deploy (Vercel/Fly.io/Railway)
- [ ] Elegir proveedor y crear app
- [ ] Configurar variables de entorno
- [ ] Estrategia de migraciones:
  - [ ] `prisma migrate deploy` en arranque
- [ ] Configurar dominio y SSL
- [ ] Probar `/health` y flujo de voto

---

## 18) Post‑lanzamiento
- [ ] Monitoreo de errores y performance
- [ ] Recolección de feedback (encuesta meta)
- [ ] Roadmap de mejoras:
  - [ ] Editor avanzado de encuestas
  - [ ] Multi‑idioma
  - [ ] Roles y permisos
  - [ ] Firma criptográfica de resultados (opcional)

---

## 19) Scripts recomendados (`package.json`)
- [ ] `"dev": "vite dev"`
- [ ] `"dev:docker": "docker compose up -d db && vite dev"`
- [ ] `"build": "vite build"`
- [ ] `"preview": "vite preview"`
- [ ] `"lint": "eslint ."`
- [ ] `"format": "prettier -w ."`
- [ ] `"migrate": "prisma migrate dev"`
- [ ] `"migrate:deploy": "prisma migrate deploy"`
- [ ] `"seed": "ts-node prisma/seed.ts"`
- [ ] `"test": "vitest run"`
- [ ] `"e2e": "playwright test"`

---

## 20) Checklist final antes de producción
- [ ] Admin detrás de cookie segura + rate‑limit adicional en rutas sensibles
- [ ] `RESULTS_PUSH` verificado (SSE o WS) bajo carga
- [ ] CAPTCHA operativo si habilitado
- [ ] Backups de DB y política de retención
- [ ] Política de privacidad (sin PII en logs/resultados)
- [ ] README actualizado con instalación, migraciones, seeds y troubleshooting

---
