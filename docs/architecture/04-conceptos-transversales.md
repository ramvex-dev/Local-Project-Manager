# 04. Conceptos Transversales

Conceptos que atraviesan varios módulos y que todos los documentos y el código deben interpretar
igual. Los valores concretos del catálogo y del layout base están en
[07-plantillas-base.md](07-plantillas-base.md); los campos de cada archivo en
[06-modelo-de-dominio.md](06-modelo-de-dominio.md).

## Catálogo frente a layout

El sistema separa dos preguntas:

| Concepto     | Pregunta que responde                                       | Clase           | Sabe del otro |
| ------------ | ----------------------------------------------------------- | --------------- | ------------- |
| **Catálogo** | ¿Qué valores puede tener un proyecto?                       | `Catalog`       | No            |
| **Layout**   | ¿Qué directorios y archivos crea `new` dentro del proyecto? | `ProjectLayout` | No            |

El catálogo es vocabulario: estados, prioridades, categorías, tipos, lenguajes, frameworks y tags.
El layout es esqueleto: subdirectorios y archivos iniciales. Un workspace tiene un único catálogo y
uno o más layouts. La palabra **template** se reserva para el almacén global
(`templates/catalogs/`, `templates/layouts/`); dentro del workspace se habla de "catálogo del
workspace" y de "layouts del workspace".

## Almacén global y copias del workspace

El almacén global (`~/.local_project_manager/` o `$LPM_HOME`) contiene definiciones reutilizables.
Un workspace recibe **copias** de ellas y nunca vuelve a consultar el original:

| Origen global                        | Copia en el workspace                 | Cuándo se copia               | Evoluciona después                     |
| ------------------------------------ | ------------------------------------- | ----------------------------- | -------------------------------------- |
| `templates/catalogs/<id>.yaml`       | bloque `catalog` de `.workspace.yaml` | Al crear el workspace         | Sola, mediante `workspace catalog *`   |
| `templates/layouts/base.yaml`        | `.layouts/base.yaml`                  | Al crear el workspace         | Sola, mediante `workspace layout edit` |
| `templates/layouts/custom/<id>.yaml` | `.layouts/<id>.yaml`                  | Con `workspace layout import` | Sola                                   |

Consecuencias:

- Modificar un catálogo o layout global no cambia ningún workspace (RN-604).
- Modificar la copia de un workspace no cambia el original ni otros workspaces (RN-603).
- `source_catalog_id` en `.workspace.yaml` es solo informativo (RN-605).
- No existen revisiones, historial ni procedencia por proyecto. Cada archivo es la única versión de
  sí mismo. Para reutilizar los cambios de un workspace en otro hay que crear un catálogo o layout
  global nuevo a partir de él.

## Identificadores normalizados

Todos los identificadores del catálogo (`statuses`, `priorities`, `categories`, tipos, lenguajes,
frameworks y tags) y los `catalog_id`, `layout_id` y `slug` siguen la misma forma:

- Minúsculas, dígitos y guiones: `[a-z0-9-]`. Los tags admiten además `+` (`api+ui`).
- Sin espacios, sin acentos, sin guiones al inicio o al final, sin guiones dobles.
- Únicos dentro de su lista o mapa.

El `slug` se genera a partir de `name` con `unicodedata.normalize("NFKD")`, eliminación de marcas
diacríticas, minúsculas y sustitución de cualquier secuencia no válida por un guion. Si el resultado
ya existe en el workspace, el asistente pide otro nombre o un slug manual.

`statuses` y `priorities` son mapas de identificador a etiqueta legible. El identificador es lo que
se guarda en `.project.json`; la etiqueta es lo que muestran los reportes.

## Patrón de rutas

`project_path_pattern` decide dónde vive cada proyecto dentro del workspace. Se fija al crear el
workspace y no cambia después (RN-606).

| Patrón                     | Ruta de `auth-service` (web/backend) | Directorios creados al crear el workspace |
| -------------------------- | ------------------------------------ | ----------------------------------------- |
| `{category}/{type}/{slug}` | `web/backend/auth-service/`          | Uno por categoría y uno por tipo          |
| `{category}/{slug}`        | `web/auth-service/`                  | Uno por categoría                         |
| `projects/{slug}`          | `projects/auth-service/`             | `projects/`                               |

Reglas:

- Debe contener `{slug}`. Solo admite los marcadores `{category}`, `{type}` y `{slug}`.
- Si contiene `{category}` y el catálogo define tipos, debe contener también `{type}`; de lo
  contrario dos tipos de la misma categoría compartirían carpeta.
- Una categoría sin tipos resuelve `{type}` como segmento vacío: `archive/old-app/`.
- `project move` recalcula la ruta con el patrón y mueve el directorio.
- `project register` exige que la ruta indicada respete el patrón.

El patrón pertenece al workspace, no al layout. Un layout describe el interior del directorio del
proyecto; nunca decide dónde está (RN-801).

## Valores obsoletos

El catálogo del workspace cambia sin romper proyectos existentes mediante `obsolete_values`, una
estructura paralela con la misma forma que el catálogo que enumera los identificadores retirados:

```yaml
obsolete_values:
    statuses: []
    priorities: []
    categories: []
    types_by_category: {}
    languages: [ruby]
    frameworks_by_language: { python: [flask] }
    default_tags_by_type: {}
```

