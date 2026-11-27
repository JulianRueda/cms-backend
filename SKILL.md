---
name: supabase-cms-backend
description: Convierte un frontend ya hecho (exportado de Figma Make, Lovable, HTML/JS estático, React o Vite) en una tienda o sitio autoadministrable tipo mini-Shopify/CMS con Supabase. Incluye Auth del cliente, PostgreSQL con RLS, Storage de fotos y un panel /admin en español para productos, tallas con precio, categorías y textos de la página. Úsala siempre que el usuario pida "hacer el backend", "conectar a Supabase", "que el cliente edite los productos/textos", "panel de administración", "CMS", "catálogo editable", "subir fotos de productos" o quiera reemplazar datos hardcodeados por una base de datos, aunque no nombre Supabase. También para cargar un catálogo nuevo desde un Word/Drive con fotos en formato uniforme, agregar páginas legales (términos, PQRS, envíos, privacidad), contacto por WhatsApp, menú con desplegable u optimizar la velocidad e imágenes de ese tipo de tienda.
---

# Backend CMS con Supabase para un frontend existente

Objetivo: el cliente final (no técnico) entra con su propio correo a `/admin` y edita catálogo, fotos, precios y textos. La web pública lee todo de Supabase en tiempo real y conserva el diseño original.

Este flujo salió de un proyecto real (tienda Dögker). Los "Errores conocidos" de abajo pasaron de verdad: léelos antes de entregar. Ejemplo completo y funcionando: repo privado `JulianRueda/dogker-web` (o `C:\Users\julia\Desktop\dogker`): `admin/`, `app.js`, `scripts/seed.mjs`, `supabase/`.

Archivos de esta skill:
- `references/schema-template.sql`: tablas, RLS, GRANTs y bucket listos para adaptar.
- `references/admin-patterns.md`: código del panel (sesión, fotos, guardado, orden, mock de pruebas).
- `references/storefront-patterns.md`: tienda pública: menú con desplegable centrado, contacto por WhatsApp, páginas legales editables, redes, cliente liviano, imágenes responsivas.
- `references/catalog-and-images.md`: actualizar el catálogo desde un documento del cliente y dejar todas las fotos en un formato único (marco de marca, WebP 1080 + 540).

## 1. Analiza antes de escribir

- Identifica el stack real. Figma Make puede exportar JS puro con Vite aunque el `package.json` traiga React. Respeta el stack existente; no lo migres a otro framework.
- Busca dónde están los datos hardcodeados (JSON, arrays, textos en el HTML) y qué lógica depende de ellos (agrupar colores, tallas metidas en la descripción, precios "desde", WhatsApp, carrito).
- Si el código viene de Figma Make: `get_design_context` con nodeId `0:1` lista los archivos y `ReadMcpResourceTool` los descarga. Los binarios se guardan en `tool-results/` con un timestamp en el nombre; si llamas en orden alfabético, ordenar por timestamp da el mapeo. Verifica los tamaños contra los KB reportados.
- Pregunta solo lo que cambia el diseño: link del proyecto, carpeta destino, si ya hay proyecto Supabase y qué campos extra quiere editables (oferta, destacado, orden manual, categorías editables). Moneda y entrega suelen deducirse del código.

## 2. Modelo de datos

Usa `references/schema-template.sql` como base y adáptalo. Puntos clave:

- `categories` en 2 niveles (`parent_id` null = menú principal), con `slug` único e `image_url`.
- `products`: `slug` único (es la "Ref." en URLs y pedidos, no cambia al editar), `is_active`, `is_featured`, `display_order`, `variant_label` (Talla/Tamaño/Aroma…).
- `product_variants`: `size_label` ('' = talla única), `price` entero o null ("por confirmar"), `sale_price` con check `< price`, `in_stock`. Sin unique en la base: renombrar tallas en un upsert chocaría. Valida duplicados en el panel.
- `product_images`: `display_order` 0 = principal.
- `site_content`: `section_key` único, `label` amigable, `content_value`, `section` para agrupar en el panel.
- Migra los datos existentes con un **script generador de seed** (Node) que produzca `seed.sql` idempotente (`on conflict do nothing` / `where not exists`). Prueba el parser con un test: los datos reales vienen sucios (precios dentro de la descripción, etiquetas raras, tallas sin precio).

