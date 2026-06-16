---
on:
  push:
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
tools:
  github:
    toolsets: [default]
  bash: true
safe-outputs:
  create-pull-request:
    title-prefix: "📄 [docs-draft] "
    branch-prefix: "docs-draft/"
    preserve-branch-name: true
    draft: false
    allowed-base-branches:
      - main
      - master
    allowed-files:
      - "docs-drafts/**"
    expires: 30
---

# auto-release-notes-draft

Actúa como **Technical Writer y Arquitecto de Documentación**. Analiza los commits del push actual y genera un **borrador de documentación** en markdown para revisión humana, antes de publicar nada en Confluence.

**⚠️ REGLA ABSOLUTA: No uses Confluence ni ningún MCP externo en este workflow. Solo crea el archivo markdown local y el PR.**

## Paso 0: Leer configuración del entorno

```bash
REPO_NAME=$(echo "$GITHUB_REPOSITORY" | cut -d'/' -f2)
SHA_SHORT=$(echo "$GITHUB_SHA" | cut -c1-7)
TODAY=$(date -u +%Y-%m-%d)
echo "GITHUB_REPOSITORY=$GITHUB_REPOSITORY"
echo "GITHUB_REF_NAME=$GITHUB_REF_NAME"
echo "GITHUB_SHA=$GITHUB_SHA"
echo "REPO_NAME=$REPO_NAME | SHA_SHORT=$SHA_SHORT | TODAY=$TODAY"
```

Usa estos valores en todos los pasos siguientes.

## Paso 1: Verificar que no existe un borrador para este SHA

```bash
SHA_SHORT=$(echo "$GITHUB_SHA" | cut -c1-7)
EXISTING=$(gh pr list --head "docs-draft/$SHA_SHORT" --state open --json number,title 2>/dev/null)
echo "PRs existentes para docs-draft/$SHA_SHORT: $EXISTING"
```

Si ya existe un PR abierto para `docs-draft/$SHA_SHORT`, **detente aquí** sin crear duplicado.

## Paso 2: Analizar los cambios del push

```bash
echo "=== Commits recientes ==="
git log --oneline -20

echo "=== Estadísticas del commit actual ==="
git diff HEAD~1 HEAD --stat 2>/dev/null || echo "[Primer commit — análisis de estructura del repositorio]"

echo "=== Diff detallado ==="
git diff HEAD~1 HEAD 2>/dev/null || find . -type f -not -path './.git/*' -not -path './docs-drafts/*' | sort | head -100

echo "=== Metadata de commits (SHA|autor|mensaje) ==="
git log --format="%H|%an|%s" -10
```

Para cada commit, extrae:
- SHA completo y corto (7 chars), tipo (`feat`/`fix`/`chore`/`refactor`/`docs`/`ci`), autor, mensaje
- Archivos modificados y su propósito en el proyecto
- **Impacto en documentación**: qué sección de docs se ve afectada y por qué razón

**Condición de parada**: Si el push solo modifica archivos en `docs-drafts/` o configuración interna (`.github/`, `.gitignore`, `.editorconfig`, linters) sin impacto documentable para usuarios del proyecto, finaliza sin crear PR e indica el motivo.

## Paso 3: Crear el archivo de borrador

Genera el contenido completo del borrador y escríbelo en: `docs-drafts/{TODAY}-{REPO_NAME}-{SHA_SHORT}.md`

El archivo debe seguir esta estructura:

