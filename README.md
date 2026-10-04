# Sara · WhatsApp CRM con agente IA

CRM de WhatsApp con agente IA (Sara), panel de conversaciones, calificación automática de leads y follow-up automático.

**Stack:** Next.js 16 (App Router) · Supabase (DB + Auth magic link) · WhatsApp Business API (Meta) · OpenRouter (DeepSeek) · Resend · Netlify (hosting + scheduled/background functions).

## Puesta en marcha

1. **Supabase**: creá el proyecto y ejecutá completo [`supabase/schema.sql`](supabase/schema.sql) en el SQL Editor.
   - Authentication → Providers → Email (magic link) habilitado.
   - Authentication → URL Configuration → Site URL `https://tu-sitio.netlify.app` y Redirect URL `https://tu-sitio.netlify.app/api/auth/callback` (sumá también `http://localhost:3000/api/auth/callback` para desarrollo).
2. **Variables de entorno**: copiá `.env.example` a `.env.local` y completalas.
3. **Local**:
   ```bash
   npm install
   npm run dev
   ```
4. **Deploy en Netlify**:
   - Subí el repo a GitHub/GitLab → Netlify → *Add new project* → *Import an existing project*. Netlify detecta Next.js y toma la config de [`netlify.toml`](netlify.toml) (build `npm run build`, Node 22).
   - *Project configuration → Environment variables*: cargá todas las variables (scope **Builds + Functions**). `NEXT_PUBLIC_APP_URL` = la URL final del sitio.
   - Alternativa por CLI: `npx netlify-cli deploy --build --prod`.
5. **Meta Developers** → WhatsApp → Configuración → Webhook:
   - URL: `https://tu-sitio.netlify.app/api/webhook`
   - Token de verificación: el mismo valor que `WHATSAPP_VERIFY_TOKEN`
   - Suscribirse a `messages` (los estados de entrega/lectura llegan dentro de este campo como `statuses`).
   - Recomendado: cargá el **App Secret** (Configuración de la app → Básica) en `WHATSAPP_APP_SECRET` para validar la firma de cada webhook.
6. Mandá un WhatsApp al número y verificá que Sara responde. En `/settings` → "Integraciones" ves qué variables faltan.

## Variables de entorno

| Variable | Uso |
| --- | --- |
| `WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_ACCESS_TOKEN`, `WHATSAPP_VERIFY_TOKEN` | WhatsApp Cloud API |
| `WHATSAPP_APP_SECRET` | (opcional) valida `X-Hub-Signature-256` |
| `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` | acceso servidor (ignora RLS) |
| `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` | auth (magic link) |
| `OPENROUTER_API_KEY`, `OPENROUTER_MODEL` | modelo IA (default `deepseek/deepseek-chat-v3-0324`) |
| `RESEND_API_KEY`, `RESEND_FROM`, `NOTIFY_EMAIL` | alertas y digest (varios emails separados por coma) |
| `CRON_SECRET` | protege `followup-worker` y los disparos manuales de `/api/cron/*` (`Authorization: Bearer …`) |
| `ALLOWED_EMAIL_DOMAIN` | solo `@ese-dominio` puede entrar al panel |
| `NEXT_PUBLIC_APP_URL` | URL pública (links en emails y redirect del magic link) |

## Cómo funciona

**Mensaje entrante** (`POST /api/webhook`): valida firma → responde 200 al instante y procesa en background con `after()` → deduplica por `wamid` → crea/busca contacto (guarda origen del anuncio Click-to-WhatsApp y `ctwa_clid`) → guarda el mensaje → si no está bloqueado y el bot está ON: marca como leído + "escribiendo…", espera 2,5 s para agrupar ráfagas, toma los últimos 20 mensajes, genera la respuesta con DeepSeek, la envía y la guarda con status `sent` → califica el lead (3+ mensajes del usuario) y si es HOT manda email inmediato.

**Estados** (`statuses[]`): actualiza `sent → delivered → read` sin retroceder (o `failed`).

**Derivación a humano**: si Sara detecta molestia agrega un marcador interno; el sistema lo quita del mensaje, **apaga el bot** para ese contacto y envía un email.

**Crons (Netlify Functions en [`netlify/functions`](netlify/functions))**
- `cron-notify` — scheduled, 21:00 UTC: digest con los leads calificados en las últimas 24 h.
- `cron-followup` — scheduled, 13/17/21 UTC. Como las scheduled functions cortan a los 30 s, solo dispara la background function `followup-worker` (hasta 15 min), que busca conversaciones cuyo último mensaje es de Sara hace 5+ h, dentro de las 23 h del último mensaje del usuario, solo entre 10 y 20 h de Argentina, y manda un único follow-up por cada silencio del usuario.
- La lógica vive en [`src/lib/jobs.ts`](src/lib/jobs.ts). Para probar a mano: botón **Run now** en Netlify → Functions, o `GET /api/cron/followup` / `/api/cron/notify` con `Authorization: Bearer $CRON_SECRET`.
- Las scheduled functions solo corren en el deploy de producción (no en deploy previews).

**Límites de Netlify a tener en cuenta**: funciones síncronas 60 s (webhook, test-chat), scheduled 30 s, background 15 min. El webhook responde al instante y termina el trabajo con `after()`, que normalmente tarda 10–15 s.

**Panel**
- `/dashboard`: KPIs, actividad de mensajes vs período anterior, temas, leads recientes, calificación, países (por prefijo), fuentes de anuncios.
- `/conversations`: lista con buscador y filtros (hot/warm/cold/bot off/bloqueados), se actualiza sola cada 5 s. Por conversación: historial, ticks de entrega, Bot ON/OFF, bloquear, recalificar, envío manual.
- `/settings`: editor del system prompt + link de Calendly. Las reglas irrompibles se agregan siempre y no se pueden editar.
- `/test-chat`: chat de prueba con el mismo modelo y reglas, con calificación en vivo y opción de probar un prompt borrador.

## Decisiones sobre el schema

El schema base se respeta tal cual; se agregó:
- `leads.topic`: tema detectado al calificar (métrica de "temas").
- Vista `conversation_overview` y función `dashboard_daily_messages` para no traer miles de filas al panel.
- Configuración por contacto en `settings`: `bot_disabled:<contact_id>` y `followup_at:<contact_id>`.
- RLS activado **sin políticas**: todo pasa por el servidor con la service role después de validar sesión y dominio.

## Estructura

```
src/
  proxy.ts                    # auth + restricción por dominio (Next 16: ex-middleware)
  app/
    api/webhook               # verificación + mensajes + estados
    api/cron/{notify,followup}
    api/auth/{callback,signout}
    api/test-chat
    login/                    # magic link
    (panel)/{dashboard,conversations,settings,test-chat}
  lib/
    agent.ts                  # prompt, reglas irrompibles, respuesta y follow-up
    qualify.ts                # calificación hot/warm/cold
    conversation.ts           # flujo de mensajes entrantes / envío
    jobs.ts                   # digest diario + follow-ups (usado por Netlify y /api/cron)
    whatsapp.ts  ai.ts  notify.ts  settings.ts  dashboard.ts
netlify/functions/
  cron-notify.mts             # scheduled 21 UTC
  cron-followup.mts           # scheduled 13/17/21 UTC → followup-worker
  followup-worker.mts         # background (15 min)
netlify.toml
supabase/schema.sql
```
