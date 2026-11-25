---
name: supabase-cms-backend
description: Convierte un frontend ya hecho (exportado de Figma Make, Lovable, HTML/JS estático, React o Vite) en una tienda o sitio autoadministrable tipo mini-Shopify/CMS con Supabase. Incluye Auth del cliente, PostgreSQL con RLS, Storage de fotos y un panel /admin en español para productos, tallas con precio, categorías y textos de la página. Úsala siempre que el usuario pida "hacer el backend", "conectar a Supabase", "que el cliente edite los productos/textos", "panel de administración", "CMS", "catálogo editable", "subir fotos de productos" o quiera reemplazar datos hardcodeados por una base de datos, aunque no nombre Supabase.
---

# Backend CMS con Supabase para un frontend existente

Objetivo: el cliente final (no técnico) entra con su propio correo a `/admin` y edita catálogo, fotos, precios y textos. La web pública lee todo de Supabase en tiempo real y conserva el diseño original.

Este flujo salió de un proyecto real (tienda Dögker). Los "Errores conocidos" de abajo pasaron de verdad: léelos antes de entregar. Ejemplo completo y funcionando: repo privado `JulianRueda/dogker-web` (o `C:\Users\julia\Desktop\dogker`): `admin/`, `app.js`, `scripts/seed.mjs`, `supabase/`.

Archivos de esta skill:
- `references/schema-template.sql`: tablas, RLS, GRANTs y bucket listos para adaptar.
- `references/admin-patterns.md`: código del panel (sesión, fotos, guardado, orden, mock de pruebas).

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
- Editor de producto: filas dinámicas de talla + precio + oferta + disponible (agregar, quitar, reordenar); fotos con vista previa inmediata, compresión en el navegador (≤1600 px, JPEG 0.85, fondo blanco para PNG), reordenar y "principal".
- Orden de guardado seguro: subir fotos nuevas → upsert de filas con ids generados en el cliente (`crypto.randomUUID()`) → borrar las quitadas al final. Un fallo a mitad de camino no debe perder datos.
- Reordenar con flechas: dos `update` intercambiando `display_order`. No uses upsert parcial: los NOT NULL del insert rompen el upsert aunque haya conflicto.
- Textos: combina los valores por defecto del código con lo que hay en la base. Upsert `onConflict: 'section_key'` solo de lo modificado. Así funciona aunque el seed de textos no se haya corrido.
- Login con "¿Olvidaste tu contraseña?" (`resetPasswordForEmail` + evento `PASSWORD_RECOVERY` → `updateUser`). Así el cliente no depende de la cuenta del desarrollador.

## 5. Conectar el frontend público

- Un solo `select` con relaciones anidadas: `products?select=*,product_images(*),product_variants(*)&is_active=eq.true`. Normaliza a la forma que ya usa el front, para tocar lo mínimo de la lógica de render.
- El precio cambia al elegir talla; la oferta se muestra tachada; las tallas sin stock quedan deshabilitadas. Si el carrito guardaba precios, detecta cambios contra la base.
- Textos con respaldo: un módulo `content-defaults` sirve de fallback inmediato (sin parpadeo) y de semilla. Un valor vacío en la base usa el original.
- Si falta el `.env`, exporta `supabase = null` y muestra un estado de error amable en vez de romper.

## 6. Verificación (hazla, no la supongas)

1. `npm test` (parsers/seed) y `npm run build`.
2. Sin Supabase todavía: monta un **mock local de PostgREST** (http en Node que devuelve JSON según la tabla y respeta `Accept: application/vnd.pgrst.object` para `.single()`) con `.env.local` apuntando a él. Prueba la tienda (precio por talla, oferta, carrito, mensaje de WhatsApp) y las pantallas del admin. Para el admin, inyecta una sesión falsa en `localStorage` (`sb-<host>-auth-token`). Borra `.env.local` y la sesión al terminar.
3. Con Supabase real, usando la publishable key vía curl:
   - conteos por tabla con `Prefer: count=exact` + `Range: 0-0` → deben coincidir con el seed;
   - `PATCH`/`DELETE` anónimos → 401; subida anónima a Storage → "row-level security".
4. Pide al usuario una prueba de guardado real en `/admin` con el usuario del cliente. Tú no debes tener su contraseña.

## 7. Entrega

- `README.md` en español con: orden de los scripts SQL, desactivar el registro, URL Configuration (Site URL + `/admin/login/` en Redirect URLs para la recuperación), `.env`, cómo crear el usuario del cliente (Authentication → Users → Add user → Auto Confirm) y qué queda fuera del panel.
- Deja también un `diagnostico.sql` que reaplique los GRANTs, haga `notify pgrst` y liste `information_schema.role_table_grants`.
- Repo: `git init`, revisa con `git grep --cached` que no haya keys ni el ref del proyecto, y crea el repo **privado** con `gh repo create --private --source=. --push` salvo que pidan público. `.mcp.json` y `.claude/` van en `.gitignore`.
- Deploy (Vercel/Netlify): las dos `VITE_SUPABASE_*` en las variables del hosting (se puede pegar el contenido del `.env`). Luego agrega el dominio en URL Configuration.

## Errores conocidos

| Síntoma | Causa | Arreglo |
|---|---|---|
| `permission denied for table X` (401, 42501) con RLS correcto | Proyecto nuevo sin GRANTs | `grant select … to anon, authenticated; grant insert, update, delete … to authenticated; notify pgrst, 'reload schema';` |
| Supabase advierte "creates tables without enabling RLS" | El revisor no lee `execute format(...)` dentro de `do $$` | Falsa alarma: usar "Run and enable RLS" |
| Supabase advierte "destructive operations" | `drop policy if exists` | Normal para scripts re-ejecutables |
| `require(logic.js)` no exporta nada | `"type": "module"` vuelve ESM a los `.js` | El módulo asigna a `globalThis.X`; importarlo por efecto |
| `<script>` clásico 404 tras `vite build` | Vite no empaqueta scripts sin `type="module"` | Importarlo desde el entry module |
| La build falla con top-level await | Target por defecto es2020 | `build.target: "es2022"` |
| Variables vacías en producción | No se configuraron en el hosting | Agregar las `VITE_*` en el panel del hosting y redeploy |
