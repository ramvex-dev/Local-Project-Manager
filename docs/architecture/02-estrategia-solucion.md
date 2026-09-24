# 02. Estrategia de Solución

Responsabilidades, modularidad y flujo de datos del paquete `local_project_manager`, alineados con
los RF y RN de [01-contexto-y-objetivos.md](01-contexto-y-objetivos.md). El contrato de los modelos
está en [06-modelo-de-dominio.md](06-modelo-de-dominio.md), la estructura física en
[03-vista-bloques.md](03-vista-bloques.md) y las tecnologías en
[05-stack-tecnologico.md](05-stack-tecnologico.md).

## Principios

1. **Dos conceptos, cuatro clases.** El catálogo (vocabulario) y el layout (esqueleto) son
   independientes. El workspace une ambos y el proyecto es el resultado. No hay más clases de
   dominio que `Catalog`, `Workspace`, `ProjectLayout` y `Project`.
2. **El esquema es la clase.** No existe ningún esquema declarativo en YAML ni JSON. El contrato de
   cada archivo lo definen su dataclass y su validador. Las reglas cruzadas (tipos dentro de
   categorías, frameworks dentro de lenguajes) viven en código.
3. **Copiar, no referenciar.** El workspace recibe copias del catálogo y del layout base y nunca
   vuelve a consultar el almacén global. No hay revisiones, historial ni procedencia por proyecto.
4. **Nada se rompe por cambiar el catálogo.** Un valor en uso no se elimina, se marca obsoleto.
   Ninguna operación de catálogo reescribe un `.project.json`.
5. **Los modelos no tocan disco.** Rutas, YAML y JSON son responsabilidad exclusiva de `storage/`.
   Los servicios validan y coordinan. Los comandos orquestan. La interfaz solo pregunta y muestra.

---

## Capas

```text
cli.py ─▶ ui/ ─▶ commands/ ─▶ services/ ─▶ storage/ ─▶ disco
                     │            │
                     └── models.py ┘        reports/ ◀── rebuild_service
```

| Capa         | Paquete         | Conoce                                  | No conoce                         |
| ------------ | --------------- | --------------------------------------- | --------------------------------- |
| Dominio      | `models.py`     | Campos, relaciones, `to_dict/from_dict` | Rutas, disco, terminal            |
| Persistencia | `storage/`      | Rutas, YAML, JSON, escritura atómica    | Reglas de negocio, terminal       |
| Servicios    | `services/`     | Invariantes, reconciliación, scaffold   | Terminal, formato de los archivos |
| Reportes     | `reports/`      | Jinja2 y Markdown de `meta/`            | Reglas de negocio                 |
| Casos de uso | `commands/`     | Orquestación de servicios y stores      | Formato de archivos               |
| Interfaz     | `cli.py`, `ui/` | Typer, Rich, menú y formularios         | Disco                             |

---

## Dominio (`models.py`)

| Clase           | Representa                                                      | Persistencia                                                        |
| --------------- | --------------------------------------------------------------- | ------------------------------------------------------------------- |
| `Catalog`       | Vocabulario de valores permitidos.                              | `templates/catalogs/*.yaml` y bloque `catalog` de `.workspace.yaml` |
| `Workspace`     | Identidad del workspace, patrón de rutas, catálogo y obsoletos. | `.workspace.yaml`                                                   |
| `ProjectLayout` | Directorios y archivos iniciales que materializa `new`.         | `templates/layouts/*.yaml` y `.layouts/*.yaml`                      |
| `Project`       | Fuente de verdad de un proyecto.                                | `.project.json`                                                     |

Las cuatro son `dataclasses` planas. `Project` expone `to_dict()` y `from_dict()` para convertir
entre la clase plana y el JSON anidado mediante una tabla declarativa `BLOCKS` (ver 06). Las otras
tres se serializan campo a campo.

---

## Persistencia (`storage/`)

