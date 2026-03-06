name: Intent Compiler
description: Traduce especificaciones de lenguaje natural en casos de prueba, documentación, esqueletos de código y criterios de aceptación
tags: [nlp, testing, documentation, prototyping, ai-assisted, code-generation]
version: 1.2.0
author: OpenClaw Team
requires:
  - python>=3.9
  - openai>=1.0.0
  - jinja2>=3.0.0
  - pyyaml>=6.0
  - git
env:
  OPENCLAW_INTENT_MODEL: "gpt-4"
  OPENCLAW_INTENT_TEMP: "0.3"
  OPENCLAW_INTENT_MAX_TOKENS: "2000"
```

# Intent Compiler

## Propósito

El Intent Compiler transforma especificaciones de lenguaje natural no estructuradas en artefactos concretos y ejecutables. Conecta la brecha entre requisitos de alto nivel y salidas listas para implementación.

### Casos de Uso Reales

- **Generación de Pruebas**: Convierte "Como usuario, quiero iniciar sesión con correo y contraseña" en esqueletos de pruebas Jest/Pytest con aserciones
- **Producción de Documentación**: Transforma docstrings de funciones y patrones de uso en README.md completos, documentos de referencia de API o entradas de changelog
- **Andamiaje de Código**: Genera interfaces TypeScript, esqueletos de componentes React o rutas Express a partir de descripciones de endpoints
- **Definición de Criterios de Aceptación**: Analiza descripciones de características tipo Gherkin en escenarios estructurados
- **Verificación de Compatibilidad Hacia Atrás**: Genera matrices de prueba para validar el cumplimiento del contrato de API
- **Síntesis de Planes de Migración**: Crea guías de migración paso a paso de arquitectura legada a nueva

## Alcance

### Comandos Principales

```
intent-compiler generate \
  --type <test|docs|scaffold|criteria|migration> \
  --input <file|string> \
  --output <path> \
  [--template <name>] \
  [--context <dir>] \
  [--dry-run]

intent-compiler validate \
  --spec <file> \
  --against <commit|branch>

intent-compiler interactive \
  --mode <refinement|iteration> \
  --seed <initial-prompt>

intent-compiler suggest \
  --based-on <error-output|test-failure> \
  --context <file://path>
```

### Banderas CLI

- `--type`: Categoría de artefacto (requerido)
- `--input`: Especificación fuente - puede ser texto plano, markdown o archivo de características YAML
- `--output`: Ruta de destino; usa plantillas si es directorio
- `--template`: Nombre de plantilla Jinja2 (por defecto: `default_<type>.j2`)
- `--context`: Archivos/directorios adicionales para contexto RAG
- `--dry-run`: Previsualizar sin escribir archivos
- `--based-on`: Para comando `suggest` - salida de prueba fallida
- `--mode`: Bucle interactivo de refinamiento (`refinement` para ciclos editar-sugerir, `iteration` para múltiples variaciones)

## Proceso de Trabajo Detallado

### 1. Análisis de Intención
- Cargar entrada desde archivo o stdin
- Detectar tipo de intención si no se proporciona usando clasificador (test/docs/scaffold/criteria)
- Extraer entidades de dominio (ej: "usuario", "orden", "pago")
- Identificar restricciones (asíncrono, validación, permisos)

### 2. Ensamblaje de Contexto
- Si se proporciona `--context`, escanear archivos para patrones de código relacionados
- Construir prompt con:
  - Especificación original
  - Stack tecnológico detectado de `package.json`, `requirements.txt` o `Cargo.toml`
  - Ejemplos de pruebas existentes si están presentes
  - Fragmentos de guía de estilo si se encuentran

### 3. Fase de Generación
- Construir prompt de OpenAI con mensajes de sistema desde `templates/system_<type>.txt`
- Llamar a LLM con temperatura del entorno (por defecto 0.3 para salidas deterministas)
- Transmitir respuesta a formato intermedio estructurado (JSON) antes de aplicar plantilla
- Aplicar plantilla Jinja2 para producir artefacto final

### 4. Validación y Linting
- Para pruebas: verificar imports correctos, convenciones de nombres de pruebas
- Para docs: verificar jerarquía de encabezados, sintaxis de bloques de código
- Para scaffold: asegurar que no haya errores de sintaxis en código generado (verificación de parseo básica)
- Enviar advertencias a stderr, retornar código de salida 1 si hay fallos críticos

### 5. Escritura de Salida
- Crear directorios padres si es necesario
- Escribir archivo con permisos 644
- Si la salida es directorio: generar múltiples archivos scaffold (ej: `test/<component>.spec.ts`, `docs/api/<module>.md`)

### Ejemplo de Flujo de Trabajo (Generación de Pruebas)

```bash
# Usuario escribe especificación en archivo 'feature.md':
# "Login endpoint accepts POST /auth/login with email and password, returns JWT on success"

