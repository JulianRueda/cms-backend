# Actualizar el catálogo y dejar las fotos en un formato único

Probado en Dögker (noviembre 2026): 84 fichas actualizadas, 45 productos nuevos y 79 fotos. Archivos reales: `scripts/fotos.mjs`, `scripts/catalogo-nov.mjs` y `tests/catalogo.test.mjs` en `JulianRueda/dogker-web`.

## 1. Traer el documento y las fotos

- Documento de Google Drive (Word o Doc): `read_file_content(fileId)` devuelve texto. Guárdalo en `supabase/catalogo-AAAA-MM.txt`.
- Carpeta de fotos: no la listes con `search_files` (devuelve 1 archivo por página). Pide al usuario: clic derecho en la carpeta → Descargar → .zip en Descargas. Descomprímela en el scratchpad:
  ```bash
  mkdir -p "$S/zip" && cd "$S/zip" && unzip -q -o "/c/Users/<u>/Downloads/<archivo>.zip"
  ```
- Revisa tamaños y transparencia antes de decidir el formato (`sharp(f).metadata()` y `.stats().isOpaque`).

## 2. Hoja de contactos (ver todo de un vistazo)

```js
// node sheet.cjs <dir> <salida.jpg> <inicio> <cantidad>  → imprime "índice archivo"
const sharp=require("sharp"),fs=require("fs"),path=require("path");
const [dir,out,start,count]=[process.argv[2],process.argv[3],+process.argv[4],+process.argv[5]];
const files=fs.readdirSync(dir).sort().slice(start,start+count);const T=220,C=6;
(async()=>{const tiles=await Promise.all(files.map(async(f,i)=>{const img=await sharp(path.join(dir,f)).resize(T,T,{fit:"contain",background:"#888"}).png().toBuffer();
const lab=Buffer.from(`<svg width="${T}" height="26"><rect width="${T}" height="26" fill="#000"/><text x="4" y="18" font-size="15" fill="#fff" font-family="Arial">${start+i}</text></svg>`);
return sharp(img).composite([{input:lab,top:0,left:0}]).png().toBuffer();}));
await sharp({create:{width:C*T,height:Math.ceil(tiles.length/C)*T,channels:3,background:"#fff"}})
 .composite(tiles.map((b,i)=>({input:b,left:(i%C)*T,top:Math.floor(i/C)*T}))).jpeg({quality:70}).toFile(out);
files.forEach((f,i)=>console.log(start+i,f));})();
```
Léela con Read (imagen) y empareja índice → producto/color. Hazlo también con las fotos ya procesadas para revisar el resultado.

## 3. Formato único: reconstruir el marco de la marca

Si las fotos actuales comparten un marco o fondo de marca (p. ej. un patrón alrededor y el centro blanco), se reconstruye **sin el archivo original**: el patrón es igual en todas las fotos y los productos cambian, así que el píxel más claro entre muchas fotos deja el patrón y vuelve blanco el centro.

```js
let acc=null;
for (const f of files) {
  const { data } = await sharp(f).resize(1080,1080).removeAlpha().raw().toBuffer({ resolveWithObject: true });
  if (!acc) acc = Buffer.from(data); else for (let i=0;i<acc.length;i++) if (data[i]>acc[i]) acc[i]=data[i];
}
await sharp(acc,{raw:{width:1080,height:1080,channels:3}}).png().toFile("scripts/assets/marco.png");
```
Mide la zona blanca interior (en Dögker: ~860 px) y deja el producto en una caja algo menor (780 px), centrado.

## 4. Procesar fotos (`scripts/fotos.mjs`)

```js
export async function writePair(input, base) { // 1080 + miniatura 540, WebP
  const img = sharp(input).flatten({ background: "#ffffff" });
  await img.clone().resize(1080,1080,{fit:"cover"}).webp({quality:80,effort:5}).toFile(`${IMG_DIR}${base}.webp`);
  await img.clone().resize(540,540,{fit:"cover"}).webp({quality:78,effort:5}).toFile(`${IMG_DIR}${base}-540.webp`);
}
export async function framed(file, base) { // foto de estudio -> recorte del blanco sobrante -> centrada sobre el marco
  const product = await sharp(file).flatten({background:"#ffffff"}).trim({background:"#ffffff",threshold:18})
    .resize(780,780,{fit:"inside"}).toBuffer();
  const { width, height } = await sharp(product).metadata();
  const composed = await sharp(FRAME).composite([{ input: product, left: Math.round(540-width/2), top: Math.round(540-height/2) }]).png().toBuffer();
  await writePair(composed, base);
}
// Ejecutado solo: convierte todo .jpg/.png de public/imagenes a WebP y borra el original.
```
- `sharp` va como devDependency (Vercel la instala sin problema).
- Resultado en Dögker: 140 KB → 76 KB (1080) y 29 KB (540) por foto.
- Antes de borrar los .jpg, haz que la web y el panel usen `pic()` (ver storefront-patterns.md); si no, la base sigue apuntando a `.jpg`.

## 4b. Agregar más fotos a productos que ya existen (cientos de fotos sin nombre de producto)