Un store por archivo. Cada uno resuelve rutas a partir de `config.py`, lee y escribe con
`yaml.safe_load`/`safe_dump` o `json`, y delega la escritura en `atomic.py` (RN-402).

| Módulo               | Clase            | Responsabilidad                                                                                                                                                       |
| -------------------- | ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `atomic.py`          |                  | `write_text(path, content)`: temporal en el mismo directorio y `os.replace`.                                                                                          |
| `catalog_store.py`   | `CatalogStore`   | Listar, cargar, guardar y borrar catálogos globales. Materializa `base.yaml` desde `resources/`. Rechaza escribir sobre `base.yaml`.                                  |
| `layout_store.py`    | `LayoutStore`    | Igual que el anterior para layouts, en dos ámbitos: almacén global y `.layouts/` del workspace.                                                                       |
| `workspace_store.py` | `WorkspaceStore` | Cargar y guardar `.workspace.yaml`. `exists(path)` decide si una ruta es un workspace gestionado.                                                                     |
| `project_store.py`   | `ProjectStore`   | Cargar, guardar y borrar `.project.json`; `scan()` recorre el workspace y devuelve todos los `Project`; `resolve_path(workspace, project)` aplica el patrón de rutas. |

Los stores devuelven modelos ya construidos, nunca diccionarios. Un archivo con forma inválida
produce un error de carga con la ruta y el campo que falla.

---

## Servicios (`services/`)

| Módulo                 | Responsabilidad                                                                                                                                                                                        | RN principales                           |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------- |
| `catalog_validator.py` | Invariantes de `Catalog`: normalización, unicidad, listas obligatorias no vacías, claves de mapas coherentes.                                                                                          | RN-105, RN-106                           |
| `layout_validator.py`  | Invariantes de `ProjectLayout`: rutas relativas sin `..`, sin `.project.json`, marcadores conocidos.                                                                                                   | RN-802, RN-803, RN-804                   |
| `project_validator.py` | `Project` frente a `Workspace`: valores vigentes u obsoletos ya asignados, slug, tipo según categoría, fechas, dependencias sin ciclos.                                                                | RN-102..107, RN-201, RN-202, RN-501..504 |
| `obsolete_service.py`  | `remove(value)`, `restore(value)`, `review()`: comprueban el uso real en los proyectos y ajustan `obsolete_values`.                                                                                    | RN-701..705                              |
| `scaffold_service.py`  | Resuelve la ruta del proyecto con el patrón, materializa un `ProjectLayout` sustituyendo marcadores y ejecuta `git init` si procede. También crea los directorios de categorías y tipos del workspace. | RN-606, RN-801, RN-805                   |
| `rebuild_service.py`   | Escanea proyectos, calcula agregados y llama a `reports/` para regenerar `meta/` de forma atómica.                                                                                                     | RN-401, RN-402                           |

Los validadores devuelven una lista de errores legibles, no lanzan en el primer fallo, para que la
interfaz muestre todos los problemas de un formulario a la vez.

---

## Reportes (`reports/`)

`meta_io.py` renderiza con Jinja2 las plantillas de `reports/templates/`:

| Plantilla              | Salida                                        | RF     |
| ---------------------- | --------------------------------------------- | ------ |
| `global-index.md.j2`   | `meta/global-index.md`                        | RF-301 |
| `category-index.md.j2` | `meta/category/{category}/category-index.md`  | RF-302 |
| `project-data.md.j2`   | `meta/category/{category}/projects/{slug}.md` | RF-303 |

Recibe los `Project` ya cargados y el `Workspace` para traducir identificadores a etiquetas
(`statuses`, `priorities`) y para marcar valores obsoletos. Jinja2 se usa exclusivamente aquí: los
catálogos y los layouts son datos YAML, no plantillas Jinja.

---

## Casos de uso (`commands/`)

Un módulo por grupo de subcomandos de Typer. Cada función orquesta stores y servicios y delega en
`ui/` toda interacción.

