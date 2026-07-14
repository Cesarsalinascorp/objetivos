# Panel Digital SAVAL 2026 — Contexto del proyecto

## Qué es esto

Panel de seguimiento de objetivos del equipo digital de SAVAL (César, Dani, Tati, Juan).
Es un solo archivo HTML (`index.html`, vanilla JS, sin build ni framework) desplegado
como sitio estático en GitHub Pages: https://cesarsalinascorp.github.io/objetivos/

Repo: `Cesarsalinascorp/objetivos` (rama `main`, deploy automático vía GitHub Pages al
hacer push a `main`).

## Backend: Supabase

- Auth y datos vía Supabase (`SUPABASE_URL`, `SUPABASE_KEY` en el propio `index.html`, línea ~515).
- Tabla principal: `panel_datos` (clave/valor) — ahí se guarda todo el estado editable
  (marcas de completado, notas, valores de campos, etc.) vía REST API de Supabase.
- El login "Solo para editar el panel" es contra Supabase Auth (`SB_AUTH_URL`).
- **Cuidado con el plan gratuito de Supabase**: el proyecto se pausa automáticamente si
  no hay actividad por ~7 días. Si el panel empieza a tirar "Email o contraseña
  incorrectos" sin razón aparente, lo primero es revisar si el proyecto está pausado en
  el dashboard de Supabase.

## Estructura de datos de objetivos

Cada persona tiene un array `objetivos`, cada objetivo tiene `subs` (subtareas), y cada
sub tiene `campos` (métricas individuales). Tipos de campo (`tipo`):
- `entregable`: se marca completado + fecha + link (ej. envío de un reporte)
- `puntual`: valor fijo mensual a comparar contra una meta
- `acum`: acumulativo, la meta se distribuye linealmente entre `inicio` y diciembre
- `select` / `alin_pct`: casos especiales (reputación IP, alineación de envíos por UNE)

## Sistema de visibilidad por fecha/frecuencia (agregado recientemente)

Las subtareas (`sub`) pueden tener:
- `visInicio`: mes (1-12) en que empieza a mostrarse. Si no está definido, siempre visible.
- `visFin`: mes en que deja de mostrarse (default 12).
- `visFreq`: cada cuántos meses se muestra a partir de `visInicio` (default 1 = todos los meses).

Función clave: `subVisibleEnMes(sub, m)` — decide si una sub se renderiza ese mes.
`semSub()` también respeta esto: si la sub no aplica ese mes, devuelve `'gris'` en vez
de penalizar el semáforo general del objetivo padre.

Ejemplo real ya implementado: Digital News Ecuador y Costa Rica tienen
`visInicio:7, visFin:12, visFreq:2` → solo aparecen en julio, septiembre y noviembre
(primera entrega semestral en julio, luego bimensual).

## Semáforo (colores de avance)

- `metaEsperada(campo, m)`: calcula cuánto se "esperaría" tener avanzado a un mes dado.
- `semCampo(c, v, m)`: color de un campo individual (gris/rojo/amarillo/verde/sobre).
- `semSub(sub, p, m)`: color agregado de una subtarea.
- `semObj(obj, p, m)`: color agregado de un objetivo completo (para el conteo de arriba
  del panel: verde/amarillo/rojo).

## Pendiente / cosas a mejorar (mencionadas en conversación, no resueltas aún)

- La barra de **% de avance** (no el semáforo, el número) de items `entregable` todavía
  asume que "debería" haber una entrega cada mes transcurrido desde `inicio`, sin
  considerar `visFreq`. Para casos bimensuales como Ecuador/CR, esto puede mostrar un %
  más bajo del real en meses donde no correspondía entregar. Ajustar en `avancePct()`
  si se pide.

## Deploy

No hay CI/CD más allá de GitHub Pages sirviendo directo desde `main`. Cualquier commit
a `main` se refleja en el sitio en 1-5 minutos. No hay tests automatizados — validar
sintaxis JS manualmente antes de pushear:

```bash
python3 -c "
import re
html = open('index.html', encoding='utf-8').read()
scripts = re.findall(r'<script[^>]*>(.*?)</script>', html, re.S)
open('/tmp/combined.js','w',encoding='utf-8').write('\n'.join(scripts))
"
node --check /tmp/combined.js
```

## Otros contextos de César (no directamente parte de este proyecto)

- Publicista / Especialista Performance Digital en Laboratorios SAVAL desde abril 2025.
- Este panel es para uso interno del equipo (César, Dani, Tati, Juan), no tiene relación
  con Attempo (otro proyecto propio de César, SaaS de agendamiento — no mezclar contexto).
