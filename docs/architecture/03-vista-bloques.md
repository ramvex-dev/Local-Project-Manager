# 03. Vista de Bloques

Estructura estática del sistema: el almacén global del CLI, el workspace en disco, el proyecto que
genera cada `new`, el paquete Python y los documentos derivados de `meta/`.

Los campos exactos de cada archivo se especifican en
[06-modelo-de-dominio.md](06-modelo-de-dominio.md). El contenido de las plantillas base está en
[07-plantillas-base.md](07-plantillas-base.md).

## Relaciones del modelo

```text
Workspace ──contiene──────────▶ Catalog            (copia independiente del global)
Workspace ──dispone de────────▶ 1..* ProjectLayout (archivos en .layouts/)
Project   ──se valida contra──▶ Workspace.catalog + Workspace.obsolete_values
Project   ──se materializa con▶ un ProjectLayout elegido en `new`
Project   ──se ubica según────▶ Workspace.project_path_pattern
```

`.project.json` contiene datos originales. `meta/` contiene documentos derivados que se regeneran.
`.workspace.yaml` contiene el workspace y su catálogo, no los proyectos.

---

## Almacén global del CLI

Vive fuera de cualquier workspace, en `~/.local_project_manager/` o en la ruta indicada por
`LPM_HOME` para instalaciones portables. Se transporta entre máquinas copiando la carpeta.

```text
~/.local_project_manager/
└── templates/
    ├── catalogs/
    │   ├── base.yaml                  # catálogo base, inmutable (RN-601)
    │   └── custom/
    │       ├── web-projects.yaml      # catálogo custom, editable desde el CLI (RN-602)
    │       └── personal.yaml
    └── layouts/
        ├── base.yaml                  # layout base, inmutable (RN-806)
        └── custom/
            ├── python-package.yaml    # layout custom, editable desde el CLI
            └── web-app.yaml
```

El identificador de cada catálogo o layout es el nombre del archivo sin extensión. Que sea base o
custom se deriva de la carpeta, no se guarda dentro del YAML.

---

## Estructura del workspace

La estructura no es fija: la determinan el catálogo copiado y el patrón de rutas elegidos al crear
el workspace. Con el catálogo base y el patrón por defecto `{category}/{type}/{slug}`:

El diagrama visual [workspace-structure-diagram.puml](diagrams/workspace-structure-diagram.puml)
resume la relación entre la estructura física, los metadatos fuente y los índices derivados.

```text
<workspace>/
├── .workspace.yaml                    # Workspace: identidad, patrón, catálogo y obsoletos
├── .layouts/
│   ├── base.yaml                      # copia del layout global base, editable
│   └── python-package.yaml            # layouts custom de este workspace
│
├── web/
│   ├── backend/
│   │   └── auth-service/              # proyecto: {category}/{type}/{slug}
│   │       └── .project.json
│   ├── frontend/
│   └── fullstack/
├── mobile/
│   ├── native/
│   ├── flutter/
│   └── react-native/
├── desktop/
│   ├── gui/
│   └── cli/
├── automation/
│   ├── script/
│   ├── bot/
│   └── workflow/
├── scraping/
│   ├── static/
│   └── crawler/
├── devops/
│   ├── docker/
│   ├── kubernetes/
│   └── terraform/
├── shared/
│   ├── library/
│   ├── ui-component/
│   ├── snippet/
│   └── template/
├── lab/
│   ├── course/
│   ├── poc/
│   ├── experiment/
│   └── kata/
├── archive/                           # categoría sin tipos: los proyectos van directos
│
└── meta/                              # derivado, se regenera en cada rebuild
    ├── global-index.md
    └── category/
        ├── web/
        │   ├── category-index.md
        │   └── projects/              # solo si la categoría tiene proyectos (RF-304)
        │       └── auth-service.md
        ├── mobile/
        │   └── category-index.md
        └── ...                        # una carpeta por categoría del catálogo
```

Reglas de materialización:

- Al crear el workspace se crea un directorio por categoría y, si el patrón contiene `{type}`, uno
  por tipo dentro de su categoría. Una categoría sin tipos recibe los proyectos directamente.
- Al añadir una categoría o un tipo al catálogo del workspace se crean sus directorios y se
  regeneran los índices. Al eliminarlos no se borra ningún directorio.
- `meta/category/` tiene una carpeta por categoría del catálogo, incluidas las obsoletas mientras
  tengan proyectos.

---

## Estructura del proyecto generado

`new` materializa el layout elegido en la ruta que resulta del patrón del workspace. Con el layout
base:

```text
<workspace>/web/backend/auth-service/
├── .project.json                      # lo escribe siempre el CLI, nunca el layout (RN-802)
├── README.md                          # del layout, con marcadores sustituidos
├── CHANGELOG.md
├── .gitignore
├── src/
├── docs/
└── test/
```

Un layout custom puede omitir cualquiera de estos elementos salvo `.project.json`, añadir archivos
como `pyproject.toml` o `LICENSE`, declarar rutas con subdirectorios como `src/{slug}/__init__.py`
y activar `git init`.

`project register <ruta>` no materializa nada: solo escribe `.project.json` en un directorio ya
existente que respete el patrón de rutas (RF-205, RN-505).

---

## Estructura del paquete

