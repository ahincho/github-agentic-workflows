# github-agentic-workflows

Pipeline de documentación automatizada con **Human-in-the-Loop (HITL)** que mantiene viva la base de conocimiento en Confluence. Ante cada `push` a `main`, un agente LLM (GitHub Copilot con Claude Sonnet) analiza los cambios, genera un borrador markdown y abre un Pull Request para revisión humana. Al mergear el PR, el agente publica directamente en Confluence vía el MCP de Atlassian Rovo.

---

## Arquitectura — Pipeline en 2 Fases

```
push → main
    │
    ▼
┌─────────────────────────────────┐
│  Phase 1: auto-release-notes-   │
│  draft (caller)                 │
│                                 │
│  • Lee datos pre-cargados       │
│    (DataOps: git log/diff/gh)   │
│  • Genera docs-drafts/*.md      │
│  • Actualiza README.md          │
│  • Abre PR docs-draft/{sha}     │
└──────────────┬──────────────────┘
               │ PR abierto para revisión humana
               ▼
        👤 Revisor edita / aprueba
               │
               ▼ merge
┌─────────────────────────────────┐
│  Phase 2: auto-release-notes-   │
│  publish                        │
│                                 │
│  • Lee draft aprobado           │
│  • Busca páginas existentes     │
│    en Confluence (CQL)          │
│  • Crea o actualiza páginas     │
│    vía Atlassian Rovo MCP       │
└─────────────────────────────────┘
```

### Páginas que mantiene en Confluence (espacio `PLA`)

| Página | Descripción |
|--------|-------------|
| `[repo] Release Notes` | Changelog con tabla de commits linkeados a GitHub |
| `[repo] Getting Started` | Guía de instalación y primeros pasos |
| `[repo] Arquitectura / DevOps` | Stack técnico, CI/CD, estructura del proyecto |
| `[repo] API e Interfaces` | Endpoints y contratos de la API |

---

## Stack técnico

| Capa | Tecnología |
|------|-----------|
| Framework backend | NestJS v11 (TypeScript) |
| Runtime | Node.js |
| CI/CD agéntico | GitHub Agentic Workflows (`gh aw`) |
| Modelo LLM | Claude Sonnet 4.5 (via GitHub Copilot) |
| Documentación destino | Confluence Cloud (Atlassian Rovo MCP) |
| Optimización | DataOps pre-fetch (git log/diff fuera del sandbox LLM) |

---

## Estructura del proyecto

```
.github/
├── workflows/
│   ├── auto-release-notes-caller.md       # Phase 1 — genera borrador + abre PR
│   ├── auto-release-notes-caller.lock.yml # Compilado (no editar)
│   ├── auto-release-notes.md              # Phase 2 — publica en Confluence
│   └── auto-release-notes.lock.yml        # Compilado (no editar)
├── skills/
│   └── agentic-workflows/SKILL.md
docs/
├── atlassian-admin-setup.md               # Guía para habilitar Rovo MCP
└── agentic-workflows.md                   # Documentación técnica detallada
docs-drafts/                               # Borradores generados automáticamente
src/                                       # Código fuente NestJS
```

---

## Requisitos previos

### Secrets del repositorio

| Secret | Valor |
|--------|-------|
| `COPILOT_GITHUB_TOKEN` | Token de GitHub Copilot con acceso al modelo |
| `ATLASSIAN_MCP_BASIC_TOKEN` | `base64(email:api_token)` — token de Atlassian |

### Variables del repositorio

| Variable | Valor de ejemplo |
|----------|-----------------|
| `COPILOT_MODEL` | `claude-sonnet-4.5` |
| `CONFLUENCE_BASE_URL` | `https://tu-org.atlassian.net` |
| `CONFLUENCE_CLOUD_ID` | UUID del tenant de Atlassian |
| `CONFLUENCE_SPACE_KEY` | Clave del espacio (ej. `PLA`) |
| `CONFLUENCE_PARENT_PAGE_ID` | ID de la página padre en Confluence |

### Permisos de GitHub Actions

En **Settings → Actions → General → Workflow permissions**:
- ☑ Read and write permissions
- ☑ Allow GitHub Actions to create and approve pull requests

### Atlassian Admin

El administrador de Atlassian debe habilitar el toggle de API token:  
**Atlassian Administration → Rovo → Rovo MCP server → Authentication → API token**

Ver [docs/atlassian-admin-setup.md](docs/atlassian-admin-setup.md) para el paso a paso.

---

## Desarrollo local

```bash
# Instalar dependencias
npm install

# Iniciar en modo desarrollo
npm run start:dev

# Tests unitarios
npm run test

# Tests e2e
npm run test:e2e

# Compilar para producción
npm run start:prod
```

### Compilar los workflows agénticos

```bash
# Requiere gh CLI + extensión gh-aw
gh aw compile .github/workflows/auto-release-notes-caller.md
gh aw compile .github/workflows/auto-release-notes.md
```

---

## Cómo funciona el pipeline

1. **Push a `main`** → Phase 1 se dispara automáticamente
2. **DataOps** — el runner ejecuta `git log`, `git diff` y `gh pr list` fuera del sandbox LLM, escribiendo JSON compacto en `/tmp/gh-aw/data/`
3. **Agente** — Claude Sonnet lee los datos pre-cargados, evalúa si el cambio amerita documentación y genera el borrador en `docs-drafts/`
4. **PR automático** — `safe_outputs` abre el PR `docs-draft/{sha}` con los archivos permitidos (`docs-drafts/**`, `README.md`)
5. **Revisión humana** — el equipo revisa, edita si necesario, y mergea
6. **Phase 2** — al mergear, el agente lee el draft aprobado y publica/actualiza páginas en Confluence vía MCP

> Si el push no contiene cambios con impacto documentable (ej. commits de CI o configuración interna), el agente emite `noop` sin crear PR.

---

## Documentación en Confluence

Las páginas publicadas están disponibles en el espacio `PLA` de `inlearning.atlassian.net` bajo la página padre configurada en `CONFLUENCE_PARENT_PAGE_ID`.
- Visualize your application graph and interact with the NestJS application in real-time using [NestJS Devtools](https://devtools.nestjs.com).
- Need help with your project (part-time to full-time)? Check out our official [enterprise support](https://enterprise.nestjs.com).
- To stay in the loop and get updates, follow us on [X](https://x.com/nestframework) and [LinkedIn](https://linkedin.com/company/nestjs).
- Looking for a job, or have a job to offer? Check out our official [Jobs board](https://jobs.nestjs.com).

## Support

Nest is an MIT-licensed open source project. It can grow thanks to the sponsors and support by the amazing backers. If you'd like to join them, please [read more here](https://docs.nestjs.com/support).

## Stay in touch

- Author - [Kamil Myśliwiec](https://twitter.com/kammysliwiec)
- Website - [https://nestjs.com](https://nestjs.com/)
- Twitter - [@nestframework](https://twitter.com/nestframework)

## License

Nest is [MIT licensed](https://github.com/nestjs/nest/blob/master/LICENSE).
