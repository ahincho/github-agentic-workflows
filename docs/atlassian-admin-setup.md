# Configuración de Atlassian MCP — Guía para Administrador

Esta guía describe cómo un administrador de la organización Atlassian puede configurar la autenticación mediante **service account** para el flujo de trabajo `auto-release-notes`. Esta es la opción recomendada para entornos CI/CD ya que no está ligada a la cuenta personal de ningún usuario.

> **Configuración actual del repositorio:** Los workflows usan `Authorization: Basic` con el secret `ATLASSIAN_MCP_BASIC_TOKEN`. Una vez que el admin complete esta guía, se deberá cambiar el workflow a `Authorization: Bearer` y el secret a `ATLASSIAN_ROVO_MCP_TOKEN`.

---

## Paso 1 — Habilitar autenticación por API token en Rovo MCP

Por defecto, Rovo MCP solo acepta OAuth 2.1. El admin debe habilitar el acceso por token:

1. Ir a **[Atlassian Administration](https://admin.atlassian.com)** e ingresar a la organización.
2. Navegar a **Rovo → Rovo MCP server**.
3. En la sección **Authentication**, activar el toggle **"API token"**.

Sin este paso, ningún tipo de token (personal ni service account) funcionará para acceso headless.

---

## Paso 2 — Crear una service account

Una service account es una cuenta de Atlassian no ligada a ninguna persona física, ideal para CI/CD.

1. En **Atlassian Administration**, ir a **User management → Users**.
2. Seleccionar **Invite user** y crear un usuario con un email de servicio (por ejemplo: `svc-github-workflows@inlearning.com`).
3. Asignarle los permisos mínimos necesarios en **Confluence**:
   - Acceso al espacio `PLA` (o el espacio objetivo).
   - Permiso para **crear y editar páginas**.

---

## Paso 3 — Generar un API key para la service account

1. Desde la sesión del administrador, ir a **Atlassian Administration → API keys** (o equivalente según el plan).
2. Crear un nuevo API key para la service account, seleccionando los scopes necesarios para Rovo MCP (Confluence read/write).
3. Copiar el API key generado — **solo se muestra una vez**.

---

## Paso 4 — Configurar el secret en GitHub

Con el API key en mano:

1. Ir al repositorio o a la organización en GitHub → **Settings → Secrets and variables → Actions**.
2. Crear (o actualizar) el secret:
   - **Nombre:** `ATLASSIAN_ROVO_MCP_TOKEN`
   - **Valor:** el API key generado en el paso anterior (sin codificar en base64).

---

## Paso 5 — Actualizar el workflow a Bearer

Una vez configurado el secret, actualizar el header en ambos workflows:

**`.github/workflows/auto-release-notes.md`** y **`auto-release-notes-caller.md`**:

```yaml
mcp-servers:
  atlassian:
    type: http
    url: "https://mcp.atlassian.com/v1/mcp/authv2"
    headers:
      Authorization: "Bearer ${{ secrets.ATLASSIAN_ROVO_MCP_TOKEN }}"
```

Luego recompilar y hacer push:

```bash
gh aw compile .github/workflows/auto-release-notes.md
gh aw compile .github/workflows/auto-release-notes-caller.md
git add .
git commit -m "fix: switch to Bearer auth with service account"
git push
```

---

## Resumen comparativo de opciones de autenticación

| | Token personal (actual) | Service account (recomendado) |
|--|--|--|
| **Auth header** | `Basic <base64(email:token)>` | `Bearer <api_key>` |
| **Secret** | `ATLASSIAN_MCP_BASIC_TOKEN` | `ATLASSIAN_ROVO_MCP_TOKEN` |
| **Quién lo crea** | Cualquier usuario con cuenta Atlassian | Administrador de la organización |
| **Ligado a persona** | Sí — si el usuario sale, el token deja de funcionar | No — independiente de usuarios |
| **Idóneo para CI/CD** | Solo para pruebas / desarrollo | Sí |
| **Requiere acción del admin** | Solo habilitar el toggle de API token | Sí — toggle + service account + API key |

---

## Referencias

- [Control Atlassian Rovo MCP server settings](https://support.atlassian.com/security-and-access-policies/docs/control-atlassian-rovo-mcp-server-settings/)
- [Configuring authentication via API token](https://support.atlassian.com/atlassian-rovo-mcp-server/docs/configuring-authentication-via-api-token/)
- [Atlassian Rovo MCP Server — GitHub](https://github.com/atlassian/atlassian-mcp-server)