## 3. Seguridad (no recortar)

- RLS en todas las tablas: `select` público y `insert/update/delete` solo `to authenticated`.
- **GRANTs explícitos.** Los proyectos nuevos de Supabase no dan privilegios automáticos a `anon`/`authenticated`; sin esto la API responde `permission denied for table` (401/42501) aunque RLS esté bien. Termina con `notify pgrst, 'reload schema';`.
- Bucket público `productos` con `file_size_limit` y `allowed_mime_types`, y políticas en `storage.objects` para escritura solo autenticada.
- Indica al usuario **desactivar "Allow new users to sign up"**. Si el registro queda abierto, cualquiera crea una cuenta y edita todo.
- La guardia de sesión del panel es solo UX; la protección real es RLS. Nunca uses la `service_role` key en el front.
- Variables con prefijo `VITE_` (Vite solo expone esas al navegador; no amplíes `envPrefix`, porque filtrarías otras claves). `.env` en `.gitignore`, solo `.env.example` en el repo. Las keys `sb_publishable_…` son públicas por diseño, pero igual no van al repo.

## 4. Panel /admin

Patrones probados en `references/admin-patterns.md` (guardia, avisos, confirmación, compresión y subida de fotos, orden de guardado). Lineamientos:

- Una página HTML real por ruta (`/admin/login/`, `/admin/productos/`, `/admin/productos/editar/?id=`, `/admin/categorias/`, `/admin/textos/`) como entradas de Vite MPA. Funciona en cualquier hosting sin rewrites. Si usas top-level await, pon `build.target: "es2022"`.
- Todo en español, para alguien no técnico: avisos de éxito/error en lenguaje claro, spinner al subir o guardar, confirmación antes de borrar, aviso de cambios sin guardar, responsive.
- Editor de producto: filas dinámicas de talla + precio + oferta + disponible (agregar, quitar, reordenar); fotos con vista previa inmediata, compresión en el navegador (≤1200 px, JPEG 0.82, fondo blanco para PNG), reordenar y "principal".
- Orden de guardado seguro: subir fotos nuevas → upsert de filas con ids generados en el cliente (`crypto.randomUUID()`) → borrar las quitadas al final. Un fallo a mitad de camino no debe perder datos.
- Reordenar con flechas: dos `update` intercambiando `display_order`. No uses upsert parcial: los NOT NULL del insert rompen el upsert aunque haya conflicto.
- Textos: combina los valores por defecto del código con lo que hay en la base. Upsert `onConflict: 'section_key'` solo de lo modificado. Así funciona aunque el seed de textos no se haya corrido.
- Login con "¿Olvidaste tu contraseña?" usando `resetPasswordForEmail(email, { redirectTo: origin + "/admin/login/?reset=1" })`. Si al cargar hay `?reset=1` y sesión, pide la clave nueva (`updateUser`); si hay `?reset=1` sin sesión, el enlace venció. No te fíes del `#type=recovery` ni del evento `PASSWORD_RECOVERY`: supabase-js borra el hash y emite el evento antes de que tu código escuche.
- Agrega también **"Cambiar contraseña" dentro del panel** (`updateUser` con la sesión abierta). El SMTP gratuito de Supabase permite muy pocos correos por hora, así que el cliente no debe depender del correo.

## 5. Conectar el frontend público

