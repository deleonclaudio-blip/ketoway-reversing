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
| `/messages` | GET (i) | mensajes/notificaciones in-app |

> **`/subscriptions/relevant`** no va en esta tabla: usa otra base (**funnel API**, no `CLOUD_FUNCTIONS_URL`) — ver §5.
| `/v1/content/mains/schemas/1/locales/…` | GET (i) | contenido (planes/comidas) por locale |
| `/v1/content/challenges/schemas/1/locales/…` | GET (i) | challenges |
| `/v1/config/upsells/schemas/1` | GET (i) | config de upsells/paywall |
| `/v1/recaptchaParams`, `/v2/recaptchaConfig` | GET (i) | parámetros reCAPTCHA |
| `/v2/passwordPolicy` | GET (i) | política de contraseñas |

### No pertenecen a flmng (no confundir)
- `/v1/subscribers`, `/v1/receipts` → **RevenueCat** (`api.revenuecat.com`), `Authorization: Bearer appl_…`.
- `/v1/events` → **RevenueCat** (`POST https://e.revenue.cat/v1/events`, o `httpConfig.eventsURL`): ingestión de eventos del SDK (paywall/compras). Cuerpo incluye `apiKey`, `appUserId`, `eventsUrl`, `silent`.
- `/v1/token`, `identitytoolkit` → **Firebase / GCP Identity Toolkit**.

### 2.1 Datos de usuario — Firebase Realtime Database (NO es flmng HTTP)

Los datos por usuario **no pasan por el API HTTP**: el cliente los lee/escribe directo contra **Firebase RTDB** bajo `users/<uid>/…` (uid = Firebase Auth). Confirmado en el bundle (`database.ref(...).onValue/get/update`).

| Nodo RTDB | Operaciones | Función(es) |
|---|---|---|
| `users/<uid>/userDetails` | `onValue` (listener), `get` (one-shot), `update` (patch) | `useListenToMWRTDB`, `updateUserProperty(key,value)` |
| `users/<uid>/fasting/plan` | `update` | `chooseFastingPlan(plan)` |
| `users/<uid>/fastingHistory` | `get` | — |
| `users/<uid>/weightHistory` | `get`, `update` | — |

**`updateUserProperty(key, value)`** (equivalente al "PUT" del perfil):
```js
database.ref('users/' + uid + '/userDetails').update({ [key]: value })
```
Patch de **un campo a la vez** (así el quiz de onboarding guarda cada respuesta: `startingWeight`, `goalWeight`, `gender`, `height`, `age`, `activityLevel`, `unitsOfMeasure`, `numberOfWorkoutsPerWeek` — los mismos de `DEFAULT_USER_DETAILS`).

**`chooseFastingPlan(plan)`** (hook `useChooseFastingPlan`, `mutationKey: ['fasting']`):
```js
database.ref('users/' + uid + '/fasting/plan').update(plan)
```
El `plan` es uno de los presets del catálogo **`FASTING_PLANS`**:
```js
[
  { title: "14:10", level: 1, fastingHours: 14, eatingHours: 10 },
  { title: "16:8",  level: 1, fastingHours: 16, eatingHours: 8  },
  { title: "18:6",  level: 2, fastingHours: 18, eatingHours: 6  },
  { title: "20:4",  level: 3, fastingHours: 20, eatingHours: 4  },
]
```
Forma: `{ title: string, level: number, fastingHours: number, eatingHours: number }`.
`startTime`/`endTime`/`isActive` **no** van acá — son de la sesión de ayuno activa (hook `useStartFasting`), otro nodo.

> Corrección respecto a una versión previa: `/userDetails`, `/fasting`, `/fasting/plan` y `/fastingHistory` **no son rutas HTTP de flmng** sino nodos de Firebase RTDB.

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
Son **semillas por defecto**; el quiz de onboarding las sobreescribe luego (vía `updateUserProperty` → Firebase RTDB `users/<uid>/userDetails`, ver §2.1). Los valores (80/70/172/55) son placeholders, no datos reales del usuario.

---

## 5. Suscripciones y paywall

### Funnel API (base distinta de flmng)
```
GET clients/<client_code>/subscriptions/relevant?origin=<project_origin>
```
- Base `createFunnelApiBase` (`projectConfig`), método GET, query param `origin`.
- Devuelve la suscripción "relevante" del **funnel web** (compra previa por Paddle/Stripe antes de instalar).

### Gate premium (RevenueCat)
Constantes del módulo:
- **Entitlements:** `pro` (premium), `coach_access` (función coach).
- **Productos:** `in_app_coach_1m`, `in_app_coach_3m` (coach 1 y 3 meses).
- **Offering:** `paywall_b` (variante A/B del paywall).
- **Reconciliación funnel:** claves `relevantFunnelSubscriptionCheck`, `funnelSubscriptionCheck`.

**Regla:** usuario premium ⇔ RevenueCat `customerInfo.entitlements.active['pro']` activo (`getCustomerInfo()` / `Purchases`). El coach se desbloquea con `coach_access`. Las compras del funnel web se reconcilian con el usuario de la app vía las claves `*FunnelSubscriptionCheck` + `/subscriptions/relevant`.

---

## 6. Enums del perfil

