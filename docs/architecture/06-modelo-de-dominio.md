# 06. Modelo de Dominio y Contratos de Datos

Referencia única de las clases de dominio de `local_project_manager` y de los archivos en los que
se persisten. Define qué representa cada clase, qué campos tiene, qué invariantes cumple y cómo se
serializa. El comportamiento del sistema lo definen los RF y RN de
[01-contexto-y-objetivos.md](01-contexto-y-objetivos.md); los conceptos compartidos están en
[04-conceptos-transversales.md](04-conceptos-transversales.md).

## Las cuatro clases

No se usa base de datos. Hay una clase por archivo persistente. Son `dataclasses` planas de la
biblioteca estándar, no conocen rutas absolutas ni tocan disco (RN-304).

| Clase           | Representa                                                      | Persistencia                                                        | Mutabilidad                                            |
| --------------- | --------------------------------------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------ |
| `Catalog`       | El vocabulario de valores permitidos.                           | `templates/catalogs/*.yaml` y bloque `catalog` de `.workspace.yaml` | Base inmutable; custom y copia del workspace editables |
| `Workspace`     | Identidad del workspace, patrón de rutas, catálogo y obsoletos. | `.workspace.yaml`                                                   | Mutable mediante comandos validados                    |
| `ProjectLayout` | Directorios y archivos iniciales que materializa `new`.         | `templates/layouts/*.yaml` y `.layouts/*.yaml`                      | Base global inmutable; el resto editable               |
| `Project`       | La fuente de verdad de un proyecto.                             | `.project.json`                                                     | Mutable mediante comandos validados                    |

### Relaciones

```text
Workspace ──contiene──────────▶ Catalog            (copia independiente del global)
Workspace ──dispone de────────▶ 1..* ProjectLayout (archivos en .layouts/, no embebidos)
Project   ──se valida contra──▶ Workspace.catalog + Workspace.obsolete_values
Project   ──se materializa con▶ un ProjectLayout elegido en `new`
Project   ──se ubica según────▶ Workspace.project_path_pattern
```

`Project` no guarda referencia al layout con el que nació ni al catálogo contra el que se validó.
Sin revisiones no habría nada que resolver con esa referencia: siempre se valida contra el único
catálogo activo del workspace. Si alguno de sus valores está en `obsolete_values`, se muestra como
obsoleto de forma derivada, sin escribirlo en `.project.json` (RN-705).

---

## `Catalog`

Contenido validable de un catálogo. Es la misma clase para `templates/catalogs/base.yaml`,
`templates/catalogs/custom/<id>.yaml` y el bloque `catalog` de `.workspace.yaml`.

```python
@dataclass
class Catalog:
    schema_version: str                              # versión del contrato del catálogo
    catalog_id: str                                  # coincide con el nombre del archivo
    name: str
    statuses: dict[str, str]                         # id -> etiqueta legible
    priorities: dict[str, str]                       # id -> etiqueta legible
    categories: list[str]
    types_by_category: dict[str, list[str]]          # toda categoría es clave, aunque sea []
    languages: list[str]
    description: str = ""
    frameworks_by_language: dict[str, list[str]] = field(default_factory=dict)
    default_tags_by_type: dict[str, list[str]] = field(default_factory=dict)
```

Invariantes (`catalog_validator`, RN-105):

- Todos los identificadores están normalizados y son únicos dentro de su lista o mapa.
- `statuses`, `priorities`, `categories` y `languages` no están vacíos.
- Las claves de `types_by_category` son exactamente `categories`.
- Las claves de `frameworks_by_language` están contenidas en `languages`.
- Las claves de `default_tags_by_type` están contenidas en la unión de todos los tipos.
- `catalog_id` coincide con el nombre del archivo del que se carga. `base` está reservado.

Si es base o custom no se guarda en el archivo: se deriva de su carpeta.

```yaml
schema_version: "1.0"
catalog_id: "base"
name: "Catálogo base"
description: "Vocabulario inicial de Local Project Manager"
statuses:
    todo: "Pendiente de iniciar"
    in_progress: "En desarrollo activo"
priorities:
    high: "Importante, próximo a completar"
    medium: "Progreso normal"
categories: [web, mobile, archive]
types_by_category:
    web: [backend, frontend, fullstack]
    mobile: [native, flutter, react-native]
    archive: []
languages: [python, typescript]
frameworks_by_language:
    python: [django, fastapi, flask, typer]
    typescript: [react, nextjs, angular, nestjs]
default_tags_by_type:
    backend: [api, rest, auth, crud]
    frontend: [spa, dashboard, form, responsive]
```

