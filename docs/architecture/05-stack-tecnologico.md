# 05. Stack tecnológico

Resumen de las tecnologías de `local_project_manager`. Las versiones se mantienen únicamente en
[`pyproject.toml`](../../pyproject.toml).

## Stack principal

| Área                | Tecnología                       | Uso principal                                                 |
| ------------------- | -------------------------------- | ------------------------------------------------------------- |
| Lenguaje            | Python 3.11+                     | Implementación de la CLI y del dominio                        |
| CLI                 | Typer                            | Subcomandos, opciones, ayuda y códigos de salida              |
| Terminal            | Rich                             | Menú interactivo, formularios, tablas y confirmaciones        |
| Manifiestos         | JSON (`json`)                    | Lectura y escritura de `.project.json`                        |
| Catálogos y layouts | PyYAML                           | Lectura y escritura de `.workspace.yaml`, catálogos y layouts |
| Reportes            | Jinja2                           | Generación de Markdown en `meta/`                             |
| Git                 | CLI de Git mediante `subprocess` | `git init` y `git clone`                                      |

Jinja2 se usa exclusivamente para `meta/`. Los archivos de layout usan marcadores simples
(`{name}`, `{slug}`, ...) sustituidos con `str.format_map`, no Jinja2: un layout es un dato YAML,
no una plantilla de programa.

## Biblioteca estándar

Se prioriza la biblioteca estándar para mantener el proyecto pequeño y fácil de mantener.

| Módulos                             | Uso                                                                                  |
| ----------------------------------- | ------------------------------------------------------------------------------------ |
| `pathlib`                           | Rutas del almacén global, del workspace y de los proyectos                           |
| `dataclasses`, `typing`             | Las cuatro clases de dominio y sus conversiones                                      |
| `json`                              | Persistencia de `.project.json`                                                      |
| `tempfile`, `os.replace`            | Escrituras atómicas (RN-402)                                                         |
| `shutil`                            | Crear, mover y eliminar directorios de proyectos                                     |
| `datetime`                          | Fechas ISO 8601                                                                      |
| `re`, `unicodedata`                 | Normalización de identificadores y generación de slugs                               |
| `string.Formatter`                  | Detección de marcadores desconocidos en layouts                                      |
| `collections.Counter`               | Conteos de los índices                                                               |
| `importlib.resources`               | Semillas empaquetadas `resources/catalogs/base.yaml` y `resources/layouts/base.yaml` |
| `importlib.metadata`                | Versión única del paquete                                                            |
| `os.environ`, `pathlib.Path.home()` | Resolución del almacén global (`LPM_HOME` o `~/.local_project_manager/`)             |
| `subprocess`                        | Invocación de Git                                                                    |
| `logging`                           | Diagnóstico, con handler de Rich                                                     |

## Herramientas de desarrollo

- `uv`: entorno, dependencias y ejecución de comandos.
- `hatchling`: construcción del paquete con layout `src/`.
- `pytest` y `pytest-cov`: pruebas y cobertura.
- `Ruff`: lint y formato; la regla `PTH` obliga a `pathlib`.
- `mypy` en modo estricto: comprobación estática de tipos.
- `pre-commit`: validaciones antes de cada commit.

## Decisiones de arquitectura

- Typer y Rich para una CLI tipada que funciona tanto por subcomandos como por menú interactivo.
- Los modelos son `dataclasses` con validación explícita en servicios. Sin pydantic ni jsonschema:
  el esquema de cada archivo es su clase y su validador.
- PyYAML con `safe_load` y `safe_dump` para todo lo que no sea el manifiesto; los catálogos y
  layouts son datos, no código.
- Git se integra mediante la CLI instalada y `subprocess`, sin GitPython.
- Toda escritura es atómica.
- El almacén global se resuelve en `~/.local_project_manager/` con `LPM_HOME` como alternativa
  portable.
- Python 3.11 es la versión mínima soportada.

## Evaluadas y no adoptadas

| Tecnología     | Motivo                                                                                                 |
| -------------- | ------------------------------------------------------------------------------------------------------ |
| pydantic       | Añade una dependencia pesada para cuatro dataclasses con reglas cruzadas que igualmente van en código. |
| jsonschema     | No expresa las reglas cruzadas del catálogo; sería una segunda fuente de verdad.                       |
| GitPython      | Dos operaciones de Git no justifican la dependencia.                                                   |
| ruamel.yaml    | No hace falta conservar comentarios: los YAML los escribe el CLI.                                      |
| python-slugify | `unicodedata` y `re` cubren el caso.                                                                   |
| questionary    | Rich ya cubre prompts y selección.                                                                     |
| platformdirs   | Una única ruta por defecto más una variable de entorno es suficiente.                                  |

## Referencias

- [Estrategia de solución](02-estrategia-solucion.md)
- [Plan de implementación](../planning/plan-implementacion.md)