- **`Gender`**: `{ female: "female", male: "male" }`
- **`UnitsOfMeasure`**: `{ Metric: "metric", Imperial: "imperial" }`
- **`ActivityLevelGroup`**: `{ low: "low", medium: "medium", high: "high" }`
- **`activityLevel`**: índice 0–5 (slider del quiz), agrupado:

  | activityLevel | grupo | levelValue |
  |---|---|---|
  | 0, 1 | low | 1 |
  | 2, 3 | medium | 2 |
  | 4, 5 | high | 3 |

También hay enums `AgeGroup` y `UnitOfLength`. El perfil calcula el **BMR de Mifflin-St Jeor** (`10·peso + 6.25·altura − 5·edad ± sexo`) para las metas calóricas.

---

## 7. Quiz → metas calóricas → plan

El quiz de onboarding recolecta `weight`, `height`, `age`, `gender`, `activityLevel`, `goalWeight`, `mealsADay` (guardados en `userDetails`, RTDB). De ahí se calculan las metas:

### `getMinMaxCaloriesIntake(profile)`
1. **BMR (Mifflin-St Jeor):** `10·peso + 6.25·altura − 5·edad`
2. **Ajuste sexo:** `female → −161`, `male → +161`
3. **TDEE = BMR × mult. actividad:** `activityLevel 1 → ×1.2`, `2 → ×1.3`, `3 → ×1.4`
4. **Banda con déficit:**
   - `calories_to = TDEE − 500`
   - `calories_from = TDEE − 575`  (banda de 75 kcal, déficit ~500)
5. **Clamp:** piso mínimo (≈1500/1650 para female) para no bajar de un mínimo saludable.

### Request de generación de plan de comidas
```json
{
  "type": "...",
  "days": "<PLAN_AMOUNT_OF_DAYS>",
  "height": 0, "weight": 0, "target_weight": "<goalWeight>",
  "calories_from": 0, "calories_to": 0,
  "metric_system": "metric",
  "age": 0, "gender": "...",
  "repasts": "<mealsADay || 5>",
  "activity_level": "...",
  "force_plan_regeneration": true
}
```

> Notas: el ajuste de sexo usa ±161 (el Mifflin estándar es male +5 / female −161; esta app usa ±161). El piso para `male` no se trazó por completo.

---

## 8. Contenido de los planes (recetas/comidas)

### Content API (flmng, por locale)
```
GET {baseUrl}/v1/content/mains/schemas/1/locales/<locale>        # comidas/recetas
GET {baseUrl}/v1/content/challenges/schemas/1/locales/<locale>   # challenges
```

### Modelo de receta (campos confirmados)
```js
recipe = {
  name,
  calories, protein, fat, carbs,     // macros
  ingredients: [ { /* nombre */, amount } ],
  howToMake,                         // instrucciones
  fileName: name + ".pdf"            // export a PDF (libpdfium)
  // recipeLabels: etiquetas localizadas (UI/PDF), no valores
}
```

### Organización
- Cache (React Query): `recipes`, `single-recipe`, `recipe-alternatives`, `recipe-favorites`.
- Comidas por tipo: `breakfast`, `lunch`, `dinner`, `snack`; plan estructurado por `day`.
- Exportación de recetas a **PDF**.

### Esquema anidado (plan → día → recetas)
```
Plan
 └─ plan_day: N                         # día numerado del plan
     └─ getClientRecipes({ plan_day, languageCode }) → recipes[]
         recipe {
           client_recipe_id,
           title, imageUrl,
           meal,                        # tipo (breakfast/lunch/dinner/snack)
           order,                       # orden dentro del día
           isFavorite,
           calories, protein, fat, carbs,
           ingredients: [ { amount, ... } ],
           howToMake
         }
         + alternativas: getRecipeAlternatives / getSingleAlternativeRecipe
```
**Content client:** `getClientRecipes({plan_day, languageCode})`, `getSingleRecipe({client_recipe_id})`, `getRecipeAlternatives(...)`, `getSingleAlternativeRecipe(...)`. `HEAD /v1/content/mains/schemas/1/locales/<locale>` = chequeo de disponibilidad por idioma.

> El **path/base HTTP exacto** de `getClientRecipes` no se fijó (es un content client por parámetros, no ruta literal); el anidamiento plan_day→recipes→alternativas y los campos de receta sí están confirmados.

---

## 4. Pendientes / preguntas abiertas

1. **Métodos no confirmados** (marcados "i"): verificar GET/PUT/POST exacto por endpoint en el IR.
2. ~~Payloads de `/userDetails`, `/fasting/plan`, `/subscriptions/relevant`, `/v1/events`~~ — **resuelto** (§2.1 RTDB; §5 funnel API + RevenueCat). `/v1/events` = RevenueCat, no flmng.
3. ~~Enums `Gender`, `UnitsOfMeasure`, `activityLevel`~~ — **resuelto** (§6).
4. **Headers completos** por request (además de `Authorization`): reCAPTCHA token, versión de app, locale.
5. ~~Lógica de paywall/premium~~ — **resuelto** (§5: entitlement `pro` de RevenueCat + funnel API `/subscriptions/relevant`).

---

*Reconstruido por análisis estático del bundle Hermes. No se incluye el binario ni el bundle decompilado en este repo (material propietario de Tall Five / Ketoway).*