El contenido completo del catálogo base está en [07-plantillas-base.md](07-plantillas-base.md).

---

## `Workspace`

Representa el documento `.workspace.yaml` completo.

```python
@dataclass
class Workspace:
    schema_version: str
    workspace_id: str                                # identificador estable, generado al crear
    name: str
    created_at: str                                  # ISO 8601
    updated_at: str                                  # ISO 8601, lo actualiza cada guardado
    source_catalog_id: str                           # informativo, nunca se usa para validar (RN-605)
    project_path_pattern: str                        # "{category}/{type}/{slug}" (RN-606)
    catalog: Catalog
    obsolete_values: ObsoleteValues                  # misma forma que Catalog, vacío por defecto
```

`ObsoleteValues` es un alias de tipo, no una clase: `dict[str, list[str] | dict[str, list[str]]]`
con exactamente las claves `statuses`, `priorities`, `categories`, `types_by_category`,
`languages`, `frameworks_by_language` y `default_tags_by_type`. Cada valor replica la forma de la
lista o mapa homónimo del catálogo pero solo enumera identificadores.

Invariantes:

- `catalog` valida como `Catalog`.
- `project_path_pattern` valida según [04](04-conceptos-transversales.md#patrón-de-rutas).
- Todo identificador de `obsolete_values` existe en la lista o mapa correspondiente de `catalog`.
- Todas las claves de `obsolete_values` están presentes, aunque vacías.

```yaml
schema_version: "1.0"
workspace_id: "ws-7f3a2c"
name: "dev"
created_at: "2026-09-21"
updated_at: "2026-09-21"
source_catalog_id: "base"
project_path_pattern: "{category}/{type}/{slug}"
catalog:
    schema_version: "1.0"
    catalog_id: "base"
    name: "Catálogo base"
    description: "Vocabulario inicial de Local Project Manager"
    statuses: { ... }
    priorities: { ... }
    categories: [web, mobile, ...]
    types_by_category: { ... }
    languages: [...]
    frameworks_by_language: { ... }
    default_tags_by_type: { ... }
obsolete_values:
    statuses: []
    priorities: []
    categories: []
    types_by_category: {}
    languages: []
    frameworks_by_language: {}
    default_tags_by_type: {}
```

Una ruta es un workspace gestionado solo si contiene un `.workspace.yaml` que carga y valida
(RF-103).

---

## `ProjectLayout`

Describe qué materializa `new` dentro del directorio del proyecto. No sabe dónde está ese
directorio (RN-801).

```python
@dataclass
class ProjectLayout:
    schema_version: str
    layout_id: str                                   # coincide con el nombre del archivo
    name: str
    directories: list[str]                           # rutas relativas, p. ej. ["src", "docs", "test"]
    files: dict[str, str]                            # ruta relativa -> contenido íntegro del archivo
    description: str = ""
    init_git: bool = False                           # si `new` ejecuta git init
```

Invariantes (`layout_validator`, RN-802..804):

- Toda ruta de `directories` y `files` es relativa, no contiene `..` y no es `.project.json`.
- Los marcadores usados en rutas y contenidos pertenecen a la lista permitida.
- `layout_id` normalizado; `base` está reservado.

Cada valor de `files` es el texto completo del archivo, con tantas líneas como haga falta. En YAML
se escribe con bloque literal (`|`). El CLI no interpreta el contenido: sustituye marcadores y lo
escribe. Un valor vacío crea un archivo vacío.

```yaml
schema_version: "1.0"
layout_id: "python-package"
name: "Paquete Python"
description: "Paquete instalable con pyproject, tests y docs"
directories: [tests, docs]
files:
    pyproject.toml: |
        [project]
        name = "{slug}"
        version = "0.1.0"
        description = "{description}"
        requires-python = ">=3.11"
        authors = [{{ name = "Manu Ramos" }}]

        [tool.ruff]
        line-length = 100
    src/{slug}/__init__.py: |
        """{name}."""

        __version__ = "0.1.0"
    README.md: |
        # {name}

        {description}
    .gitignore: |
        .venv/
        __pycache__/
        dist/
init_git: true
```

El contenido del layout base está en [07-plantillas-base.md](07-plantillas-base.md).

---

## `Project`

La única fuente de verdad de un proyecto (RN-101). Los índices y fichas de `meta/` son derivados.

```python
@dataclass
class Project:
    manifest_version: str                            # versión del formato de .project.json (RN-504)
    # obligatorios
    name: str                                        # nombre legible, editable
    slug: str                                        # único, inmutable, nombre del directorio (RN-102)
    category: str
    type: str                                        # "" si la categoría no define tipos (RN-104)
    status: str
    priority: str
    language: str
    created: str                                     # ISO 8601 (RN-502)
    # opcionales, siempre presentes en el JSON con "" o [] (RN-501)
    description: str = ""
    language_version: str = ""
    framework: str = ""
    framework_version: str = ""
    database: str = ""                               # texto libre
    tags: list[str] = field(default_factory=list)
    repo: str = ""
    notes: str = ""
    modified: str = ""                               # lo actualiza el CLI en cada edición
    archived_at: str = ""                            # solo con status archived
    depends_on: list[str] = field(default_factory=list)   # slugs existentes, sin ciclos (RN-503)

    def to_dict(self) -> dict[str, object]: ...
    @classmethod
    def from_dict(cls, data: dict[str, object]) -> "Project": ...
```

`Project` no guarda su ruta: se deriva del workspace, `category`, `type` y `slug` mediante el
patrón de rutas.

### Clase plana, JSON anidado

La clase es plana y el JSON anidado, a propósito. El código consume campos sueltos: el validador
comprueba `status`, `category`, `type` y `language`, que en el JSON viven en tres bloques distintos;
los filtros de `project list` son `--status` y `--category`; las plantillas de `meta/` escriben
`project.status`. Anidar la clase obligaría a crear siete dataclasses sin comportamiento propio y a
reconstruir bloques en cada edición. El anidado del JSON es legibilidad del archivo.

La correspondencia la resuelven `to_dict()` y `from_dict()` a partir de una tabla declarativa:

```python
BLOCKS: dict[str, tuple[str, ...]] = {
    "": ("manifest_version",),
    "identity": ("name", "slug"),
    "classification": ("category", "type"),
    "classification.stack": ("language", "language_version", "framework",
                             "framework_version", "database"),
    "state": ("status", "priority"),
    "metadata": ("description", "tags", "repo", "notes"),
    "dates": ("created", "modified", "archived_at"),
    "relations": ("depends_on",),
}
```

`to_dict()` recorre la tabla y coloca cada atributo en su bloque; `from_dict()` hace el camino
inverso y falla con un error claro si falta una clave. Dos tests protegen el mapeo: la tabla cubre
exactamente los campos de la dataclass, y un ciclo de ida y vuelta no pierde datos.

### Contrato de `.project.json`

JSON con indentación de cuatro espacios y `ensure_ascii=False`:

```json
{
    "manifest_version": "1.0",
    "identity": {
        "name": "Auth Service",
        "slug": "auth-service"
    },
    "classification": {
        "category": "web",
        "type": "backend",
        "stack": {
            "language": "python",
            "language_version": "3.12",
            "framework": "fastapi",
            "framework_version": "0.110",
            "database": "postgresql"
        }
    },
    "state": {
        "status": "in_progress",
        "priority": "high"
    },
    "metadata": {
        "description": "Microservicio de autenticación JWT",
        "tags": ["auth", "jwt"],
        "repo": "",
        "notes": ""
    },
    "dates": {
        "created": "2026-09-21",
        "modified": "",
        "archived_at": ""
    },
    "relations": {
        "depends_on": []
    }
}
```

| Bloque JSON            | Campos de `Project`                                                          |
| ---------------------- | ---------------------------------------------------------------------------- |
| raíz                   | `manifest_version`                                                           |
| `identity`             | `name`, `slug`                                                               |
| `classification`       | `category`, `type`                                                           |
| `classification.stack` | `language`, `language_version`, `framework`, `framework_version`, `database` |
| `state`                | `status`, `priority`                                                         |
| `metadata`             | `description`, `tags`, `repo`, `notes`                                       |
| `dates`                | `created`, `modified`, `archived_at`                                         |
| `relations`            | `depends_on`                                                                 |

---

## Límites de responsabilidad

- Los modelos representan datos y relaciones; `Project` además convierte a y desde su JSON.
- `storage/` persiste cada modelo en su archivo y resuelve rutas.
- `services/` valida invariantes y coordina obsoletos, scaffold y rebuild.
- `commands/` orquesta los casos de uso.
- `meta/` se genera desde los modelos y nunca es fuente de datos.