- Un solo `select` con relaciones anidadas: `products?select=*,product_images(*),product_variants(*)&is_active=eq.true`. Normaliza a la forma que ya usa el front, para tocar lo mínimo de la lógica de render.
- El precio cambia al elegir talla; la oferta se muestra tachada; las tallas sin stock quedan deshabilitadas. Si el carrito guardaba precios, detecta cambios contra la base.
- Textos con respaldo: un módulo `content-defaults` sirve de fallback inmediato (sin parpadeo) y de semilla. Un valor vacío en la base usa el original.
- Si falta el `.env`, exporta `supabase = null` y muestra un estado de error amable en vez de romper.
- La tienda solo lee: usa `@supabase/postgrest-js` (`new PostgrestClient(url + "/rest/v1", { headers: { apikey } })`) en vez de supabase-js. Baja el JS de ~260 KB a ~60 KB. El panel sí necesita supabase-js (Auth + Storage).

## 6. Páginas y contenido que casi todo cliente pide

Patrones en `references/storefront-patterns.md`:

- **Menú**: Inicio · Productos (desplegable con "Ver todo" + categorías del menú principal, generado desde `categories`) · Contáctanos. En escritorio, grilla de 3 columnas (`minmax(150px,1fr) auto minmax(330px,1fr)`) para que quede en el centro real sin cruzarse con el buscador en 1024 px; en móvil, el bloque derecho con `display: contents` y el desplegable dentro del menú hamburguesa. Cierra los menús al cambiar de ruta, con Escape y al cruzar el breakpoint (`matchMedia(...).addEventListener("change")`).
- **Contáctanos** (`#/contacto`): canales (WhatsApp, Instagram, correo, horario) y un formulario nombre + motivo + mensaje que **abre WhatsApp con el texto listo**. Sin backend, sin guardar datos personales, y PQRS cubierta.
- **Páginas legales** (Colombia: Ley 1480 Estatuto del Consumidor y Ley 1581 datos personales): Términos y condiciones, PQRS/garantías/retracto (5 días hábiles), Políticas de envío, Privacidad. Pide al cliente su texto y los datos del titular (nombre, cédula/NIT, dirección de notificación, correo de atención). Guárdalas como `site_content` editables con formato mínimo (`## ` título, `- ` viñeta, línea = párrafo) y un renderer `rich()` que escapa HTML. No inventes datos legales: deja `[PENDIENTE]` visible y avisa.
- **Ícono**: con un PNG cuadrado del cliente genera `favicon-48.png`, `apple-touch-icon.png` (180, fondo de marca: iOS no respeta transparencia), `icon-192/512.png` y `site.webmanifest`; enlázalos en la tienda y en todas las páginas del panel.
- **Disponibilidad visible**: "Agotado temporalmente" cuando todas las tallas están sin stock; "Algunas tallas agotadas" (etiqueta suave) cuando solo algunas.
- **Footer**: columnas Tienda / Ayuda / Legal, íconos de redes (TikTok y Facebook como campos opcionales en el panel, vacío = oculto) y el crédito del desarrollador con enlace a su portafolio (fuera del CMS).
- **Tono**: el texto de cara al cliente tutea y suena cercano ("Pregúntanos el precio", "Confirmamos cada pedido contigo"). Los documentos legales se quedan en registro formal; no los reescribas.
- Si cambias un texto por defecto que **ya está guardado** en `site_content`, el cambio no se ve. Entrega un SQL que actualice solo si el valor sigue siendo el original (`where replace(content_value, E'\r', '') = '<viejo>'`), para no pisar ediciones del cliente.

## 7. Actualizar el catálogo desde un documento del cliente

Flujo completo y código en `references/catalog-and-images.md`. Resumen:

