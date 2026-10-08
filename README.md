# VibeFilm — SaaS de *vibecoding* para cine

Estudio self-serve donde el usuario describe un corto por chat y un agente escribe el
guion, fija a los personajes, genera los planos y los monta. El proyecto no es un
timeline: es un documento JSON versionado (`film.json`), así que un cambio pedido por
chat se aplica como un **diff** y solo se vuelve a renderizar lo que quedó inválido.

**Estado:** diseño de producto y mockup. Sin código de aplicación todavía.

- Mockup navegable (6 pantallas): https://claude.ai/artifact/D5ecGFWCcLn5NAshZuf7bK *(privado; hay que compartirlo para que lo abran otros)*
- Fuentes del mockup: [`mockup/project/`](mockup/project) — artboards `.dc.html` + `canvas.json`

## Decisiones tomadas

| Eje | Decisión |
|---|---|
| Producto | Estudio self-serve (prompt → storyboard → planos → MP4) |
| Cliente inicial | Creadores indie / YouTube |
| Infraestructura | Solo APIs de terceros, sin GPUs propias |
| Diferenciador | *Film-as-code*: `film.json` versionado + re-render selectivo |

## Arquitectura

| Capa | Elección | Motivo |
|---|---|---|
| App | Next.js App Router + TypeScript + Tailwind/shadcn (Vercel) | Streaming UI para el chat agéntico |
| Datos / Auth | Supabase (Postgres + RLS + Auth) + Drizzle | RLS ahorra media capa de autorización |
| Orquestación | Inngest (alt. Trigger.dev) | Renders de 1–6 min: workflows durables, reintentos, concurrencia por usuario |
| Router de modelos | fal.ai | Una key para Kling/Seedance/LTX/Flux/ElevenLabs, webhooks, no factura cold starts ni fallos |
| Agentes | Claude Opus 5 (`claude-opus-5`) para guion, shot list y diffs; Haiku 5.5 para prompts y moderación | Structured outputs → `film.json` validado con Zod |
| Vídeo | Preview: LTX-2 Fast · Final: Kling 3.0 Pro · Hero: Veo 3.1 | Dos tiers de calidad = margen |
| Audio | ElevenLabs (Text-to-Dialogue + SFX) | Música: ver *Riesgos* |
| Ensamblado | Remotion Lambda | Timeline declarativo: rótulos, subtítulos, 9:16 |
| Entrega | Cloudflare R2 + Stream | Egress cero; con vídeo, S3 es prohibitivo |
| Pagos | Stripe (suscripción + top-ups) | |
| Observabilidad | Sentry + PostHog + tabla propia de costes por job | |

### Pipeline

```
prompt ──► [Opus 5: guionista]      → beats, tono, duración
       ──► [Opus 5: script doctor]  → shot list JSON (lente, ángulo, luz, movimiento, diálogo)
       ──► [Flux]                   → character bible: 3-4 stills por personaje
       ──► [Flux]                   → keyframe por plano, usando el character sheet
       ──► [LTX-2 Fast]             → PREVIEW 480p de los 12 planos (~3 $, iterable)
       ──► usuario aprueba o itera por chat
       ──► [Kling 3.0 Pro]          → FINAL 1080p solo de los planos aprobados
       ──► [ElevenLabs]             → diálogo + SFX + música
       ──► [Remotion Lambda]        → MP4 + SRT + 9:16
```

La consistencia de personaje se resuelve fijando primero los stills de referencia e
inyectándolos como *reference images* (Kling Elements / «ingredients» de Veo). Nunca
se pide el personaje por texto en cada plano.

## Economía unitaria

Coste de **1 minuto entregado** (12 planos × 5 s), con precios de fal de octubre de 2026:

