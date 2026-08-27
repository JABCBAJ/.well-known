# Plan: acceso a Home Connect (horno Bosch) y uso en Node-RED para gestión de cargas

Objetivo: detectar en tiempo real cuándo el horno Bosch entra en funcionamiento y
publicar ese estado en Node-RED para modular el resto de cargas de la casa
(carga del coche, termo, bomba de calor…) sin superar la potencia contratada.

---

## 0. Lo primero: qué te da la API y qué NO

Esto condiciona todo el diseño, así que conviene tenerlo claro antes de empezar.

**Lo que SÍ da la API de Home Connect:**

| Señal | Qué significa |
|---|---|
| `CONNECTED` / `DISCONNECTED` | El electrodoméstico aparece/desaparece de la nube. Es presencia de red, **no** que esté cocinando. |
| `BSH.Common.Setting.PowerState` | `Off`, `On`, `Standby`. Esta es la señal de "el usuario ha encendido el horno". |
| `BSH.Common.Status.OperationState` | `Inactive`, `Ready`, `DelayedStart`, `Run`, `Pause`, `ActionRequired`, `Finished`. |
| `BSH.Common.Root.ActiveProgram` | Programa en marcha (incl. pirólisis, que es el peor caso de consumo). |
| `BSH.Common.Option.ElapsedProgramTime` / `RemainingProgramTime` / `ProgramProgress` | Progreso, útil para estimar cuánto durará la carga. |
| `Cooking.Oven.Status.CurrentCavityTemperature` | Temperatura de cavidad. Sirve para inferir si está en rampa de calentamiento o manteniendo. |
| `Cooking.Oven.Event.PreheatFinished` | Fin del precalentamiento → a partir de aquí el consumo medio cae bastante. |

**Lo que NO da la API:** vatios. Home Connect no expone consumo instantáneo del
horno. Cualquier cifra de potencia hay que **estimarla** a partir del estado, o
medirla con un contador real.

**Consecuencia de diseño:** el plan usa dos fuentes complementarias.

1. **Home Connect (nube)** → señal *semántica* y *anticipatoria*: "el usuario ha
   encendido el horno", "ha programado un inicio diferido a las 20:30",
   "queda 1 h de programa". Latencia típica de 1–3 s, pero depende de internet.
2. **Un medidor real** (pinza/Shelly EM en el circuito del horno, o el medidor
   general que ya uses) → verdad de campo en vatios, latencia <1 s y local.

Si el objetivo es **evitar que salte el ICP**, la protección dura tiene que
apoyarse en el medidor local; Home Connect aporta la anticipación y el contexto
(saber que el horno va a arrancar y cuánto va a durar). Si el objetivo es
optimizar/priorizar cargas sin riesgo de corte, con Home Connect solo ya vale.

---

## Fase 1 — Alta y registro de la aplicación

