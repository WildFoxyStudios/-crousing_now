# Cruising Now

Backend **Rust** + app **Kotlin Multiplatform** (Android + iOS) para un clon de
Grindr, diseñado para ser **rápido**, **mantenible** y de **coste de infraestructura
prácticamente cero**.

```
gey/
├── server/          Backend Rust (axum + sqlx + PostgreSQL/PostGIS + Redis)
│   ├── migrations/  Esquema completo: 50 tablas, 138 índices
│   └── src/
├── app/             App Kotlin Multiplatform
│   ├── shared/core/ Dominio, red, repositorios, DI  (compartido 100%)
│   └── shared/ui/   Pantallas Compose Multiplatform (compartido 100%)
├── androidApp/      Entry point Android
├── iosApp/          Entry point iOS
└── docker-compose.yml
```

---

## Arranque rápido

Requisitos: **Docker Desktop** y **Rust 1.99** o superior (el resto se instala dentro
del contenedor).

```bash
cp .env.example .env
docker compose --profile local-s3 up -d --build
```

| Servicio | URL |
|---|---|
| API | http://localhost:8080 |
| OpenAPI / Swagger | http://localhost:8080/docs |
| Mailpit (correos) | http://localhost:8025 |
| MinIO (ficheros) | http://localhost:9001 · `minioadmin` / `minioadmin` |
| PostgreSQL | `localhost:5432` |
| Redis | `localhost:6379` |

Las migraciones se aplican **automáticamente** al arrancar el backend.

---

## Decisiones de arquitectura y por qué

### El coste cero no es casualidad: es una decisión de diseño

| Palanca | Elección | Ahorro |
|---|---|---|
| Media | **Uploads presignados** a S3/R2. El binario de vídeo **nunca pasa por el servidor** | Sin ancho de banda de subida ni RAM para 50 MB de vídeo |
| Egress | **Cloudflare R2** no cobra salida | Un usuario que ve 100 MB/día de fotos no cuesta nada |
| Base de datos | **PostgreSQL + PostGIS** en una sola máquina | Un motor cubre relacional + geoespacial; sin Mongo + PostGIS + Elasticsearch |
| Búsqueda | `pg_trgm` + GIN en vez de Elasticsearch | Un servicio menos = un servicio menos que mantener y pagar |
| Presencia | **Redis con TTL de 90 s** en vez de un broker | Sin estado que sincronizar; si Redis cae, la app sigue |
| Fan-out realtime | **Redis pub/sub** entre réplicas | Escalar es añadir réplicas sin tocar código |

### Descubrimiento por cercanía

La query del core del producto está en `server/src/repo.rs::discover_nearby`:

- `ST_DWithin` con `&&` sobre un índice **GIST** parcial (`WHERE is_live`), así que
  Postgres sólo toca las filas con posición viva.
- Índice **H3 resolución 9** (~174 m) precalculado en `profiles.h3_index` y en
  `user_locations.h3_index`, para poder prefiltrar celdas antes del cálculo.
- Distancia calculada por **PostGIS**, no en Rust: en SQL el radio es indexable;
   obligaría a traer candidatos de más.
- Radio máximo **acotado a 200 km**: más allá los resultados son ruido y el
  planner empieza a degradarse.
- Paginación por **keyset** (`distance_m, user_id`), nunca `OFFSET`.

### Chat en tiempo real

```
cliente A ──ws──┐                        ┌──ws── cliente B
                ├── nodo 1 ──redis pub───┤
cliente C ──ws──┘        (sub)            └──ws── cliente D
```

Cada nodo mantiene sus conexiones en memoria (`DashMap`) y está suscrito a Redis.
Si el nodo que recibe un mensaje **no** es el que tiene abierta la WebSocket del
destinatario, el evento viaja por pub/sub. Añadir réplicas no requiere cambios de
código.

Backpressure: cada conexión tiene un canal de 256 mensajes. Si el cliente no lee,
se **descarta** el evento en lugar de crecer sin límite en RAM.

Reconexión: backoff exponencial con **jitter completo**. Sin jitter, 50 000
clientes reconectan en el mismo milisegundo cuando cae el servidor y lo tumban
otra vez al arrancar.

### Seguridad

- **Argon2id** con parámetros re-calibrables. El hash se **rehace en vuelo** cuando
  subes el coste: los usuarios no notan nada y las contraseñas viejas siguen valiendo.
- **Refresh tokens opacos**, no JWT. Se guarda sólo su SHA-256: una fuga de la base
  no permite forjar sesiones. La rotación por *familia* detecta *reuse* y revoca
  la cadena entera de golpe.
- **Access tokens de 15 min**. Cortos a propósito: si son más largos, el *logout*
  no puede ser real.
- **Lista blanca de MIME**. SVG, HTML y ejecutables se rechazan en el servidor.
- **Ningún error 5xx filtra detalles**: SQL, rutas del sistema y nombres de bucket
  nunca salen hacia el cliente.
- **Las cabeceras de autorización nunca se registran** en los logs, ni en
  desarrollo.