intent-compiler generate \
  --type test \
  --input feature.md \
  --output tests/auth/ \
  --template react-query
```

Proceso:
1. Parseo: identifica `POST /auth/login`, parámetros `email`, `password`, respuesta `JWT`
2. Contexto: encuentra `package.json` con `@testing-library/react` y `msw`
3. Generación: LLM produce prueba Jest + React Testing Library con mock service worker
4. Validación: verifica rutas de importación, nombres de prueba (`describe('auth login', ...)`)
5. Escritura: `tests/auth/login.test.tsx`

## Reglas de Oro

### Deben Seguirse

1. **Prompts Atómicos**: Cada generación debe apuntar a un solo tipo de intención; nunca mezclar test + docs en una llamada
2. **Versionado de Plantillas**: Almacenar plantillas en `~/.openclaw/templates/` con versionado semántico; la actualización requiere migración manual
3. **Limitación de Contexto**: Máximo 5 archivos de contexto por ejecución para evitar overflow de tokens; usar `--context-dir` con patrones `.intentignore`
4. **Puerta de Revisión**: Nunca escribir directamente en directorios de producción (ej: `src/`, `app/`); la salida por defecto debe ser `generated/` o `tmp/`
5. **Idempotencia**: Re-ejecutar con misma entrada debe producir salida idéntica bit a bit cuando `--temp=0`
6. **Seguridad**: Nunca generar operaciones destructivas (rm -rf, drop database) sin bandera `--force` explícita y revisión humana
7. **Atribución**: Incluir encabezado de comentario con marca de tiempo de generación, hash de entrada y versión de plantilla

### Anti-Patrones a Evitar

- No generar microservicios completos a partir de especificaciones de una frase
- No hacer commit automático de código generado; siempre dejar al usuario
- No inferir lógica de negocio; limitarse a patrones estructurales
- Evitar valores hardcodeados; usar placeholders como `{{API_BASE_URL}}`
- Nunca exponer API keys en ejemplos generados

## Ejemplos

### Ejemplo 1: Generación de Esqueleto de Pruebas

**Entrada** (`specs/search.feature.md`):
```markdown
Feature: Product search
  As a shopper
  I want to search products by name
  So that I can find items quickly

  Scenario: Search returns matching products
    Given the catalog contains "widget", "gadget", "doohickey"
    When I search for "wid"
    Then I see "widget" in results
```

**Comando**:
```bash
intent-compiler generate \
  --type test \
  --input specs/search.feature.md \
  --output tests/unit/product_search.test.ts \
  --template cucumber-ts
```

**Salida** (`tests/unit/product_search.test.ts`):
```typescript
// Generated by Intent Compiler v1.2.0 | Template: cucumber-ts
// Input: specs/search.feature.md (sha256: a3f8...)
// DO NOT EDIT MANUALLY - regenerate via intent-compiler

import { test, expect } from '@playwright/test';

test.describe('Product search', () => {
  test('Search returns matching products', async ({ page }) => {
    // Given the catalog contains "widget", "gadget", "doohickey"
    await page.goto('/admin/seed?items=widget,gadget,doohickey');

    // When I search for "wid"
    await page.fill('[data-testid=search-input]', 'wid');
    await page.click('[data-testid=search-submit]');

    // Then I see "widget" in results
    await expect(page.locator('[data-testid=product-card]')
      .filter({ hasText: 'widget' })).toBeVisible();
  });
});
```

### Ejemplo 2: Generación de Documentación de API

**Entrada** (`src/api/v1/users.ts`):
```typescript
/**
 * GET /api/v1/users
 * Query: ?role=admin&page=1
 * Returns paginated user list
 */
