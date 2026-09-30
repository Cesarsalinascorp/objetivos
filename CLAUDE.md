# Panel Digital SAVAL 2026 — Contexto del proyecto

> Última actualización de este archivo: 30 sep 2026. Escrito para dar contexto completo
> a una sesión de Claude Code nueva (César trabaja este proyecto desde dos entornos
> distintos: uno via GitHub token por turno, y otro Ubuntu local conectado a Claude).
> Si algo acá no calza con el código real, el código manda — este archivo puede quedar
> desactualizado si se siguen haciendo ajustes sin actualizarlo.

## Qué es esto

Panel de seguimiento de objetivos del equipo digital de SAVAL (César, Dani, Tati, Juan).
Es un **solo archivo HTML** (`index.html`, ~1885 líneas, vanilla JS, sin build ni
framework) desplegado como sitio estático en GitHub Pages:
https://cesarsalinascorp.github.io/objetivos/

Repo: `Cesarsalinascorp/objetivos` (rama `main`, deploy automático vía GitHub Pages al
hacer push a `main`, se refleja en 1-5 min).

César es Publicista / Especialista Performance Digital en Laboratorios SAVAL (desde
abril 2025). Este panel es de uso interno del equipo (César, Dani, Tati, Juan) — no
tiene relación con Attempo (otro proyecto propio de César, SaaS de agendamiento; no
mezclar contexto).

## Backend: Supabase

- Auth y datos vía Supabase (`SUPABASE_URL`, `SUPABASE_KEY` en el propio `index.html`).
- Tabla principal: `panel_datos` (clave/valor) — ahí se guarda todo el estado editable
  (marcas de completado, notas, valores de campos) vía REST API de Supabase.
- El login "Solo para editar el panel" es contra Supabase Auth.
- **Plan gratuito de Supabase se pausa automáticamente tras ~7 días sin actividad.** Si
  el panel tira "Email o contraseña incorrectos" sin razón aparente, lo primero es
  revisar si el proyecto está pausado en el dashboard de Supabase (toma unos minutos en
  reactivarse tras despausar).
- **Guardado con feedback real**: `cloudSet()` retorna `true`/`false`. Si el guardado
  falla (ej. sesión admin expirada), se muestra un toast rojo de advertencia
  (`showAuthWarning()`) en vez de un falso "Guardado ✓". `flushSyncQueue()` también
  surfacea errores reales de Supabase vía `showSyncError(status, detalle)`.
- Si algo "no se guarda" y no es un problema de sesión, sospechar primero de **caché del
  navegador / GitHub Pages** antes que de un bug de lógica — ya pasó varias veces. El
  `<head>` tiene meta tags no-cache, pero igual puede requerir hard-refresh
  (DevTools → "Vaciar caché y volver a cargar de manera forzada") o modo incógnito para
  confirmar que un cambio recién pusheado ya está visible.

## Estructura de datos (`const objData`, línea ~683 de `index.html`)

```
objData = {
  cesar: { nombre:'César', objetivos:[ {...}, ... ] },
  dani:  { nombre:'Dani',  objetivos:[ ... ] },
  tati:  { nombre:'Tati',  objetivos:[ ... ] },
  juan:  { nombre:'Juan',  objetivos:[ ... ] },
}
```

Cada **objetivo**: `{id, num, titulo, open, visInicio?, visFin?, sinPonderacion?, subs:[...]}`
Cada **sub** (subtarea/sub-objetivo): `{id, titulo, peso, visInicio?, visFin?, visFreq?,
une_detalle?, une_lista?, une_lista_mes?, campos:[...]}`
Cada **campo**: `{key, lbl, meta, meta_lbl, tipo, inicio, hint?, visInicio?, visFin?,
invertido?, unico?, formato?, pondera?, opciones?, target?}`

### Tipos de campo (`tipo`)

- `entregable`: checkbox de completado + fecha + link. Opcional `unico:true` → entrega
  NO recurrente (ej. Looker, Digital News Ecuador/CR de julio): una vez marcada
  completada en cualquier mes dentro de su ventana, cuenta como 100% para siempre (no se
  re-evalúa mes a mes).
