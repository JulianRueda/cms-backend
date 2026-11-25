# Patrones del panel /admin (JS puro + supabase-js v2)

Fragmentos probados. Adáptalos al stack del proyecto (en React, las mismas funciones viven en hooks o utilidades).

## Cliente y env

```js
// src/supabase.js
import { createClient } from "@supabase/supabase-js";
const url = import.meta.env.VITE_SUPABASE_URL, key = import.meta.env.VITE_SUPABASE_ANON_KEY;
export const supabase = url && key ? createClient(url, key) : null; // null => pantalla amable, no crash
export const BUCKET = "productos";
```

## Guardia de sesión (UX; la seguridad real es RLS)

```js
export async function requireSession() {
  const { data: { session } } = await supabase.auth.getSession();
  if (!session) { location.replace(`/admin/login/?next=${encodeURIComponent(location.pathname + location.search)}`); throw new Error("sin sesión"); }
  supabase.auth.onAuthStateChange((_ev, s) => { if (!s) location.replace("/admin/login/"); });
  document.body.classList.remove("booting"); // body.booting oculta el contenido y muestra un spinner hasta aquí
  return session;
}
```

En el login, acepta solo `next` que empiece por `/admin/` (evita redirecciones abiertas).

## Recuperar contraseña

```js
await supabase.auth.resetPasswordForEmail(email, { redirectTo: location.origin + "/admin/login/" });
supabase.auth.onAuthStateChange((ev) => { if (ev === "PASSWORD_RECOVERY") showNewPasswordForm(); });
// ... luego: await supabase.auth.updateUser({ password });
```

La Redirect URL tiene que estar permitida en Authentication → URL Configuration.

## Errores en lenguaje del cliente

```js
export function errMsg(e) {
  const m = e?.message || String(e);
  if (/JWT|session|not authenticated|row-level security/i.test(m)) return "Tu sesión expiró. Vuelve a ingresar.";
  if (/Failed to fetch|NetworkError/i.test(m)) return "Sin conexión. Revisa tu internet e intenta de nuevo.";
  if (/foreign key/i.test(m)) return "No se puede borrar: hay elementos que dependen de este.";
  return m;
}
```

## Confirmación antes de borrar (`<dialog>` nativo)

```js
export function confirmDialog({ title, text, ok = "Eliminar" }) {
  return new Promise((resolve) => {
    const d = document.createElement("dialog");
    d.innerHTML = `<h3>${esc(title)}</h3><p>${esc(text)}</p><button value="no">Cancelar</button><button value="si">${esc(ok)}</button>`;
    document.body.append(d);
    d.querySelectorAll("button").forEach((b) => (b.onclick = () => d.close(b.value)));
    d.onclose = () => { resolve(d.returnValue === "si"); d.remove(); };
    d.showModal();
  });
}
```

En el texto, ofrece la alternativa suave: "Si solo quieres ocultarlo, desmarca «Visible»".

## Fotos: validar, comprimir y subir

