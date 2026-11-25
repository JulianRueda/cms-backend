# cms-backend: skill de Claude Code

Skill `supabase-cms-backend`: convierte un frontend existente (Figma Make, Lovable, HTML/JS, React o Vite) en una tienda o sitio autoadministrable con **Supabase** (Auth + PostgreSQL con RLS + Storage) y un panel `/admin` en español para productos, tallas con precio, fotos, categorías y textos.

## Instalar

```powershell
git clone https://github.com/JulianRueda/cms-backend.git "$HOME\.claude\skills\supabase-cms-backend"
```

En macOS o Linux: `git clone https://github.com/JulianRueda/cms-backend.git ~/.claude/skills/supabase-cms-backend`

Luego, en Claude Code escribe `/supabase-cms-backend`, o pide algo como "hazle el backend con Supabase para que el cliente edite el catálogo".

## Contenido

- `SKILL.md`: el flujo completo (análisis, modelo de datos, seguridad, panel, verificación, entrega) y los errores conocidos.
- `references/schema-template.sql`: tablas, RLS, GRANTs y bucket de fotos, probados en producción.
- `references/admin-patterns.md`: código del panel (sesión, fotos, guardado seguro, orden, mock de pruebas).

Para actualizar la skill: `git pull` dentro de la carpeta.
