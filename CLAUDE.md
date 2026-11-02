# CLAUDE.md

Guía para sesiones de Claude Code en este repositorio y en los de los servicios.

## Qué es este repositorio

`pdf-extractext` sube un PDF, extrae su texto y genera un resumen con IA en la nube
(OpenRouter); el resumen se guarda en MongoDB, se muestra en la interfaz y se descarga
como `.docx`. La configuración del servidor vive en Consul, y cada usuario puede elegir
modelo, API key y máximo de tokens desde la página Ajustes (en su navegador).

El proyecto está dividido en microservicios, cada uno en su repositorio de la
organización `PDF-extract-organization`. **Este repositorio no tiene código**: es la
portada, con la documentación general (`README.md`, `docs/`), la épica de la migración
(#64) y los issues transversales.

| Repositorio | Contenido |
|---|---|
| `pdf-extractext-web` | Interfaz y orquestador (BFF) |
| `pdf-extractext-extractor` | PDF → texto |
| `pdf-extractext-summarizer` | Texto → resumen con OpenRouter |
| `pdf-extractext-summaries` | Historial en MongoDB |
| `pdf-extractext-exporter` | Markdown → `.docx` |
| `pdf-extractext-config` | CRUD sobre el KV de Consul (`py-consul`) |
| `pdf-extractext-orchestration` | `docker-compose` con Traefik y Consul, y pruebas e2e |

La arquitectura completa está en `docs/estructura.md`.

### El monolito anterior

El código del monolito se retiró en #63 y queda en la etiqueta **`monolito-v1`**
(`git checkout monolito-v1`). El profesor lo pidió como entrega previa y la decisión
**puede cambiar**: antes de actuar sobre el monolito o sobre esa etiqueta, preguntar.
Nunca borrar ni mover la etiqueta.

### Trabajar en los servicios

Los siete repos se clonan como carpetas hermanas de este. Para levantar y probar el
stack completo, desde `pdf-extractext-orchestration/`:

```bash
cp .env.example .env    # MONGO_ROOT_PASSWORD obligatoria; OPENROUTER_API_KEY opcional (también desde Ajustes)
docker compose -f docker-compose.yml -f docker-compose.build.yml up -d --build --wait
uv run pytest -rs       # pruebas e2e, todas entrando por Traefik (http://pdf.localhost)
```

Sin `docker-compose.build.yml` usa las imágenes de ghcr, que la CI de cada servicio
publica al integrar en `main`. Traefik debe ser v3.6 o posterior (la 3.5 no funciona
con Docker Engine 29).

Todos los servicios siguen la misma estructura y las mismas reglas:

- Capas `app/presentation`, `app/application` (casos de uso, puertos y entidades; no
  importa librerías externas) y `app/infrastructure` (adaptadores). Configuración en
  `app/core` y cableado en `app/dependencies.py`, que importan los routers y `main`.
- `uv sync --group dev` y `uv run pytest`: sin red, sin base de datos y sin API key.
  Lo que necesita servicios reales se marca `@pytest.mark.integration` y se salta solo.
- TDD: un commit con los tests en rojo y otro con la implementación en verde.
- Los valores de cada resumen se resuelven: ajustes del usuario → Consul (vía `config`)
  → `.env` de `summarizer`. Nunca guardar API keys en Consul ni en el servidor.

### Convenciones

- Documentación, comentarios y mensajes de commit **en español**.
- Los mensajes de commit citan el issue (`#63`) para que GitHub lo enlace. En los repos
  de los servicios, el número es el del issue de ese repo.
- `main` y `develop` existen en todos los repos; los cambios entran por pull request a
  `develop`. Al mergear en `develop`, `Closes #N` no cierra el issue: hay que cerrarlo
  a mano.
- Principios: **KISS, DRY, YAGNI, SOLID** y **TDD**.

## Pull requests y merges con develop

Estas reglas son obligatorias. El criterio de fondo: **nunca modificar el estado del
repositorio antes de que la persona haya visto lo que va a pasar.**

### 1. Informar antes de actuar

Antes de aceptar un pull request o de hacer merge con `develop`, primero se informa.
No se ejecuta `git merge`, `gh pr merge` ni se resuelve ningún conflicto hasta que la
persona lo haya autorizado explícitamente después de leer el informe.

### 2. Conflictos de merge: listarlos antes de tocar nada

Los conflictos son normales y no son motivo de rechazo por sí solos. Pero **se listan
antes de arreglarlos**, nunca después:

1. Detectar los conflictos sin dejar el repositorio a medio mergear (por ejemplo con
   `git merge --no-commit --no-ff` seguido de `git merge --abort`, o `git merge-tree`).
2. Presentar la lista completa: qué archivos están en conflicto, en qué zonas y qué
   propone cada lado.
3. Explicar cómo se resolvería cada uno y qué se perdería en cada caso.
4. Esperar la aprobación.
5. Solo entonces resolver y mergear.

Si al resolverlos aparece un conflicto que no estaba en la lista inicial, se para y se
vuelve a informar.

### 3. Cuándo recomendar que NO se acepte un pull request

Se informa para **no aceptar** el pull request cuando tiene conflictos **y además**
incumple los principios del proyecto:

- **YAGNI**: añade funcionalidad, configuración o abstracciones que nadie ha pedido.
- **TDD**: no trae tests para el comportamiento nuevo, o los tests existentes fallan.
- **KISS**: resuelve de forma más complicada de lo que el problema requiere.
- **DRY**: duplica lógica que ya existe en el proyecto.

En ese caso el informe debe decir con claridad qué principio se incumple y en qué
archivo y línea, no una valoración genérica. La decisión final es siempre de la
persona: se recomienda, no se cierra el PR por cuenta propia.

Un PR con conflictos que respeta los principios se puede mergear tras el paso 2. Un PR
sin conflictos que incumple los principios se comenta, pero no se bloquea por esta
regla.

### 4. Qué incluye el informe de un pull request

- Qué hace el PR y qué archivos toca.
- Si tiene conflictos con `develop`, y la lista detallada del punto 2.
- Si los tests pasan (`uv run pytest`), ejecutados de verdad, no supuestos.
- Revisión frente a YAGNI, TDD, KISS y DRY, con referencias concretas al código.
- Una recomendación explícita: aceptar, aceptar con cambios, o no aceptar.

### 5. Borrado de ramas

No borrar ramas remotas sin confirmación explícita para cada una. Antes, comprobar
siempre si tienen commits sin mergear (`git rev-list --count origin/develop..origin/<rama>`)
e informar de qué trabajo se perdería. Recordar que las referencias `origin/*` locales
pueden estar obsoletas: consultar el remoto real con `git ls-remote --heads origin`.
