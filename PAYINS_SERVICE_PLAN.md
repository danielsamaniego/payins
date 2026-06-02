# Plan: **Payins** — Orquestador de Pasarelas Independiente

**Fecha:** 2026-04-17
**Autor:** daniel.samaniego@kunfupay.com
**Estado:** Draft v2 (reescritura desde cero, sin herencia del sistema de referencia)
**Inspiración arquitectónica estricta:** [Wallet](../Wallet/) (DDD + Hexagonal + CQRS sobre Hono, PostgreSQL + Prisma, montos enteros, 100% coverage)
**Inspiración funcional:** Stripe, Adyen, Mollie, Checkout.com, MercadoPago — extrayendo lo bueno y mejorando las carencias

---

## 0. TL;DR

**Payins** es un orquestador de pagos de entrada, multi‑tenant, multi‑país, multi‑moneda, multi‑proveedor, totalmente independiente. Recibe órdenes de pago de cualquier consumidor y las ejecuta contra múltiples proveedores (Stripe, Ebanx, …) por múltiples métodos (card, Pix, Boleto, PSE, Yape, Nequi, PLIN, MercadoPago, bank transfer, …) en múltiples países.

Maneja, con el mismo peso, estas responsabilidades (visión completa del servicio). El marcador **[hoy]** indica lo ya construido; **[roadmap]** lo planificado aún no implementado:

1. **Pagos one‑time** (con sus reintentos, refunds, disputas). **[hoy]** Payment (agnóstico) + Attempt + Refund existen; reintento/fallback = nuevo Attempt sobre el mismo Payment. Las disputas se **parsean pero no se accionan** en v1 **[roadmap]**.
2. **Suscripciones** en sus tres arquetipos: auto‑proveedor (Stripe), managed (Yape/Nequi/tarjeta guardada), reminder. **[roadmap]**
3. **Payment Links** — URLs reutilizables que generan checkouts bajo demanda. **[roadmap]**
4. **Webhooks entrantes** — recibe, verifica firma, normaliza, deduplica. **[hoy]** El dispatcher acciona CAPTURED/FAILED/REFUNDED(+parcial); chargebacks y checkout‑expired se parsean pero no se accionan en v1.
5. **Webhooks salientes** — emite eventos firmados a endpoints que los integradores registran. **[hoy]** Enum cerrado de 5 eventos; encolado **directo** (el outbox `DomainEvent` está construido pero dormido).
6. **Contratos comerciales** — por `Account`, con términos de comisión y markup por combinación método × país × moneda. **[hoy]**
7. **Instrumentos guardados** — tokens de tarjeta y mandates de wallet, reusables. **[roadmap]**

**Principios no‑negociables** (idénticos a Wallet):

- **Montos siempre enteros** (`BigInt` en código, `BIGINT` en DB, minor units). Nada de `number` ni floats.
- **Porcentajes en basis points** (entero 0‑10000, donde 10000 = 100%). Nada de decimales en comisiones.
- **IDs UUID v7** (RFC 9562), generados en código vía `IIDGenerator`; la DB nunca genera IDs; nunca UUID v4. Los prefijos semánticos (`pay_…`, `sub_…`, `inv_…`) se usan solo como pista de legibilidad en ejemplos; el valor almacenado es un string UUID v7.
- **Timestamps Unix‑ms (BigInt)** como representación única interna y en DB; ISO‑8601 UTC solo en los payloads públicos de webhooks salientes (§9).
- **ISO‑3166‑1 alpha‑2** para países, **ISO‑4217** para monedas, validados por Zod.
- **Idempotencia end‑to‑end** (header `Idempotency-Key` obligatorio, dedupe de webhooks entrantes).
- **Optimistic locking** (`version` en todos los aggregates con concurrencia).
- **Zero‑knowledge** del consumidor. Ningún archivo del repo referencia productos externos.
- **100% coverage** enforced. BDD unit + E2E contra mocks/sandboxes.

**Limpieza desde cero:** el vocabulario, las abstracciones y el modelo de datos se diseñan partiendo de convenciones profesionales (Stripe/Adyen/Mollie). **Ningún nombre, tabla, ni enum** del sistema de referencia (`_payments`, `_paymentsIntegration` en Kunfupay) se hereda. Esa base solo se usa como insumo de análisis para no repetir sus deudas.

---

## 1. Objetivos y no‑objetivos

### 1.1 Objetivos v1

| # | Objetivo |
|---|---|
| O1 | Servicio independiente (repo propio, DB propia, despliegue propio), platform‑agnostic. |
| O2 | Abstracción **Provider ↔ PaymentMethod ↔ Country ↔ Currency ↔ Flow ↔ Recurrence** correcta desde el día 1, con registry declarativo de capabilities. |
| O3 | Dos proveedores activos en v1: **Stripe** y **Ebanx**. MercadoPago se sirve como método de pago (`mercadopago_wallet`) a través de Ebanx, sin integración directa. Arquitectura abierta a añadir Adyen, Worldpay, dLocal, Checkout.com o partnership directo con MP sin tocar dominio. |
| O4 | Pagos one‑time, suscripciones (3 arquetipos), payment links, refunds, disputas como ciudadanos de primera clase. |
| O5 | Contratos con términos comerciales por tupla (método × país × moneda), con prioridad por especificidad. |
| O6 | Webhooks entrantes (verificación, parseo, dedupe, dispatch) y salientes (firma HMAC, reintentos exponenciales, DLQ). |
| O7 | Observabilidad: canonical log, tracking‑id, sensitive‑key filter, OpenAPI auto‑generado. |
| O8 | 100% test coverage con BDD unit + E2E contra mocks/sandboxes; security suite ≥12 categorías. |
| O9 | Multi‑tenant robusto desde la fase 1 (tests de aislamiento obligatorios). |

### 1.2 No‑objetivos v1

- **Integración con ningún consumidor concreto** (Kunfupay u otro). V1 termina cuando pasa su propia sandbox; no cuando alguien lo usa.
- **Frontend / UI de checkout (revisado):** Payins **sí** entrega ahora una **UI de checkout hosted** (`apps/checkout`) y un **dashboard superadmin** (`apps/dashboard`) en un repo independiente (`Kunfupay-Payins-Front/`, ver §16). Lo único que sigue siendo responsabilidad del integrador es **embeber esa UI en su propio sitio**, vía el widget embebible (`payins.js` + iframe con `frame-ancestors` restringido por plataforma).
- **Payouts.** Fuera del perímetro.
- **Settlement de balances.** Es responsabilidad del consumidor, vía webhook saliente (típicamente contra su Wallet o ledger propio).
- **Reporting / BI.** Exponemos queries bien modeladas; dashboards los arma el consumidor.
- **Acquiring propio / licencias regulatorias.** Somos orquestador, no adquirente.
- **Proveedores legacy** (Astropay, Redsys, PayPal, Payretailers, dLocal Go, Monei, Payid19, Ibis, VPay). Se evalúan en v2 según demanda real.

### 1.3 Scope de proveedores, métodos y países en v1

Datos de referencia **en código, no en DB**: proveedores, métodos de pago, países y monedas viven como enums TypeScript + const maps en `utils/kernel/` (`catalog/`, `money/`). No hay tablas `provider`/`payment_method`/`country`/`currency` ni "seed" de catálogo (ver datamodel §3). Las columnas tipo `provider_slug` / `payment_method_slug` son `TEXT` validado por Zod en el boundary, nunca foreign‑keyed.

**Proveedores (v1 activos):**

| slug | display | países típicos |
|---|---|---|
| `stripe` | Stripe | GLOBAL (priorizamos EU, US, LATAM donde aplica) |
| `ebanx` | Ebanx | AR, BO, BR, CL, CO, EC, MX, PE, UY |

MercadoPago **no** es un provider separado: se sirve como método `mercadopago_wallet` a través de Ebanx (ver matriz abajo).

**Métodos (taxonomía lógica, agnóstica de proveedor):**

| slug | category | countries típicos |
|---|---|---|
| `card` | CARD | GLOBAL |
| `pix` | INSTANT_TRANSFER | BR |
| `boleto` | VOUCHER | BR |
| `pse` | BANK_TRANSFER | CO |
| `yape` | WALLET | PE |
| `nequi` | WALLET | CO |
| `plin` | WALLET | PE |
| `mercadopago_wallet` | WALLET | AR, BR, CL, CO, MX, PE, UY |
| `bank_transfer` | BANK_TRANSFER | varios |

**Matriz `(provider × method × country × currency)` en v1** — seed inicial:

| provider | method | country | currency | flow | recurrence |
|---|---|---|---|---|---|
| stripe | card | GLOBAL | USD, EUR, GBP, BRL, MXN, … | REDIRECT, ONSITE_TOKEN | NONE, AUTO_PROVIDER |
| ebanx | card | AR, BO, BR, CL, CO, EC, MX, PE, UY | local | REDIRECT, ONSITE_TOKEN | NONE, MANAGED |
| ebanx | mercadopago_wallet | AR, BR, CL, CO, MX, PE, UY | local | REDIRECT | NONE |
| ebanx | pix | BR | BRL | REDIRECT_ASYNC | NONE |
| ebanx | boleto | BR | BRL | REDIRECT_ASYNC | NONE |
| ebanx | pse | CO | COP | REDIRECT_ASYNC | NONE |
| ebanx | yape | PE | PEN | ENROLLMENT | NONE, MANAGED |
| ebanx | nequi | CO | COP | ENROLLMENT | NONE, MANAGED |
| ebanx | plin | PE | PEN | ENROLLMENT | NONE, MANAGED |

Esta matriz se materializa en la tabla `ProviderCapability` (DB), poblada por tenant. Añadir una celda habilitada es un `INSERT` en `ProviderCapability`, no un cambio de código. (El catálogo de slugs de provider/método y las listas de país/moneda sí son código — ver datamodel §3.)

---

## 2. Lecciones del sistema de referencia (solo análisis)

El sistema actual de payins en Kunfupay ([src/_payments/back/](../Kunfupay/Kunfupay-Nextjs/src/_payments/back/), [src/_paymentsIntegration/back/](../Kunfupay/Kunfupay-Nextjs/src/_paymentsIntegration/back/)) se usa exclusivamente como **caso de estudio**. De ahí extraemos:

**Qué conservamos (conceptualmente, con nombres nuevos):**
- Tres arquetipos de recurrencia (auto‑proveedor, managed, reminder).
- Separación entre "catálogo global" y "configuración por cuenta".
- Adapters por proveedor con verificación de firma aparte del parseo.
- Normalización de eventos entrantes a un enum único.
- Outbound webhook dispatcher con reintentos.

**Qué descartamos (y por qué):**

| Concepto descartado | Razón |
|---|---|
| `Gateway` como nombre | Demasiado cargado, ambiguo. Los PSPs profesionales se llaman **providers**. |
| `GatewayContract` como "credenciales de un proveedor" | El nombre "contract" es valioso para los contratos comerciales reales. Renombramos credenciales a **ProviderConnection**. |
| `GatewayMethod` | Colapsa demasiadas dimensiones en un único registro. Lo separamos en **PaymentMethod** (catálogo) + **ProviderCapability** (habilitación). |
| `PaymentTemplate`, `UserPaymentTemplate`, `ContractTemplate` | Tres entidades que hacen lo mismo mal. Sustituidas por **Contract** + **ContractTerm** (comercial) y **ProviderCapability** (técnico). |
| `DirectPayment` | Nombre confuso. **Payment** a secas. |
| `ClientCardToken` + `ClientWalletEnrollment` | Dos tablas para la misma idea. Unificadas en **Instrument** con campo `type`. |
| `CheckoutSession` + `SubscriptionSession` | Se unifican en **Checkout** con campo `mode`. |
| `Sale` acoplado | Payins no conoce el modelo de negocio del consumidor; usa `reference` opaca. |
| Factory de processors hardcoded en código | Sustituida por **ProviderAdapterRegistry** + módulo por provider compuesto de handlers por flujo (§6). |
| Cron de suscripciones fuera del bounded context | En v1 el cron vive dentro de `subscription`. |
| `SubscriptionManagementType` como único campo de control | Desagregado en **FlowType** + **RecurrenceStrategy** (2 dimensiones independientes). |

**Deudas técnicas a no repetir:**
1. Factory solo resuelve un proveedor.
2. Sin DI explícita en use cases.
3. Webhook handler comentado.
4. Sin idempotency keys.
5. Mezcla de verificación de firma + parseo en el mismo adapter.
6. Settlement hardcoded en use case de pago.
7. Retry logic hardcoded en cron.
8. Sin validación de aislamiento multi‑tenant en la capa de persistencia.
9. Amounts usando tipos imprecisos en algunos lugares.
10. Sin auditoría de eventos de dominio (no se puede reconstruir "qué pasó" post‑mortem).

---

## 3. Principios arquitectónicos (fidelidad a Wallet)

### 3.1 Stack

| Capa | Tecnología |
|---|---|
| Lenguaje | TypeScript 5.x strict |
| HTTP framework | Hono + hono-openapi |
| DB | PostgreSQL 16 |
| ORM | Prisma |
| Validación | Zod en todos los bordes (API, env, webhooks) |
| Logging | Pino + `SensitiveKeysFilter` + `SafeLogger` |
| Testing | Vitest (unit + e2e) + `vitest-mock-extended`, BDD Given/When/Then |
| Lint/Format | Biome |
| Container | Docker + docker-compose (dev + test aislados en puertos distintos) |
| Crypto credenciales | libsodium sealed boxes (master key en env / KMS) |
| Hashing secrets | argon2id (secrets de API key) |
| IDs | UUID v7 (uuidv7), generados en app vía `IIDGenerator`; nunca UUID v4 |
| Lock distribuido | Redis vía ioredis (tcp) o `@upstash/redis` (rest) — capa externa de concurrencia (`LockRunner`) |
| Deploy | Node 22 Alpine multi‑stage |
| Package mgr | pnpm |

### 3.2 Patrón (copia textual del patrón Wallet)

- **DDD + Hexagonal + CQRS**.
- **Domain** puro: entities, value objects, aggregate roots, errors de dominio, interfaces de puertos. Cero dependencias de libs externas (ni Prisma, ni Hono, ni axios).
- **Application**: comandos (writes) y queries (reads). Orquestan dominio a través de puertos.
- **Infrastructure**: adapters concretos (Prisma repositories, HTTP clients de proveedores, webhook verifiers, outbound dispatcher).
- **Presentation**: handlers HTTP (Hono), schemas Zod, specs OpenAPI.
- **No event sourcing**. Sí un **event log** persistente para auditoría y, **a futuro**, para alimentar webhooks salientes. **Estado hoy:** el outbox (`common/events`: entidad `DomainEvent` + puerto `IEventPublisher` + `PrismaEventStore`, tabla `domain_events`) está **construido pero no cableado** — ningún use case emite `DomainEvent` todavía. En v1 los webhooks salientes se encolan **directamente** por el dispatcher de webhooks entrantes (`OutboundWebhookDelivery.eventId` es nullable y queda `null` en este camino directo). El proyector evento→delivery es trabajo futuro (§15.2 Fase 2).
- **Concurrencia de dos capas** (idéntica a Wallet): capa externa = lock distribuido `LockRunner` (`utils/application/lock.runner.ts`, respaldado por Redis/Upstash) que envuelve la transacción; capa interna = optimistic locking (`version`) + aislamiento `Serializable`. El `TransactionManager` reintenta la capa interna hasta **3 intentos** (backoff exponencial determinista, 30ms / 60ms, sin jitter; `MAX_RETRIES=3` en `prisma.transaction.manager.ts`); agotado → `409 VERSION_CONFLICT`. Timeout de espera del lock → `409 LOCK_CONTENDED`. Lock FIRST, tx SECOND. Prefijos de key: `payment-lock:<id>`, `subscription-lock:<id>`, `checkout-lock:<id>`, `invoice-lock:<id>`. Para comandos keyed por recurso derivado (un refund recibe `paymentId`), se valida la propiedad del tenant **fuera del lock** antes de bloquear. Si Redis no está disponible, el runner cae transparentemente al optimistic locking.
- **TrackingId + canonical log** por request.
- **Idempotency middleware** atómico (acquire → execute → complete).
- **AppError + ErrorKind + Code** homogéneo.

