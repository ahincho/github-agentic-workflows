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
  model: ${{ vars.COPILOT_MODEL || 'claude-sonnet-4.5' }}
network:
  allowed:
    - defaults
    - github
tools:
  github:
    mode: gh-proxy
  bash: true
steps:
  - name: Pre-fetch commit data
    env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    run: |
      mkdir -p /tmp/gh-aw/data
      REPO_NAME=$(echo "$GITHUB_REPOSITORY" | cut -d'/' -f2)
      SHA_SHORT=$(echo "$GITHUB_SHA" | cut -c1-7)
      TODAY=$(date -u +%Y-%m-%d)
      TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

      # Commits recientes con metadata estructurada
      git log --format='{"sha":"%H","sha_short":"%h","author":"%an","date":"%ai","subject":"%s"}' -20 \
        | jq -s '.' > /tmp/gh-aw/data/commits.json

      # Estadísticas del diff actual
      git diff HEAD~1 HEAD --stat 2>/dev/null \
        | tail -1 > /tmp/gh-aw/data/diff_stat.txt \
        || echo "first-commit" > /tmp/gh-aw/data/diff_stat.txt

      # Diff completo (limitado a 8000 chars para no saturar contexto)
      git diff HEAD~1 HEAD 2>/dev/null \
        | head -c 8000 > /tmp/gh-aw/data/diff.patch \
        || find . -type f -not -path './.git/*' -not -path './docs-drafts/*' \
             | sort | head -100 > /tmp/gh-aw/data/diff.patch

      # Verificar si ya existe un PR de borrador para este SHA
      EXISTING_PR=$(gh pr list \
        --head "docs-draft/$SHA_SHORT" \
        --state open \
        --json number,title \
        --repo "$GITHUB_REPOSITORY" 2>/dev/null || echo "[]")

      # Metadata del contexto
      jq -n \
        --arg repo "$GITHUB_REPOSITORY" \
        --arg repo_name "$REPO_NAME" \
        --arg sha "$GITHUB_SHA" \
        --arg sha_short "$SHA_SHORT" \
        --arg branch "$GITHUB_REF_NAME" \
        --arg today "$TODAY" \
        --arg timestamp "$TIMESTAMP" \
        --argjson existing_prs "$EXISTING_PR" \
        '{
          repository: $repo,
          repo_name: $repo_name,
          sha: $sha,
          sha_short: $sha_short,
          branch: $branch,
          today: $today,
          timestamp: $timestamp,
          draft_filename: ("docs-drafts/" + $today + "-" + $repo_name + "-" + $sha_short + ".md"),
          pr_branch: $sha_short,
          existing_draft_prs: $existing_prs
        }' > /tmp/gh-aw/data/context.json

      echo "=== Context ==="
      cat /tmp/gh-aw/data/context.json
      echo "=== Commits (top 5) ==="
      jq '.[0:5]' /tmp/gh-aw/data/commits.json
      echo "=== Diff stat ==="
      cat /tmp/gh-aw/data/diff_stat.txt
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
      - "README.md"
    expires: 30
---

# auto-release-notes-draft

Actúa como **Technical Writer y Arquitecto de Documentación**. Genera un **borrador de documentación** en markdown para revisión humana, antes de publicar en Confluence.

**⚠️ REGLAS ABSOLUTAS:**
- No uses Confluence ni ningún MCP externo.
- No corras `git log` ni `git diff` — los datos ya están pre-cargados en `/tmp/gh-aw/data/`.

## Paso 0: Leer los datos pre-cargados

```bash
cat /tmp/gh-aw/data/context.json
cat /tmp/gh-aw/data/commits.json
cat /tmp/gh-aw/data/diff_stat.txt
cat /tmp/gh-aw/data/diff.patch
```

Del archivo `context.json` extrae y usa siempre:
- `repo_name`, `sha`, `sha_short`, `branch`, `today`, `timestamp`
- `draft_filename` — ruta exacta donde escribir el borrador
- `pr_branch` — nombre del branch para el PR (sin el prefijo `docs-draft/`)
- `existing_draft_prs` — si tiene elementos, **detente aquí sin crear nada** (ya existe un PR para este commit)

## Paso 1: Evaluar si el push merece documentación

Analiza `commits.json` y `diff.patch`. **Detente sin crear PR** si el push solo toca:
- Archivos en `docs-drafts/`
- Archivos de configuración interna (`.github/`, `.gitignore`, `.editorconfig`, linters, lock files)

Si hay cambios documentables, continúa.

## Paso 2: Actualizar README.md (si aplica)

Si los commits incluyen cambios en la estructura del proyecto, nuevos módulos, comandos de instalación/uso o cualquier información relevante para un desarrollador nuevo, actualiza `README.md` en el directorio raíz con esa información. El README debe mantenerse al día con el estado real del proyecto.

Si no hay cambios que justifiquen actualizar el README, omite este paso.

## Paso 3: Crear el archivo de borrador

Escribe el archivo en la ruta exacta de `draft_filename`. Usa Python para escribirlo:

```bash
python3 - <<'PYEOF'
import json, sys
ctx = json.load(open('/tmp/gh-aw/data/context.json'))
# escribe el contenido markdown completo en ctx['draft_filename']
PYEOF
```

O usa un heredoc bash. El archivo debe tener esta estructura (sin placeholders):

```markdown
---
generated_at: {timestamp}
repository: {repository}
trigger_commit: {sha}
branch: {branch}
status: pending-review
---

# Borrador: {repo_name}

> ⚠️ Requiere revisión humana. Edita si es necesario, luego mergea para publicar en Confluence.

## ¿Por qué se generó este borrador?

{Explica qué cambió y por qué justifica actualizar la documentación.}
Commit: [{sha_short}](https://github.com/{repository}/commit/{sha})

## Commits que originan esta actualización

| SHA | Tipo | Mensaje | Autor |
|-----|------|---------|-------|
| [`sha_short`](https://github.com/{repo}/commit/{sha}) | feat/fix/chore | mensaje | autor |

## Cambios propuestos

### 📝 Release Notes
{Changelog concreto basado en los commits. Omite si no aplica.}

### 🔌 API / Interfaces
{Solo si hay nuevos endpoints o contratos. Omite si no aplica.}

### 🏗️ Arquitectura / DevOps
{Solo si hay cambios en CI/CD, Dockerfile, infra. Omite si no aplica.}

### 📚 Onboarding / Getting Started
{Solo si hay cambios en README u onboarding. Omite si no aplica.}

---
> 🤖 Generado por `auto-release-notes-draft`
```

Verifica que el archivo se creó:

```bash
cat {draft_filename}
```

## Paso 4: Crear el Pull Request

Usa `create_pull_request` con:
- `branch`: el valor de `pr_branch` del context.json
- `base`: el valor de `branch` del context.json (`main` o `master`)
- `title`: descripción concisa (el prefijo `📄 [docs-draft]` se añade automáticamente)
- `body`: tabla de commits + instrucciones de revisión (mergear = publicar en Confluence, cerrar = descartar)

## Reglas finales

1. No uses Confluence ni ningún MCP externo.
2. No incluyas secretos ni tokens en el borrador.
3. Si `existing_draft_prs` tiene elementos en el Paso 0, detente sin crear nada.
4. Si el push no tiene impacto documentable, usa `noop`.
5. Todo el contenido debe ser concreto — sin placeholders ni textos genéricos.