1. Crear cuenta en el **Home Connect Developer Portal**
   (<https://developer.home-connect.com>) usando **la misma cuenta de correo
   con la que está registrado el horno en la app Home Connect**. Si no coincide,
   la app no verá el electrodoméstico.
2. Comprobar en la app móvil que el horno está emparejado y online.
3. En el portal → *Applications* → **Register Application**:
   - **Application ID / Client ID**: lo genera el portal.
   - **OAuth Flow**: elegir **Device Flow** (ver Fase 2).
   - **Home Connect User Account for Testing**: tu propio correo. Esto crea la
     app en modo "un solo usuario", que es justo lo que necesitas; no hace falta
     pedir revisión de partner.
   - **Scopes**: pedir el mínimo → `IdentifyAppliance Monitor Settings`.
     **No pidas `Control` ni `Oven-Control`** si no vas a mandar órdenes al
     horno: evitas el riesgo de arrancarlo remotamente por un fallo de flujo.
4. Guardar `client_id` y, si el portal lo genera, `client_secret`.

---

## Fase 2 — Autorización OAuth2

Dos caminos posibles. **Recomendado: Device Flow.**

### Opción A (recomendada): Device Flow

No necesita que Node-RED sea accesible desde internet ni URL de redirección.
Se autoriza una vez, a mano, y luego se vive de refresh tokens.

```bash
# 1) Pedir código de dispositivo
curl -s -X POST https://api.home-connect.com/security/oauth/device_authorization \
  -d "client_id=TU_CLIENT_ID" \
  -d "scope=IdentifyAppliance Monitor Settings"
# → { "device_code": "...", "user_code": "ABCD-1234",
#     "verification_uri_complete": "https://api.home-connect.com/security/oauth/device_verify?...",
#     "interval": 5, "expires_in": 600 }

# 2) Abrir verification_uri_complete en el navegador y aceptar con tu cuenta Home Connect

# 3) Canjear el device_code por tokens (repetir cada `interval` s hasta que deje
#    de responder authorization_pending)
curl -s -X POST https://api.home-connect.com/security/oauth/token \
  -d "grant_type=device_code" \
  -d "device_code=EL_DEVICE_CODE" \
  -d "client_id=TU_CLIENT_ID"
```

Notas:
- Si `grant_type=device_code` devuelve `unsupported_grant_type`, usar el valor
  estándar `urn:ietf:params:oauth:grant-type:device_code`.
- Errores esperables mientras esperas la confirmación en el navegador:
  `authorization_pending` (sigue esperando), `slow_down` (aumenta el intervalo),
  `access_denied`, `expired_token` (vuelve al paso 1).

### Opción B: Authorization Code

Requiere una `redirect_uri` registrada. Ventaja: ya tienes un sitio estático
publicado en GitHub Pages (este mismo repo, `jabcbaj.github.io/.well-known`),
así que podrías registrar ahí una página de callback que se limite a mostrar el
`code` para pegarlo a mano. Es más pasos y no aporta nada frente a Device Flow
en un despliegue de un solo usuario. Úsalo solo si el portal no te ofrece
Device Flow para tu tipo de app.

### Gestión de tokens (crítico)

- `access_token`: vida ~24 h (`expires_in: 86400`).
- `refresh_token`: vida larga (semanas/meses), **rota en cada refresco** — hay
  que guardar siempre el último.
- **Persistencia obligatoria**: si guardas los tokens solo en memoria, cada
  reinicio de Node-RED te obliga a re-autorizar a mano. Guardar en context
  store de tipo `file` (o en un fichero JSON con permisos restringidos).

```js
// settings.js de Node-RED
contextStorage: {
  default: "memoryOnly",
  memoryOnly: { module: "memory" },
  file: { module: "localfilesystem" }   // usar con: global.get("hc_tokens", "file")
}
```

Refresco:

```bash
curl -s -X POST https://api.home-connect.com/security/oauth/token \
  -d "grant_type=refresh_token" \
  -d "refresh_token=EL_REFRESH_TOKEN" \
  -d "client_id=TU_CLIENT_ID"          # + client_secret si tu app lo tiene
```

Refrescar **proactivamente** cada 12 h y, además, siempre que una llamada
devuelva `401`.

---

## Fase 3 — Descubrir el `haId` del horno

```bash
curl -s https://api.home-connect.com/api/homeappliances \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Accept: application/vnd.bsh.sdk.v1+json"
```

Devuelve la lista con `haId`, `type` (`Oven`), `brand`, `connected`. Anota el
`haId` del horno; es fijo, guárdalo como variable de entorno o global de
Node-RED.

Estado actual en cualquier momento (útil al arrancar, para sincronizar):

```bash
curl -s https://api.home-connect.com/api/homeappliances/$HAID/status   -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.bsh.sdk.v1+json"
curl -s https://api.home-connect.com/api/homeappliances/$HAID/settings -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.bsh.sdk.v1+json"
```

---

## Fase 4 — Stream de eventos (SSE)

**No hagas polling.** Hay cuota diaria de peticiones (del orden de 1.000/día por
cliente y cuenta) y un límite de ~10 req/s. Abrir el SSE cuenta como **1**
petición y luego los mensajes del stream no consumen cuota. Además hay un máximo
de **10 canales de eventos simultáneos** por cliente/cuenta: no dejes streams
zombis abiertos.

```bash
curl -N https://api.home-connect.com/api/homeappliances/$HAID/events \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Accept: text/event-stream"
```

- `/api/homeappliances/events` (sin `haId`) → todos los electrodomésticos.
- `/api/homeappliances/{haId}/events` → solo el horno. **Usa este**: menos ruido.

Tipos de evento que llegan:

| `event:` | Contenido |
|---|---|
| `KEEP-ALIVE` | Latido. Si dejas de recibirlo >2 min, reconecta. |
| `CONNECTED` / `DISCONNECTED` | El horno entra/sale de la nube. |
| `STATUS` | Cambios de `OperationState`, `DoorState`, temperatura de cavidad… |
| `NOTIFY` | Cambios de settings (`PowerState`) y de opciones de programa. |
| `EVENT` | Eventos puntuales: `PreheatFinished`, `ProgramFinished`, alarmas. |

Formato del payload (`data:`): JSON con `haId` y un array `items[]`, cada uno con
`key`, `value`, `unit`, `timestamp`.

**Robustez del stream:**
- Reconectar con backoff exponencial (1 s → 2 → 4 … máx. 60 s) ante cierre o
  error.
- Reciclar la conexión cada ~15 min aunque parezca viva (evita streams zombis
  que ya no reciben nada).
- Al reconectar, hacer **una** llamada a `/status` y `/settings` para
  resincronizar, porque los cambios ocurridos mientras estabas caído no se
  reenvían.
- Si el stream devuelve `401`, refrescar token y reabrir.

---

## Fase 5 — Máquina de estados en Node-RED

Traducir el flujo de eventos a un único objeto de estado que el resto de flujos
pueda consultar (`global.set("horno", …)`).

```
                 PowerState=On            OperationState=Run
   APAGADO ────────────────────▶ EN_ESPERA ─────────────────▶ CALENTANDO
      ▲                              │                            │
      │ PowerState=Off/Standby       │ DelayedStart               │ PreheatFinished
      │                              ▼                            ▼
      └────────────── ProgramFinished ◀──── MANTENIENDO ◀─────────┘
```

Estado publicado:

```js
{
  conectado: true,           // CONNECTED / DISCONNECTED
  encendido: true,           // PowerState
  estado: "CALENTANDO",      // APAGADO | EN_ESPERA | DIFERIDO | CALENTANDO | MANTENIENDO | PAUSA | FIN
  programa: "Cooking.Oven.Program.HeatingMode.HotAir",
  temperaturaObjetivo: 200,
  temperaturaCavidad: 143,
  finEstimado: "2026-08-27T20:41:00Z",
  potenciaEstimadaW: 3000,   // ← estimación, no medida
  ts: 1756312345
}
```

**Estimación de potencia** (ajustar con tus propias medidas la primera semana):

| Estado | Estimación típica horno Bosch |
|---|---|
| `APAGADO` / `EN_ESPERA` | 2–5 W (standby) |
| `CALENTANDO` (rampa de precalentamiento) | 2.500–3.600 W continuos |
| `MANTENIENDO` (tras `PreheatFinished`) | 1.000–1.800 W medios, ciclando ON/OFF |
| Pirólisis | ~3.000 W sostenidos durante 1,5–3 h ← el caso peor |
| Grill máximo | según modelo, hasta ~3.000 W |

Marca el resultado explícitamente como estimación (`fuente: "estimado"` vs
`fuente: "medido"`) para que la lógica de cargas sepa de qué se fía.

---

## Fase 6 — Lógica de gestión de cargas

1. **Tabla de prioridades** (de menor a mayor prioridad de corte):
   horno y cocina = intocables → bomba de calor / termo = modulables →
   carga del coche = primera en recortar.
2. **Presupuesto de potencia**: `disponible = P_contratada * 0,9 − P_medida_actual`.
   El 0,9 es margen de seguridad; ajústalo a tu instalación.
3. **Anticipación**: al recibir `PowerState=On` o `DelayedStart`, reservar ya la
   potencia estimada del horno *antes* de que empiece a consumir. Ahí está el
   valor real de Home Connect frente a un simple medidor: te enteras con
   segundos (o con el inicio diferido, con horas) de antelación.
4. **Histéresis y temporización**: no reaccionar a cambios <30 s; al liberar
   potencia (`ProgramFinished`), esperar 60–120 s antes de devolver carga al
   coche, para no encadenar arranques.
5. **Fallback**: si el estado del horno tiene más de N minutos de antigüedad o
   `conectado=false`, asumir el peor caso (horno consumiendo) o pasar el control
   íntegro al medidor local. Nunca dejar que un fallo de la nube se traduzca en
   "el horno no consume".
6. **Registro**: guardar cada transición en InfluxDB/CSV con la potencia medida
   simultánea, para calibrar las estimaciones de la Fase 5.

---

## Fase 7 — Implementación en Node-RED: tres opciones

| Opción | Cómo | Pros | Contras |
|---|---|---|---|
| **A. `node-red-contrib-homeconnect`** | Nodo de config `home-connect-auth` + nodo `request` que emite eventos | Rápido de montar, sin código | Paquete de terceros poco mantenido; conviene revisar que siga funcionando |
| **B. Nodos genéricos** | `function` con `https` en `libs` para el SSE + `http request` para el refresco de token | Control total, sin dependencias, reconexión a tu gusto | Hay que escribir la lógica (está en el esqueleto adjunto) |
| **C. Vía Home Assistant** | Integración oficial `home_connect` en HA + `node-red-contrib-home-assistant-websocket` | HA ya resuelve OAuth, refresco y reconexión | Solo tiene sentido si ya usas HA |

Recomendación: **B** si Node-RED es tu único orquestador (es el que trae el
esqueleto de este repo), **C** si ya tienes Home Assistant montado.

Fichero adjunto: [`flow-esqueleto.json`](./flow-esqueleto.json) — flujo mínimo
importable con refresco de token, SSE con reconexión y máquina de estados. **Sin
probar contra un horno real**: revísalo y ajústalo antes de usarlo en producción.

---

## Fase 8 — Pruebas

1. **Simulador**: el portal de desarrollador ofrece electrodomésticos simulados.
   Permite montar y depurar todo el flujo (auth, SSE, máquina de estados) sin
   tocar el horno de verdad. Ojo: los `haId` del simulador son distintos.
2. **Contra el horno real**, en este orden:
   - Encender/apagar el horno → verificar `PowerState`.
   - Programa corto a 200 °C → verificar `Run`, `PreheatFinished`, curva de
     temperatura y `ProgramFinished`.
   - Abrir la puerta a mitad → verificar `Pause` / `DoorState`.
   - Cortar el WiFi del horno → verificar `DISCONNECTED` y que el fallback de la
     Fase 6 actúa.
   - Reiniciar Node-RED con el horno en marcha → verificar que resincroniza vía
     `/status` y no pierde el estado.
3. **Calibración**: una semana registrando estado + potencia medida, y ajustar la
   tabla de la Fase 5.

---

## Riesgos y decisiones abiertas

- **Dependencia de la nube**: si cae internet o la API de BSH, te quedas sin
  señal. Por eso el medidor local no es opcional si hay riesgo de corte de ICP.
- **Semántica de "conectado"**: el horno mantiene el WiFi en standby, así que
  `CONNECTED` normalmente solo aparece tras un corte de luz o un reinicio del
  aparato. La señal útil para gestión de cargas es `PowerState` +
  `OperationState`, no `CONNECTED`.
- **Cuotas**: cuota diaria de peticiones y máx. 10 canales de eventos. Un bucle
  de reconexión mal hecho te puede dejar bloqueado el resto del día.
- **Secretos**: `client_id`, `client_secret` y los tokens **no** deben acabar en
  este repositorio (es público). Usar variables de entorno o el context store en
  fichero, fuera de git.
- **Scope `Control`**: no pedirlo mientras no haga falta.

## Decisiones que faltan por confirmar

- ¿Hay ya un medidor de potencia general/por circuito en la instalación, o hay
  que añadirlo?
- ¿Node-RED corre en un equipo siempre encendido con almacenamiento persistente?
- ¿Se usa Home Assistant además de Node-RED (elegiría la opción C)?
- ¿Cuál es la potencia contratada y qué cargas concretas se van a modular?