```text
src/local_project_manager/
├── __init__.py               # __version__
├── __main__.py               # python -m local_project_manager
├── cli.py                    # app Typer: grupos de subcomandos, --workspace, menú si no hay args
├── config.py                 # resolución de rutas del workspace y del almacén global (RF-501)
├── constants.py              # nombres de archivos, MANIFEST_VERSION, marcadores, ids reservados
├── models.py                 # Catalog, Workspace, ProjectLayout, Project
│
├── storage/                  # persistencia: rutas, YAML, JSON, escritura atómica
│   ├── __init__.py
│   ├── atomic.py
│   ├── catalog_store.py
│   ├── layout_store.py
│   ├── workspace_store.py
│   └── project_store.py
│
├── services/                 # validación, obsoletos, scaffold, rebuild
│   ├── __init__.py
│   ├── catalog_validator.py
│   ├── layout_validator.py
│   ├── project_validator.py
│   ├── obsolete_service.py
│   ├── scaffold_service.py
│   └── rebuild_service.py
│
├── reports/                  # Markdown de meta/ con Jinja2
│   ├── __init__.py
│   ├── meta_io.py
│   └── templates/
│       ├── global-index.md.j2
│       ├── category-index.md.j2
│       └── project-data.md.j2
│
├── commands/                 # casos de uso, un módulo por grupo de subcomandos
│   ├── __init__.py
│   ├── workspace.py
│   ├── project.py
│   ├── catalog.py
│   └── layout.py
│
├── ui/                       # menú interactivo y formularios Rich
│   ├── __init__.py
│   ├── menu.py
│   └── prompts.py
│
├── utils/
│   ├── __init__.py
│   ├── slug.py
│   ├── dates.py
│   └── git.py
│
└── resources/                # semillas empaquetadas, leídas con importlib.resources
    ├── catalogs/base.yaml
    └── layouts/base.yaml
```

Las responsabilidades de cada módulo se describen en
[02-estrategia-solucion.md](02-estrategia-solucion.md).

---

## Ejemplos de reportes generados (`meta/`)

Mockups del contenido que produce `reports/meta_io.py` (RF-301 a RF-304), usados como referencia
estructural para las pruebas. Las etiquetas de estados y prioridades salen del catálogo del
workspace; los valores obsoletos se marcan con `(obsoleto)`.

### `meta/global-index.md`

```markdown
# GLOBAL INDEX

## 📊 Workspace Dashboard

| Status               | Count | #   | Priority | Count |
| -------------------- | ----- | --- | -------- | ----- |
| Pendiente de iniciar | 0     | #   | Critical | 0     |
| En desarrollo activo | 1     | #   | High     | 1     |
| Completado           | 0     | #   | Medium   | 0     |
| Pausado              | 0     | #   | Low      | 0     |
| Archivado            | 0     | #   | None     | 0     |
| **Total Projects**   | 1     | #   |          |       |

## 🗂️ Distribution

### By Category

| Category | Count |
| -------- | ----- |
| web      | 1     |

### By Language / Framework

| Language / Framework | Count |
| -------------------- | ----- |
| python / fastapi     | 1     |

## 🚀 All Projects

| Project                                     | Category | Type    | Stack            | Status      | Priority |
| ------------------------------------------- | -------- | ------- | ---------------- | ----------- | -------- |
| [Auth Service](../web/backend/auth-service) | web      | backend | python / fastapi | in_progress | high     |
```

### `meta/category/{category}/category-index.md`

```markdown
# WEB CATEGORY INDEX

## 📊 Category Dashboard

| Status                 | Count | #   | Priority | Count |
| ---------------------- | ----- | --- | -------- | ----- |
| Pendiente de iniciar   | 0     | #   | Critical | 0     |
| En desarrollo activo   | 1     | #   | High     | 1     |
| Completado             | 0     | #   | Medium   | 0     |
| Pausado                | 0     | #   | Low      | 0     |
| Archivado              | 0     | #   | None     | 0     |
| **Total Web Projects** | 1     | #   |          |       |

## 💻 Tech Stacks in this Category

| Type     | Count | #   | Language / Framework | Count |
| -------- | ----- | --- | -------------------- | ----- |
| backend  | 1     | #   | python / fastapi     | 1     |
| frontend | 0     | #   |                      |       |

## 📂 Web Projects Directory

| Project                                           | Type    | Stack            | Status      | Priority | Description                        |
| ------------------------------------------------- | ------- | ---------------- | ----------- | -------- | ---------------------------------- |
| [Auth Service](../../../web/backend/auth-service) | backend | python / fastapi | in_progress | high     | Microservicio de autenticación JWT |
```

### `meta/category/{category}/projects/{slug}.md`

```markdown
# Auth Service

## Identidad

- name: Auth Service
- slug: auth-service
- manifest_version: 1.0

## Clasificación

- category: web
- type: backend
- language: python 3.12
- framework: fastapi 0.110
- database: postgresql

## Estado

- status: in_progress (En desarrollo activo)
- priority: high (Importante, próximo a completar)

## Metadata

- description: Microservicio de autenticación JWT
- tags: auth, jwt
- repo:
- notes:

## Fechas

- created: 2026-09-21
- modified:
- archived_at:

## Relaciones

- depends_on: (ninguna)
```
