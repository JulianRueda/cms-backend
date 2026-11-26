# Patrones de la tienda pública (JS puro + Vite)

## Cliente liviano de solo lectura

```js
// src/db.js — ~25 KB en vez de ~230 KB de supabase-js
import { PostgrestClient } from "@supabase/postgrest-js"; // agrégala explícita en package.json
const url = import.meta.env.VITE_SUPABASE_URL, key = import.meta.env.VITE_SUPABASE_ANON_KEY;
export const db = url && key ? new PostgrestClient(`${url}/rest/v1`, { headers: { apikey: key } }) : null;
```
Misma API: `db.from("products").select("*, product_images(*), product_variants(*)").eq("is_active", true)`.

## Fotos: helper `pic()` y srcset

```js
// Fotos propias (/imagenes/…) existen como .webp (1080) y -540.webp; las del bucket se usan tal cual.
const pic = (url, small) => /^\/imagenes\/[^?#]+\.(jpe?g|png|webp)$/i.test(url || "")
  ? url.replace(/\.(jpe?g|png|webp)$/i, small ? "-540.webp" : ".webp") : url;
const srcset = (u) => (pic(u, 1) !== u ? `srcset="${pic(u, 1)} 540w, ${pic(u)} 1080w"` : "");
// tarjeta: <img loading="lazy" decoding="async" src="${pic(u,1)}" ${srcset(u)} sizes="(min-width:1200px) 25vw, (min-width:768px) 33vw, 50vw" width="540" height="600">
// al cambiar de foto en la galería: quita el srcset antes de asignar el src.
```
Úsalo en la tienda **y** en el panel (lista, editor, categorías).

## Textos editables con formato mínimo

```js
// "## " título, "- " viñeta, otra línea = párrafo. Escapa siempre.
function rich(k) {
  let html = "", list = false;
  for (const l of lines(k)) {
    const item = l.startsWith("- ");
    if (list && !item) { html += "</ul>"; list = false; }
    if (item) { if (!list) { html += "<ul>"; list = true; } html += `<li>${esc(l.slice(2))}</li>`; }
    else if (l.startsWith("## ")) html += `<h2>${esc(l.slice(3))}</h2>`;
    else html += `<p>${esc(l)}</p>`;
  }
  return html + (list ? "</ul>" : "");
}
```
`lines()` hace `split("\n")` + `trim()`; así también tolera `\r\n`.

## Textos legales fuera del bundle inicial

- `legal-texts.js` exporta `LEGAL = { page_terminos, page_pqrs, page_envios, page_privacidad }`.
- En `content-defaults.js` esas entradas llevan `undefined` como valor (no `null`) y `DEFAULTS` las filtra.
- El panel y el seed las completan con `([section, key, label, value = LEGAL[key]]) => …`.
- La tienda las carga al abrir la página:
```js
if (esLegal && !("page_terminos" in DEFAULTS)) {
  main.innerHTML = "Cargando…";
  import("./legal-texts.js").then(({ LEGAL }) => { Object.assign(DEFAULTS, LEGAL); if (location.hash.includes(page)) renderInfo(page); });
  return;
}
```
Páginas: `#/info/terminos`, `#/info/pqrs` (con botón "Radicar una PQRS" → contacto), `#/info/envios` (tabla de horarios + texto) y `#/info/privacidad`, más chips para saltar entre ellas. Redirige el enlace viejo (`politicas` → `terminos`).

## Menú con desplegable "Productos" centrado

```js
nav.innerHTML = `<a href="#/">Inicio</a>
  <div class="dd"><button class="dd__btn" id="prodBtn" aria-expanded="false" aria-controls="prodMenu">Productos<svg class="ic"><use href="#i-chev"/></svg></button>
    <div class="dd__menu" id="prodMenu"><a href="#/c/todo"><b>Ver todo</b></a>${DEPTS.map(d=>`<a href="#/c/${d.slug}">${esc(d.name)}</a>`).join("")}</div></div>
  <a href="#/contacto">Contáctanos</a>`;
// clic afuera (escritorio), Escape, cambio de ruta y cruce de breakpoint -> cerrar
matchMedia("(min-width:1024px)").addEventListener("change", closeMenus);
```
```css
.header__end{display:contents}                                   /* móvil: buscador y carrito en el flujo */
@media (min-width:1024px){
  .header__row{display:grid;grid-template-columns:minmax(150px,1fr) auto minmax(330px,1fr);gap:24px}
  .logo{justify-self:start}
  .header__end{display:flex;justify-content:flex-end;align-items:center;gap:12px}
  .dd__menu{position:absolute;top:calc(100% + 10px);left:-16px;min-width:230px; /* tarjeta flotante */}
}
.dd__menu{display:none} .dd.open .dd__menu{display:flex;flex-direction:column}
```
HTML: envuelve buscador + carrito en `<div class="header__end">`. Mide el centrado: `navCenter` vs `clientWidth/2` (la barra de desplazamiento explica ~8 px).

## Contáctanos sin backend

Canales (WhatsApp, Instagram, correo si existe, horario, ciudad) y un formulario que arma el mensaje y abre WhatsApp:
```js
form.onsubmit = (e) => { e.preventDefault();
  const text = `Hola <Marca>, soy ${nombre}.\nMotivo: ${motivo}\n\n${mensaje}`;
  window.open(`https://wa.me/${WA}?text=${encodeURIComponent(text)}`, "_blank", "noopener"); };
```
Motivos: consulta de producto, mi pedido, petición, queja, reclamo o garantía, sugerencia. Aclara "No guardamos tus datos en este formulario" y enlaza PQRS y privacidad.

## Redes sociales (opcionales desde el panel)

```js
function socials() {
  const tk = t("tiktok_user").replace(/^@/, "").trim(), fb = t("facebook_url").trim();
  return [[`https://www.instagram.com/${t("instagram_user")}/`, "ig", "Instagram"], [`https://wa.me/${t("whatsapp_number")}`, "wa", "WhatsApp"],
    tk && [`https://www.tiktok.com/@${tk}`, "tiktok", "TikTok"], /^https?:\/\//.test(fb) && [fb, "fb", "Facebook"]].filter(Boolean);
}
```
Íconos redondos de 44 px en el footer, bajo la descripción de la marca. Crédito fijo: `Desarrollado por <a href="https://julirueda.com/">Julián Rueda</a>`.

## Copy cercano

- Tutea: "Pregúntanos el precio", "Pregúntanos cuándo llega", "Ups, no pudimos cargar el catálogo… escríbenos y te ayudamos", "Confirmamos cada pedido contigo por WhatsApp".
- Evita el lenguaje de trámite ("solicitud pendiente de confirmación", "Consultar").
- Lo legal se queda formal.
- Los textos que ya están en la base se actualizan con SQL condicionado al valor viejo (ver SKILL.md §6).

## Caché y conexiones

```json
{ "headers": [
  { "source": "/assets/(.*)",   "headers": [{ "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }] },
  { "source": "/imagenes/(.*)", "headers": [{ "key": "Cache-Control", "value": "public, max-age=2592000, stale-while-revalidate=86400" }] } ] }
```
`<link rel="preconnect" href="%VITE_SUPABASE_URL%" crossorigin>` en `index.html`.