| Módulo         | Subcomandos                                                                                                 | RF                                    |
| -------------- | ----------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| `workspace.py` | `create`, `rebuild`, `catalog show/add/remove/restore/review`, `layout list/show/create/edit/delete/import` | RF-101..103, RF-606..609, RF-702..706 |
| `project.py`   | `new`, `register`, `edit`, `move`, `delete`, `list`, `search`                                               | RF-201..205, RF-401..404              |
| `catalog.py`   | `list`, `show`, `create`, `edit`, `delete` (almacén global)                                                 | RF-601..605                           |
| `layout.py`    | `list`, `show`, `create`, `edit`, `delete` (almacén global)                                                 | RF-701..705                           |

Los comandos que mutan proyectos o categorías terminan llamando a `rebuild_service` (RN-401). Los
comandos de borrado piden confirmación mediante `ui/` (RN-302).

---

## Interfaz (`cli.py`, `ui/`)

- `cli.py` declara la aplicación Typer, los grupos de subcomandos y la opción global `--workspace`.
  Sin argumentos lanza el menú interactivo.
- `ui/menu.py` implementa el menú descrito en [08-interfaz-cli.md](08-interfaz-cli.md). Cada
  opción invoca la misma función de `commands/` que el subcomando equivalente.
- `ui/prompts.py` construye los formularios guiados con Rich: selección entre valores vigentes,
  marcado de obsoletos, confirmaciones y visualización de errores de validación.

---

## Apoyo (`config.py`, `constants.py`, `utils/`, `resources/`)

- `config.py`: resuelve la ruta del workspace (`--workspace` o cwd) y la del almacén global
  (`LPM_HOME` o `~/.local_project_manager/`). Materializa la estructura del almacén si no existe.
- `constants.py`: nombres de archivos y directorios, `MANIFEST_VERSION`, lista de marcadores,
  identificadores reservados (`base`).
- `utils/slug.py`: genera y valida slugs (`unicodedata` + `re`).
- `utils/dates.py`: fechas ISO 8601.
- `utils/git.py`: `git init` y `git clone` mediante `subprocess`.
- `resources/`: `catalogs/base.yaml` y `layouts/base.yaml` empaquetados, leídos con
  `importlib.resources` para materializar el almacén global la primera vez.

---

## Flujos principales

### Crear un workspace

1. `ui` pide directorio, catálogo global y patrón de rutas.
2. `CatalogStore.load(catalog_id)` y `catalog_validator` comprueban el catálogo.
3. Se construye `Workspace` con el catálogo copiado, `obsolete_values` vacío y las fechas.
4. `WorkspaceStore.save()` escribe `.workspace.yaml`.
5. `LayoutStore.load_global("base")` y `LayoutStore.save_workspace()` crean `.layouts/base.yaml`.
6. `scaffold_service.create_category_dirs()` crea los directorios de categorías y tipos.
7. `rebuild_service.run()` genera los índices en cero.

### Crear un proyecto

1. `WorkspaceStore.load()` valida que la ruta es un workspace.
2. `ui` ofrece los layouts de `.layouts/` y los valores vigentes del catálogo.
3. Se construye `Project`; `project_validator` lo comprueba contra `Workspace`.
4. `ProjectStore.resolve_path()` calcula la ruta con el patrón.
5. `scaffold_service.materialize(layout, project, path)` crea directorios y archivos, y `git init`
   si el layout lo pide.
6. `ProjectStore.save()` escribe `.project.json`.
7. `rebuild_service.run()`.

### Eliminar un valor del catálogo del workspace

1. `ProjectStore.scan()` carga todos los proyectos.
2. `obsolete_service.remove(value)` decide: sin uso, eliminación física; con uso, alta en
   `obsolete_values`.
3. `catalog_validator` comprueba el catálogo resultante.
4. `WorkspaceStore.save()`. Si el valor era una categoría o un tipo, `rebuild_service.run()`.
