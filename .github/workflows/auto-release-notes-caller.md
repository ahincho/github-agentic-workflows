---
on:
  push:
    branches: [main]
permissions:
  contents: read
  issues: read
  pull-requests: read
engine: copilot
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
      Authorization: "Bearer ${{ secrets.ATLASSIAN_ROVO_MCP_TOKEN }}"
    allowed:
      - getConfluencePage
      - searchConfluenceUsingCql
      - getConfluenceSpaces
      - getPagesInConfluenceSpace
      - getConfluencePageDescendants
      - createConfluencePage
      - updateConfluencePage
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

# auto-release-notes-caller

Actúa como Technical Writer experto. Mantén la documentación del proyecto actualizada en Confluence como fuente de conocimiento centralizada.

## 1. Leer documentación existente

Antes de analizar el código, obtén el nombre del repositorio y lee el estado actual en Confluence:

```bash
# El nombre del repositorio está disponible como env var nativa de GitHub Actions
echo "Repo: $GITHUB_REPOSITORY"
# Formato: owner/nombre-repo → extrae solo el nombre:
echo "$GITHUB_REPOSITORY" | cut -d'/' -f2
```

- Usa `getConfluenceSpaces` para descubrir los espacios disponibles e identificar el espacio destino.
- Usa `searchConfluenceUsingCql` con la query:
  `title ~ "[nombre-del-repo]" AND type = page AND space.type = "global"`
  donde `nombre-del-repo` es el valor extraído de `$GITHUB_REPOSITORY`.
- Para cada página encontrada, usa `getConfluencePage` para leer su contenido actual.
- Identifica qué páginas ya existen: Release Notes, Arquitectura, API, Onboarding.

## 2. Analizar cambios

```bash
git log --oneline -20
git diff HEAD~1 HEAD --stat 2>/dev/null || find . -type f -not -path './.git/*' | sort | head -100
git diff HEAD~1 HEAD 2>/dev/null
```

## 3. Decidir qué actualizar

| Tipo de cambio | Página a actualizar |
|----------------|---------------------|
| Código fuente, features, fixes | Release Notes / Changelog |
| Nuevos endpoints o contratos | API / Interfaces |
| README, docs/ | Onboarding / Getting Started |
| CI/CD, infra, Dockerfile | Arquitectura / DevOps |

**Si la página existe:** léela, actualiza solo las secciones afectadas, preserva el resto.  
**Si no existe:** créala completa con estructura base.

## 4. Publicar en Confluence

- Página nueva → `createConfluencePage` con `spaceId`, `title` (`[repo] Tipo`), y `body` en Confluence Storage Format (XHTML).
- Página existente → `updateConfluencePage` con `pageId`, `body` actualizado y `version + 1`.

## Reglas

- No incluyas información sensible (tokens, passwords, secrets).
- Preserva el historial de cambios: agrega nuevas entradas al inicio, no borres las existentes.
- Títulos con prefijo `[nombre-repo]`.
- Fechas en ISO 8601 (`YYYY-MM-DD HH:mm UTC`).
- Omite secciones sin cambios.
- Si el MCP falla, crea un issue de GitHub con el contenido para revisión manual.