export async function listUsers(req: Request) { ... }

/**
 * POST /api/v1/users
 * Body: { email: string, name: string }
 * Returns created user with id
 */
export async function createUser(req: Request) { ... }
```

**Comando**:
```bash
intent-compiler generate \
  --type docs \
  --input src/api/v1/users.ts \
  --output docs/api/v1/users.md \
  --template openapi-json
```

**Salida** (`docs/api/v1/users.md`):
```markdown
# Users API (v1)

Base URL: `/api/v1`

## List Users

`GET /users`

Query Parameters:
| Name  | Type   | Required | Description           |
|-------|--------|----------|-----------------------|
| role  | string | false    | Filter by role        |
| page  | number | false    | Page number (default: 1) |

Response (200):
```json
{
  "data": [{ "id": "uuid", "email": "...", "name": "..." }],
  "page": 1,
  "total": 150
}
```

## Create User

`POST /users`

Request Body:
```json
{
  "email": "string",
  "name": "string"
}
```

Response (201):
```json
{
  "id": "uuid",
  "email": "...",
  "name": "..."
}
```
```

### Ejemplo 3: Andamiaje de Código con Contexto

**Comando**:
```bash
intent-compiler generate \
  --type scaffold \
  --input "Create a Next.js 14 page for /dashboard/settings with form fields: theme, notifications, language" \
  --output app/dashboard/settings/page.tsx \
  --context app/(auth)/layout.tsx,components/ui/Form.tsx \
  --template next14-page
```

**Salida Generada**: Incluye patrón `useServerAction`, imports del componente Form existente, respeta estructura de layout.

## Comandos de Rollback

### Rollback a Nivel de Archivo

```bash
# Deshacer última generación si está bajo git
git checkout HEAD -- app/dashboard/settings/page.tsx

# O restaurar desde respaldo de marca de tiempo
cp ~/.openclaw/backups/20240315_143022/app/dashboard/settings/page.tsx \
   app/dashboard/settings/page.tsx
