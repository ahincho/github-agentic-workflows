---
generated_at: 2026-06-16T20:16:21Z
repository: ahincho/github-agentic-workflows
trigger_commit: c4b9425328b2f45d1ee20a98a1f3510549e96ef1
branch: main
status: pending-review
---

# Borrador: github-agentic-workflows

> ⚠️ Requiere revisión humana. Edita si es necesario, luego mergea para publicar en Confluence.

## ¿Por qué se generó este borrador?

Este borrador documenta la **inicialización completa del proyecto NestJS** con un sistema de workflows agentic para automatizar la generación de documentación. El commit fusiona la configuración inicial del proyecto, incluyendo estructura de carpetas, workflows de GitHub Actions, agentes de IA, y configuración de integración con Confluence.

Commit: [`c4b9425`](https://github.com/ahincho/github-agentic-workflows/commit/c4b9425328b2f45d1ee20a98a1f3510549e96ef1)

## Commits que originan esta actualización

| SHA | Tipo | Mensaje | Autor |
|-----|------|---------|-------|
| [`c4b9425`](https://github.com/ahincho/github-agentic-workflows/commit/c4b9425328b2f45d1ee20a98a1f3510549e96ef1) | merge | merge: approve docs-draft for NestJS scaffold | Angel Eduardo Hincho Jove |

## Cambios propuestos

### 📝 Release Notes

**v0.1.0 - Inicialización del Proyecto**

- ✨ **Nuevo proyecto NestJS**: Scaffold completo con TypeScript, ESLint, Prettier
- 🤖 **Sistema de workflows agentic**: Automatización de generación de documentación mediante GitHub Actions
- 📄 **Integración con Confluence**: Publisher automático para sincronizar borradores aprobados
- 🧪 **Suite de testing**: Jest configurado para unit tests y e2e tests
- 📚 **Documentación inicial**: README del proyecto, guías de setup de Atlassian Admin

**Archivos principales creados** (32 archivos, 15,511 líneas añadidas):
- Estructura base de NestJS: `src/main.ts`, `app.module.ts`, `app.controller.ts`, `app.service.ts`
- Workflows: `auto-release-notes.lock.yml`, `auto-release-notes-caller.lock.yml`
- Agentes IA: `.github/agents/agentic-workflows.md`, `.github/skills/`
- Configuración: `package.json`, `tsconfig.json`, `eslint.config.mjs`, `nest-cli.json`

### 🔌 API / Interfaces

**Endpoints iniciales del proyecto NestJS:**

- `GET /` - Health check endpoint
  - Controlador: `AppController.getHello()`
  - Servicio: `AppService.getHello()`
  - Response: `"Hello World!"`

**Workflows disponibles:**

1. **auto-release-notes** - Workflow principal que genera borradores de documentación
   - Trigger: `push` a `main`
   - Salida: Pull Request con borrador en markdown en `docs-drafts/`

2. **auto-release-notes-caller** - Wrapper para invocar el workflow principal
   - Trigger: `push` a `main` (excluye cambios solo en `docs-drafts/`)

### 🏗️ Arquitectura / DevOps

**Stack tecnológico:**
- **Runtime**: Node.js con TypeScript 5
- **Framework**: NestJS 10.4.15
- **Testing**: Jest 29.7.0
- **Linting**: ESLint 9 con flat config
- **Formatting**: Prettier 3

**CI/CD:**
- GitHub Actions workflows para auto-generación de documentación
- Lock files versionados para reproducibilidad
- Integración con MCP (Model Context Protocol) para agentes IA

**Estructura del proyecto:**
```
.
├── .github/
│   ├── agents/           # Definiciones de agentes IA
│   ├── skills/           # Skills reutilizables para workflows
│   └── workflows/        # GitHub Actions workflows
├── docs/                 # Documentación del proyecto
├── docs-drafts/          # Borradores generados automáticamente
├── src/                  # Código fuente NestJS
└── test/                 # Tests e2e
```

**Dependencias clave:**
- `@nestjs/core`, `@nestjs/common`, `@nestjs/platform-express`
- `rxjs` para programación reactiva
- `reflect-metadata` para decoradores TypeScript

### 📚 Onboarding / Getting Started

**Pre-requisitos:**
- Node.js (versión recomendada: 18 o superior)
- npm o yarn
- Git

**Instalación:**
```bash
# Clonar el repositorio
git clone https://github.com/ahincho/github-agentic-workflows.git
cd github-agentic-workflows

# Instalar dependencias
npm install

# Desarrollo local
npm run start:dev

# Tests
npm test                # Unit tests
npm run test:e2e        # End-to-end tests
npm run test:cov        # Coverage report
```

**Estructura de comandos npm:**
- `npm run build` - Compila el proyecto TypeScript
- `npm run format` - Formatea código con Prettier
- `npm run lint` - Ejecuta ESLint
- `npm start` - Inicia la aplicación en modo producción

**Configuración de workflows:**

Los workflows agentic requieren configuración de Confluence (consultar `docs/atlassian-admin-setup.md`):
1. Crear aplicación OAuth en Atlassian Developer Console
2. Configurar secretos en GitHub: `CONFLUENCE_CLOUD_ID`, `CONFLUENCE_CLIENT_ID`, `CONFLUENCE_CLIENT_SECRET`
3. Los borradores generados en `docs-drafts/` se publican automáticamente al mergear el PR

**Recursos adicionales:**
- 📖 `README.md` - Visión general del proyecto
- 📖 `docs/README-project.md` - Documentación técnica detallada
- 📖 `docs/atlassian-admin-setup.md` - Guía de configuración de Confluence
- 🤖 `.github/agents/agentic-workflows.md` - Documentación de agentes IA

---
> 🤖 Generado por `auto-release-notes-draft` el 2026-06-16T20:16:21Z