### 3.3 Reglas de oro

1. Los aggregates son **inmutables**: `withStatus()`, `withAttempt()` devuelven nueva instancia.
2. Amounts siempre `bigint` (minor units). Conversión en boundary: input/output via Zod transform a/desde string decimal si el consumidor lo prefiere (pero DB y código interno: siempre entero).
3. Percentajes en **basis points** (entero, 0‑10000). `250` = `2.50%`. Redondeo half‑even al convertir a decimal en formato de salida.
4. Multi‑tenant: el `Platform` es el tenant. Toda query va filtrada por `platformId` y, en entidades account‑scoped, también por `accountId` (ambos `NOT NULL`). Tests de aislamiento por endpoint: aislamiento de Platform **y** de Account.
5. Timestamps Unix‑ms (`BigInt`) como representación única; nunca `Date.now()` en dominio; recibe un `IClock` inyectado.
6. Concurrencia de dos capas (ver §3.2): optimistic lock con `version` + `LockRunner` distribuido en aggregates con concurrencia (Payment, Subscription, Checkout, Invoice, Contract, Instrument).
7. Ningún archivo del dominio importa `@prisma/client`, `hono`, `axios`.
8. No hay herencia de clases en dominio; composición + interfaces + discriminated unions.

---

## 4. Bounded contexts (estricto DDD)

**9 bounded contexts de pago + 1 BC cross-cutting `iam` (admin auth) + 3 features
cross-cutting en `common/`**. La convención de carpetas es idéntica a Wallet: cada BC y
cada `common/*` respeta `domain/ / application/ / infrastructure/`. Reglas de import
detalladas viven en [Kunfupay-Payins-Back/docs/datamodel.md §3.1](Kunfupay-Payins-Back/docs/datamodel.md)
("Layering & placement rules").

> **`iam` (Identity & Access Management):** BC que **posee la autenticación del dashboard
> superadmin** (`apps/dashboard`). Expone un puerto **swappable `IAdminAuthenticator`**
> (login, validar sesión, resolver rol) con **adapter nativo** ahora (`AdminUser` +
> `AdminSession` + `Role`, password hashing **argon2id**, tokens de sesión seguros) y
> swappable más tarde a un proveedor (Auth0/Clerk/Firebase/Supabase) sin tocar dominio ni
> dashboard. **Hoy solo existe un `ADMIN_API_KEY` estático** (§13) para las rutas
> `/v1/admin/*`; `iam` es **requisito previo del dashboard** (§16). Roles: `superadmin` ahora;
> roles platform‑scoped en Fase 2. Datamodel: [Kunfupay-Payins-Back/docs/datamodel.md](Kunfupay-Payins-Back/docs/datamodel.md) § `iam`.

```
src/
├── app.ts / config.ts / index.ts / wiring.ts
│
├── utils/                              # toolkit puro; NO use cases; NO tablas
│   ├── kernel/                         # domain-safe, zero deps
│   │   ├── appError.ts
│   │   ├── context.ts
│   │   ├── bigint.ts
│   │   ├── listing.ts
│   │   ├── clock.port.ts
│   │   ├── observability/              # ILogger port, CanonicalAccumulator
│   │   ├── money/                      # Money VO, Currency enum, Country enum
│   │   └── catalog/                    # ProviderSlug, PaymentMethodSlug enums + maps
│   ├── application/                    # ports de nivel aplicación
│   │   ├── cqrs.ts
│   │   ├── id.generator.ts
│   │   ├── transaction.manager.ts
│   │   ├── event-publisher.port.ts
│   │   └── provider-ports/             # IRedirectFlowHandler, IWebhookVerifier, ...
│   └── infrastructure/                 # implementaciones concretas (Pino, Prisma, Hono, …)
│
├── common/                             # features cross-cutting con DDD completo (NO BCs)
│   ├── idempotency/
│   ├── events/                         # DomainEvent (outbox append-only)
│   └── inboundWebhooks/                # InboundWebhookEvent (ingest + dedupe)
│
├── platform/                           # BC 1: Platform + Account
├── routing/                            # BC 2: ProviderConnection + ProviderCapability
├── contract/                           # BC 3: Contract + ContractTerm
├── customer/                           # BC 4: Customer
├── instrument/                         # BC 5: Instrument
├── checkout/                           # BC 6: Checkout + PaymentLink
├── payment/                            # BC 7: Payment + Attempt + Refund + Dispute
├── subscription/                       # BC 8: Plan + Subscription + Invoice
├── webhook-outbound/                   # BC 9: WebhookEndpoint + WebhookDelivery
│
└── provider-adapters/                  # módulos por provider, handlers por flujo
    ├── stripe/
    └── ebanx/
```

**Reglas de dependencia (acíclicas)** — enforzadas por
[`Kunfupay-Payins-Back/scripts/check-layer-violations.cjs`](Kunfupay-Payins-Back/scripts/check-layer-violations.cjs):

| Layer | Puede importar | Prohibido |
|---|---|---|
| `utils/kernel/` | TS stdlib | todo lo demás |
| `utils/application/` | `utils/kernel/` | infra, libs, BCs, `common/*` |
| `utils/infrastructure/` | `utils/kernel/`, `utils/application/`, libs | BCs, `common/*/domain/` |
| `common/*/domain/` | `utils/kernel/` | libs, BCs, otros `common/*` |
| `common/*/application/` | `utils/kernel/`, `utils/application/`, propia domain | libs, BCs, otros `common/*` |
| `common/*/infrastructure/` | lo anterior + libs | BCs |
| `<bc>/domain/` | `utils/kernel/` | libs, infra, otros BCs, `common/*/domain/` |
| `<bc>/application/` | `utils/kernel/`, `utils/application/`, propia domain, `common/*/application/` como cliente | libs, otros BCs |
| `<bc>/infrastructure/` | lo anterior + libs | otros BCs |
| `provider-adapters/<slug>/` | `utils/kernel/`, `utils/application/provider-ports/`, libs del SDK | **cualquier BC**, otros `provider-adapters/<otro>/`, `common/*` |

La última fila es la **Anti-Corruption Layer estructural**: shapes de SDK de provider
nunca cruzan a un BC.

---

## 5. Modelo de dominio (esquemas detallados)

Convenciones comunes a todos los esquemas:

- `id`: string **UUID v7** (RFC 9562), generado en app vía `IIDGenerator`; nunca lo genera la DB; nunca UUID v4. Los prefijos semánticos (`plat_…`, `acc_…`, `pay_…`, `sub_…`, `inv_…`, `ins_…`, `con_…`, `cus_…`, `cap_…`, `ctr_…`, `chk_…`, `lnk_…`, `evt_…`, `whd_…`, `iwe_…`) se mantienen **solo como pista de legibilidad** en los ejemplos; el valor almacenado es un string UUID v7.
- `createdAt`, `updatedAt`: `BIGINT (Unix ms)`.
- Money: `amount BIGINT NOT NULL` (minor units) + `currency CHAR(3) NOT NULL`.
- Percentajes: `INT NOT NULL` en basis points.
- Soft‑delete: **no**. Eliminación es `status = ARCHIVED` cuando aplica.
- `version INT NOT NULL DEFAULT 0` en aggregates con concurrencia.

### 5.1 Bounded context: `platform` (Platform + Account)

El **`Platform` es el tenant** y se autentica vía API key (no hay tabla `ApiCredential` separada: los campos de la key viven **en** el `Platform`). Todo `Platform` posee **uno o más `Account`s**; al registrar el Platform se auto‑crea un `Account` por defecto. El `Account` es la **raíz de propiedad** de contracts, customers, instruments, checkouts, payments, subscriptions y plans. **No existe un flag `mode`**: simple vs agregador emerge del número de Accounts. Toda entidad account‑scoped lleva `platformId` **y** `accountId` como FKs `NOT NULL`.

#### Platform

| campo | tipo | notas |
|---|---|---|
| id | `plat_UUIDv7` | PK |
| name | `TEXT` | |
| apiKeyHash | `TEXT` | argon2id del secret de la key actual |
| apiKeyId | `TEXT UNIQUE` | prefijo público visible (`pk_live_…` / `pk_test_…`) |
| status | `ENUM` | `ACTIVE \| SUSPENDED \| REVOKED` (revoked es terminal) |
| livemode | `BOOLEAN` | `false` = sandbox, `true` = producción |
| createdAt, updatedAt | `BIGINT (Unix ms)` | |

**Invariante:** la última credencial de API activa no puede auto‑revocarse vía API pública (bloqueo auto‑lockout). Todo `Platform` posee **al menos un** Account.

#### Account

Representa a un merchant dentro de un Platform. **No tiene API key propia**; el Platform autentica y especifica `accountId` por operación. Es la raíz de propiedad de contracts, customers, instruments, plans, payments, subscriptions, checkouts y payment links.

| campo | tipo | notas |
|---|---|---|
| id | `acc_UUIDv7` | PK |
| platformId | FK → `Platform` | |
| externalReference | `TEXT` | id propio del Platform para el account; para el default auto‑creado es `"default"` |
| displayName | `TEXT` | |
| status | `ENUM` | `ACTIVE \| SUSPENDED \| ARCHIVED` |
| country | `CHAR(2)` nullable | hint de routing |
| email | `TEXT` nullable | para notificaciones de disputa / compliance |
| metadata | `JSONB` | ≤ 4KB |
| version | `INT` | |
| createdAt, updatedAt | `BIGINT (Unix ms)` | |

`UNIQUE (platformId, externalReference)`.

**Invariantes:**
- Un Platform nunca queda con cero Accounts: siempre debe quedar al menos uno `status != ARCHIVED`.
- Toda escritura account‑scoped verifica `account.platformId = platform.id` (aislamiento de Account — categoría de test de seguridad).
- Un Account `ARCHIVED` no puede ser destino de nuevos Payments/Subscriptions, pero los existentes continúan su ciclo de vida (refunds, renewals, disputas).
- **Ergonomía API — Account default implícito:** si la request omite `accountId` y el Platform tiene exactamente un Account activo, el sistema lo resuelve automáticamente; con múltiples Accounts, error `ACCOUNT_REQUIRED`.

### 5.2 Datos de referencia (en código, NO en DB)

Proveedores, métodos de pago, países y monedas **no se persisten**. Viven como enums TypeScript + const maps en `utils/kernel/` (`catalog/` para `ProviderSlug`/`PaymentMethodSlug`, `money/` para `Country`/`Currency`); ver datamodel §3. No hay BC `catalog` ni migración de catálogo.

Por qué en código y no en DB: son **definiciones de tipo**, no datos operativos. Cada `ProviderSlug` tiene una clase adapter; cada `PaymentMethodSlug` tiene reglas de flujo/`NextAction` en código; `Country`/`Currency` son listas ISO fijas. Si fueran filas de DB, cada `switch(slug)` perdería type‑safety y un typo en una migración sería un incidente de producción.

```ts
export const PROVIDER_SLUGS = ["stripe", "ebanx"] as const;
export type ProviderSlug = (typeof PROVIDER_SLUGS)[number];

export const PAYMENT_METHOD_SLUGS = [
  "card", "yape", "nequi", "plin", "pix", "boleto", "pse",
  "mercadopago_wallet", "bank_transfer",
] as const;
export type PaymentMethodSlug = (typeof PAYMENT_METHOD_SLUGS)[number];
```

Las columnas que en otros diseños habrían sido FK a estas tablas almacenan `TEXT` (`provider_connections.provider_slug`, `provider_capabilities.payment_method_slug`), validado por Zod (`z.enum(PROVIDER_SLUGS)`, etc.) en el boundary, **nunca foreign‑keyed**. `Country` = ISO‑3166‑1 alpha‑2; `Currency` = ISO‑4217 con `minorUnits` por moneda en el const map.

### 5.3 Bounded context: `routing` (ProviderConnection + ProviderCapability)

`routing` cubre la configuración de proveedores por tenant: credenciales (`ProviderConnection`) + matriz de routing (`ProviderCapability`). Ambas son **Platform‑scoped** (infraestructura compartida por todos los Accounts del Platform), no account‑scoped.

#### ProviderConnection

Credenciales del `(Platform × provider)`. Encriptadas at‑rest. **`platformId` es nullable:** `NULL` denota una conexión global/compartida (Merchant‑of‑Record, default de Payins); con valor es BYOC (tenant‑owned). El resolver prefiere la conexión del tenant.

| campo | tipo | notas |
|---|---|---|
| id | `con_UUIDv7` | PK |
| platformId | FK → `Platform` **nullable** | `NULL` = global/MoR compartida; con valor = BYOC del tenant |
| providerSlug | `TEXT` | validado contra el enum `ProviderSlug` (sin FK; catálogo en código) |
| label | `TEXT` | "Ebanx Retail", "Stripe Backup" (permite varias del mismo provider) |
| credentialsCiphertext | `BYTEA` | libsodium sealed box del JSON de credenciales. **El secret de webhook del provider** (para verificar firmas entrantes) vive **dentro** de este blob encriptado — no hay columna dedicada. |
| credentialsFingerprint | `TEXT` | SHA‑256 hex del plaintext, visible al usuario para distinguir versiones |
| livemode | `BOOLEAN` | default `true` |
| status | `ENUM` (`TEXT`) | `active \| revoked` (default `active`) |
| version | `INT` | |
| createdAt, updatedAt | `BIGINT (Unix ms)` | |

**Invariantes:**
- `UNIQUE (platformId, providerSlug, label)`; además un índice parcial único garantiza "una sola conexión global por provider" (`platform_id IS NULL`).
- Credenciales nunca en logs (filtro obligatorio).
- El secret de verificación de webhooks entrantes **no** es una columna aparte: forma parte de las credenciales encriptadas.

#### ProviderCapability

La **matriz de routing**. Es una tabla de DB poblada por tenant.

| campo | tipo | notas |
|---|---|---|
| id | `cap_UUIDv7` | PK |
| connectionId | FK → `ProviderConnection` | |
| paymentMethodSlug | `TEXT` | validado contra el enum `PaymentMethodSlug` (sin FK; catálogo en código) |
| country | `CHAR(2)` | ISO-2 |
| currency | `CHAR(3)` | ISO-3 |
| flowType | `ENUM` (`TEXT`) | `REDIRECT \| ONSITE_TOKEN \| REDIRECT_ASYNC \| DISPLAY \| ENROLLMENT \| PUSH_TO_APP` (enum `FLOW_TYPES` en `utils/kernel/flow-types.ts`; `ONSITE_DIRECT` es trabajo futuro, no existe aún) |
| recurrenceStrategy | `ENUM` | `NONE \| AUTO_PROVIDER \| MANAGED \| REMINDER` |
| priority | `INT` | default `100`; menor = preferido cuando varias capabilities matchean |
| minAmount | `BIGINT` nullable | en currency minor units |
| maxAmount | `BIGINT` nullable | |
| config | `JSONB` | parámetros específicos del proveedor (ej. `payment_type_code` de Ebanx, `require_3ds`) |
| label | `TEXT` | label legible (ej. "Ebanx Card PE 1‑500 PEN") |
| buyerFieldRequirements | `JSONB` | campos del comprador requeridos/opcionales para este método |
| enabled | `BOOLEAN` | default `true` |
| fallbackBehavior | `ENUM` (`TEXT`) | `NEVER_FALLBACK \| FALLBACK_ON_ERROR` |
| healthScore | `INT` | 0‑100, mantenido por job de background; default `100` |
| platformId | FK → `Platform` **nullable** | `NULL` = global; con valor = override del tenant (futuro) |
| version | `INT` | |
| createdAt, updatedAt | `BIGINT (Unix ms)` | |

