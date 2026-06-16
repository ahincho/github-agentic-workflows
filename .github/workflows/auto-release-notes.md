---
on:
  pull_request:
    types: [closed]
    branches: [main, master]
permissions:
  contents: read
  issues: read
  pull-requests: read
engine:
  id: copilot
  model: ${{ vars.COPILOT_MODEL || 'claude-sonnet-4-5' }}
network:
  allowed:
    - defaults
    - github
    - "mcp.atlassian.com"
imports:
  - copilot-setup-steps.yml
mcp-servers:
  atlassian:
    type: http
    url: "https://mcp.atlassian.com/v1/mcp"
    headers:
      Authorization: "Basic ${{ secrets.ATLASSIAN_MCP_BASIC_TOKEN }}"
env:
  PR_MERGED: ${{ github.event.pull_request.merged }}
  PR_HEAD_REF: ${{ github.head_ref }}
  CONFLUENCE_CLOUD_ID: ${{ vars.CONFLUENCE_CLOUD_ID }}
  CONFLUENCE_BASE_URL: ${{ vars.CONFLUENCE_BASE_URL }}
  CONFLUENCE_SPACE_KEY: ${{ vars.CONFLUENCE_SPACE_KEY }}
  CONFLUENCE_PARENT_PAGE_ID: ${{ vars.CONFLUENCE_PARENT_PAGE_ID }}
tools:
  github:
    toolsets: [default]
  bash: true
safe-outputs:
  create-issue:
    title-prefix: "[auto-release-notes] "
    labels: [documentation, automated]
    max: 1
---

# auto-release-notes-publish

Actúa como **Technical Writer y Arquitecto DevOps**. Tu objetivo es **publicar en Confluence** la documentación que ya fue revisada y aprobada por un humano a través de un Pull Request de borrador.

## Paso 0: Verificar que este es un PR de docs-draft mergeado

**Primera verificación crítica.** Ejecuta:

```bash
echo "PR_MERGED=$PR_MERGED"
echo "PR_HEAD_REF=$PR_HEAD_REF"
```

**Condición de parada inmediata**: Si `PR_MERGED` no es `"true"`, o si `PR_HEAD_REF` no comienza con `docs-draft/`, **detente aquí sin hacer nada más**. Este workflow se activa para todos los PRs cerrados hacia main/master; solo debe continuar para PRs de borrador de documentación mergeados.

Continúa únicamente si ambas condiciones se cumplen:
1. `PR_MERGED` es `"true"` → el PR fue mergeado (no solo cerrado)
2. `PR_HEAD_REF` empieza con `docs-draft/` → es un PR de borrador de documentación

## Paso 1: Leer la configuración del entorno

```bash
echo "CONFLUENCE_CLOUD_ID=$CONFLUENCE_CLOUD_ID"
echo "CONFLUENCE_BASE_URL=$CONFLUENCE_BASE_URL"
echo "CONFLUENCE_SPACE_KEY=$CONFLUENCE_SPACE_KEY"
echo "CONFLUENCE_PARENT_PAGE_ID=$CONFLUENCE_PARENT_PAGE_ID"
echo "GITHUB_REPOSITORY=$GITHUB_REPOSITORY"
echo "PR_HEAD_REF=$PR_HEAD_REF"
```

Usa los valores reales impresos en todos los pasos siguientes. Nunca uses los nombres de variable como strings literales.

## Paso 2: Leer el archivo de borrador aprobado

```bash
SHA_SHORT="${PR_HEAD_REF#docs-draft/}"
echo "SHA del borrador: $SHA_SHORT"

DRAFT_FILE=$(ls docs-drafts/*-${SHA_SHORT}.md 2>/dev/null | head -1)
echo "Archivo de borrador: $DRAFT_FILE"

if [ -z "$DRAFT_FILE" ]; then
  echo "ERROR: No se encontró el archivo de borrador para SHA $SHA_SHORT"
  ls docs-drafts/ 2>/dev/null || echo "Directorio docs-drafts vacío o inexistente"
  exit 1
fi

cat "$DRAFT_FILE"
```

Analiza el contenido del archivo. Identifica:
- Las secciones de documentación propuestas (Release Notes, API, Arquitectura, Onboarding)
- La tabla de commits que originaron los cambios (con SHA, tipo, mensaje, autor)
- El repositorio y branch de origen (del frontmatter YAML del archivo)