```js
export async function compressImage(file) {
  if (!/^image\/(jpeg|png|webp)$/.test(file.type)) throw new Error(`«${file.name}»: usa JPG, PNG o WebP.`);
  if (file.size > 15 * 1024 * 1024) throw new Error(`«${file.name}» pesa más de 15 MB.`);
  const bmp = await createImageBitmap(file);
  const s = Math.min(1, 1600 / Math.max(bmp.width, bmp.height));
  const c = Object.assign(document.createElement("canvas"), { width: Math.round(bmp.width * s), height: Math.round(bmp.height * s) });
  const ctx = c.getContext("2d"); ctx.fillStyle = "#fff"; ctx.fillRect(0, 0, c.width, c.height); // PNG transparente
  ctx.drawImage(bmp, 0, 0, c.width, c.height);
  const blob = await new Promise((r) => c.toBlob(r, "image/jpeg", 0.85)); // JPEG: compatible en todos los navegadores
  if (!blob || blob.size > 5 * 1024 * 1024) throw new Error(`«${file.name}» sigue pesando demasiado.`);
  return blob;
}
export async function uploadImage(blob, folder) {
  const path = `${folder}/${crypto.randomUUID()}.jpg`;
  const { error } = await supabase.storage.from(BUCKET).upload(path, blob, { contentType: "image/jpeg", cacheControl: "31536000" });
  if (error) throw error;
  return supabase.storage.from(BUCKET).getPublicUrl(path).data.publicUrl;
}
export async function removeStored(urls) { // solo borra lo que vive en el bucket (no las fotos estáticas originales)
  const marker = `/storage/v1/object/public/${BUCKET}/`;
  const paths = urls.filter((u) => u?.includes(marker)).map((u) => decodeURIComponent(u.split(marker)[1]));
  if (paths.length) await supabase.storage.from(BUCKET).remove(paths);
}
```

Flujo en el editor: al elegir archivos muestra la vista previa con `URL.createObjectURL(file)` y el estado "Optimizando…", comprime y reemplaza la vista previa por el blob. **Sube solo al pulsar Guardar.**

## Guardar un producto sin perder datos

```js
// 1) producto: insert (slug único + display_order) o update
// 2) subir blobs nuevos -> url; cada foto nueva recibe id = crypto.randomUUID()
await supabase.from("product_images").upsert(images.map((im, i) => ({ id: im.id, product_id, image_url: im.url, display_order: i })));
await supabase.from("product_variants").upsert(variants.map((v, i) => ({ id: v.id, product_id, size_label: v.size_label.trim(), price: v.price, sale_price: v.sale_price, in_stock: v.in_stock, display_order: i })));
// 3) recién ahora: borrar ids que estaban al cargar y ya no están (+ removeStored de esas fotos)
```

Valida antes de enviar: nombre obligatorio; tallas sin duplicados; si hay varias, ninguna vacía; oferta menor que el precio; fotos ya optimizadas.

## Slug único

```js
export async function uniqueSlug(table, name) {
  const base = slugify(name) || "item";
  const { data } = await supabase.from(table).select("slug").like("slug", `${base}%`);
  const taken = new Set((data || []).map((r) => r.slug));
  let s = base, i = 2; while (taken.has(s)) s = `${base}-${i++}`;
  return s;
}
```

## Reordenar (flechas)

```js
const [oa, ob] = a.display_order === b.display_order ? [b.display_order + dir, b.display_order] : [b.display_order, a.display_order];
await Promise.all([
  supabase.from("products").update({ display_order: oa }).eq("id", a.id),
  supabase.from("products").update({ display_order: ob }).eq("id", b.id),
]);
```

Desactiva las flechas mientras haya búsqueda o filtro activos: así el orden siempre se refiere a la lista completa.

## Precios COP en inputs

```js
const parseCOP = (s) => { const d = String(s ?? "").replace(/\D/g, ""); return d ? Number(d) : null; };
const fmtCOP = (n) => (n == null ? "" : Math.round(n).toLocaleString("es-CO").replace(/,/g, "."));
// input type="text" inputmode="numeric"; oninput -> parseCOP; onblur -> fmtCOP
```

## Mock de PostgREST para probar sin Supabase

Servidor http en Node (puerto 54321): devuelve el arreglo de la tabla según el último segmento de la URL, filtra `id=eq.X` y, si `Accept` incluye `vnd.pgrst.object`, devuelve el primer elemento. Agrega los headers CORS (`*`) y responde a OPTIONS. Apunta `.env.local` a `http://localhost:54321`. Para el admin, inyecta en `localStorage['sb-localhost-auth-token']` un objeto `{access_token, refresh_token, expires_at: ahora+3600, user:{id,email}}`. Solo sirve para lecturas; el guardado se prueba contra el Supabase real.