**Invariantes:**
- `currency` y `country` deben estar en las listas soportadas del provider (definidas en el catálogo de código `utils/kernel/`, ver §5.2).
- Unicidad por la tupla `(platformId, providerSlug, paymentMethodSlug, country, currency, flowType, recurrenceStrategy)`; la disjunción de rangos de monto es operacional, no DB‑enforced.

**Resolución en tiempo de pago**: dado `(platformId, paymentMethodSlug, country, currency, amount, flowType, recurrenceStrategy)`, se seleccionan las capabilities con `enabled=true`, dentro de rango de monto. El resolver (`providerCapability.repo.ts`) ordena por `platformId ASC (nulls last) → priority DESC → healthScore DESC → createdAt ASC`; la primera gana y las siguientes forman la cadena de fallback. (Nota: el comentario del schema dice "LOWER priority = preferred / ORDER BY priority ASC", pero el adapter ejecutado ordena `priority DESC` — inconsistencia interna conocida del código.) Si ninguna matchea → error de routing (`ErrProviderCapabilityNotFound`).

### 5.4 Bounded context: `contract`

Términos comerciales por `Account`. Separados de la configuración técnica.

#### Contract

| campo | tipo | notas |
|---|---|---|
| id | `ctr_UUIDv7` | PK |
| platformId | FK | denormalizado; siempre coincide con `Account.platformId` |
| accountId | FK → `Account` | account propietario |
| name | `TEXT` | "Contrato 2026 Q2" |
| status | `ENUM` | `DRAFT \| ACTIVE \| TERMINATED` |
| effectiveFrom | `BIGINT (Unix ms)` | |
| effectiveUntil | `BIGINT (Unix ms)` nullable | null = vigente indefinidamente |
| currency | `CHAR(3)` | currency de facturación de las comisiones (columna `currency`; v1 debe igualar `Payment.currency`). Solo `Payment` tiene una columna `commissionCurrency`. |
| version | `INT` | optimistic lock |
| createdAt, updatedAt | `BIGINT (Unix ms)` | |

**Invariante:** un `Account` tiene **a lo sumo una** Contract `ACTIVE` a la vez. Al activar una nueva, la anterior pasa a `TERMINATED` con `effectiveUntil = now()`.

#### ContractTerm

Un contrato se compone de múltiples términos. Cada término define comisión/markup para un **scope** (tupla opcional). El término más específico gana.

| campo | tipo | notas |
|---|---|---|
| id | `trm_UUIDv7` | PK |
| contractId | FK → `Contract` | |
| scopePaymentMethodSlug | nullable `TEXT` | null = cualquiera |
| scopeProviderSlug | nullable `TEXT` | null = cualquiera |
| scopeCountry | nullable `CHAR(2)` | null = cualquiera |
| scopeCurrency | nullable `CHAR(3)` | null = cualquiera |
| commissionPercentBps | `INT` | basis points sobre el `amount` del pago |
| commissionFixed | `BIGINT` | en `Contract.currency`, minor units |
| markupPercentBps | `INT` | basis points añadidos al amount que ve el cliente (si el consumidor delega la suma a Payins; default 0) |
| markupFixed | `BIGINT` | |
| minCommission | `BIGINT` nullable | piso |
| maxCommission | `BIGINT` nullable | techo |
| specificity | `INT` computed | número de campos scope no‑null (0‑4); tiebreaker |
| priority | `INT` | override manual del specificity |
| createdAt, updatedAt | | |

**Resolución de término aplicable a un pago**:
1. Candidatos: términos cuyos `scope*` son null **o** matchean el pago.
2. Ordenar por `(priority DESC, specificity DESC, createdAt DESC)`.
3. Primero gana.
4. Si ninguno matchea y no hay término wildcard: pago se rechaza con `NO_CONTRACT_TERM`.