- `puntual`: valor que se reingresa cada mes y se compara contra `meta` fija (o contra
  `metaEsperada()` si aplica). Opcional `invertido:true` → menor es mejor (ej. CPC:
  sobrecumplido si `v <= meta`). Opcional `formato:'mmss'` → se ingresa como texto
  "m:ss" (ej. "1:32") y se guarda internamente en segundos totales vía
  `setOVFromMMSS()`.
- `acum`: acumulativo hacia una meta de fin de año; `metaEsperada()` prorratea
  linealmente entre `inicio` y diciembre (o `visFin` del campo si aplica).
- `select`: dropdown con `opciones` (ej. reputación IP: Pésima/Mala/Normal/Buena). Por
  defecto es **solo informativo** (no pondera en el promedio). Con `pondera:true` sí
  pondera: el valor 1-4 se traduce a 25/50/75/100% de cumplimiento (usado en el
  objetivo de Juan de reputación IP, a pedido explícito de César con esa fórmula
  exacta).
- `alin_pct`: campo binario 0/1 ("¿enviaste al menos 1 mail alineado a esta UNE este
  mes?"), con `target` (90 o 80 normalmente) representando el % de alineación esperado
  del GRUPO de UNEs, no del campo individual.

### Sistema de visibilidad por fecha/frecuencia

A nivel de **objetivo**: `visInicio`/`visFin` ocultan el objetivo completo fuera de su
ventana (ej. objetivos de Juan que recién arrancan en septiembre).

A nivel de **sub**: `visInicio`/`visFin`/`visFreq` — función `subVisibleEnMes(sub,m)`.
`visFreq` = cada cuántos meses se muestra (default 1 = todos). Ej. patrón semestral→
bimensual ya no se usa activamente (Digital News Ecuador/CR quedó como entrega ÚNICA de
julio, `visInicio:7, visFin:7`), pero el mecanismo de `visFreq` sigue soportado por si
se vuelve a necesitar.

A nivel de **campo individual** (más reciente, independiente del sub):
`campoVisibleEnMes(c,m)` — permite que un campo puntual dentro de un sub visible
aparezca solo en cierto rango (ej. "Sesiones Meta Ads" de Dani, canal nuevo desde junio:
`visInicio:6, visFin:12` a nivel de CAMPO, mientras el resto de los campos del mismo sub
son siempre visibles).

`sinPonderacion:true` (a nivel de objetivo) → el objetivo se muestra pero queda
EXCLUIDO de los promedios ponderados. Se usa en el objetivo 5 de César (Arquitectura):
es opcional, aún no iniciado formalmente, pero César pidió mantenerlo visible sin que
afecte su cumplimiento global. (Ojo: existe también un flag viejo `sub.activo===false`
en `subVisibleEnMes` que oculta una sub por completo — es un mecanismo distinto,
heredado de un intento anterior; `sinPonderacion` es el patrón preferido ahora porque
mantiene visibilidad.)

## Semáforo y cálculo de avance — funciones clave

- `metaEsperada(campo, m)` — cuánto "debería" llevar un campo a un mes dado. `null` si
  aún no aplica (antes de `inicio`). Para `acum` prorratea linealmente
  `meta * (mesesTranscurridos/mesesDisponibles)`; para `puntual` devuelve la meta fija.
- `tiempoAnual(m)` — % de AÑO calendario completo transcurrido (`m/12*100`). Ya casi no
  se usa directo en el render; quedó como base/fallback.
- **`tiempoParaObj(obj, m)`** (fix más importante del proyecto) — % transcurrido de la
  VENTANA PROPIA del objetivo (`visInicio`→`visFin`), no del año calendario completo.
  Un objetivo que arranca en junio (ventana jun-dic = 7 meses) en julio lleva 2/7≈29%,
  NO 7/12≈58%. Antes de este fix, el "ritmo esperado" mostrado para objetivos que
  arrancan a mitad de año estaba sistemáticamente inflado, haciendo ver como
  "rezagados" objetivos que en realidad iban bien. Se usa en 3 lugares del render: el
  header de la tarjeta de objetivo, `renderPerfilCard()` (tarjetas por objetivo + el
  promedio global "cumplimiento global"), y `renderAnualPanel()` (filas de
  "Cumplimiento anual").
- `semCampo(c, v, m, subId, p)` — color de un campo individual. Firma con `subId` y `p`
  explícitos (antes usaba un `c._subid` roto que dejaba TODOS los entregables en gris
  permanentemente — ya corregido).
- `semSub(sub, p, m)` — color de una subtarea. Se deriva **directamente de
  `avancePct()`** (el mismo % que ve el usuario en pantalla): ≥100 sobre, ≥90 verde, ≥70
  amarillo, si no rojo. (Antes tomaba el peor campo individual, lo que producía
  contradicciones visuales tipo "108% de avance pero semáforo amarillo" — corregido
  explícitamente a pedido de César, aplicado "en todo el panel".)
- `semObj(obj, p, m)` — color agregado del objetivo completo, derivado de sus subs.
- `avancePct(sub, p, m)` — % de avance ponderado del mes/rango actual (respeta
  `campoVisibleEnMes`, `unico`, `pondera` en selects, `invertido`).
- `avanceAnualPct(sub, p, finMes)` — % de avance ANUAL acumulado de una sub, con
  `finMes` (default 12, normalmente `obj.visFin`) para no exigir cumplimiento más allá
  de la ventana real del objetivo. `entregable`/`puntual`/`alin_pct` se miden como
  "meses cumplidos / meses totales del período" (unificado — antes `acum` usaba lógica
  de "mejor valor histórico" y el resto usaba "¿alguna vez se cumplió?", lo que producía
  números muy dispares para conceptos similares).
- `avanceAnualObj(obj, p)` — promedio ponderado de avance anual del objetivo completo;
  salta subs cuyo `visInicio` aún no llega.
- `alertMsg(sub, p, m, mes)` — texto de alerta con detalle concreto (qué campos
  cumplidos/pendientes, brecha numérica). Maneja `invertido` correctamente (CPC bajo la
  meta = cumple), incluye entregables en el conteo "N de M campos en meta" (antes los
  excluía → mostraba "0 de 1" con el campo ya cumplido), y compara contra
  `metaEsperada()` prorrateada en vez de la meta anual fija (antes mostraba brechas
  falsas en campos `acum` que iban bien pero aún no llegaban a la meta final de
  diciembre).

## UNE (Unidad de Negocio Estratégica) — listas reutilizables

Constantes cerca del inicio de `objData`: `UNES_5`, `UNES_7`, `UNES_8_MARZO`,
`UNES_8_INF_RESP`.

- Objetivo 3 de César (Mailing Médicos, `visInicio:2, visFin:6`, este objetivo se
  traspasa completo a Juan desde julio): `une_lista:UNES_5` como base (OFTA, OTC, INF,
  GUN, NEU), con override **solo para marzo** vía `une_lista_mes:{3:UNES_8_MARZO}` (se
  agregan CAR, GAS y Marca SAVAL únicamente ese mes), y desde abril `UNES_7` fijo (se
  suman CAR y GAS permanentemente, sin Marca SAVAL).
  ⚠️ Ojo con este patrón: la primera vez fijé por error la lista de 8 UNEs como
  PERMANENTE; César corrigió explícitamente que el cambio era SOLO para marzo. Si se
  vuelve a tocar este bloque, confirmar bien el alcance temporal antes de aplicar.
- Objetivo 5 de Juan (ex-objetivo 6, Mailing Médicos traspasado de César + journeys):
  `une_lista:UNES_7` desde julio, con `une_lista_mes` para agosto-diciembre usando
  `UNES_8_INF_RESP` — desde agosto se agrega RESP como celda separada de INF, pero
  ambas **comparten el mismo cupo** de cumplimiento (`group:'inf_resp'` en las celdas
  correspondientes): si hay envío en INF o en RESP (o ambos), ese cupo cuenta como
  cumplido — no se exige tener ambos.
- `envios_une` (conteo de UNEs con envío ese mes) se **calcula automáticamente** desde
  el detalle de grilla por UNE (`unesConEnvio`), el input correspondiente es readonly —
  antes requería doble digitación manual, lo cual generaba inconsistencias.

## Estado actual de los objetivos por persona (resumen, no exhaustivo — ver `index.html` para el detalle campo por campo)

**César** (5 objetivos):
1. Digital News Chile (mensual) + Ecuador/CR (entrega única julio 2026, `unico:true`).
2. Campaña BS 2026 (`visInicio:6`): Looker (`unico:true`), sesiones/usuarios/leads
   acumulados vs metas 2025→2026 reales, duración de sesión en formato `mmss`.
3. Mailing Médicos (`visInicio:2, visFin:6` — traspasa a Juan desde julio): envíos por
   UNE + reputación IP + alineación 360° (UNEs principales target 90%, secundarias
   target 80%).
4. Medición + recomendaciones growth al equipo (`visInicio:6`).
5. Arquitectura (`sinPonderacion:true` — visible, opcional, sin afectar el promedio).

**Dani** (5 objetivos): tráfico orgánico BS +7%, AWON (CPC `invertido:true`, Sesiones
Meta Ads como canal nuevo desde junio vía `visInicio` a nivel de CAMPO), visibilidad
farmacias (`visInicio:6`), SFMC Mailing Pacientes (IP + OR/CTOR + alineación 360°),
Campaña BS PMO (`visInicio:6`).

**Tati** (5 objetivos): todos con campos tipo `acum` de % de avance de estrategia
(BBDD médicos Ecuador/CR, Digital News Ecuador/CR con insights, campaña BS, Arquitectura
desde el lado comercial, Salesforce + consentimiento médico legal, clima laboral de la
subgerencia).

**Juan** (5 objetivos, reestructurado varias veces a pedido de César con tablas reales
pegadas en el chat):
1. Cobertura HCP (`visInicio:9`).
2. Centralización BBDD médicos (`visInicio:9`).
3. Consentimiento médico, dividido en 3 fases con ventanas propias: base de apoyo de
   prueba (julio, `visInicio:7,visFin:7`), inicio de pruebas (agosto,
   `visInicio:8`), centro de preferencias (desde septiembre, `visInicio:9`).
4. Puentes (`visInicio:8`).
5. SFMC Mailing Médicos (ex-objetivo 6, el viejo objetivo 5 se ELIMINÓ y su sub de
   journeys/OR/CTOR se fusionó acá): envíos por UNE + reputación IP `pondera:true` +
   alineación 360° + journeys complejos por UNE (`visInicio:8, visFin:11` — noviembre,
   no diciembre).

## Deploy y validación

No hay CI/CD más allá de GitHub Pages sirviendo directo desde `main`. No hay tests
automatizados — validar sintaxis JS manualmente antes de cada push:

```bash
python3 -c "
import re
html = open('index.html', encoding='utf-8').read()
scripts = re.findall(r'<script[^>]*>(.*?)</script>', html, re.S)
open('/tmp/combined.js','w',encoding='utf-8').write('\n'.join(scripts))
"
node --check /tmp/combined.js
```

### Push / autenticación

César normalmente pega un **fine-grained GitHub PAT** (`ghp_...`, scope
Contents:Read/write solo sobre este repo) al inicio de cada sesión/turno cuando hace
falta pushear. El patrón correcto es usarlo **inline y una sola vez** en el propio
comando de push, sin guardarlo ni reusarlo entre turnos distintos (si expira o cambia,
pedir uno nuevo):

```bash
git push https://<TOKEN>@github.com/Cesarsalinascorp/objetivos.git main
```

Si el repo local no tiene identidad de git configurada, hay que setearla una vez (ya
pasó en una sesión anterior — el commit fallaba por esto):

```bash
git config user.email "cesar@local"
git config user.name "Cesar"
```

## Trabajo pendiente / en duda

- No hay tareas técnicas pendientes explícitas conocidas al momento de escribir esto.
- Última duda no 100% cerrada: César preguntó por qué el objetivo 1 (Digital News)
  mostraba 87% de avance en julio cuando "cumplió con todos los digital news en julio"
  y esperaba ver 58% (el "ritmo esperado" de `tiempoParaObj`). Se le explicó (con
  analogía de cuenta de ahorros) que 87% real ≥ 58% esperado = sobrecumplimiento, y que
  mostrar 58% en vez del real ocultaría el sobrecumplimiento genuino (Ecuador/CR al
  100% en su única entrega de julio). Quedó sin objeción visible, pero vale confirmar
  si vuelve a salir el tema en una sesión nueva — puede que la expectativa real sea otra
  visualización (ej. mostrar "real" Y "esperado" lado a lado de forma más explícita en
  vez de solo un número).
- Otro entregable de esta conversación, sin relación directa con `index.html`: un Excel
  (`oportunidades_mejora.xlsx`) construido como tracker de oportunidades de mejora
  growth-marketing detectadas al revisar los objetivos de todo el equipo. Fue entregado
  directamente a César como archivo descargable — no vive en este repo.
</content>