### Moderación en tres capas

1. **Proveedor externo** (Hive): clasifica NSFW en imágenes y vídeos.
2. **Reglas internas**: veto duro y sin red para material declarado de menores,
   dimensiones sub-mínimas (la técnica estándar para eludir filtros) y oversized.
3. **Post-publicación**: reports de la comunidad.

Política de fallo configurable (`MODERATION_FAIL_OPEN`): *fail-closed* bloquea el
contenido si el proveedor cae; *fail-open* prioriza no tumbar el producto. En
producción, arrancar sin moderación configurada **es un error de arranque**.

---

## Variables de entorno

Copia `.env.example` a `.env`. Todas tienen default de desarrollo salvo las
marcadas como obligatorias en producción.

### Imprescindibles en producción

| Variable | Para qué |
|---|---|
| `DATABASE_URL` | Cadena de conexión a Postgres |
| `REDIS_URL` | Redis |
| `JWT_ACCESS_SECRET` | ≥32 bytes. `openssl rand -base64 48` |
| `JWT_REFRESH_SECRET` | ≥32 bytes, **distinto** del anterior |
| `MODERATION_PROVIDER` | `hive`. `none` hace fallar el arranque en producción |

### Servicios externos que conectaremos

| Servicio | Variables | Estado |
|---|---|---|
| **Cloudflare R2** | `S3_ENDPOINT`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `S3_BUCKET` | Pendiente de tus credenciales |
| **Firebase FCM** | `FCM_PROJECT_ID`, `FCM_CLIENT_EMAIL`, `FCM_PRIVATE_KEY_PATH` | Pendiente |
| **Apple APNs** | `APNS_KEY_ID`, `APNS_TEAM_ID`, `APNS_BUNDLE_ID`, `APNS_PRIVATE_KEY_PATH` | Pendiente |
| **Hive AI** | `HIVE_API_KEY`, `HIVE_API_URL` | Pendiente (hasta entonces, `MODERATION_PROVIDER=stub`) |
| **Email** | `SMTP_*`, `EMAIL_PROVIDER` | Mailpit local mientras tanto |

**Seguridad de las claves**: las claves privadas (`.p8` de APNs, service account de
Firebase) se leen de **fichero** vía `*_PRIVATE_KEY_PATH`, no de una variable de
entorno. En producción eso mantiene el material criptográfico fuera del entorno
del proceso.

---

## Stack, con versiones exactas

Todas verificadas contra crates.io y Maven Central el **14/10/2026**. Sin
`+`, sin `latest.release`, sin rangos abiertos.

### Backend

| Crate | Versión | | Crate | Versión |
|---|---|---|---|---|
| `axum` | 0.8.9 | | `argon2` | 0.6.0 |
| `sqlx` | 0.9.0 | | `jsonwebtoken` | 11.1.0 |
| `tokio` | 1.53.3 | | `uuid` | 1.28.0 |
| `tower` | 0.5.3 | | `blake3` | 1.8.7 |
| `tower-http` | 0.7.1 | | `dashmap` | 6.2.1 |
| `deadpool-redis` | 0.23.1 | | `h3o` | 0.11.0 |
| `redis` | 1.7.1 | | `reqwest` | 0.13.5 |
| `utoipa` | 6.0.0 | | `aws-sdk-s3` | 1.152.0 |
| `tracing` | 0.1.44 | | `thiserror` | 2.0.21 |

Rust: **1.99**, edition **2024**.

### App

| Librería | Versión | | Librería | Versión |
|---|---|---|---|---|
| Kotlin | 2.4.21 | | Compose Multiplatform | 1.12.1 |
| AGP | 9.4.1 | | Compose Material3 | **1.9.0** ⚠ |
| Ktor | 3.6.0 | | kotlinx-coroutines | 1.11.0 |
| kotlinx-serialization | 1.11.0 | | kotlinx-datetime | 0.8.0 |
| Koin | 4.2.2 | | SQLDelight | 2.4.1 |
| Coil | 3.6.3 | | navigation-compose | 2.9.2 |
| Turbine | 1.2.1 | | Kotest | 6.2.5 |

⚠ **El material3 de JetBrains no sigue la línea de Compose Multiplatform.** Su
última estable es 1.9.0 mientras runtime/ui/foundation van por 1.12.1; pedir
`material3:1.12.1` falla con `Could not find`. Por eso lleva versión propia en
`libs.versions.toml`.

Gradle **9.6**. No hay target `iosX64`: Xcode 16 eliminó ese simulador y las
dependencias ya no lo publican.

---

## Testing

```bash
# Backend: tests unitarios (no requieren base de datos)
make test-rs

# Calidad completa del backend
make ci          # fmt + clippy + check + test

# App: todo el módulo KMP compilado y testeado dentro de Docker
scripts/kt.sh sync
scripts/kt.sh gradle :shared:core:jvmTest
scripts/kt.sh gradle :shared:core:compileKotlinIosSimulatorArm64
scripts/kt.sh gradle :androidApp:assembleDebug

# Smoke test end-to-end contra el stack levantado
make smoke
```