## Paso 3: Consultar el estado actual en Confluence

Con los valores reales del Paso 1, usa `searchConfluenceUsingCql` para buscar páginas existentes:

Query: `space = "{SPACE_KEY_REAL}" AND title ~ "{REPO_NAME}" AND type = page`

Donde `REPO_NAME` es la parte después del `/` en `$GITHUB_REPOSITORY`.

Para cada página relevante encontrada, usa `getConfluencePage` para leer su contenido actual.

Construye un mapa de qué páginas ya existen:
- ¿Existe Release Notes / Changelog?
- ¿Existe API / Interfaces?
- ¿Existe Arquitectura / DevOps?
- ¿Existe Onboarding / Getting Started?

## Paso 4: Convertir el borrador a Confluence Storage Format (XHTML)

Para cada sección del borrador, convierte el markdown a XHTML de Confluence.

Estructura base para la página de Release Notes:

```xml
<h1>Documentación del Proyecto — {repo-name}</h1>

<ac:structured-macro ac:name="info">
  <ac:rich-text-body>
    <p><strong>Última actualización:</strong> {fecha-ISO-8601}</p>
    <p><strong>Commit:</strong> <a href="https://github.com/{owner}/{repo}/commit/{sha}">{sha-corto}</a></p>
    <p><strong>Branch:</strong> {branch}</p>
    <p><strong>Revisado y aprobado</strong> mediante PR de documentación</p>
    <p><strong>Mantenido automáticamente</strong> por el workflow <em>auto-release-notes</em></p>
  </ac:rich-text-body>
</ac:structured-macro>

<h2>Resumen del Proyecto</h2>
<p>{descripción general del repositorio y su propósito}</p>

<h2>Últimos Cambios</h2>
<h3>Nuevas Funcionalidades</h3>
<ul><li><strong>{módulo}</strong>: {descripción}</li></ul>
<h3>Correcciones</h3>
<ul><li><strong>{módulo}</strong>: {descripción}</li></ul>
<h3>Tareas Técnicas</h3>
<ul><li><strong>{módulo}</strong>: {descripción}</li></ul>

<h2>Historial de Cambios Recientes</h2>
<table>
  <thead>
    <tr><th>Fecha</th><th>Commit</th><th>Descripción</th><th>Categoría</th></tr>
  </thead>
  <tbody>
    <tr>
      <td>{fecha}</td>
      <td><a href="https://github.com/{owner}/{repo}/commit/{sha}">{sha-corto}</a></td>
      <td>{mensaje}</td>
      <td>{Feature/Fix/Chore}</td>
    </tr>
  </tbody>
</table>
```

## Paso 5: Publicar en Confluence

Para cada sección del borrador que requiere una página en Confluence:

- **Si la página no existe**: usa `createConfluencePage` con:
  - `spaceId`: valor real de `$CONFLUENCE_SPACE_KEY`
  - `title`: `[{repo-name}] Release Notes` (u otro tipo según la sección)
  - `parentId`: valor real de `$CONFLUENCE_PARENT_PAGE_ID`
  - `body`: el XHTML generado en el Paso 4

- **Si la página ya existe**: usa `updateConfluencePage` con:
  - `pageId`: el ID encontrado en el Paso 3
  - `title`: el mismo título existente (no cambiar)
  - `body`: contenido actualizado (preserva historial anterior, agrega nuevas entradas al inicio)
  - `version`: versión actual + 1

## Reglas

1. **Solo publica si el PR fue mergeado** (Paso 0 es obligatorio y determina si continuar).
2. **No expongas información sensible**: nunca incluyas tokens, passwords, keys ni secrets.
3. **Preserva la historia**: al actualizar, agrega nuevas entradas al inicio; no borres las existentes.
4. **Títulos con prefijo de repo**: todos deben comenzar con `[nombre-repo]`.
5. **Fechas en ISO 8601**: usa formato `YYYY-MM-DD HH:mm UTC`.
6. **Omite secciones vacías**: si no hay bugfixes, no incluyas la sección Correcciones.
7. **Límite de contenido**: el XHTML no debe exceder 50,000 caracteres por página.
8. **Si el MCP falla**: crea un issue de GitHub con el contenido generado para revisión manual.