| Partida | Coste |
|---|---|
| Keyframes (12 img) | 0,45 $ |
| Vídeo final Kling 3.0 Pro (0,112 $/s × 60) | 6,72 $ |
| Previews LTX-2 Fast (2 pasadas) | 0,60 $ |
| Agentes (Opus 5 + Haiku) | 0,30 $ |
| Voz + 6 SFX + 1 min de música | 1,05 $ |
| Ensamblado | 0,05 $ |
| **Subtotal** | **≈ 9,20 $** |
| × factor de descarte real (~1,7) | **≈ 15 $ / min** |

Consecuencias de diseño, no negociables:

1. **Créditos prepago, nunca «ilimitado».** 1 crédito = 0,01 $ de COGS, pero el crédito
   es opaco para poder cambiar de modelo sin tocar precios.
2. **Ledger append-only con hold / capture / release.** Se reserva al encolar, se cobra
   al terminar y se devuelve lo no usado. Sin esto, los reintentos se comen el margen.
3. **Previews casi gratis, final caro.** Convierte la iteración (el enganche) en coste
   marginal ridículo.
4. **Re-render selectivo** como ventaja de coste, no solo de UX.

### Planes

| Plan | Precio | Créditos | Vídeo final aprox. |
|---|---|---|---|
| Free | 0 $ | 400 una vez | ~25 s · marca de agua · sin uso comercial |
| Creator | 29 $/mes | 1.000 | ~45 s/mes · derechos comerciales |
| Pro | 99 $/mes | 3.300 | ~2,3 min/mes · sin marca de agua · 3 colaboradores |
| Top-up | 39 $ | 1.000 | sin caducidad |

## Riesgos a resolver desde el día 1

- **Música:** la licencia estándar de ElevenLabs Music exige licencia adicional para
  cine, TV, publicidad y distribución. El público objetivo monetiza en YouTube → hay que
  ofrecer sustitución por pista de librería licenciada.
- **Derechos comerciales por modelo:** Kling (Kuaishou) y ByteDance tienen ToS distintos
  a Google. La promesa de uso comercial solo puede ser tan amplia como el modelo más
  restrictivo del tier.
- **Moderación obligatoria:** filtrar prompt *e* imagen de referencia; bloquear personas
  reales identificables y menores. Un incidente cierra la cuenta del proveedor.
- **Procedencia:** metadatos C2PA + marca de agua en los tiers bajos.
- **Dependencia de proveedor:** la API de Sora se cerró el 24/09/2026. Todo detrás de una
  interfaz `VideoProvider` con dos implementaciones desde el principio.

## Roadmap a beta

| Semana | Entrega |
|---|---|
| 1 | Schema + RLS, Stripe con créditos y ledger, `VideoProvider` sobre fal, pipeline fijo end-to-end por CLI |
| 2 | Agentes con structured output: `film.json`, shot list, character bible. Workflows en Inngest |
| 3 | UI: chat + grid de planos + preview, diffs y re-render selectivo |
| 4 | Ensamblado Remotion, export MP4/SRT, moderación, watermark, límites de concurrencia, landing |
| 5 | Beta cerrada de 30 creadores midiendo COGS por minuto entregado y % de planos descartados |

## El mockup

Seis pantallas en `mockup/project/`, navegables entre sí:

| Archivo | Pantalla |
|---|---|
| `Main.dc.html` | Estudio: chat agéntico y tarjeta de diff sobre `film.json` |
| `Storyboard.dc.html` | Shot list por beats con estado, modelo y coste por plano |
| `Personajes.dc.html` | Character bible y consistencia medida |
| `Timeline.dc.html` | Pistas de montaje y opciones de export |
| `Creditos.dc.html` | KPI del mes y libro de créditos (reserva → cobro → devolución) |
| `Landing.dc.html` | Landing y precios |

Son artboards del tipo *Design* de Claude Artifacts (`.dc.html`): HTML autocontenido con
un bloque de lógica. El reparto, el corto («Lluvia en Malasaña») y las cifras de uso son
ficticios; los precios de los modelos, no.