| Operación | Comando                     | Comprueba proyectos | Efecto                                                                    |
| --------- | --------------------------- | ------------------- | ------------------------------------------------------------------------- |
| Añadir    | `workspace catalog add`     | No                  | Agrega el valor a su lista o mapa. Disponible de inmediato.               |
| Eliminar  | `workspace catalog remove`  | Sí                  | Sin uso: borrado físico. Con uso: permanece y entra en `obsolete_values`. |
| Restaurar | `workspace catalog restore` | No                  | Sale de `obsolete_values` y vuelve a ofrecerse.                           |
| Revisar   | `workspace catalog review`  | Sí                  | Reconcilia `obsolete_values` con el uso real (ver abajo).                 |

Un valor obsoleto:

- No se ofrece en formularios de creación ni de edición para proyectos nuevos.
- Sigue siendo válido para los proyectos que ya lo tienen. Se muestra en fichas, listados y
  búsquedas marcado como `(obsoleto)`.
- Si se edita un proyecto que lo usa, el asistente lo indica y ofrece mantenerlo o cambiarlo.
- Las sugerencias interactivas lo excluyen por defecto y lo incluyen con `--include-obsolete`.

La **revisión** existe porque `.workspace.yaml` y los `.project.json` se pueden editar a mano:

- Un valor en `obsolete_values` que ya no usa ningún proyecto se retira de la lista y se elimina
  físicamente del catálogo, completando la limpieza que quedó pendiente.
- Un valor que algún proyecto usa pero que no está ni en el catálogo ni en `obsolete_values` se
  restituye al catálogo y se marca obsoleto, para que el proyecto no quede huérfano.

Ninguna de las cuatro operaciones reescribe un `.project.json` (RN-705). El estado de obsoleto de
un valor concreto de un proyecto se deriva al mostrarlo comparando con `obsolete_values`.

Eliminar una categoría o un tipo con proyectos sigue la misma regla: se marcan obsoletos y sus
directorios permanecen. `meta/category/<category>/` se sigue generando mientras haya proyectos.

## Marcadores de layout

Los contenidos de `files` y las rutas de `directories` y `files` de un layout admiten marcadores
que `new` sustituye con los datos del proyecto:

| Marcador        | Valor                           | Ejemplo        |
| --------------- | ------------------------------- | -------------- |
| `{name}`        | `identity.name`                 | `Auth Service` |
| `{slug}`        | `identity.slug`                 | `auth-service` |
| `{description}` | `metadata.description`          | texto libre    |
| `{category}`    | `classification.category`       | `web`          |
| `{type}`        | `classification.type`           | `backend`      |
| `{language}`    | `classification.stack.language` | `python`       |

- Una llave literal se escribe doblada: `{{` y `}}`. Es necesario en TOML con tablas inline
  (`authors = [{{ name = "..." }}]`) o en cualquier archivo con llaves.
- Un marcador desconocido invalida el layout en la validación, no en la materialización.
- El CLI no interpreta el contenido de los archivos: solo sustituye marcadores y escribe el texto.
- Un layout se materializa una sola vez, al crear el proyecto (RN-805).

## Validación

No hay esquemas declarativos. Cada clase tiene un validador en `services/` que devuelve la lista
completa de errores legibles:

| Validador           | Comprueba                                                                                                                                                                                                                                                                                    |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `catalog_validator` | Identificadores normalizados y únicos; `statuses`, `priorities`, `categories` y `languages` no vacíos; claves de `types_by_category` iguales a `categories`; claves de `frameworks_by_language` contenidas en `languages`; claves de `default_tags_by_type` contenidas en la unión de tipos. |
| `layout_validator`  | Rutas relativas sin `..`; ninguna ruta es `.project.json`; marcadores conocidos en rutas y contenidos; `layout_id` normalizado.                                                                                                                                                              |
| `project_validator` | Campos obligatorios; slug válido y único; `category`, `type`, `status`, `priority`, `language` y `framework` vigentes u obsoletos ya asignados; `type` vacío si la categoría no define tipos; fechas ISO; `archived_at` solo con `archived`; `depends_on` existentes y sin ciclos.           |
| Patrón de rutas     | Contiene `{slug}`; solo marcadores permitidos; `{type}` presente si hay `{category}` y el catálogo define tipos.                                                                                                                                                                             |

Los validadores se ejecutan siempre antes de guardar y también al cargar: un `.workspace.yaml` que
no valida convierte la ruta en "no gestionada" (RF-103); un `.project.json` que no valida se
reporta en `rebuild` y se excluye de los índices con un aviso.

## Escritura atómica

Todo archivo persistido se escribe en un temporal del mismo directorio y se renombra con
`os.replace` (RN-402). Aplica a `.project.json`, `.workspace.yaml`, catálogos, layouts y todos los
documentos de `meta/`. Los temporales no forman parte del modelo ni sobreviven a la operación.
`rebuild` genera todo `meta/` en un directorio temporal y lo intercambia al final, para que un
fallo a medias no deje índices inconsistentes.

## Resolución de rutas

| Ruta           | Precedencia                                    | Depende de un workspace                    |
| -------------- | ---------------------------------------------- | ------------------------------------------ |
| Workspace      | 1. `--workspace <ruta>`; 2. directorio actual  | Sí: debe contener `.workspace.yaml` válido |
| Almacén global | 1. `$LPM_HOME`; 2. `~/.local_project_manager/` | No                                         |

Ambas se calculan en `config.py` en tiempo de ejecución. Los comandos de catálogos y layouts
globales funcionan sin workspace.

## Documentos derivados

`meta/` es un caché legible de los `.project.json`. Se regenera completo tras cada mutación
(RN-401) y con `workspace rebuild`. Nunca se edita a mano y nunca es fuente de datos: si un índice
y un manifiesto discrepan, el manifiesto tiene razón.