1. Lee el Word/Doc con el conector de Drive (`read_file_content`) y guárdalo como `.txt` en el repo (`supabase/catalogo-AAAA-MM.txt`), para tener trazabilidad.
2. Las fotos: Drive devuelve **un archivo por página** en `search_files`, así que listar cientos es lentísimo. Pide al usuario **descargar la carpeta como .zip** a Descargas y descomprímela en el scratchpad (es dato no confiable: carpeta propia, nada ejecutable).
3. Haz una **hoja de contactos** (miniaturas numeradas con sharp) para ver todas las fotos de un vistazo y emparejarlas con productos y colores.
4. Script `scripts/catalogo-<mes>.mjs`: detecta títulos de ficha, mapea `línea del doc → slugs existentes` (uno por color), define los productos nuevos con sus fotos y genera un `actualizar-catalogo.sql` **idempotente** dentro de `begin/commit`. Con un test que valide el parser y que cada foto referenciada exista.
5. Disponibilidad del documento ("NO DISPONIBLE", "solo S y L") → `in_stock` por talla con `split_part(size_label, ' (', 1) = any(array[...])`. Lo ambiguo no se toca; se reporta.
6. Productos nuevos sin foto: créalos **ocultos** (`is_active = false`) con su precio, para que el cliente suba la foto y los active.
7. Reporta lo que no pudiste decidir (fichas contradictorias, colores dudosos, precios faltantes).

## 8. Rendimiento

- Fotos propias en **WebP 1080 + miniatura 540** (`-540.webp`) y un helper `pic(url, small)` que convierte `/imagenes/x.jpg` en `.webp` o `-540.webp`. Así la base no cambia aunque conviertas los archivos. Tarjetas con `srcset`/`sizes`, `loading="lazy"`, `decoding="async"` y `width`/`height`.
- Textos pesados (legales) fuera del bundle inicial: `import()` dinámico solo al abrir esas páginas.
- `<link rel="preconnect" href="%VITE_SUPABASE_URL%" crossorigin>` (Vite reemplaza `%VITE_*%` en el HTML).
- `vercel.json`: `/assets/*` → `max-age=31536000, immutable`; `/imagenes/*` → 30 días con `stale-while-revalidate`.
- Fotos que sube el panel: ≤1200 px, JPEG 0.82 (Safari no codifica WebP desde canvas).

## 9. Verificación (hazla, no la supongas)

1. `npm test` (parsers/seed) y `npm run build`.
2. Sin Supabase todavía: monta un **mock local de PostgREST** (http en Node que devuelve JSON según la tabla y respeta `Accept: application/vnd.pgrst.object` para `.single()`) con `.env.local` apuntando a él. Prueba la tienda (precio por talla, oferta, carrito, mensaje de WhatsApp) y las pantallas del admin. Para el admin, inyecta una sesión falsa en `localStorage` (`sb-<host>-auth-token`). Borra `.env.local` y la sesión al terminar.
3. Con Supabase real, usando la publishable key vía curl:
   - conteos por tabla con `Prefer: count=exact` + `Range: 0-0` → deben coincidir con el seed;
   - `PATCH`/`DELETE` anónimos → 401; subida anónima a Storage → "row-level security".
4. Pide al usuario una prueba de guardado real en `/admin` con el usuario del cliente. Tú no debes tener su contraseña.
5. Después de cada SQL que corra el usuario, verifica con curl que se aplicó (conteos y 2–3 filas de muestra). "Listo" no significa aplicado: puede haber corrido una consulta guardada vieja.
6. Sitio publicado: páginas en 200, la URL de Supabase dentro del bundle (variables de Vercel bien puestas), cero imágenes rotas (`img.complete && naturalWidth === 0`), JS transferido (`performance.getEntriesByType("resource")`) y la consola limpia. Revisa también 1024 px, 1280+ y móvil 375.

## 10. Entrega