**Cómputo de comisión** (sobre `Payment.amount`, `A` en minor units):
```
commissionGross = A * commissionPercentBps / 10000 + commissionFixed
commission = clamp(commissionGross, minCommission, maxCommission)  // si aplican
```
Redondeo **half‑even** (banker's rounding) a la unidad entera. Persistido en `Payment.commissionAmount`.

**Nota:** el `markup` se aplica **antes** de crear el `Payment` si el consumidor lo delega (ej. consumidor pide "cóbrale 100€, gestiona el markup"). Por defecto, el consumidor ya viene con el `amount` final y markup = 0.

### 5.5 Bounded context: `customer` — **planificado, no implementado todavía**

> El BC `customer` y su tabla `customers` no existen aún en el código (Fase C). El diseño abajo es el target.

Wrapper opaco del usuario final. Útil para agrupar instrumentos y suscripciones.

#### Customer

| campo | tipo | notas |
|---|---|---|
| id | `cus_UUIDv7` | PK |
| platformId | FK | denormalizado |
| accountId | FK → `Account` | account propietario |
| externalReference | `TEXT` | opaque ID del consumidor (ej. `"userId=42"`) |
| email | `TEXT` nullable | opcional, para disputas |
| displayName | `TEXT` nullable | |
| country | `CHAR(2)` nullable | hint para routing |
| metadata | `JSONB` | max 4KB |
| status | `ENUM` | `ACTIVE \| ARCHIVED` |
| createdAt, updatedAt | `BIGINT (Unix ms)` | |

`UNIQUE (platformId, accountId, externalReference)` — el mismo `externalReference` puede existir bajo dos Accounts distintos.

### 5.6 Bounded context: `instrument` — **planificado, no implementado todavía**

> El BC `instrument` y su tabla `instruments` no existen aún en el código (Fase C). El diseño abajo es el target.

Unifica card tokens y wallet/bank mandates.

#### Instrument

| campo | tipo | notas |
|---|---|---|
| id | `ins_UUIDv7` | PK |
| platformId | FK | denormalizado |
| accountId | FK → `Account` | account propietario (heredado del Customer cuando existe) |
| customerId | FK → `Customer` nullable | puede ser anónimo (guest checkout) |
| connectionId | FK → `ProviderConnection` | a qué conexión está vinculado el token/mandate |
| paymentMethodSlug | `TEXT` | validado contra el enum `PaymentMethodSlug` (sin FK; catálogo en código) |
| type | `ENUM` | `CARD \| WALLET_MANDATE \| BANK_MANDATE` |
| providerInstrumentId | `TEXT` | el token o enrollment id del proveedor |
| status | `ENUM` | `PENDING \| ACTIVE \| EXPIRED \| REVOKED` |
| displayHint | `TEXT` | `"Visa ****4242"`, `"Yape ***1234"` |
| expiresAt | `BIGINT (Unix ms)` nullable | |
| lastUsedAt | `BIGINT (Unix ms)` nullable | |
| lastFailureAt | `BIGINT (Unix ms)` nullable | |
| consecutiveFailures | `INT default 0` | usado para auto‑expirar |
| deviceFingerprint | `TEXT` nullable | anti‑fraud |
| cardDetails | `JSONB` nullable | `{brand, last4, expMonth, expYear, country?}` si `type=CARD` |
| mandateDetails | `JSONB` nullable | `{issuer, phoneMasked, …}` si `type=WALLET_MANDATE` |
| createdAt, updatedAt | `BIGINT (Unix ms)` | |
| version | `INT` | |

**Invariante:** un Instrument con `consecutiveFailures >= N` (N configurable por Platform, default 3) pasa automáticamente a `EXPIRED` y se emite `instrument.expired` outbound.

**Reuso cross‑Platform / cross‑Account / cross‑customer bloqueado en la capa de query** (categorías de test de seguridad #3, #4 y #5). Un Instrument del Account A no puede usarse en un Payment que referencia al Account B, ni siquiera bajo el mismo Platform.

### 5.7 Bounded context: `checkout`

> **Estado hoy:** este BC se implementa como **`paymentSession`** (tabla `payment_sessions`), que respalda la página de checkout del comprador. La tabla `payment_sessions` real tiene los campos: `id`, `platformId`, `accountId`, `customerId?`, `mode` (`ONE_TIME \| SUBSCRIPTION_SETUP`; **v1 solo `ONE_TIME`**), `amount?`, `currency?`, `country?`, `allowedPaymentMethodSlugs?`, `saveInstrument`, `successUrl`, `cancelUrl`, `expiresAt`, `status` (`OPEN \| COMPLETED \| EXPIRED \| CANCELED`), `resolvedCapabilityId?`, `paymentId?`, `instrumentId?`, `reference?`, `metadata`, `version`, timestamps. **No existe** la tabla `payment_links` ni el aggregate `PaymentLink` — ese subdiseño (`PaymentLink`, `/l/:slug`) es **planificado, no implementado**. El diseño `Checkout`/`PaymentLink` abajo es el target conceptual.

#### Checkout (target — implementado como `PaymentSession`)

Sesión de pago o de suscripción. Unifica lo que en el sistema de referencia eran dos entidades.

| campo | tipo | notas |
|---|---|---|
| id | `chk_UUIDv7` | PK |
| platformId | FK | denormalizado |
| accountId | FK → `Account` | account propietario; propagado al Payment/Subscription resultante |
| customerId | FK nullable | |
| paymentLinkId | FK → `PaymentLink` nullable | si fue generado por un link |
| mode | `ENUM` | `ONE_TIME \| SUBSCRIPTION_SETUP` |
| amount | `BIGINT` nullable | requerido si `mode=ONE_TIME` |
| currency | `CHAR(3)` nullable | idem |
| country | `CHAR(2)` nullable | hint |
| planId | FK → `Plan` nullable | requerido si `mode=SUBSCRIPTION_SETUP` |
| trialDays | `INT` nullable | override del Plan |
| allowedPaymentMethodSlugs | `TEXT[]` nullable | null = todos los habilitados |
| saveInstrument | `BOOLEAN` | si true y paga con card/mandate, se guarda Instrument |
| successUrl | `TEXT` | HTTPS |
| cancelUrl | `TEXT` | |
| expiresAt | `BIGINT (Unix ms)` | default now + 30min |
| status | `ENUM` | `OPEN \| COMPLETED \| EXPIRED \| CANCELED` |
| resolvedCapabilityId | FK nullable | cuando se inicia flujo, qué `ProviderCapability` se eligió |
| paymentId | FK nullable | al completar, el pago creado |
| subscriptionId | FK nullable | al completar, la suscripción |
| instrumentId | FK nullable | si se guardó |
| claimToken | `TEXT` nullable | para flujos sin auth previo |
| claimTokenExpiresAt | `BIGINT (Unix ms)` nullable | |
| reference | `TEXT` nullable | idempotencia por parte del consumidor |
| metadata | `JSONB` | max 4KB |
| version | `INT` | |
| createdAt, updatedAt | `BIGINT (Unix ms)` | |

**Invariantes:**
- `UNIQUE (platformId, reference)` cuando `reference IS NOT NULL`.
- Checkout expirado es inmutable.
- `mode=ONE_TIME` ⇒ `amount` y `currency` obligatorios.
- `mode=SUBSCRIPTION_SETUP` ⇒ `planId` obligatorio.

#### PaymentLink — **planificado, no implementado todavía**

URL reutilizable que genera Checkouts. (No existe en el código actual; ver nota de §5.7.)

| campo | tipo | notas |
|---|---|---|
| id | `lnk_UUIDv7` | PK |
| platformId | FK | denormalizado |
| accountId | FK → `Account` | account propietario; propagado a cada Checkout que el link genera |
| slug | `TEXT UNIQUE` | parte de la URL pública: `pay.example.com/l/{slug}`. Se genera por default, o el consumidor propone uno (validado regex). |
| mode | `ENUM` | `ONE_TIME \| SUBSCRIPTION_SETUP` |
| amount | `BIGINT` nullable | si null y `mode=ONE_TIME` → el cliente escribe el monto |
| currency | `CHAR(3)` nullable | |
| planId | FK nullable | |
| allowedPaymentMethodSlugs | `TEXT[]` nullable | |
| allowedCountries | `TEXT[]` nullable | |
| maxRedemptions | `INT` nullable | null = ilimitado |
| redemptionsCount | `INT default 0` | |
| status | `ENUM` | `ACTIVE \| ARCHIVED \| EXHAUSTED` |
| expiresAt | `BIGINT (Unix ms)` nullable | |
| description | `TEXT` nullable | visible al pagador |
| collectCustomerEmail | `BOOLEAN` | |
| metadata | `JSONB` | |
| version | `INT` | |
| createdAt, updatedAt | `BIGINT (Unix ms)` | |

**Endpoints públicos del link:**
- `GET /l/:slug` → retorna la metadata pública (frontend del consumidor la renderiza).
- `POST /l/:slug/checkout` → crea un `Checkout` asociado y devuelve el `chk_…`. Incrementa `redemptionsCount`.

**Invariantes:**
- Si `maxRedemptions` alcanzado, `status → EXHAUSTED` y subsequentes POSTs dan 409.
- Si `expiresAt` pasó, 410 Gone.

### 5.8 Bounded context: `payment`

#### Payment (aggregate root) — **gateway‑AGNÓSTICO**

El `Payment` representa la **intención de cobro** y **no conoce el provider**: no tiene `capabilityId`, `providerSlug`, ni `providerPaymentId`. La **ruta** (qué provider/conexión/capability se usó) vive **en el `Attempt`**. Reintento/fallback = un **nuevo `Attempt`** sobre el **mismo `Payment`** — por eso la ruta no puede estar fijada en el Payment. El origen es **polimórfico y opcional** (`paymentSessionId` / `invoiceId` / `subscriptionId` / `instrumentId`, todos nullable).

| campo | tipo | notas |
|---|---|---|
| id | `pay_UUIDv7` | PK |
| platformId | FK | denormalizado |
| accountId | FK → `Account` | account propietario; dirige la resolución de Contract |
| paymentSessionId | FK nullable | origen: checkout hosted |
| invoiceId | columna escalar nullable | origen: renovación de suscripción (FK en Fase C) |
| subscriptionId | columna escalar nullable | link de conveniencia (FK en Fase C) |
| customerId | columna escalar nullable | Fase C |
| instrumentId | columna escalar nullable | si se usó credencial guardada (Fase C) |
| requestedMethodSlug | `TEXT` | método **lógico** que eligió el pagador (validado vs `PaymentMethodSlug`) |
| contractTermId | FK nullable | término SELL aplicado (se fija al capturar) |
| amount | `BIGINT` | minor units del `currency` |
| currency | `CHAR(3)` | |
| country | `CHAR(2)` nullable | |
| buyer | `JSONB` | snapshot de los campos canónicos del comprador (PII) |
| commissionAmount | `BIGINT default 0` | SELL congelada al capturar desde el `ContractTerm` |
| commissionCurrency | `CHAR(3) nullable` | currency del Contract; null hasta capturar (puede diferir del Payment) |
| status | `ENUM` | `CREATED \| AUTHORIZED \| CAPTURED \| FAILED \| EXPIRED \| REFUNDED \| PARTIALLY_REFUNDED \| CHARGED_BACK` |
| statusReason | `TEXT` nullable | motivo legible del estado actual |
| nextAction | `JSONB` nullable | `NextAction` (unión discriminada); null si no hay acción pendiente |
| totalRefundedAmount | `BIGINT default 0` | |
| reference | `TEXT` nullable | idempotencia opcional por parte del consumidor (`UNIQUE (platformId, reference) WHERE NOT NULL`) |
| metadata | `JSONB` | |
| version | `INT` | |
| createdAt, updatedAt | `BIGINT (Unix ms)` | |
| authorizedAt, capturedAt, failedAt, expiredAt, refundedAt, chargedBackAt | `BIGINT (Unix ms)` nullable | |

#### Attempt — **donde vive la ruta del provider**

Cada ejecución ROUTED contra un gateway es un `Attempt` persistido (audit + debugging + base del fallback). Es append‑only: N por `Payment`. Aquí están **todas** las columnas de ruta que el Payment ya no tiene.

| campo | tipo | notas |
|---|---|---|
| id | `att_UUIDv7` | PK |
| paymentId | FK | |
| sequence | `INT` | 1, 2, 3 (`UNIQUE (paymentId, sequence)`) |
| connectionId | FK → `ProviderConnection` | qué credenciales se usaron |
| capabilityId | FK → `ProviderCapability` | ruta elegida **para este intento** |
| providerSlug | `TEXT` | provider de este intento |
| providerPaymentId | `TEXT` nullable | id en el gateway (`UNIQUE (providerSlug, providerPaymentId)` cuando está seteado) |
| status | `ENUM` | `PENDING \| SUCCESS \| FAILED` |
| requestSnapshot | `JSONB` | payload enviado al provider (sanitizado — sin PAN/CVV) |
| responseSnapshot | `JSONB` | respuesta del provider (sanitizada) |
| errorCode | `TEXT` nullable | código normalizado interno |
| providerErrorCode | `TEXT` nullable | código crudo del provider para debugging |
| errorMessage | `TEXT` nullable | |
| providerCostAmount | `BIGINT` nullable | costo BUY congelado aquí (Fase D) |
| latencyMs | `INT` | tiempo contra el provider |
| occurredAt | `BIGINT (Unix ms)` | |

#### Refund

| campo | tipo | notas |
|---|---|---|
| id | `ref_UUIDv7` | PK |
| paymentId | FK | |
| amount | `BIGINT` | |
| currency | `CHAR(3)` | debe coincidir con `Payment.currency` |
| reason | `ENUM` | `DUPLICATE \| FRAUDULENT \| REQUESTED_BY_CUSTOMER \| PROVIDER_ERROR \| OTHER` |
| providerRefundId | `TEXT` nullable | |
| status | `ENUM` | `PENDING \| SUCCEEDED \| FAILED` |
| createdAt, updatedAt, succeededAt, failedAt | `BIGINT (Unix ms)` | |
| version | `INT` | |

**Invariantes:**
- `SUM(refunds.amount WHERE status=SUCCEEDED) <= Payment.amount`.
- El refund que iguale el total → `Payment.status = REFUNDED`.
- Refund parcial → `Payment.status = PARTIALLY_REFUNDED`.

#### Dispute — **planificado, no implementado todavía**

> No existe tabla `disputes` ni aggregate `Dispute` en el código actual (la migración v1 no la crea). El dispatcher entrante **parsea** chargebacks pero **no actúa** sobre ellos en v1; el manejo de disputas es trabajo futuro. El diseño de tabla abajo es el target.

| campo | tipo | notas |
|---|---|---|
| id | `dsp_UUIDv7` | PK |
| paymentId | FK | |
| amount | `BIGINT` | |
| currency | `CHAR(3)` | |
| reason | `TEXT` | string libre del provider (muchos códigos) |
| providerDisputeId | `TEXT` | |
| status | `ENUM` | `OPEN \| UNDER_REVIEW \| WON \| LOST \| ACCEPTED` |
| evidence | `JSONB` nullable | metadata; adjuntos NO se guardan aquí (opcional v2) |
| dueBy | `BIGINT (Unix ms)` nullable | fecha límite si el provider la notifica |
| createdAt, updatedAt | `BIGINT (Unix ms)` | |

### 5.9 Bounded context: `subscription` — **planificado, no implementado todavía**

> El BC `subscription` y sus tablas `plans` / `subscriptions` / `invoices` no existen aún en el código (Fase C). Los diseños abajo son el target.

#### Plan

| campo | tipo | notas |
|---|---|---|
| id | `pln_UUIDv7` | PK |
| platformId | FK | denormalizado |
| accountId | FK → `Account` | account propietario |
| name | `TEXT` | |
| amount | `BIGINT` | |
| currency | `CHAR(3)` | |
| interval | `ENUM` | `DAY \| WEEK \| MONTH \| YEAR` |
| intervalCount | `INT` | cada N intervals |
| trialDays | `INT default 0` | |
| status | `ENUM` | `ACTIVE \| ARCHIVED` |
| metadata | `JSONB` | |
| createdAt, updatedAt | `BIGINT (Unix ms)` | |

#### Subscription

| campo | tipo | notas |
|---|---|---|
| id | `sub_UUIDv7` | PK |
| platformId | FK | denormalizado |
| accountId | FK → `Account` | account propietario; heredado por Invoice y Payment |
| customerId | FK nullable | |
| planId | FK | |
| capabilityId | FK | ruta elegida al crearse |
| recurrenceStrategy | `ENUM` | `AUTO_PROVIDER \| MANAGED \| REMINDER` (heredado de capability al crear) |
| instrumentId | FK nullable | si MANAGED |
| providerSubscriptionId | `TEXT` nullable | si AUTO_PROVIDER (ej. Stripe `sub_…`) |
| status | `ENUM` | `PENDING \| ACTIVE \| PAST_DUE \| CANCELED \| EXPIRED` |
| currentCycleNumber | `INT` | 1‑based |
| currentPeriodStart | `BIGINT (Unix ms)` | |
| currentPeriodEnd | `BIGINT (Unix ms)` | |
| nextInvoiceAt | `BIGINT (Unix ms)` nullable | null si está cancelada |
| trialEndsAt | `BIGINT (Unix ms)` nullable | |
| cancelAtPeriodEnd | `BOOLEAN` | |
| canceledAt | `BIGINT (Unix ms)` nullable | |
| reference | `TEXT` nullable | idempotencia del consumidor |
| metadata | `JSONB` | |
| version | `INT` | |
| createdAt, updatedAt | `BIGINT (Unix ms)` | |

#### Invoice

Una factura por ciclo de suscripción. Se crea *antes* del cobro, queda `OPEN`, y al capturar el Payment pasa a `PAID`.

| campo | tipo | notas |
|---|---|---|
| id | `inv_UUIDv7` | PK |
| platformId | FK | denormalizado |
| subscriptionId | FK | |
| cycleNumber | `INT` | |
| amount | `BIGINT` | |
| currency | `CHAR(3)` | |
| periodStart, periodEnd | `BIGINT (Unix ms)` | |
| status | `ENUM` | `OPEN \| PAID \| UNCOLLECTIBLE \| VOIDED` |
| paymentId | FK nullable | |
| attemptsCount | `INT default 0` | incrementa por intento de cobro |
| nextAttemptAt | `BIGINT (Unix ms)` nullable | |
| dueAt | `BIGINT (Unix ms)` nullable | |
| createdAt, updatedAt, paidAt | `BIGINT (Unix ms)` | |
| version | `INT` | |

**Invariantes:**
- `UNIQUE (subscriptionId, cycleNumber)`.
- `Invoice.amount` se fija al crearla (inmutable); cambios de plan solo afectan invoices futuras.

### 5.10 BC `webhook-outbound` + features `common/events` y `common/inboundWebhooks`

`outboundWebhooks` es un BC (endpoints CRUD propios para el integrador; el plan lo nombraba `webhook-outbound`). `DomainEvent` (outbox) vive en `common/events` y `InboundWebhookEvent` en `common/inboundWebhooks` — features cross‑cutting con DDD completo, **no** BCs. `WebhookEndpoint`, `DomainEvent` e `InboundWebhookEvent` son **Platform‑scoped** (infra compartida por todos los Accounts del Platform).

#### WebhookEndpoint (outbound — registrado por el consumidor) — BC `outboundWebhooks`

El secret de firma se almacena **en plaintext** en una sola columna `signingSecret` (`TEXT`); la encriptación at‑rest (libsodium) es un hardening **diferido**, no implementado. No hay columnas `signingSecretCiphertext` ni `signingSecretFingerprint`. El integrador **provee** su propio `signing_secret` (mín. 32 chars) al registrar el endpoint; Payins **no genera ni devuelve** secret.

| campo | tipo | notas |
|---|---|---|
| id | `whe_UUIDv7` | PK |
| platformId | FK | |
| url | `TEXT` | HTTPS obligatorio |
| signingSecret | `TEXT` | **plaintext v1** (encrypt‑at‑rest diferido); provisto por el integrador |
| subscribedEvents | `JSONB` | array de `OutboundEventType` (ver §5.13); enum **cerrado** de exactamente 5 |
| status | `ENUM` (`TEXT`) | `active \| disabled` (default `active`) |
| livemode | `BOOLEAN` | default `true` |
| description | `TEXT` nullable | |
| version | `INT` | |
| createdAt, updatedAt | `BIGINT (Unix ms)` | |

#### DomainEvent — feature `common/events` (outbox, NO BC) — **construido pero no cableado**

> **Estado hoy:** el outbox existe (entidad `DomainEvent`, puerto `IEventPublisher` en `utils/application/event-publisher.port.ts`, `PrismaEventStore`, tabla `domain_events`) **pero está dormido**: ningún use case emite `DomainEvent`, y no hay proyector evento→delivery. En v1 los webhooks salientes se encolan **directamente** (ver `WebhookDelivery` abajo). El diseño "cada mutación de BC persiste un `DomainEvent` en la misma transacción" es el **target**, no el comportamiento actual.

Log append‑only diseñado para todos los eventos de dominio (auditoría + futura fuente para webhooks salientes).

| campo | tipo | notas |
|---|---|---|
| id | `evt_UUIDv7` | PK |
| platformId | FK | |
| accountId | FK nullable | seteado cuando el evento es account‑scoped (la mayoría) |
| type | `TEXT` | ej. `payment.captured` |
| aggregateType | `TEXT` | `Payment`, `Refund`, … |
| aggregateId | `TEXT` | |
| payload | `JSONB` | shape público estable del recurso |
| livemode | `BOOLEAN` | |
| occurredAt | `BIGINT (Unix ms)` | |
| publishedAt | `BIGINT (Unix ms)` nullable | cuando se proyectó por primera vez a deliveries (futuro) |

#### OutboundWebhookDelivery (outbound) — BC `outboundWebhooks`

En v1 cada delivery se **encola directamente** por el dispatcher de webhooks entrantes (sin pasar por `DomainEvent`); por eso `eventId` es **nullable** y queda `null` en este camino directo (se llenará cuando aterrice el proyector del outbox).

| campo | tipo | notas |
|---|---|---|
| id | `whd_UUIDv7` | PK |
| endpointId | FK | |
| platformId | FK | |
| eventId | FK → `DomainEvent` **nullable** | `null` en el camino directo v1; se setea cuando aterrice el proyector |
| eventType | `TEXT` | uno de los 5 tipos del enum cerrado |
| payload | `JSONB` | envelope entregado |
| status | `ENUM` (`TEXT`) | `PENDING \| DELIVERED \| FAILED \| EXHAUSTED` (terminal: `EXHAUSTED`) |
| attemptCount | `INT default 0` | |
| attempts | `JSONB` | array de `DeliveryAttempt` (inmutable una vez añadido) |
| nextRetryAt | `BIGINT (Unix ms)` nullable | |
| deliveredAt | `BIGINT (Unix ms)` nullable | |
| failedAt | `BIGINT (Unix ms)` nullable | |
| lastError | `TEXT` nullable | |
| version | `INT` | |
| createdAt, updatedAt | `BIGINT (Unix ms)` | |

**Reintentos:** 30s → 2min → 5min → 10min (`RETRY_DELAYS_MS` en `delivery.aggregate.ts`), máximo **5 intentos**; agotados → estado terminal `EXHAUSTED`. Consultable y re‑lanzable manualmente vía `POST /v1/webhook-deliveries/:id/retry`.

#### InboundWebhookEvent — feature `common/inboundWebhooks` (NO BC)

Crudo recibido del provider, persistido para dedupe y auditoría.

| campo | tipo | notas |
|---|---|---|
| id | `iwe_UUIDv7` | PK |
| connectionId | FK → `ProviderConnection` | |
| providerEventId | `TEXT` | id nativo del provider |
| rawHeaders | `JSONB` | sanitizado |
| rawBody | `BYTEA` | opcionalmente truncado |
| signatureValid | `BOOLEAN` | |
| normalizedType | `ENUM` `InboundNormalizedType` | |
| normalizedPayload | `JSONB` nullable | |
| processingStatus | `ENUM` | `RECEIVED \| PROCESSED \| IGNORED \| FAILED` |
| processedAt | `BIGINT (Unix ms)` nullable | |
| errorMessage | `TEXT` nullable | |
| receivedAt | `BIGINT (Unix ms)` | |

`UNIQUE (connectionId, providerEventId)` → dedupe idempotente.

### 5.13 Eventos de dominio (vocabulario público)

Nombres estables que los consumidores ven en webhooks salientes. Este es el **contrato** del servicio.

```
payment.created
payment.authorized
payment.captured
payment.failed
payment.expired
payment.refunded
payment.partially_refunded
payment.charged_back

refund.created
refund.succeeded
refund.failed

dispute.opened
dispute.updated
dispute.won
dispute.lost

checkout.opened
checkout.completed
checkout.expired
checkout.canceled

payment_link.created
payment_link.archived
payment_link.exhausted

instrument.created
instrument.activated
instrument.expired
instrument.revoked

subscription.created
subscription.activated
subscription.trialing
subscription.renewed
subscription.past_due
subscription.renewal_failed
subscription.canceled
subscription.expired
subscription.reminder_due

invoice.created
invoice.paid
invoice.uncollectible
invoice.voided

contract.activated
contract.terminated
```

### 5.14 Enum `InboundNormalizedType` (tipos internos, no expuestos)

Traducción desde eventos nativos de cada provider:

```
PAYMENT_CAPTURED
PAYMENT_FAILED
PAYMENT_EXPIRED
PAYMENT_REFUNDED
PAYMENT_CHARGEBACK_OPENED
PAYMENT_CHARGEBACK_UPDATED
SUBSCRIPTION_RENEWED
SUBSCRIPTION_RENEWAL_FAILED
SUBSCRIPTION_CANCELED_BY_PROVIDER
INSTRUMENT_ENROLLED
INSTRUMENT_ENROLLMENT_FAILED
```

Cada adapter mapea sus eventos nativos a este enum cerrado. El dominio solo conoce este enum.

---

## 6. Provider adapters — módulos + handlers por familia de flujo

### 6.1 Por qué no un port único

La primera versión del plan tenía un único `IProviderAdapter` con ~12 métodos opcionales (`createRedirectCheckout?`, `startEnrollment?`, `chargeWithInstrument?`, …) + un `capabilities()` declarativo. **Esto es exactamente la "abstracción inflada llena de opcionales"** que Clean Architecture / DDD advierte evitar en orquestadores de pagos:

- Cada adapter termina con 60% de métodos `undefined`.
- `capabilities()` en runtime intenta compensar la falta de type-safety.
- Stripe card (redirect + 3DS) y Ebanx Yape (enrollment + re-charge) y Ebanx Pix (display QR + webhook tardío) **no comparten invariantes reales** — forzarlos a la misma interfaz produce un modelo mentiroso.
- En Kunfupay este patrón (factory única resolviendo todos los gateways) es una de las deudas técnicas que explícitamente no queremos repetir.

### 6.2 Patrón: módulo provider + handlers por flujo

En vez de un port único, **una interfaz por familia de flujo**, cada una con su contrato honesto (sin `?` opcionales, sin métodos irrelevantes):

```ts
// src/utils/application/provider-ports/

interface IRedirectFlowHandler {
  createRedirect(ctx: CreateRedirectCtx): Promise<RedirectResult>
}

interface IRedirectAsyncFlowHandler {
  createRedirectAsync(ctx: CreateRedirectAsyncCtx): Promise<RedirectAsyncResult>
}

interface IOnsiteTokenFlowHandler {
  confirmWithProviderToken(ctx: ConfirmOnsiteCtx): Promise<ConfirmResult>
}

interface IDisplayFlowHandler {
  createDisplay(ctx: CreateDisplayCtx): Promise<DisplayArtifactResult>  // QR, voucher
}

interface IPushToAppFlowHandler {
  initiatePush(ctx: InitiatePushCtx): Promise<PushInitiationResult>
}

interface IEnrollmentFlowHandler {
  startEnrollment(ctx: StartEnrollmentCtx): Promise<EnrollmentResult>
  checkEnrollment(ctx: CheckEnrollmentCtx): Promise<EnrollmentStatus>
  chargeWithInstrument(ctx: ChargeInstrumentCtx): Promise<ConfirmResult>
  revokeInstrument(ctx: RevokeInstrumentCtx): Promise<void>
}

interface IAutoProviderSubscriptionHandler {
  createSubscription(ctx: CreateAutoSubCtx): Promise<AutoSubResult>
  cancelSubscription(ctx: CancelAutoSubCtx): Promise<void>
}

interface IRefundHandler {
  refund(ctx: RefundCtx): Promise<RefundResult>
}

interface IDisputeHandler {
  submitEvidence(ctx: DisputeEvidenceCtx): Promise<void>
}

interface IWebhookVerifier {
  verify(raw: Buffer, headers: Headers, secret: string): boolean
}

interface IWebhookParser {
  parse(raw: Buffer, headers: Headers): ParsedInboundEvent
}
```

Un **provider adapter** es un **módulo** que agrupa solo los handlers que ese provider realmente implementa:

```ts
// src/provider-adapters/stripe/index.ts
export const stripeAdapter = {
  slug: "stripe" as const,
  redirectFlow:              new StripeCheckoutRedirectHandler(deps),
  onsiteTokenFlow:           new StripeOnsiteTokenHandler(deps),
  autoProviderSubscription:  new StripeAutoSubHandler(deps),
  refund:                    new StripeRefundHandler(deps),
  disputes:                  new StripeDisputeHandler(deps),
  webhookVerifier:           new StripeWebhookVerifier(deps),
  webhookParser:             new StripeWebhookParser(deps),
  // notably absent: displayFlow, pushToAppFlow, enrollmentFlow
} satisfies ProviderAdapterModule;
```

```ts
// src/provider-adapters/ebanx/index.ts
export const ebanxAdapter = {
  slug: "ebanx" as const,
  redirectFlow:         new EbanxCardRedirectHandler(deps),
  redirectAsyncFlow:    new EbanxRedirectAsyncHandler(deps),    // Boleto, PSE
  displayFlow:          new EbanxPixDisplayHandler(deps),        // Pix QR
  enrollmentFlow:       new EbanxWalletEnrollmentHandler(deps),  // Yape, Nequi, PLIN
  refund:               new EbanxRefundHandler(deps),
  webhookVerifier:      new EbanxWebhookVerifier(deps),          // RSA-SHA1
  webhookParser:        new EbanxWebhookParser(deps),
  // notably absent: onsiteTokenFlow, autoProviderSubscription, pushToAppFlow
} satisfies ProviderAdapterModule;
```

El tipo `ProviderAdapterModule` es genérico: cualquier subconjunto de los handlers anteriores, con `slug`, `webhookVerifier` y `webhookParser` siempre requeridos (un provider sin firma de webhook no se admite).

> **Nota sobre MercadoPago:** no es provider en v1. Se sirve como método `mercadopago_wallet` a través del módulo de Ebanx (capability `ebanx × mercadopago_wallet × {AR,BR,CL,CO,MX,PE,UY} × REDIRECT × NONE`). Si en el futuro se adopta partnership directo con MP, se añade `mercadopago` a `PROVIDER_SLUGS` + un módulo `provider-adapters/mercadopago/` con sus handlers propios — sin tocar dominio ni el módulo de Ebanx.

### 6.3 Cómo se usa desde los application services

Cada `PaymentCapability` persistida en DB declara su `flowType`. El application service correspondiente pide **el handler exacto** que necesita — type‑safe, sin ramificación runtime:

```ts
// Application handler for "ONSITE_TOKEN" confirmation
class ConfirmOnsiteTokenPayment {
  async handle(cmd: ConfirmOnsiteTokenCommand) {
    const capability = await routingResolver.resolve(cmd);
    const adapter = registry.resolve(capability.providerSlug);

    if (!adapter.onsiteTokenFlow) {
      throw AppError.domainRule(
        "PROVIDER_DOES_NOT_SUPPORT_FLOW",
        `${adapter.slug} does not implement onsiteTokenFlow`,
      );
    }

    const result = await adapter.onsiteTokenFlow.confirmWithProviderToken({ ... });
    // persist Payment, emit events, etc.
  }
}
```

La verificación `if (!adapter.onsiteTokenFlow)` es la **única ramificación runtime necesaria** — no hay un zoo de `if (capability.flowType === "X")` en el dominio. Y cuando el resolver de routing arranca, valida que cada `ProviderCapability.enabled=true` tiene el handler correspondiente en su adapter módulo; si no, la capability se auto‑deshabilita con log de warning.

### 6.4 Registry y wiring

```ts
// src/provider-adapters/registry.ts
export class ProviderAdapterRegistry {
  private readonly modules = new Map<string, ProviderAdapterModule>();

  register(mod: ProviderAdapterModule): void { /* ... */ }
  resolve(slug: string): ProviderAdapterModule { /* ... */ }
}
```

```ts
// src/wiring.ts
registry.register(stripeAdapter);
registry.register(ebanxAdapter);
```

Añadir un provider = 1 módulo + N handlers concretos + 1 línea en wiring. Cada handler testea en aislamiento con su propio mock.

### 6.5 Adapter-per-method cuando la variación lo exige

El patrón **permite pero no obliga** que un provider tenga N clases handler. Si Ebanx Card redirect y Ebanx Pix display son totalmente distintos, son dos clases distintas compuestas bajo el mismo módulo. Si Stripe card redirect y Stripe card onsite pueden compartir algo interno, es decisión interna del adapter — desde fuera solo se ven los ports.

La regla: **no fuerces uniformidad donde no existe**. La misma regla se aplica a **método+provider** cuando hace falta — `EbanxYapeEnrollmentHandler` y `EbanxNequiEnrollmentHandler` pueden ser clases distintas si los shapes difieren, o la misma con parámetros si son simétricos.

### 6.6 Matriz de capabilities por provider (v1)

Documenta qué módulos implementa cada handler. Sirve de referencia y test de compilación (si un módulo dice que soporta Yape pero su adapter no tiene `enrollmentFlow`, fails).

| Handler \ Provider     | Stripe | Ebanx |
|---|:---:|:---:|
| `redirectFlow`         | ✅ (Checkout) | ✅ (Card, redirect) |
| `redirectAsyncFlow`    | — | ✅ (Boleto, PSE) |
| `onsiteTokenFlow`      | ✅ (Elements + PaymentIntent) | — |
| `displayFlow`          | — | ✅ (Pix QR) |
| `pushToAppFlow`        | — | ✅ (`mercadopago_wallet`, MP vía Ebanx) |
| `enrollmentFlow`       | — | ✅ (Yape, Nequi, PLIN) |
| `autoProviderSubscription` | ✅ | — |
| `refund`               | ✅ | ✅ |
| `disputes`             | ✅ | — (v1) |
| `webhookVerifier`      | ✅ HMAC-SHA256 | ✅ RSA-SHA1 |
| `webhookParser`        | ✅ | ✅ |

Esta matriz es ejecutable (los types lo garantizan en compile time), no un comentario en papel.

### 6.7 Anti-corruption layer — regla dura

Cada handler es la **ACL** de su provider: traduce shapes/errores/eventos del provider al modelo canónico de Payins.

**Regla inviolable:** **ningún shape o tipo de provider cruza fuera de `src/provider-adapters/<slug>/`**. Si un `stripe.PaymentIntent` aparece en `src/payment/application/` o `src/checkout/domain/`, es bug — el adapter falló en su responsabilidad.

Los tipos que el application layer ve son todos canónicos:
- `RedirectResult`, `ConfirmResult`, `EnrollmentResult`, `DisplayArtifactResult`, `AutoSubResult`, `RefundResult`, `ParsedInboundEvent`.

Esto se enforza con el script [Kunfupay-Payins-Back/scripts/check-layer-violations.cjs](Kunfupay-Payins-Back/scripts/check-layer-violations.cjs): nada en `src/payment/`, `src/subscription/`, `src/checkout/`, `src/instrument/` puede importar desde `src/provider-adapters/<slug>/`. Solo imports de `src/utils/application/provider-ports/` (que son interfaces puras) son válidos.

### 6.8 Resumen de principios

1. **Abstrae por invariante de negocio (flujo)**, no por integración (provider).
2. **Un port por familia de flujo**, cada uno con contrato honesto, sin `?` opcionales.
3. **Un provider es un módulo compuesto** de los handlers que implementa — nunca más.
4. **Input discriminado por flujo** (ver sección 7 / endpoint `/confirm`): `ConfirmPaymentInput` es un union discriminado por `flow`, no un DTO gigante.
5. **Estado canónico mínimo** (ya lo tenemos con `PaymentStatus`). No espejamos microestados del provider.
6. **ACL estricta**: shapes de provider no cruzan fuera del adapter. Enforzado por lint.
7. **No unifiques a la fuerza** métodos que difieren en input, shape, ciclo, semántica. Acepta la variación real.

---

## 7. Flujos end‑to‑end

> **Nota de estado.** Los flujos abajo son el **target conceptual** y usan nombres de ruta del diseño original (`/v1/checkouts`, `/v1/subscriptions`, `/v1/payments/:id/refund`, `/l/:slug`). El **surface HTTP real montado hoy** es distinto (ver §8): el checkout vive en `/v1/payment-sessions` (+ `/c/*` público), los BCs de payment/refund/subscription **no tienen rutas HTTP** todavía, y no existen rutas de payment‑links. Además, donde estos flujos dicen "emite domain events → webhook dispatcher", la realidad v1 es que el dispatcher de webhooks **entrantes** encola la entrega **saliente directamente** (el outbox `DomainEvent` está construido pero dormido, ver §3.2 / §5.10).

### 7.1 Pago one‑time on‑site (card tokenizada)

```
consumer → POST /v1/checkouts { mode:ONE_TIME, amount, currency, country, allowed_methods:["card"] }
         ← 201 { id:chk_…, expiresAt }

(browser: SDK de Stripe tokeniza → "pm_abc")

consumer → POST /v1/checkouts/:id/confirm { paymentMethod:"card", providerToken:"pm_abc", saveInstrument:true }
         ↓
         • routing resolve → capability (stripe+card+US+USD+ONSITE_TOKEN+NONE)
         • crea Payment status=CREATED
         • adapter.confirmOnsitePayment → { captured, providerPaymentId }
         • Payment.status = CAPTURED, guarda Attempt #1
         • computa commission con ContractTerm aplicable → guarda en Payment
         • si saveInstrument: crea Instrument status=ACTIVE
         • emite domain events: checkout.completed, payment.created, payment.captured, instrument.created
         ← 200 { paymentId:pay_…, status:CAPTURED }
         
webhook dispatcher → POST consumer_endpoint  "payment.captured"  firmado  (HMAC X-Payins-Signature)
```

### 7.2 Pago one‑time redirect async (Pix / Boleto / PSE)

```
consumer → POST /v1/checkouts { mode:ONE_TIME, amount, currency, country:BR, allowed_methods:["pix"] }
consumer → POST /v1/checkouts/:id/start
         ↓ adapter.createRedirectCheckout → { redirectUrl, qrPayload?, expiresAt }
         ← 200 { redirectUrl, … }
(cliente paga en Pix)

provider → POST /v1/webhooks/:provider  (firma específica del provider)
         ↓ adapter.verifyWebhook
         ↓ adapter.parseWebhook → PAYMENT_CAPTURED
         ↓ InboundWebhookEvent persistido (dedupe por providerEventId)
         ↓ Payment.status = CAPTURED, Checkout.status = COMPLETED
         ↓ emite payment.captured, checkout.completed

outbound dispatcher → consumer_endpoint firmado
```

### 7.3 Suscripción AUTO_PROVIDER (Stripe)

```
consumer → POST /v1/subscriptions { planId, customerId, via_checkout:true }
         ↓ crea Subscription status=PENDING
         ↓ crea Checkout mode=SUBSCRIPTION_SETUP
         ↓ adapter.createAutoProviderSubscription → { checkoutUrl, providerSubscriptionId }
         ← 201 { checkoutUrl }
(cliente completa en Stripe)

stripe webhook customer.subscription.created → SUBSCRIPTION_RENEWED (cycle 0)
         ↓ Subscription.status = ACTIVE, providerSubscriptionId guardado
         
stripe webhook invoice.payment_succeeded (cada mes) → SUBSCRIPTION_RENEWED
         ↓ crea Invoice + Payment CAPTURED, incrementa cycleNumber
         ↓ emite subscription.renewed, invoice.paid, payment.captured
```

### 7.4 Suscripción MANAGED (Yape)

```
Setup:
  consumer → POST /v1/instruments/enroll { paymentMethod:"yape", country:"PE", phoneNumber }
           ↓ adapter.startEnrollment → { redirectUrl, enrollmentId }
           ← 200 { instrumentId:ins_…, status:PENDING, redirectUrl }
  (usuario autoriza en Yape)
  provider webhook → INSTRUMENT_ENROLLED → Instrument.status = ACTIVE, emite instrument.activated
  
Crear suscripción:
  consumer → POST /v1/subscriptions { planId, customerId, instrumentId }
           ↓ crea Subscription status=PENDING, recurrenceStrategy=MANAGED
           ↓ crea Invoice #1
           ↓ adapter.chargeWithInstrument → CAPTURED
           ↓ Payment, Invoice → PAID, Subscription → ACTIVE
           ↓ calcula nextInvoiceAt
           ← 201 { subscriptionId, firstPaymentId }

Renovación (cron nightly):
  GET /v1/internal/cron/subscriptions/renew  (roadmap — no montado; auth Authorization: Bearer <CRON_SECRET>)
         ↓ para cada Subscription con nextInvoiceAt <= now AND status IN (ACTIVE, PAST_DUE):
           - idempotency key = (subscriptionId, currentCycleNumber)
           - crea Invoice si no existe
           - adapter.chargeWithInstrument
           - SUCCESS → Payment CAPTURED, Invoice PAID, avanza cycle, subscription.renewed
           - FAILED → status = PAST_DUE, programa reintento por IRetryStrategy
           - si consecutiveFailures del Instrument >= threshold → Instrument EXPIRED, Subscription CANCELED, emite subscription.canceled
```

### 7.5 Payment Link

```
consumer → POST /v1/payment-links { mode:ONE_TIME, amount:10000, currency:"PEN", slug:"cafe-mensual" }
         ← 201 { id:lnk_…, publicUrl:"https://pay.example.com/l/cafe-mensual" }

cliente → GET /l/cafe-mensual (público)
        ← 200 { description, amount, currency, methods }

cliente → POST /l/cafe-mensual/checkout { paymentMethod:"yape", country:"PE", email }
        ↓ crea Checkout asociado al link
        ↓ redirige al flujo estándar (ONSITE_TOKEN o ENROLLMENT según método)
```

### 7.6 Refund

```
consumer → POST /v1/payments/:id/refund { amount, reason:"REQUESTED_BY_CUSTOMER" }
         ↓ valida estado CAPTURED, valida amount <= remaining
         ↓ crea Refund status=PENDING
         ↓ adapter.refund
         ↓ SUCCESS → Refund SUCCEEDED, Payment → REFUNDED o PARTIALLY_REFUNDED
         ↓ emite refund.succeeded, payment.refunded/partially_refunded
         ← 200 { refundId:ref_…, status:SUCCEEDED }
```

---

## 8. API HTTP (v1)

Prefijo `/v1`. **Auth real:** header `X-API-Key: <apiKeyId>.<secret>` (no `Authorization: Bearer`); el Account se selecciona vía header `X-Account-Id` (no en el body). Key ausente → `MISSING_API_KEY`; key malformada/desconocida/revocada → `INVALID_API_KEY`. Todo POST admite `Idempotency-Key`.

> **Nota de estado — surface realmente montado (`src/app.ts`).** Hoy el back monta: `/v1/platforms`, `/v1/payment-sessions` (+ `GET /:id/available-methods`, `POST /:id/confirm`), `/v1/webhooks/:provider` (entrantes), `/v1/webhook-endpoints`, `/v1/webhook-deliveries`, el checkout público bajo `/c/*`, los crons bajo `/internal/cron/*`, más `/health`, `/openapi`, `/docs`. **No existen** (son roadmap): `/v1/checkouts`, `/v1/payments`, `/v1/refunds`, `/v1/subscriptions`, `/v1/plans`, `/v1/invoices`, `/v1/instruments`, `/v1/customers`, `/v1/payment-links`, `/v1/payment-methods`, `/v1/events`, `/v1/admin/*`, `/l/:slug`, `/pay/:id`, ni `/ready`. El BC `payment` **no tiene adapter HTTP**. La tabla abajo es el **target**; las filas no listadas en esta nota están sin montar.
>
> El confirm real es `POST /v1/payment-sessions/:id/confirm` y devuelve una **unión discriminada** por `kind` (`{kind:'redirect'}` o `{kind:'onsite_token_prepared'}`), sin envelope `next_action`.

### 8.1 Consumer‑facing (target — ver nota de estado para lo realmente montado)

| Método | Ruta | Propósito |
|---|---|---|
| POST | `/v1/checkouts` | crear checkout one‑time o subscription‑setup |
| GET | `/v1/checkouts/:id` | consultar |
| POST | `/v1/checkouts/:id/start` | iniciar flujo redirect (devuelve redirectUrl) |
| POST | `/v1/checkouts/:id/confirm` | confirmar on‑site (con providerToken) |
| POST | `/v1/checkouts/:id/cancel` | |
| POST | `/v1/payments` | cobro directo (requiere `instrumentId` o `providerToken`) |
| GET | `/v1/payments/:id` | |
| GET | `/v1/payments` | listado paginado filtrable |
| POST | `/v1/payments/:id/refund` | |
| GET | `/v1/payments/:id/attempts` | detalle audit |
| GET | `/v1/refunds/:id` | |
| GET | `/v1/disputes/:id` | |
| POST | `/v1/payment-links` | |
| GET | `/v1/payment-links/:id` | |
| POST | `/v1/payment-links/:id/archive` | |
| GET | `/v1/plans` / `POST` / `GET /:id` / `POST /:id/archive` | |
| POST | `/v1/subscriptions` | |
| GET | `/v1/subscriptions/:id` | |
| POST | `/v1/subscriptions/:id/cancel` | `{mode: "immediate" \| "at_period_end"}` |
| GET | `/v1/invoices/:id` | |
| GET | `/v1/invoices` | listar |
| POST | `/v1/invoices/:id/retry` | forzar reintento manual |
| POST | `/v1/instruments/enroll` | |
| GET | `/v1/instruments/:id` | |
| POST | `/v1/instruments/:id/revoke` | |
| POST | `/v1/customers` / `GET /:id` / `PATCH /:id` / `POST /:id/archive` | |
| GET | `/v1/payment-methods` | lista de métodos habilitados para el Platform, filtrable por country/currency |
| POST | `/v1/webhook-endpoints` | |
| GET | `/v1/webhook-endpoints` | |
| POST | `/v1/webhook-endpoints/:id/disable` | |
| GET | `/v1/events` | consulta de DomainEvents (read model público) |
| POST | `/v1/events/:id/replay` | re‑entregar a todos los endpoints suscritos |

### 8.2 Admin / internal

| Método | Ruta | Propósito | Auth |
|---|---|---|---|
| POST | `/v1/admin/platforms` | crear Platform (auto‑crea Account `"default"`) | admin token |
| POST | `/v1/admin/platforms/:id/api-keys` | rotar/generar apiKey (sobre el Platform) | |
| POST | `/v1/admin/platforms/:id/accounts` | crear Account adicional (agregadores) | |
| POST | `/v1/admin/platforms/:id/connections` | setup ProviderConnection (Platform‑scoped) | |
| POST | `/v1/admin/platforms/:id/capabilities` | seed ProviderCapability | |
| POST | `/v1/admin/platforms/:id/accounts/:accountId/contracts` | crear Contract + Terms (account‑scoped) | |

### 8.3 Público (sin auth)

| Método | Ruta | Propósito |
|---|---|---|
| GET | `/l/:slug` | metadata de Payment Link |
| POST | `/l/:slug/checkout` | crear Checkout |
| GET | `/pay/:chk_id` | resolver y redirigir (alternativa al slug) |

### 8.4 Webhooks entrantes

| Método | Ruta | Propósito |
|---|---|---|
| POST | `/v1/webhooks/:provider` | provider notifica; firma verificada por adapter. No hay segmento `:connectionId` en la URL (el `connectionId` se usa solo para dedupe interno). |

### 8.5 Crons (protegido con `Authorization: Bearer <CRON_SECRET>`)

El **único cron montado hoy** es un `GET` sin prefijo `/v1` (montado vía
`app.basePath("/internal")` en `src/app.ts`), autenticado por
`Authorization: Bearer <CRON_SECRET>` (no existe ningún header `X-Cron-Secret` en el
código). Las demás filas son **roadmap — no montadas** (sus BCs no están construidos).

| Método | Ruta | Estado |
|---|---|---|
| GET | `/internal/cron/cleanup-idempotency` | montado hoy |
| POST | `/v1/internal/cron/subscriptions/renew` | roadmap — no montado |
| POST | `/v1/internal/cron/subscriptions/reminders` | roadmap — no montado |
| POST | `/v1/internal/cron/checkouts/expire` | roadmap — no montado |
| POST | `/v1/internal/cron/instruments/health-check` | roadmap — no montado |
| POST | `/v1/internal/cron/webhook-deliveries/retry` | roadmap — no montado |

### 8.6 Sistema

Montados hoy: `/health` (devuelve `{status, version, db}`, con `status:'degraded'` ante fallo de DB), `/docs` (Scalar) y `/openapi` (spec OpenAPI 3.1). **No existe** `/ready` (es roadmap).

---

## 9. Webhooks salientes — contrato público

### 9.1 Firma

**Dos headers** (no uno solo): `X-Payins-Timestamp: <unix_ms>` y `X-Payins-Signature: v1=<hex_hmac_sha256>`.

El HMAC‑SHA256 se calcula sobre `${timestamp}.${raw_body}` con el `signingSecret` del endpoint (provisto por el integrador al registrar). Los eventos suscritos son un enum **cerrado de exactamente 5**: `payment.captured`, `payment.failed`, `payment.expired`, `refund.succeeded`, `refund.failed` (no se aceptan wildcards como `payment.*`).

### 9.2 Shape del payload

El envelope entregado es:

```json
{
  "event_id": "evt_018f...",
  "event_type": "payment.captured",
  "created_at": 1776422096789,
  "delivered_at": 1776422096900,
  "attempt": 1,
  "platform_id": "plat_018f...",
  "payload": { /* shape público estable del recurso */ }
}
```

Los timestamps (`created_at`, `delivered_at`) son **Unix‑ms como `number`**, no strings ISO‑8601 — coherente con la representación única end‑to‑end del servicio.

### 9.3 Entrega y reintentos

- 5 intentos: 30s → 2min → 5min → 10min (ver §5.10); agotados → estado terminal `EXHAUSTED`.
- Orden de entrega **no** garantizado.
- Replay manual disponible vía API: `POST /v1/webhook-deliveries/:id/retry`.

---

## 10. Seguridad

1. **API keys**: prefix visible (`apiKeyId`) + argon2id del secret (`apiKeyHash`), ambos sobre el `Platform`. Revocación inmediata.
2. **Provider credentials**: encriptadas at‑rest con libsodium sealed box. Master key en env/KMS. Nunca logged.
3. **Webhook signing secrets (salientes)**: el integrador **provee** su propio secret (mín. 32 chars) al registrar el endpoint; Payins no lo genera ni lo devuelve. Se almacena **en plaintext** en v1 (`WebhookEndpoint.signingSecret`); encrypt‑at‑rest es un hardening diferido.
4. **Webhooks entrantes**: firma verificada **antes** de parsear. Body raw capturado antes de cualquier parser JSON. Adapters custom por provider.
5. **PCI**: Payins **nunca** acepta PAN/CVV. Solo tokens del provider. Regla de gate: `rg -iE "pan|cvv|card.?number"` fuera de `src/provider-adapters/` debe ser vacío. **Estado:** esta es una regla de política; el gate **aún no está cableado** en CI (CI solo corre biome + tsc + tests; pre‑commit corre biome + `check-layer-violations.cjs` + `check-financial-patterns.cjs`).
6. **TLS** obligatorio en prod.
7. **Rate limiting**: 100 req/min por `connectionId` en `/v1/webhooks/…`; 600 req/min por API key en API consumer.
8. **Idempotency** en todos los mutadores.
9. **Concurrencia de dos capas**: lock distribuido externo (`LockRunner`, Redis) + optimistic locking interno (`version`) + `Serializable`, en Payment, Subscription, Checkout, Invoice, Contract, Instrument. **Lock FIRST, tx SECOND**; keys `payment-lock:<id>`, `subscription-lock:<id>`, `checkout-lock:<id>`, `invoice-lock:<id>`. La propiedad del tenant de keys derivadas se valida **fuera del lock** (un key cross‑tenant sin validar es un vector de DoS).
10. **Reintentos de transacción**: el `TransactionManager` reintenta la capa interna hasta **3 intentos** (backoff exponencial determinista 30ms / 60ms, sin jitter; `MAX_RETRIES=3`); agotado → `409 VERSION_CONFLICT`; timeout del lock externo → `409 LOCK_CONTENDED`. Ambos son transitorios: el cliente reintenta con la misma idempotency key.
11. **Sensitive key filter** en logs (recursivo): `pan, cvv, card_number, secret, token, bearer, authorization, cookie, api_key, api_key_hash, signing_secret, credentials, password, phone, email`.
12. **Tenant isolation**: el `Platform` es el tenant. Cada repositorio recibe `platformId` (y, en entidades account‑scoped, `accountId`) en el scope de la request; queries sin esos filtros fallan en tests. Aislamiento de Platform **y** de Account (toda escritura account‑scoped verifica `account.platformId = platform.id`).

---

## 11. Testing

Estándar idéntico a Wallet.

- **Unit**: BDD Given/When/Then, Vitest + `vitest-mock-extended`. Mocks solo de ports del dominio.
- **E2E**: docker-compose up de Postgres + app + mocks (stripe-mock oficial, ebanx-mock custom). Corren flujos completos.
- **Coverage**: 100% statements/branches/functions/lines enforced.
- **Security suite** (≥12 categorías obligatorias):
  1. API key required + valid.
  2. **Tenant isolation** (Platform A no ve recursos del Platform B).
  3. **Account isolation** + **Instrument reuse cross‑tenant/cross‑Account** bloqueado (un Account no accede a recursos de otro Account del mismo Platform).
  4. **Instrument reuse cross‑customer** dentro del mismo Account bloqueado.
  5. Idempotency replay.
  6. Optimistic lock conflict + lock contention (`LOCK_CONTENDED`).
  7. Sensitive logging audit.
  8. Webhook entrante: firma inválida → 401.
  9. Webhook entrante: replay (timestamp viejo) → 401.
  10. Amount validation: negativos, overflow, minor units erróneas.
  11. Currency mismatch payment vs subscription.
  12. Rate limiting.
  13. SQL injection en query params.
  14. Webhook saliente: firma verificable por el consumidor.

---

## 12. Observabilidad

- **Logs**: Pino JSON → stdout. Canonical log por request (`tracking_id`, `platform_id`, `account_id`, `method`, `path`, `status`, `duration_ms`).
- **Eventos**: cada emisión de DomainEvent se loguea con `event_type`, `aggregate_id`.
- **Sensitive keys filter** documentado en `utils/infrastructure/observability/` (chain `PinoAdapter → SensitiveKeysFilter → SafeLogger`).
- **Métricas** (v1.1): Prometheus counters por provider/method/status, histogram de latencia por provider.

---

## 13. Despliegue

Docker multi‑stage Node 22 Alpine. `docker-compose.dev.yml` + `docker-compose.test.yml` con puertos aislados. Migraciones Prisma con `prisma migrate deploy`. Crons en Vercel Cron / EventBridge protegidos con `Authorization: Bearer <CRON_SECRET>` (el único cron montado hoy es `GET /internal/cron/cleanup-idempotency`, sin prefijo `/v1`; ver §8.5). Master key de credenciales en secret manager.

### Env vars

Variables realmente leídas por `src/config.ts` (+ overrides de los adapters):

```
DATABASE_URL
DIRECT_URL
HTTP_PORT
LOG_LEVEL
CRON_SECRET
CREDENTIAL_MASTER_KEY      # libsodium
REDIS_URL                  # lock distribuido (cuando PAYINS_LOCK_ENABLED=true)
PAYINS_LOCK_ENABLED
PAYINS_LOCK_TTL_MS
PAYINS_LOCK_WAIT_MS
PAYINS_LOCK_RETRY_MS
PAYINS_LOCK_TRANSPORT      # redis | upstash-rest
STRIPE_API_BASE_URL        # override (e2e → stripe-mock); unset en prod
EBANX_API_BASE_URL         # override (e2e → ebanx-mock); unset en prod
```

> **Planificadas, aún NO consumidas por el código:** `LIVEMODE`, `ADMIN_API_KEY`, `BASE_PUBLIC_URL` no se leen en ningún lugar de `src/`. `livemode` es una columna por‑`ProviderConnection`, no un toggle global de env; `ADMIN_API_KEY` correspondería al BC `iam`/rutas `/v1/admin/*` aún no construidas; `BASE_PUBLIC_URL` sería para slugs de payment‑links no implementados. No las marques como requeridas.

`docker:dev` y `start:local` aplican migraciones con `pnpm db:deploy` (`prisma migrate deploy`), **no** `db:push`. Ambos compose levantan también Redis (`:1468`).

---

## 14. Contrato de integración genérico (consumidores)

Payins no conoce a ningún consumidor. La integración es:

1. **Onboarding**: admin crea `Platform` (con su API key — `apiKeyId`/`apiKeyHash` en el propio Platform, sin tabla `ApiCredential` separada) + `ProviderConnection`(s) + `ProviderCapability`(ies) + `Contract` + `WebhookEndpoint`. Al crear el `Platform` se auto‑crea un `Account` por defecto; integradores agregadores (marketplace) crean tantos Accounts adicionales como sub‑merchants tengan, cada uno con su propio Contract.
2. **Uso**: el consumidor llama la API con su key; manda `reference` para idempotencia propia; recibe webhooks firmados.
3. **Reconciliación**: el consumidor reconcilia por `reference`. Payins no sabe qué representa esa `reference`.

Qué haga el consumidor con `payment.captured` (acreditar Wallet, marcar Sale, enviar email, …) es 100% su problema.

### 14.1 Modelo uniforme de `Account`

Todo `Platform` tiene **uno o más `Account`s**. Contratos, customers, instruments, pagos, suscripciones y planes siempre cuelgan de un `Account` — nunca del `Platform` directamente. Esto es uniforme sin importar el modelo de negocio del integrador:

El Account se selecciona vía el header **`X-Account-Id`** (no en el body):

| Tipo de integrador | Accounts por Platform | Notas |
|---|---|---|
| Simple / direct      | **1** (auto‑creado al registrar el Platform) | `X-Account-Id` puede omitirse; el sistema resuelve al Account activo más antiguo. |
| Agregador / marketplace | **N** (uno por sub‑merchant)              | `X-Account-Id` debe indicarse para apuntar al sub‑merchant correcto. Cada Account tiene su contrato comercial. |

**No existe un flag `mode` en el Platform.** La distinción emerge del número de Accounts. Esto mantiene el schema uniforme (`accountId FK NOT NULL` en todas las entidades account‑scoped) y el mismo código sirve para ambos casos de uso. `ProviderConnection` y `ProviderCapability` siguen siendo del Platform — los Accounts comparten la infraestructura con proveedores. (Nota: la auto‑creación del Account `"default"` al registrar un Platform es comportamiento **target**; el use case de registro de Platform aún no está construido.)

Detalle completo del modelo en [Kunfupay-Payins-Back/docs/datamodel.md](Kunfupay-Payins-Back/docs/datamodel.md) secciones 1.2 (Account), 5 (Contract resolution) y 16 (invariantes).

---

## 15. Plan de ejecución (fases incrementales)

### 15.1 Principios de incremento

1. Una fase = un entregable deployable (main verde, coverage 100%, docs, OpenAPI).
2. **Arquitectura cerrada tras la fase de payment/contract.** Desde las fases de integración de proveedores solo se **añade** (adapter nuevo o método nuevo en adapter existente). Si hay que tocar dominio: se para y se refactoriza.
3. Una integración a la vez. Métodos que comparten flow idéntico pueden agruparse; si divergen, se separan.
4. **Checkpoint de abstracción**: tras el Provider #2, diff de `src/{routing,contract,payment,subscription,checkout,instrument,customer}/domain/` y `src/webhook-outbound/domain/` debe ser vacío. Si no → refactor del port antes de continuar. (No hay BC `catalog`: providers/métodos/países/monedas son enums en `utils/kernel/`.)
5. Feature flag por provider (desactivable por env).
6. `docs/providers/<slug>.md` obligatorio por provider.

### 15.2 Fases

El orden y los contenidos siguen el mapa de migraciones canónico en
[Kunfupay-Payins-Back/docs/datamodel.md §17](Kunfupay-Payins-Back/docs/datamodel.md). Cada
fase añade exactamente las migraciones indicadas; no hay migración de `catalog`
(providers/métodos/países/monedas son enums en `utils/kernel/`).

> **Estado real del esquema.** Los nombres lógicos de migración por fase abajo
> (`001_init`, `002_routing`, …) son el **mapa de planificación**. En el repo hoy existe
> **una sola migración** consolidada, `prisma/migrations/20260601161132_core_payins`
> (nombre con timestamp, sin secuencia zero‑padded), que crea las **15 tablas** del core:
> `platforms`, `accounts`, `provider_connections`, `provider_capabilities`, `contracts`,
> `contract_terms`, `payment_sessions`, `payments`, `attempts`, `refunds`, `domain_events`,
> `webhook_endpoints`, `outbound_webhook_deliveries`, `inbound_webhook_events`,
> `idempotency_records`. **No** crea `customers`, `instruments`, `checkouts`,
> `subscriptions`, `invoices`, `plans`, `payment_links` ni `disputes`. Prisma 7: los
> comandos de migración usan `--config prisma/prisma.config.ts`.

#### Fase 0 — Bootstrap (~semana 1)
Repo, Docker, CI, lint, tests triviales verdes, `/health` (`/ready` queda como roadmap, no se construyó), docs base en el repo
del back (`Kunfupay-Payins-Back/docs/projectbrief.md`, `Kunfupay-Payins-Back/docs/domain.md`,
`Kunfupay-Payins-Back/docs/architecture/*`, `Kunfupay-Payins-Back/AGENTS.md`,
`Kunfupay-Payins-Back/CLAUDE.md`). Toolkit `utils/`: `Money`, `Currency`, `Country`, enums
de catálogo, `AppError`/`ErrorKind`, `ILogger` chain, `IClock`, `IIDGenerator` (UUID v7),
`TransactionManager`, `LockRunner`/`IDistributedLock` (Redis/Upstash), middlewares
(`apiKeyAuth`, `idempotency`, `trackingCanonical`, `requestResponseLog`, rate limit).

#### Fase 1 — Baseline `platform` + `idempotency` + `routing` (~semanas 2‑3)
- Migraciones `001_init` (`platforms` + `idempotency_records`, baseline) + `002_routing` (`provider_connections` + `provider_capabilities`).
- `platform/`: entidad `Platform` (con `apiKeyId`/`apiKeyHash` propios), `apiKeyAuth` validando la key. (El `Account` llega en Fase 2.)
- `routing/`: `ProviderConnection` (CRUD admin, encriptación libsodium) + `ProviderCapability` (CRUD admin) + resolver de capability aplicable. Ports por familia de flujo en `utils/application/provider-ports/` (`IRedirectFlowHandler`, `IOnsiteTokenFlowHandler`, `IEnrollmentFlowHandler`, `IDisplayFlowHandler`, `IPushToAppFlowHandler`, `IAutoProviderSubscriptionHandler`, `IRefundHandler`, `IDisputeHandler`, `IWebhookVerifier`, `IWebhookParser`). Ver §6. `ProviderAdapterRegistry` + `NoopProviderAdapter` (solo tests).
- Security tests: tenant isolation (≥2 Platforms en cada e2e), sensitive logging, rate limiting.

#### Fase 2 — `account` + eventos + webhooks I/O + `customer` + `instrument` + `contract` (~semanas 4‑5)
- Migraciones `003_account` (`accounts` + auto‑create del Account `"default"` para cada Platform), `004_events` (`domain_events` en `common/events`), `005_inbound_webhooks` (`inbound_webhook_events` en `common/inboundWebhooks`), `006_webhook_outbound` (`webhook_endpoints` + `webhook_deliveries`, BC), `007_customer`, `008_instrument`, `009_contract`.
- `account`: `Account` como raíz de propiedad; resolución de Account default implícito.
- `common/events`: `DomainEvent` outbox + `IEventPublisher` + `PrismaEventStore`. **Estado:** construido pero **no cableado** — ningún use case emite eventos aún y los webhooks salientes se encolan directamente; el proyector evento→delivery queda pendiente (ver §3.2 / §5.10).
- `common/inboundWebhooks`: `InboundWebhookEvent` + dedupe `(connectionId, providerEventId)` + ingress `POST /v1/webhooks/:provider` con NoopAdapter (el `connectionId` es interno al dedupe, no es segmento de URL).
- `webhook-outbound`: `WebhookEndpoint` CRUD consumer‑facing + `WebhookDelivery` dispatcher con reintentos + cron retry.
- `customer`, `instrument` (entidades + repos account‑scoped), `contract` + `ContractTerm` + resolver de término aplicable.

#### Fase 2b — `iam` (admin auth para el dashboard) (~semana 5‑6)
**Requisito previo del dashboard (`apps/dashboard`, en el repo del front).** Hoy las rutas
`/v1/admin/*` se protegen solo con un `ADMIN_API_KEY` estático (§13); esta fase introduce
autenticación real de administradores.
- Migración `iam` (`admin_users` + `admin_sessions`): entidades `AdminUser` (email `UNIQUE`,
  `passwordHash` argon2id, `role`, `status`, timestamps Unix‑ms BIGINT, `version`),
  `AdminSession` (`adminUserId` FK, `tokenHash`, `expiresAt`, `createdAt`) y el value object
  `Role`. IDs UUID v7.
- BC `iam`: puerto **`IAdminAuthenticator`** (login, validar sesión, resolver rol) + **adapter
  nativo** (admin users + sessions + roles, hashing argon2id, tokens de sesión seguros),
  swappable a un proveedor (Auth0/Clerk/Firebase/Supabase) sin tocar dominio ni dashboard.
- Rol `superadmin` ahora; roles platform‑scoped (self‑service del integrador) en Fase 2 del
  dashboard. Ver §16 y [Kunfupay-Payins-Back/docs/datamodel.md](Kunfupay-Payins-Back/docs/datamodel.md) § `iam`.

#### Fase 3 — `checkout` + `payment` core (~semanas 6‑7)
**Fase que cierra la arquitectura (junto con `contract`).**
- Migraciones `010_checkout` (`checkouts` + `payment_links`) + `011_payment` (`payments` + `attempts` + `refunds` + `disputes`).
- `Checkout` + `PaymentLink` (mode=ONE_TIME solo de momento).
- `Payment` + `Attempt` + `Refund` + `Dispute` (entidades + repos + comandos/queries).
- Cómputo de commission al capturar usando el `Contract`/`ContractTerm` resuelto (contra NoopAdapter simulando capture síncrono).
- OpenAPI completo publicado en `/docs`.
- E2E: flujo ONE_TIME completo contra NoopAdapter, con commission calculada.

**DoD especial:** desde aquí, diff de `domain/` queda congelado salvo refactor explícito.

#### Fase 4 — `subscription` + `plan` + `invoice` (~semana 8)
- Migración `012_subscription` (`plans` + `subscriptions` + `invoices`).
- Entidades `Plan`, `Subscription`, `Invoice` + comandos/queries base (sin provider aún; cobro contra NoopAdapter).
- Cron `/v1/internal/cron/subscriptions/renew` y `/expire` de checkouts listos para enchufar providers.

#### Fase 5 — Provider #1: Stripe (card redirect + on‑site) (~semana 9)
- `StripeAdapter`: `createRedirect`, `confirmWithProviderToken`, `verifyWebhook` (HMAC SHA‑256), `parseWebhook`, `refund`.
- Normalización `checkout.session.completed` / `payment_intent.*` → `PAYMENT_CAPTURED|FAILED`.
- `docs/providers/stripe.md`.
- E2E con `stripe-mock`.

#### Fase 6 — Provider #2: Ebanx (card redirect) (~semana 10)
- `EbanxAdapter`: `createRedirect` con card.
- Webhook RSA‑SHA1 verifier + parser (hash codes → query API).
- `docs/providers/ebanx.md`.
- E2E con ebanx-mock custom o sandbox real.
- **Checkpoint de abstracción**: `git diff src/*/domain/` debe ser vacío tras añadir el segundo provider. Si no, se aborta y se refactoriza el port antes de continuar.

#### Fase 7 — Stripe AUTO_PROVIDER subscriptions (~semana 11)
- Comando `CreateSubscription` con `recurrenceStrategy=AUTO_PROVIDER` sobre las entidades `Plan`/`Subscription`/`Invoice` (de Fase 4).
- Stripe: `createSubscription` (`IAutoProviderSubscriptionHandler`), parseo de `customer.subscription.*` / `invoice.*`.
- Cron `/v1/internal/cron/subscriptions/renew` (AUTO_PROVIDER lo ignora; se deja listo para MANAGED en Fase 11).
- E2E: suscripción Stripe mensual, 2 ciclos simulados.

#### Fase 8 — Ebanx: `mercadopago_wallet` (~2‑3 días semana 11)
- Extiende `EbanxAdapter` con el flow `pushToAppFlow` (o `redirectFlow` según docs de Ebanx para MP) para el método `mercadopago_wallet`.
- Capability seed: `ebanx × mercadopago_wallet × {AR,BR,CL,CO,MX,PE,UY} × REDIRECT × NONE`.
- MercadoPago queda disponible para los tenants sin partnership directo con MP.

#### Fase 9 — Ebanx async methods: Pix + Boleto + PSE (~semana 12)
- Extiende `EbanxAdapter` (`createRedirectAsync` / `createDisplay` para Pix QR) con `payment_type_code` según method.
- Manejo de `PAYMENT_EXPIRED` para boleto.
- E2E por method.

#### Fase 10 — Ebanx Yape enrollment one‑time (~semana 13)
- Ebanx: `IEnrollmentFlowHandler` (`startEnrollment`, `checkEnrollment`, `chargeWithInstrument`, `revokeInstrument`) sobre la entidad `Instrument` (de Fase 2).
- Comandos `EnrollInstrument`, `RevokeInstrument` enchufados al adapter de Ebanx.
- Ebanx: enrollment Yape + charge one‑time.
- E2E: enroll → charge.

#### Fase 11 — Ebanx Yape MANAGED subscription (~semana 14)
- Comando `CreateSubscription` con `recurrenceStrategy=MANAGED`.
- Comando idempotente `RenewSubscription` (key: `subscriptionId + currentCycleNumber`).
- Cron renewal ejecutando MANAGED.
- `IRetryStrategy` + `DefaultRetryStrategy` configurable.
- E2E: 3 ciclos, uno con fallo transitorio recuperado, otro con fallos consecutivos → instrument EXPIRED, subscription CANCELED.

#### Fase 12 — Ebanx Nequi + PLIN MANAGED (~2‑3 días de semana 15)
- Extiende `EbanxAdapter` con métodos nequi/plin recurrentes.
- Prueba empírica: ¿se hace en ≤3 días? Si no → diagnóstico de deuda en el adapter.

#### Fase 13 — REMINDER + Disputes + hardening (~semana 15‑16)
- `recurrenceStrategy=REMINDER` + cron que emite `subscription.reminder_due` con ventana configurable.
- `Dispute` handling en Stripe (`charge.dispute.*`).
- Security suite completa (≥12 categorías; las 14 enumeradas arriba en §15.3).
- Load test k6 1000 rps sostenidos 10 min.
- PCI audit de CI.

**DoD v1:** servicio autónomo completo. Stripe + Ebanx con sus métodos (card, Pix, Boleto, PSE, Yape, Nequi, PLIN, MercadoPago vía Ebanx) y recurrences soportados. Documentación completa. Adopción por consumidores = proyecto aparte.

### 15.3 Definition of Done común a toda fase

- Coverage 100% enforced.
- `pnpm lint` + `pnpm tsc --noEmit` sin errores.
- Unit + E2E verdes en CI.
- OpenAPI actualizado.
- Docs tocados (todos en el repo del back salvo indicación): `Kunfupay-Payins-Back/docs/providers/<slug>.md` si fase de provider; `Kunfupay-Payins-Back/docs/domain.md` si cambió dominio; el `AGENTS.md` del repo afectado si cambió convención.
- Security tests relevantes verdes.
- Feature flag para nuevas integraciones, OFF por default.
- Fase ≥ 5: `git diff src/*/domain/` vacío, o PR incluye ADR en `docs/architecture/` justificando el cambio.

### 15.4 Plantilla `docs/providers/<slug>.md`

```md
# Provider: <slug>

## Resumen
displayName, países, monedas soportadas, links a docs oficiales.

## Handlers implementados
lista de los ports del provider module que este adapter expone (`redirectFlow`,
`onsiteTokenFlow`, `enrollmentFlow`, `displayFlow`, `pushToAppFlow`,
`autoProviderSubscription`, `refund`, `disputes`, `webhookVerifier`, `webhookParser`) y
los métodos / países / monedas cubiertos por cada uno.

## Autenticación
shape del JSON de `credentials` en `ProviderConnection`.

## Webhooks entrantes
algoritmo de firma, headers, normalización provider_event → InboundNormalizedType, campo para dedupe.

## Sandbox / mock
cómo levantar el mock en docker-compose.test.yml, cuentas de prueba.

## Edge cases
rate limits, delays, errores recurrentes.

## Tests
paths unit + e2e.

## Changelog
```

### 15.5 Estimación total

~15‑16 semanas serie. Las fases 5‑13 aceptan paralelismo limitado siguiendo la regla "una integración por rama".

---

## 16. Frontend / Hosted-UI (repo independiente)

Payins ya no es solo un backend "API‑only": incluye también una **UI de checkout hosted** y
un **dashboard superadmin**. **No es un monorepo:** el frontend vive en un repo independiente
(`Kunfupay-Payins-Front/`), hermano del back (`Kunfupay-Payins-Back/`), bajo un paraguas
documental (`Kunfupay-Payins/`). Cada repo se instala, construye, prueba y despliega de forma
independiente. Detalle completo del front en
[Kunfupay-Payins-Front/docs/frontend-architecture.md](Kunfupay-Payins-Front/docs/frontend-architecture.md);
resumen:

**Layout (dos repos bajo un paraguas):**

```
Kunfupay-Payins/                    # paraguas (sólo AGENTS, CLAUDE, README, este plan)
├── Kunfupay-Payins-Back/           # repo independiente — servicio Hono (ÚNICO hogar del dominio)
│   ├── src/                        # capa de aplicación + dominio + adapters
│   ├── prisma/ · scripts/ · docs/ · test/ · api/
│   ├── Dockerfile{,.dev} · docker-compose{,.dev,.test}.yml · vercel.json
│   └── package.json · pnpm-lock.yaml · .npmrc · biome.json · .husky/
└── Kunfupay-Payins-Front/          # repo independiente — workspace pnpm + Turborepo interno
    ├── apps/
    │   ├── checkout/               # Next.js — público, payer‑facing (UI hosted/embebible)
    │   └── dashboard/              # Next.js — superadmin autenticado
    ├── packages/                   # @payins/{api-client,types,money} — internos a este repo
    ├── docs/ · scripts/ · docker-compose.dev.yml
    └── package.json · pnpm-lock.yaml · pnpm-workspace.yaml · turbo.json · .npmrc · biome.json · .husky/
```

**Las dos apps Next.js (App Router + React + TS + Tailwind):**
- **`apps/checkout`** — público. Renderiza nativamente cada `FlowType` (tarjeta vía Stripe
  Elements/Ebanx fields, Pix QR, voucher de Boleto, enrollment Yape/Nequi, redirect
  genérico). Rutas: landing del payment‑link `/l/:slug` y checkout `/c/:token`. **PCI:** el
  SDK del proveedor tokeniza en el navegador; Payins **nunca** ve PAN. **Embebible** vía un
  `payins.js` que monta un iframe (`frame-ancestors` restringido por plataforma); themeable
  por plataforma + i18n. Mayormente Server Components, mínimo JS de cliente.
- **`apps/dashboard`** — superadmin autenticado: configurar métodos de pago, capabilities y
  contratos de comisión; ver plataformas integradoras; observabilidad de pagos/suscripciones/
  disputas; inspeccionar y **re‑lanzar** entregas de webhooks. Usa **TanStack Query**. Fase 2:
  el mismo dashboard con un rol reducido **platform‑scoped** para self‑service del integrador
  (aislado por `platformId`/`accountId`).

**Seam frontend↔backend (`@payins/api-client`):** único punto de integración. Es un cliente
**tipado, generado del spec OpenAPI 3.1 del backend** (`/openapi`, vía `hono-openapi` + Zod).
Una sola fuente de verdad del contrato ⇒ type‑safety end‑to‑end: un cambio de API que rompa a
los fronts **falla en compile time**, no en runtime. Los fronts **nunca** importan dominio del
backend; el server side de Next es solo un **BFF fino** (SSR + sesión del dashboard). El
`@payins/api-client` vive **dentro del repo del front** como package interno del workspace —
no se comparte cross‑repo, se regenera desde el OpenAPI cuando cambia el contrato.

**Auth del dashboard — `iam`:** la autenticación la **posee el backend** en el BC `iam` (ver
§4 y la Fase 2b en §15), detrás del puerto swappable `IAdminAuthenticator`; adapter nativo
(argon2id) ahora, swappable a proveedor después. Hoy solo hay `ADMIN_API_KEY` estático.

**Política conservadora de `packages/` (internos al front):** se comparte **solo lo que no
puede divergir** entre las dos apps Next.js — `api-client` (generado), `types` (contratos) y
`money` (minor‑units / basis‑points / ISO). **No** hay librería de componentes UI compartida
todavía: se arranca con tokens Tailwind y se aplica la **regla de tres** antes de extraer.

**Tooling y deploy:** **Biome** como único linter/formatter dentro de cada repo (cada uno
tiene su propio `biome.json`); el `.npmrc` con `node-linker=hoisted` vive en el back para que
el cliente generado de Prisma resuelva (el front lo replica por comodidad, no por necesidad
estructural). **Deploy:** **3 proyectos Vercel en total** — 1 desde el repo del back (Root
Directory = `.`) y 2 desde el repo del front (Root Directories = `apps/checkout`,
`apps/dashboard`). Cada repo deploya independientemente. Localmente, cada repo trae sus
propios `docker-compose*.yml` (Postgres + back en el back; checkout + dashboard en el front,
apuntando al back vía `host.docker.internal:1464`).

---

## 17. Riesgos

| Riesgo | Impacto | Mitigación |
|---|---|---|
| Fuga de abstracción al añadir provider N+1 | Alto | Checkpoint Fase 6, ADR obligatorio si hay que tocar dominio. |
| Diseño sesgado por un consumidor imaginario | Medio | Regla de CI: `rg -i "kunfupay\|sale\|order\|product"` en `src/` debe ser vacío. |
| PCI scope creep | Alto | Gate de CI: no PAN/CVV fuera de adapters. |
| Provider credentials leak | Crítico | libsodium at‑rest, KMS en prod, filter en logs, rotación documentada. |
| Cron doble ejecución | Medio | Idempotency key `(subscriptionId, currentCycleNumber)`. |
| Sandbox de Ebanx/MP débil | Medio | Mocks propios incluidos en repo; budget de 1‑2 días dentro de cada fase. |
| Contract wildcard mal definido causa cobros sin comisión | Alto | `NO_CONTRACT_TERM` rechaza pagos sin término matcheable. Seed obliga al menos a un término wildcard por contrato `ACTIVE`. |
| Redondeo de basis points genera discrepancias | Medio | Redondeo half‑even documentado, test property‑based. |
| Amounts en distintas monedas entre Payment y Contract | Medio | Si `Contract.currency != Payment.currency`, se rechaza con `CONTRACT_CURRENCY_MISMATCH` en v1. v2 agregará FX. |

---

## 18. Decisiones pendientes

1. **Repo**: **dos repos independientes** bajo el paraguas documental `Kunfupay-Payins/` (hermano de `Wallet/`). `Kunfupay-Payins-Back/` (Hono, standalone) y `Kunfupay-Payins-Front/` (workspace pnpm + Turborepo interno con `apps/{checkout,dashboard}` Next.js y `packages/` `@payins/{api-client,types,money}` privados a este repo). Sin plumbing al nivel del paraguas. — Confirmado.
2. **DB**: instancia separada en prod; schema `payins` compartiendo instancia con Wallet en dev.
3. **Hosting**: el mismo que use Wallet.
4. **Mock de Ebanx**: construir en Fase 9 (1‑2 días).
5. **FX entre currencies**: fuera de v1.
6. **Comisiones v1**: solo en `Contract.currency` === `Payment.currency`. v2 añade conversión.
7. **Retention de `OutboundWebhookDelivery` en estado terminal `EXHAUSTED`**: 90 días default.
8. **Multi‑tenant desde día 1**: sí, con ≥2 Platforms en e2e.
9. **Payment Link domain**: subdominio dedicado `pay.example.com` o path `/l/…` dentro del mismo host. — Recomendación: path en v1, subdominio opcional en v2.
10. **Comisiones: accounting treatment**: en v1 la commission se **calcula y registra** pero **no se cobra** al consumidor (Payins no debita). v2 puede añadir Wallet‑style ledger para cobro.

**Resueltas / alineadas con Wallet** (ya no son decisiones abiertas):
11. **Modelo de tenancy**: `Platform` (tenant, API key propia) + `Account` (raíz de propiedad account‑scoped); **sin** `Organization` ni tabla `ApiCredential` separada, **sin** flag `mode`. — Resuelto.
12. **IDs**: **UUID v7** (RFC 9562) generados en app vía `IIDGenerator`; nunca UUID v4; los prefijos semánticos son solo pista de legibilidad. — Resuelto (era ULID).
13. **Datos de referencia**: providers/métodos/países/monedas son **enums en `utils/kernel/`**, no tablas DB; no hay BC `catalog` ni migración de catálogo. — Resuelto.
14. **Concurrencia**: modelo de **dos capas** (`LockRunner` distribuido Redis + optimistic locking `version` + `Serializable`, hasta **3 intentos** internos con backoff exponencial determinista 30ms/60ms, sin jitter). — Resuelto.
15. **Timestamps**: **Unix‑ms (BigInt)** como representación única interna y en DB; ISO‑8601 solo en payloads públicos de webhooks. — Resuelto (antes se modelaban como columnas de fecha/hora con zona e ISO‑8601 interno).

---

## 19. Referencias

**Arquitectura de inspiración (patrón a replicar):**
- [Wallet/](../Wallet/) · [AGENTS.md](../Wallet/AGENTS.md) · [docs/architecture/backend-architecture.md](../Wallet/docs/architecture/backend-architecture.md)

**Inspiración funcional (convenciones de API y modelado):**
- Stripe API reference (PaymentIntents, Setup Intents, Payment Links, Subscriptions, Invoices, Disputes, Events).
- Mollie API (Payments, Mandates, Subscriptions, Profiles).
- Adyen (Payment Methods Resources, Recurring).
- Checkout.com (Payment Sessions, Instruments).

**Caso de estudio (analítico, no dependencia):**
- [Kunfupay-Nextjs/src/_payments/back/](../Kunfupay/Kunfupay-Nextjs/src/_payments/back/)
- [Kunfupay-Nextjs/src/_paymentsIntegration/back/](../Kunfupay/Kunfupay-Nextjs/src/_paymentsIntegration/back/)
- [Kunfupay-Nextjs/src/_payouts/back/](../Kunfupay/Kunfupay-Nextjs/src/_payouts/back/) (referencia interna limpia)
- [YAPE_SUBSCRIPTIONS_ROADMAP.md](../Kunfupay/Kunfupay-Nextjs/YAPE_SUBSCRIPTIONS_ROADMAP.md) — para entender el flujo Yape enrollment + subscription.
- [SUBS_WITHOUT_AUTH_ROADMAP.md](../Kunfupay/Kunfupay-Nextjs/SUBS_WITHOUT_AUTH_ROADMAP.md) — para entender el patrón claim token.
- [KUNFUPAY_WALLET_MIGRATION.md](../KUNFUPAY_WALLET_MIGRATION.md) — precedente de separación de servicio.

Ninguna de estas referencias condiciona el diseño ni el calendario de Payins.

---

## 20. Checklist de salud del plan

- [x] Multi‑currency nativo (campo `currency` en Payment/Subscription/ProviderCapability/Plan/Contract).
- [x] Multi‑provider (Registry + port declarativo).
- [x] Multi‑país (campo `country` en Capability/Contract/Customer/Payment).
- [x] Multi‑método con mapeo many‑to‑many contra providers (Capability).
- [x] Flujos distintos (`FlowType`: REDIRECT, ONSITE_TOKEN, REDIRECT_ASYNC, DISPLAY, ENROLLMENT, PUSH_TO_APP — los 6 reales del enum `FLOW_TYPES`).
- [x] Tres arquetipos de recurrencia (`RecurrenceStrategy`: NONE, AUTO_PROVIDER, MANAGED, REMINDER).
- [~] Pagos one‑time + refunds: **construidos**. Suscripciones, payment links y disputes: **modelados como ciudadanos de primera clase en el diseño, aún no implementados** (roadmap).
- [x] **Contratos comerciales con comisión/markup por scope** (Contract + ContractTerm).
- [x] **Montos siempre enteros** (BIGINT minor units), porcentajes en basis points.
- [x] **IDs UUID v7** (RFC 9562) generados en app vía `IIDGenerator`; nunca UUID v4 (no ULID).
- [x] **Timestamps Unix‑ms (BigInt)** como representación única interna y en DB; ISO‑8601 solo en payloads públicos de webhooks.
- [x] **Datos de referencia en código** (`utils/kernel/`): providers/métodos/países/monedas como enums, no tablas DB; no hay BC `catalog`.
- [x] **Tenancy `Platform` + `Account`** (no `Organization`); API key sobre el `Platform`; sin flag `mode`; account‑scoped lleva `platformId` + `accountId` NOT NULL.
- [x] **Concurrencia de dos capas** (`LockRunner` distribuido Redis + optimistic locking + `Serializable`, hasta 3 intentos internos, backoff determinista 30/60ms).
- [x] Webhooks entrantes (verificación + parseo + dedupe + normalización) y salientes (firma + reintentos + replay).
- [~] Event log auditable (`DomainEvent`): tabla `domain_events` + puerto `IEventPublisher` + `PrismaEventStore` **construidos pero dormidos** (ningún use case emite eventos aún; webhooks salientes encolados directamente).
- [x] Una integración a la vez en el plan; arquitectura cerrada tras la fase de payment/contract.
- [x] Feature flag por provider; doc por provider.
- [x] Multi‑tenant desde día 1 con tests de aislamiento (Platform y Account) obligatorios.
- [x] Zero‑knowledge de cualquier consumidor en el dominio.
- [x] Vocabulario 100% nuevo — ningún nombre heredado del sistema de referencia.
- [x] Arquitectura fiel a Wallet (DDD + Hexagonal + CQRS, mismos patrones de transaction, lock distribuido, logger, idempotency, optimistic lock).