Caso real: zip con ~630 JPG de estudio numerados (`Dogker-Catalogo_ParteII-123.jpg`) + ~110 PNG con nombres informales ("Peluche gatico 2"). Resultado: 131 fotos para 76 productos.
1. **Huella por foto**: recorta el blanco (`trim`), reduce a 48×48 e histograma de color 6×6×6 sin los píxeles casi blancos. A la foto actual del producto (con marco) recórtale antes el centro (`extract` de la zona blanca). Distancia = 1 − intersección de histogramas (no depende del ángulo).
2. **Candidatas**: misma mejor coincidencia y distancia ≤ 0.15, **o** distancia ≤ 0.3 si la foto está a ±4 números de una "ancla" (distancia ≤ 0.05, que es la foto actual o su gemela): las fotos de un producto se tomaron seguidas.
3. **Series con nombre**: agrupa por el nombre sin el número final; la serie entera va al producto donde mejor coincide cualquiera de sus fotos (≤ 0.12).
4. **Variedad**: descarta las casi idénticas a la principal o entre sí (diferencia media de píxeles 32×32 ≤ 0.08) y deja máximo 4 extras. Omite productos que ya tienen varias fotos.
5. **Revisión visual obligatoria**: hojas de contactos de 8–10 filas (foto actual + candidatas). Siempre aparecen errores: cartones de otra referencia, correas o bebederos que se cuelan en un transportador del mismo color. Exclúyelos por nombre de archivo; si una exclusión deja entrar otra candidata, revisa esa también.
6. Guarda el mapeo (`supabase/fotos-extra.json`), procesa con `framed(file, "<slug>-f<n>")` y genera `fotos-extra.sql` (`insert … where not exists`, `display_order` 1..n). Verifica que cada URL del SQL tenga su `.webp` y su `-540.webp`.

## 5. Script del catálogo (`scripts/catalogo-<mes>.mjs`)

Partes:
```js
const DOC = readFileSync(".../catalogo-2026-11.txt","utf8").split("\n").map(s=>s.trim());
// Título de ficha = línea sin ":" (los campos siempre llevan "Campo: valor")
const isTitle = (l,i) => i>10 && l.length>3 && !l.includes(":") && !/^Descripciones comerciales/.test(l);
const TITLES = DOC.map((l,i)=>isTitle(l,i)?i+1:0).filter(Boolean);
const block = (line) => DOC.slice(line, (TITLES.find(t=>t>line) ?? DOC.length+1) - 1).filter(Boolean);
const field = (b,re) => b.find(l=>re.test(l))?.replace(re,"").trim() || null;
export function prices(line) {           // "Precios: XS: $48.200; S: $57.800" | "Precio: $40.700" | "Paquete de 24 huesos: $ 69.700"
  const m = /Precios?\s*:\s*([\s\S]*)$/.exec(block(line).join(" ")); if (!m) return [];
  const pairs = [...m[1].matchAll(/([A-Za-zÁÉÍÓÚáéíóúñ][\wÁÉÍÓÚáéíóúñ ]*?)\s*:\s*\$\s?([\d.]+)/g)].map(r=>({label:r[1].trim(),price:+r[2].replace(/\./g,"")}));
  if (pairs.length) return pairs;
  const one = /\$\s?([\d.]+)/.exec(m[1]); return one ? [{label:null,price:+one[1].replace(/\./g,"")}] : [];
}
const EXISTING = { 160: ["arnes-honey-azul","arnes-honey-rojo", /*…uno por color*/] };   // línea -> slugs
const STOCK = { 228: "out", 160: ["XS","S","L"], 12: "skip" };                           // disponibilidad
const NEW = [ P(slug, nombre, línea, categoría, [fotos], { label, variants, hidden, out }) ];
```
SQL generado (todo idempotente, dentro de `begin; … commit;`):
- Existentes: `update products set description, dimensions where slug in (…)`; precios con etiqueta `… and split_part(size_label,' (',1) = 'L'` (así "L (56 x 26 cm)" también coincide); sin etiqueta → todas las tallas.
- Stock: `"out"` → `in_stock=false`; lista → `in_stock = split_part(size_label,' (',1) = any(array[…])`; sin entrada → `true`; `"skip"` → no tocar (contradictorio, se reporta).
- Nuevos: `insert … on conflict (slug) do nothing`, tallas y fotos con `where not exists`. `display_order` negativo para que salgan primero. Sin foto → `is_active=false`.
- Categorías nuevas (p. ej. "Pelotas") con `on conflict (slug) do nothing`.
- Cuidado con slugs que ya existen por productos de prueba del cliente (usa otro slug).

Test (`tests/catalogo.test.mjs`): precios de 4–5 fichas raras, que cada línea del mapeo sea un título, slugs únicos, visibles con foto, sin `undefined`/`NaN` en el SQL y que cada `/imagenes/…webp` referenciado exista.

## 6. Después de que el usuario corra el SQL

```bash
c(){ curl -s -o /dev/null -D - "$U/$1" -H "apikey: $K" -H "Prefer: count=exact" -H "Range: 0-0" | grep -i content-range; }
c 'products?select=id'; c 'products?select=id&is_active=eq.true'; c 'product_images?select=id'
```
Y revisa 3 o 4 productos concretos (un precio nuevo, una talla agotada, un producto nuevo con sus fotos). Reporta los ambiguos que dejaste sin tocar.