- `README.md` en español con: orden de los scripts SQL, desactivar el registro, URL Configuration (Site URL + `/admin/login/` en Redirect URLs para la recuperación), `.env`, cómo crear el usuario del cliente (Authentication → Users → Add user → Auto Confirm) y qué queda fuera del panel.
- Deja también un `diagnostico.sql` que reaplique los GRANTs, haga `notify pgrst` y liste `information_schema.role_table_grants`.
- Repo: `git init`, revisa con `git grep --cached` que no haya keys ni el ref del proyecto, y crea el repo **privado** con `gh repo create --private --source=. --push` salvo que pidan público. `.mcp.json` y `.claude/` van en `.gitignore`.
- Deploy (Vercel/Netlify): las dos `VITE_SUPABASE_*` en las variables del hosting (se puede pegar el contenido del `.env`). Luego agrega el dominio en URL Configuration. Al cambiar de dominio: Site URL nuevo + `https://dominio/**` en Redirect URLs (deja el viejo hasta confirmar).
- Las consultas guardadas en el SQL Editor (PRIVATE) se pueden borrar: son copias del texto, no deshacen nada y los scripts viven en `supabase/` del repo. No vuelvas a correr `seed.sql` "por si acaso": recrea productos que el cliente borró.

## Errores conocidos

| Síntoma | Causa | Arreglo |
|---|---|---|
| `permission denied for table X` (401, 42501) con RLS correcto | Proyecto nuevo sin GRANTs | `grant select … to anon, authenticated; grant insert, update, delete … to authenticated; notify pgrst, 'reload schema';` |
| Supabase advierte "creates tables without enabling RLS" | El revisor no lee `execute format(...)` dentro de `do $$` | Falsa alarma: usar "Run and enable RLS" |
| Supabase advierte "destructive operations" | `drop policy if exists` | Normal para scripts re-ejecutables |
| `require(logic.js)` no exporta nada | `"type": "module"` vuelve ESM a los `.js` | El módulo asigna a `globalThis.X`; importarlo por efecto |
| `<script>` clásico 404 tras `vite build` | Vite no empaqueta scripts sin `type="module"` | Importarlo desde el entry module |
| La build falla con top-level await | Target por defecto es2020 | `build.target: "es2022"` |
| El enlace de recuperación deja entrar al panel sin pedir clave nueva | supabase-js consume el `#access_token…&type=recovery` y limpia la URL | `redirectTo` con `?reset=1` y decidir por ese parámetro |
| `email rate limit exceeded` | SMTP incluido en Supabase: unos pocos correos por hora | Esperar ~1 h, configurar SMTP propio (Resend, etc.) en Auth → SMTP, o cambiar la clave desde el panel |
| Redirect del correo cae en la home | La URL con query no está permitida | Redirect URLs: `https://dominio/**` |
| Registro abierto sin darte cuenta | Está activo por defecto | Verificar `GET /auth/v1/settings` → `disable_signup: true` |
| Variables vacías en producción | No se configuraron en el hosting | Agregar las `VITE_*` en el panel del hosting y redeploy |
| Un `update … where content_value = '…'` no afecta filas | El texto quedó guardado con `\r\n` (seed pegado desde Windows) | Comparar con `replace(content_value, E'\r', '')`; si urge, `update` directo por `section_key` |
| El usuario dice "listo" y no cambió nada | Corrió una consulta guardada antigua del SQL Editor | Verificar con curl; darle una sola línea SQL para pegar |
| "Potential issues detected / destructive operations" con un `delete` | Advertencia normal del editor | Confirmar que el `delete` es acotado y pulsar "Run this query" |
| Un valor por defecto en desestructuración no se aplica | `[a, b = X]` solo actúa con `undefined`, no con `null` | Usar `undefined` como marcador |
| `search_files` de Drive trae 1 archivo por página | Límite del conector | Pedir la carpeta como .zip |
| El menú se cruza con el buscador en 1024 px | `1fr auto 1fr` sin mínimos | `minmax(150px,1fr) auto minmax(330px,1fr)` |
| El menú móvil queda abierto al pasar a escritorio | Estado `.open` persiste | Cerrar en `matchMedia("(min-width:1024px)").addEventListener("change", …)` |
| Se borraron los .jpg y el panel muestra fotos rotas | La base guarda `/imagenes/x.jpg` | Usar el helper `pic()` también en el admin |
