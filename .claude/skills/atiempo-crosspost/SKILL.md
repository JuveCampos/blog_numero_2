---
name: atiempo-crosspost
description: >
  Detecta artículos nuevos de Juvenal en ATiempo.TV y crea sus cross-posts
  en posts/ siguiendo la plantilla estandarizada del blog. Usar cuando el
  usuario pida "agregar/reincorporar artículos de ATiempo", "sincronizar con
  ATiempo" o mencione que hay artículos nuevos de su autoría en ATiempo.TV
  u otro medio externo (Grupo Animal, México Cómo Vamos, etc.).
---

# Cross-posts de ATiempo.TV

Sincroniza el blog con los artículos de Jorge Juvenal Campos Ferreira
publicados en https://atiempo.tv/author/juvenal-campos/.

## Paso 1 — Detectar artículos faltantes

```bash
# URLs ya cross-posteadas en el blog
grep -rhoE 'https://atiempo\.tv/[^)" ]+' posts/*/index.qmd \
  | grep -viE "wp-content|author/" | sed 's/{.*//;s|/*$|/|' | sort -u \
  > /tmp/existing_urls.txt

# URLs en la página de autor (revisar página 1; si TODAS son nuevas,
# seguir con /page/2/, /page/3/... hasta encontrar solo conocidas)
curl -sL "https://atiempo.tv/author/juvenal-campos/" \
  | grep -oE 'https://atiempo\.tv/[a-z0-9/-]+/' \
  | grep -vE "author|category|/page/|tag|wp-|/feed|contacto|aviso|quienes|attachment|investigaciones|nosotros|noticias|politicas|slogan|semanario/$|columnista/$|nacional/$|apuntes-de-datos[a-z-]*/$|desarrollo-de-talento[a-z-]*/$" \
  | sort -u > /tmp/author_urls.txt

comm -23 /tmp/author_urls.txt /tmp/existing_urls.txt
```

## Paso 2 — Extraer metadatos de cada artículo nuevo

Descargar el HTML con `curl -sL` y extraer:

- **Título**: del `<h1>` de la página (NO del `og:title`, que agrega
  "| A Tiempo Medio de Comunicación").
- **Fecha**: `<meta property="article:published_time">` (tomar YYYY-MM-DD).
- **Imagen**: `<meta property="og:image">`. Convertir la URL directa
  `https://atiempo.tv/wp-content/...` al proxy con resize:
  `https://i0.wp.com/atiempo.tv/wp-content/...?resize=1000%2C563&ssl=1`
- **Lead**: primer `<p>` sustantivo (>100 caracteres) del contenido del
  artículo (`entry-content`), con etiquetas HTML removidas y entidades
  decodificadas. Copiarlo TAL CUAL, sin reescribir.

## Paso 3 — Crear el post

Carpeta: `posts/YYYY-MM-DD-<slug-del-titulo>/index.qmd` (fecha de
publicación en ATiempo; slug en minúsculas, sin tildes, con guiones).

Plantilla exacta (referencia canónica:
`posts/2026-02-03-mi-trabajo-esta-en-peligro-por-la-ia/index.qmd`):

```markdown
---
title: "<título exacto del H1>"
author: "Jorge Juvenal Campos Ferreira"
date: "YYYY-MM-DD"
categories:
  - ATiempo.TV
  - Periodismo de datos
image: "<URL i0.wp.com con ?resize=1000%2C563&ssl=1>"
description: "<primeros ~200 caracteres del lead, truncados con '...'>"
---

![<título>](<misma URL de image>)

<lead completo tal cual aparece en ATiempo>

---

[**Continúa leyendo en ATiempo.TV →**](<URL del artículo>){target="_blank" rel="noopener noreferrer"}
```

Reglas:

- NO inventar ni parafrasear texto: título y lead se copian textuales.
- Las categorías son exactamente `ATiempo.TV` y `Periodismo de datos`
  (son chips visibles en el listado del blog).
- La descripción termina con `...` si se truncó.

## Otros medios (Grupo Animal, etc.)

La misma plantilla aplica para artículos publicados en otros medios; solo
cambia:

- **Categorías**: el nombre del medio reemplaza a `ATiempo.TV` (ej.
  `Grupo Animal`) + `Periodismo de datos`. Ojo: una categoría nueva se
  vuelve chip de filtro visible en /posts.html inmediatamente.
- **Enlace final**: `[**Continúa leyendo en <Medio> →**](<URL>)...`
- **Imagen**: si el `og:image` del medio es una tarjeta genérica/vieja
  (pasó con Grupo Animal: card de 2022 en artículo de 2026), usar la
  primera figura del cuerpo del artículo en su lugar.
- El usuario normalmente proporciona la URL directa (no hay página de
  autor que rastrear); referencia:
  `posts/2026-05-19-magisterio-mexicano-mercado-laboral/index.qmd`.

## Paso 4 — Renderizar y verificar

```bash
quarto render posts/<carpeta-nueva>/index.qmd   # cada post nuevo
quarto render index.qmd                          # listado de la portada
quarto render posts.qmd                          # listado del blog
```

Verificar que las cards aparecen en `_site/posts.html` con imagen, título
y descripción.

## Paso 5 — Limpiar efectos colaterales del render y commitear

El render puede borrar HTML viejos rastreados dentro de `posts/` y dejar
artefactos en la raíz (`about.html`, `posts.html`, `gracias.html`,
`posts-listing.json`):

```bash
# Restaurar borrados accidentales
git status --short | grep "^ D" | grep -v "_site/" | sed 's/^ D //' \
  | while IFS= read -r f; do git checkout -- "$f"; done
# Eliminar artefactos de raíz
rm -f about.html gracias.html posts.html posts-listing.json
```

Commit: stagear SOLO los `posts/*/index.qmd` nuevos (nunca `_site/` ni
`.DS_Store`). Hacer `git pull` antes de `git push` (Juvenal también
commitea desde otra máquina). El push a `main` dispara el deploy en
Netlify automáticamente.
