# Ketoway — Contrato del API (reconstruido)

> Ingeniería inversa de la app **Ketoway** (`com.tallfive.ketoway` v2.6.8). **Borrador de trabajo.**

## Procedencia y método

| Dato | Valor |
|---|---|
| App | Ketoway (`com.tallfive.ketoway`), dieta keto / ayuno |
| Versión | 2.6.8 (versionCode 2006008), minSdk 26 / targetSdk 36 |
| Arquitectura | **React Native + Hermes** (bytecode v96) + Expo |
| Fuente analizada | `assets/index.android.bundle` (7.9 MB, bytecode Hermes) |
| Método | Decompilado con `hermes-dec` 0.1.8 (`hbc-decompiler`) → ~56 MB de pseudo-JS (IR de registros). Rutas extraídas por grep + trazado manual de registros. |

**Advertencia:** el decompilador produce pseudocódigo de registros (bajo nivel). Las URLs, métodos y el flujo de auth se leen con confianza; los **payloads exactos** salen de trazar registros a mano y pueden tener campos adicionales no capturados. Los 9 dex y las 29 `.so` del APK son solo el runtime RN/Expo, no la lógica.

---

## 1. Arquitectura de red

- **Backend propio:** `https://acs.flmng.tools/api` (prod) · `https://acs.sandbox-flmng.live/api` (sandbox). En el código es `config.CLOUD_FUNCTIONS_URL` → es un front de **Firebase Cloud Functions**.
- **Identidad/Auth:** Firebase Auth (email/password y `signInWithPhoneNumber` vía Identity Toolkit).
- **Suscripciones:** RevenueCat (`api.revenuecat.com`) + Paddle + Stripe.
- **Telemetría:** Sentry (DSN en el cliente) + Firebase Crashlytics. Push vía Expo (`exp.host`).

### Modelo de autenticación
- Token OAuth-style: `accessToken` + `refreshToken` + `expires_in`/`expirationTime` (se renueva).
- Respaldado por Firebase. El `accessToken` viaja en header **`Authorization`** (Bearer) hacia flmng.
- El alta (`/register`) va **sin** `Authorization` (todavía no hay sesión).

### Claves embebidas (de cliente, públicas por diseño)
- Firebase/Google API keys: `AIzaSyC-tp5B1VDFyNfllc1rglcXwTzTODQT6qQ`, `AIzaSyDKtDJSUM75JeFyp_i19VSdrlTvdiJ89Mc`
- RevenueCat public SDK key: `appl_tEGMCOfwfMtOkRAEVptuYTeSZku`
- No se hallaron secretos de servidor (p.ej. Stripe `sk_`), lo cual es lo correcto.

---

## 2. Inventario de endpoints — `acs.flmng.tools/api`

Método: **(c)** = confirmado en el código, **(i)** = inferido por nombre/semántica.

| Ruta | Método | Qué hace |
|---|---|---|
| `/register` | POST (c) | alta de usuario (ver §3) |
| `/userDetails`, `/userDetails/` | GET / PUT (i) | perfil del usuario |
| `/fasting`, `/fasting/plan` | GET (i) | plan de ayuno actual |
| `/fastingHistory`, `/fastingHistory/` | GET / POST (i) | historial de ayunos (lee y registra) |
| `/subscriptions`, `/subscriptions/relevant` | GET (i) | estado y ofertas de suscripción |
| `/messages` | GET (i) | mensajes/notificaciones in-app |
| `/v1/content/mains/schemas/1/locales/…` | GET (i) | contenido (planes/comidas) por locale |
| `/v1/content/challenges/schemas/1/locales/…` | GET (i) | challenges |
| `/v1/config/upsells/schemas/1` | GET (i) | config de upsells/paywall |
| `/v1/events` | POST (i) | telemetría de eventos |
| `/v1/recaptchaParams`, `/v2/recaptchaConfig` | GET (i) | parámetros reCAPTCHA |
| `/v2/passwordPolicy` | GET (i) | política de contraseñas |

### No pertenecen a flmng (no confundir)
- `/v1/subscribers`, `/v1/receipts` → **RevenueCat** (`api.revenuecat.com`), `Authorization: Bearer appl_…`.
- `/v1/token`, `identitytoolkit` → **Firebase / GCP Identity Toolkit**.

---

## 3. `POST /register` — contrato completo (trazado)

Origen: hook `useRegister` (`useMutation` de React Query) → `axios.post`.

- **Método:** `POST` (confirmado).
- **URL:** `config.CLOUD_FUNCTIONS_URL` + `/register`.
- **Auth:** ninguna (alta previa a la sesión).
- **Body (JSON):**
```json
{
  "userCredentials": { "email": "...", "password": "..." },
  "userRTDBProperties": {
    "weightHistory": { "<yyyyMMdd de hoy>": <startingWeight> },
    "userDetails": { ...DEFAULT_USER_DETAILS... }
  }
}
```
- `weightHistory` se indexa por la **fecha de hoy** (`formatDateTimeToyyyyMMdd(DateTime.now())`), valor = peso inicial.
- `userDetails` = `DEFAULT_USER_DETAILS`, o `DEFAULT_EXERCISE_USER_DETAILS` si `ENABLED_FEATURES.appType === 'exercise'`.
- `RTDB` = Firebase Realtime Database (la Cloud Function escribe estas props).

### Flujo post-registro (confirmado)
1. `POST /register` → la Cloud Function crea el user de Firebase Auth y escribe las props en RTDB.
2. Cliente: `signInWithEmailAndPassword(auth, email, password)` (Firebase).
3. `logRegisterEvent(user.uid)` (analytics).

### `DEFAULT_USER_DETAILS` (expandido)
Base + overrides del getter (`gender` ← `defaultUserGender` o `Gender.female`; `unitsOfMeasure` ← `Imperial`):
```js
{
  startingWeight: 80,
  goalWeight: 70,
  gender: "female",            // o defaultUserGender del build
  height: 172,
  activityLevel: 1,            // enum
  age: 55,
  unitsOfMeasure: "Imperial",  // enum
  numberOfWorkoutsPerWeek: 3
}
```

### `DEFAULT_EXERCISE_USER_DETAILS` (appType = "exercise")
```js
{
  startingWeight: (gender === female ? 68 : 90),
  gender: "female",            // o defaultUserGender
  activityLevel: 1,
  unitsOfMeasure: "Imperial",
  numberOfWorkoutsPerWeek: 3
}
```
Son **semillas por defecto**; el quiz de onboarding las sobreescribe luego (vía `PUT /userDetails`). Los valores (80/70/172/55) son placeholders, no datos reales del usuario.

---

## 4. Pendientes / preguntas abiertas

1. **Métodos no confirmados** (marcados "i"): verificar GET/PUT/POST exacto por endpoint en el IR.
2. **Payloads de** `/userDetails` (PUT), `/fasting/plan`, `/subscriptions/relevant`, `/v1/events` — trazar.
3. **Enums**: `Gender`, `UnitsOfMeasure`, valores de `activityLevel`.
4. **Headers completos** por request (además de `Authorization`): reCAPTCHA token, versión de app, locale.
5. **Lógica de paywall/premium**: cómo combina RevenueCat + `/subscriptions/relevant` para decidir acceso.

---

*Reconstruido por análisis estático del bundle Hermes. No se incluye el binario ni el bundle decompilado en este repo (material propietario de Tall Five / Ketoway).*