```
---
generated_at: {TIMESTAMP_ISO8601}
repository: {GITHUB_REPOSITORY}
trigger_commit: {GITHUB_SHA}
branch: {GITHUB_REF_NAME}
status: pending-review
---

# Borrador de Documentación: {REPO_NAME}

> ⚠️ Requiere revisión humana antes de publicarse en Confluence.
> Edita si es necesario, luego aprueba este PR para publicar automáticamente.

## ¿Por qué se generó este borrador?

{Explicación de qué cambió y por qué justifica actualizar la documentación}
Commit disparador: [{SHA_SHORT}](https://github.com/{GITHUB_REPOSITORY}/commit/{GITHUB_SHA})

## Commits que originan esta actualización

| SHA | Tipo | Mensaje | Autor | Impacto en documentación |
|-----|------|---------|-------|--------------------------|
| [{sha_corto}](https://github.com/{GITHUB_REPOSITORY}/commit/{sha_completo}) | {tipo} | {mensaje} | {autor} | {impacto específico en docs} |

## Cambios propuestos en Confluence

### 📝 Release Notes / Changelog

{Texto real de changelog basado en los commits: qué cambió, por qué importa, impacto para usuarios}

### [Solo si aplica] 🔌 API / Interfaces

{Si hay nuevos endpoints, cambios de contrato o interfaces públicas}

### [Solo si aplica] 🏗️ Arquitectura / DevOps

{Si hay cambios en CI/CD, Dockerfile, infraestructura, configuración}

### [Solo si aplica] 📚 Onboarding / Getting Started

{Si hay cambios en README u otra documentación de inicio}

---

## Instrucciones para el revisor

1. Revisa el contenido propuesto en las secciones anteriores
2. Edita este archivo si alguna descripción es incorrecta o incompleta
3. **Mergea este PR** para publicar automáticamente en Confluence
4. O **cierra el PR sin mergear** para descartar esta actualización

> 🤖 Generado por `auto-release-notes-draft`
```

Crea el directorio y escribe el archivo completo (sin placeholders — omite las secciones que no apliquen):

```bash
mkdir -p docs-drafts
REPO_NAME=$(echo "$GITHUB_REPOSITORY" | cut -d'/' -f2)
SHA_SHORT=$(echo "$GITHUB_SHA" | cut -c1-7)
TODAY=$(date -u +%Y-%m-%d)
FILENAME="docs-drafts/${TODAY}-${REPO_NAME}-${SHA_SHORT}.md"
echo "Escribiendo en: $FILENAME"
# Usa Python o heredoc para escribir el contenido completo generado
```

Verifica que el archivo se creó correctamente:

```bash
cat "$FILENAME"
```

## Paso 4: Crear el Pull Request

Usa la herramienta `create_pull_request` con estos campos:

- **`branch`**: `{SHA_SHORT}` (el prefijo `docs-draft/` se añade automáticamente)
- **`base`**: el valor de `$GITHUB_REF_NAME` (será `main` o `master`)
- **`title`**: descripción concisa de los cambios documentados (el prefijo `📄 [docs-draft]` se añade automáticamente)
- **`body`**: descripción del PR con la tabla de commits e instrucciones de revisión

El cuerpo del PR debe incluir:

```markdown
## 📋 Borrador de documentación — revisión requerida

Cambios detectados en `{GITHUB_REF_NAME}` tras el commit [`{SHA_SHORT}`](https://github.com/{GITHUB_REPOSITORY}/commit/{GITHUB_SHA}).

### Commits incluidos

| SHA | Tipo | Mensaje | Autor |
|-----|------|---------|-------|
| [`{sha}`](link) | {tipo} | {mensaje} | {autor} |

### Cómo proceder

| Acción | Resultado |
|--------|-----------|
| ✅ **Mergear** | Publica automáticamente en Confluence |
| ✏️ **Editar y mergear** | Publica la versión revisada |
| ❌ **Cerrar sin mergear** | Descarta esta actualización |

> 🤖 Generado por `auto-release-notes-draft`
```

## Reglas

1. **No uses Confluence** en ningún momento de este workflow.
2. **No incluyas secretos, tokens ni passwords** en el contenido del borrador.
3. **No crees PR duplicados**: si ya existe uno para este SHA (Paso 1), detente.
4. **No crees PR** si el push no tiene impacto documentable (solo `docs-drafts/`, `.github/`, configs).
5. **El borrador debe ser concreto**: contenido real del análisis, sin placeholders ni textos genéricos.