```

### Limpieza de Nivel de Sesión

```bash
# Eliminar todos los archivos generados de la sesión actual
intent-compiler --cleanup-temp --session-id $(cat ~/.openclaw/last_session_id)
```

### Rollback de Estado

```bash
# Revertir registro de plantillas a versión anterior (si se actualizó)
git -C ~/.openclaw/templates checkout v1.1.0
```

## Pasos de Verificación

### Verificaciones Posteriores a la Generación

1. **Validación de Sintaxis**:
   ```bash
   # Para TypeScript
   npx tsc --noEmit generated/*.ts 2>&1 | grep -q "error" && echo "FAIL"
   
   # Para Python
   python -m py_compile generated/*.py
   ```

2. **Dry-Run de Pruebas**:
   ```bash
   # Jest/Vitest
   npx vitest run --reporter=json --silent tests/generated/ > /dev/null
   echo $?  # Debería ser 0 (las pruebas se parsean correctamente)
   ```

3. **Verificación de Enlaces en Documentación**:
   ```bash
   # Verificar enlaces internos en markdown generado
   grep -Eo '\[[^]]+\]\([^)]+\)' docs/generated/*.md \
     | cut -d')' -f1 | cut -d'(' -f2 \
     | while read link; do
         if [ ! -f "$link" ] && [[ "$link" != http* ]]; then
           echo "Broken link: $link"
         fi
       done
   ```

4. **Estimación de Cobertura**:
   ```bash
   # Asegurar que las pruebas generadas ejerciten rutas requeridas
   grep -c "it\|test" tests/generated/*.test.ts | awk '{s+=$1} END {print s" test cases"}'
   ```

## Solución de Problemas

| Síntoma | Causa Probable | Solución |
|---------|----------------|----------|
| Archivo de salida vacío | Límite de cuota LLM excedido | Verificar límites de `OPENAI_API_KEY`; agregar reintento `--retries 3` |
| "Template not found" | Archivo de plantilla faltante | `ls ~/.openclaw/templates/`; instalar via `intent-compiler install-template <name>` |
| Unicode corrupto | Encoding incorrecto | Asegurar `PYTHONIOENCODING=utf-8`; regenerar con `--encoding utf-8` |
| Pruebas no descubiertas | Patrón de nombres incorrecto | Ajustar al framework: Jest usa `.test.tsx`, PyTest usa `test_*.py` |
| Generación lenta | Contexto demasiado grande | Reducir archivos `--context`; usar `.intentignore` para excluir `node_modules`, `dist` |
| Imports alucinados | LLM demasiado creativo | Fijar versión de plantilla; establecer `OPENCLAW_INTENT_TEMP=0.1` para determinismo |
| Permiso denegado | Directorio de salida no escribible | Usar `--output ./generated/` dentro del proyecto; evitar directorios del sistema |

### Modo Debug

```bash
# Mostrar prompt completo enviado al LLM
OPENCLAW_INTENT_DEBUG=1 intent-compiler generate ... 2>&1 | less

# Rastrear escrituras de archivos
strace -e openat,write -f intent-compiler generate ...
```

## Dependencias

- **Runtime**: Python 3.9+, Node.js 18+ (para ciertas plantillas)
- **Proveedor LLM**: OpenAI API (GPT-4 recomendado) o endpoint Azure OpenAI
- **Git**: Para comparaciones de diff y seguimiento de rollback
- **Opcional**: `jq` (procesamiento JSON en plantillas), `pandoc` (conversión de docs)

### Instalación

```bash
# Clonar repositorio de plantillas
git clone https://github.com/openclaw/intent-templates ~/.openclaw/templates

# Instalar paquete Python
pipx install intent-compiler

# Configurar
echo 'export OPENAI_API_KEY="sk-..."' >> ~/.bashrc
intent-compiler --install-completions  # bash/zsh/fish
```

### Configuración Mínima

```bash
mkdir -p ~/.openclaw/templates
intent-compiler verify-env
# Debería mostrar: OK: todas las plantillas requeridas presentes
```

## Variables de Entorno

- `OPENCLAW_INTENT_MODEL`: Nombre del modelo OpenAI (`gpt-4-turbo`, `gpt-3.5-turbo`)
- `OPENCLAW_INTENT_TEMP`: Temperatura de muestreo 0.0-1.0 (por defecto: 0.3)
- `OPENCLAW_INTENT_MAX_TOKENS`: Presupuesto de generación (por defecto: 2000)
- `OPENCLAW_INTENT_TEMPLATE_DIR`: Sobrescribir ubicación de plantillas por defecto
- `OPENCLAW_INTENT_DRY_RUN`: Establecer a `1` para omitir escritura de archivos globalmente
- `OPENCLAW_INTENT_LOG_FILE`: Ruta a registro de auditoría JSONL
- `OPENCLAW_INTENT_CONTEXT_LIMIT`: Máximo de archivos de contexto (por defecto: 5)

```bash
# Ejemplo .env
OPENCLAW_INTENT_MODEL="gpt-4-1106-preview"
OPENCLAW_INTENT_TEMP=0.2
OPENCLAW_INTENT_LOG_FILE="$HOME/.openclaw/logs/intent_audit.jsonl"
```

## Consejos de Rendimiento

- Cachear embeddings de contexto: `intent-compiler index-context --dir src/ --db ~/.cache/intent_context.db`
- Usar `--type test --template minimal` para generación más rápida en prototipado
- Batch de múltiples especificaciones: `find specs/ -name "*.md" -exec intent-compiler generate -i {} -o generated/ \;`
- Reutilizar JSON intermedio: `--intermediate /tmp/intent_payload.json` para separar generación de templating

## Limitaciones Conocidas

- LLM puede perder casos límite en pruebas; siempre revisar y aumentar
- Sintaxis de plantilla debe coincidir exactamente con versión del framework objetivo
- No hay visor de diff integrado; confiar en `git diff` para cambios de código generado
- Contexto grande (>10 archivos) puede exceder límites de tokens; usar indexación selectiva

## Roadmap Futuro

- Integración Git: `intent-compiler apply --commit` para auto-commit con mensajes convencionales
- Multi-modal: Soporte para especificaciones basadas en imágenes (wireframe → scaffold)
- Bucle de retroalimentación: `intent-compiler improve --from PR-comments` para afinar prompts
- Distribuido: Pool de workers para generación masiva en pipelines CI
```