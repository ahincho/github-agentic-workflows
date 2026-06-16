---
generated_at: 2026-06-16T20:50:46Z
repository: ahincho/github-agentic-workflows
trigger_commit: 93244988742c4ca5491fde10d144d030f840625f
branch: main
status: pending-review
---

# Borrador: github-agentic-workflows

> ⚠️ Requiere revisión humana. Edita si es necesario, luego mergea para publicar en Confluence.

## ¿Por qué se generó este borrador?

Este borrador documenta una actualización mayor del README que introduce la arquitectura completa del pipeline Human-in-the-Loop (HITL) para documentación automatizada. El cambio incluye diagramas de flujo, stack técnico, estructura del proyecto, requisitos previos y guía de setup completa.

Commit: [9324498](https://github.com/ahincho/github-agentic-workflows/commit/93244988742c4ca5491fde10d144d030f840625f)

## Commits que originan esta actualización

| SHA | Tipo | Mensaje | Autor |
|-----|------|---------|-------|
| [`9324498`](https://github.com/ahincho/github-agentic-workflows/commit/93244988742c4ca5491fde10d144d030f840625f) | docs | docs: update README with HITL pipeline architecture and setup guide | ahincho |

## Cambios propuestos

### 📝 Release Notes

**Versión:** 2026-06-16  
**Commit:** 9324498

#### Documentación

- ✨ **Arquitectura HITL documentada** — Agregado diagrama completo del pipeline de 2 fases (draft → review → publish)
- 📚 **Stack técnico completo** — Tabla con todas las tecnologías: NestJS, Claude Sonnet 4.5, GitHub Agentic Workflows, Atlassian Rovo MCP
- 🏗️ **Estructura del proyecto** — Árbol de directorios con descripción de cada archivo clave
- 🔧 **Guía de configuración** — Secrets, variables, permisos de GitHub Actions y setup de Atlassian Rovo MCP
- 🚀 **Comandos de desarrollo** — npm scripts para desarrollo local y compilación de workflows agénticos
- 📖 **Flujo de trabajo explicado** — Paso a paso del pipeline desde push hasta publicación en Confluence

### 📚 Onboarding / Getting Started

**Actualización del README principal**

El README ahora sirve como documentación completa del proyecto, incluyendo:

1. **Arquitectura del Pipeline**
   - Diagrama ASCII del flujo de 2 fases
   - Descripción de cada fase y sus responsabilidades
   - Páginas que mantiene en Confluence (espacio PLA)

2. **Stack Técnico**
   - Framework: NestJS v11 (TypeScript)
   - Runtime: Node.js
   - CI/CD: GitHub Agentic Workflows (`gh aw`)
   - Modelo LLM: Claude Sonnet 4.5
   - Documentación: Confluence Cloud (Atlassian Rovo MCP)

3. **Requisitos Previos**
   - **Secrets requeridos:**
     - `COPILOT_GITHUB_TOKEN` — Token de GitHub Copilot
     - `ATLASSIAN_MCP_BASIC_TOKEN` — Credenciales de Atlassian en base64
   
   - **Variables del repositorio:**
     - `COPILOT_MODEL` — Modelo LLM a usar
     - `CONFLUENCE_BASE_URL` — URL de Atlassian
     - `CONFLUENCE_CLOUD_ID` — UUID del tenant
     - `CONFLUENCE_SPACE_KEY` — Espacio de Confluence (ej. PLA)
     - `CONFLUENCE_PARENT_PAGE_ID` — ID de página padre
   
   - **Permisos de GitHub Actions:**
     - Read and write permissions
     - Create and approve pull requests

4. **Desarrollo Local**
   ```bash
   npm install
   npm run start:dev
   npm run test
   gh aw compile .github/workflows/auto-release-notes-caller.md
   ```

5. **Flujo de Trabajo Completo**
   - Push a main → DataOps pre-fetch (git log/diff)
   - Agente evalúa cambios y genera borrador
   - PR automático para revisión humana
   - Merge → publicación en Confluence vía MCP

### 🏗️ Arquitectura / DevOps

**Optimización DataOps**

El pipeline implementa una optimización clave: los datos de git (`git log`, `git diff`) y GitHub (`gh pr list`) se obtienen **fuera del sandbox LLM** y se pre-cargan en formato JSON compacto en `/tmp/gh-aw/data/`. Esto reduce el consumo de tokens y acelera la ejecución del agente.

**Estructura de archivos:**
- `.github/workflows/auto-release-notes-caller.md` — Phase 1: genera borrador + abre PR
- `.github/workflows/auto-release-notes.md` — Phase 2: publica en Confluence
- `docs-drafts/` — Borradores generados automáticamente (revisión humana)
- `docs/atlassian-admin-setup.md` — Guía para administradores de Atlassian

**Safe Outputs**

El workflow utiliza el sistema de `safe-outputs` que restringe las operaciones permitidas del agente:
- Solo puede crear PRs en branches prefijados con `docs-draft/`
- Solo puede modificar archivos en `docs-drafts/**` y `README.md`
- Máximo 1 pull request por ejecución

---
> 🤖 Generado por `auto-release-notes-draft`