La app **no se compila en la máquina del usuario**: no hay JDK ni Android SDK, y
el bind mount de Windows hace que Gradle falle con `java.io.IOException`. Todo
corre en Docker contra volúmenes Linux. `scripts/kt.sh` encapsula las cuatro
trampas del entorno y las documenta dentro del propio script.

Los tests unitarios del backend cubren hashing de contraseñas, JWT (expiración vs.
token inválido), cursores de paginación, la lógica de moderación y el hub de
WebSocket. **No requieren ni Postgres ni Redis**, así que corren en segundos.

> **Kotlin/Native compila targets iOS desde Linux.** No es lo habitual —el
> *linking* sí necesita macOS—, pero el compilador de Kotlin sí produce los klibs
> en Linux. Eso permite verificar de verdad el cinterop del Keychain en Windows y
> en CI, en vez de dejar el `actual` de iOS sin comprobar hasta tener un Mac.
> Para iterar sobre ese interop hay `scripts/kprobe.sh`, que compila un `.kt`
> suelto con `konanc` en segundos.

---

## Estado actual

### Verificado en ejecución

| Prueba | Resultado |
|---|---|
| `cargo check --lib` | **0 errores, 0 warnings** |
| `cargo clippy --all-targets --all-features -- -D warnings` | **0 warnings** |
| `cargo test --lib` | **58/58 en verde** |
| `scripts/smoke-test.sh` | **35/35 en verde** |
| `:shared:core:compileKotlinJvm` | **BUILD SUCCESSFUL** |
| `:shared:core:jvmTest` | **BUILD SUCCESSFUL — tests de contrato** |
| `:shared:core:compileKotlinIosSimulatorArm64` | **BUILD SUCCESSFUL** |
| `:shared:ui:compileKotlinJvm` | **BUILD SUCCESSFUL** |
| Esquema | 51 tablas · 139 índices · 8 migraciones |
| `/health/live`, `/health/ready` | 200 (ready comprueba Postgres **y** Redis) |
| `/api-doc/openapi.json`, `/docs` | 200 |

El smoke test recorre el camino crítico contra el servidor real: registro,
login, **rotación de refresh y detección de reuse**, perfil, rechazo de
menores, ubicación, **discovery con la distancia calculada**, match mutuo,
chat, **idempotencia de mensajes**, aislamiento entre conversaciones, bloqueo,
lista blanca de MIME y reportes.

### Bugs reales encontrados al verificar

Ninguno se habría visto leyendo el código: son fallos de contrato, de tipos de
Postgres, de crates nuevas y de APIs que sólo existen en una plataforma, y sólo
se manifiestan al ejecutar.

| Bug | Síntoma |
|---|---|
| DTOs en `snake_case` frente a un cliente Kotlin en `camelCase` | Con `ignoreUnknownKeys` no daba error: **daba campos vacíos en silencio** |
| `ON CONFLICT` sobre índices únicos parciales | 42P10 en ejecución: mensajes, conversaciones y registro de usuario |
| Parámetros `$5/$6/$7` desalineados con los `.bind()` | 500 en todos los discoveries |
| `format!("{g:?}")` → `"Man"` en vez de `"man"` | 500 al guardar género o preferencias |
| ENUM y arrays de ENUM sin cast | 500 al leer perfil, matches o discovery |
| `jsonwebtoken` sin backend criptográfico | **panic en tiempo de ejecución** |
| Parser del prefijo PHC leía la versión como costes | Rehacía todas las contraseñas en cada login |
| `String.format` y reflexión de JVM en `commonMain` | Compilaban en JVM y Android, rompían en iOS |
| `org.jetbrains.compose.material3:1.12.1` no existe | Esa librería tiene su propia línea: la última estable es 1.9.0 |
| `iosX64()` sin dependencias que lo publiquen | Xcode 16 eliminó ese simulador |
| `EncryptedSharedPreferences` obsoleto y sin dependencia declarada | El target Android de `shared/core` no compilaba |

El detalle completo está en `docs/ESTADO.md`.

### Pendiente

- **SQLDelight**: el esquema `.sqm` está escrito (caché de perfiles, medios,
  conversaciones, mensajes y cursor de discovery) pero **sin verificar**: falta
  ejecutar la generación y confirmar que compila.
- **`iosApp/`**: el proyecto de Xcode. `shared/core` y `shared/ui` ya compilan
  para los targets de iOS, pero el *linking* y el empaquetado necesitan un Mac.
- **Imagen Docker del backend**: el `Dockerfile` se reescribió pero no se ha
  construido (BuildKit compila las 370 crates sin caché compartida). El binario
  se verificó ejecutándolo contra el stack.
- **Claves de servicios externos**: R2, FCM y APNs. Se leen de fichero vía
  `*_PRIVATE_KEY_PATH`; nunca de variables de entorno ni del repositorio.

---

## Licencia

Apache-2.0.
#   - c r o u s i n g _ n o w  
 