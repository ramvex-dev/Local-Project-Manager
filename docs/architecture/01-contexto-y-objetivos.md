# 01. Contexto y Objetivos

Local Project Manager es una CLI en Python para crear y organizar workspaces de proyectos locales
de desarrollo. Cada workspace tiene su propio catálogo de valores (estados, prioridades,
categorías, tipos, lenguajes, frameworks y tags), uno o más layouts que definen el esqueleto de los
proyectos nuevos, y un sistema ligero de metadatos: un manifiesto `.project.json` por proyecto y
reportes Markdown generados en `meta/`.

Este documento reúne los Requisitos Funcionales (RF) y las Reglas de Negocio (RN) vigentes. Define
qué debe hacer el sistema, no cómo se implementa. El modelo de datos está en
[06-modelo-de-dominio.md](06-modelo-de-dominio.md), la estructura física en
[03-vista-bloques.md](03-vista-bloques.md), los conceptos compartidos en
[04-conceptos-transversales.md](04-conceptos-transversales.md) y la interfaz de usuario en
[08-interfaz-cli.md](08-interfaz-cli.md).

## Alcance

El sistema se apoya en dos conceptos independientes que viven en un **almacén global del CLI**
(`~/.local_project_manager/`, o `$LPM_HOME`), fuera de cualquier workspace:

| Concepto     | Qué define                                                                              | Archivo global                                   |
| ------------ | --------------------------------------------------------------------------------------- | ------------------------------------------------ |
| **Catálogo** | Los valores que puede tomar un proyecto. Ademas de la estructura de categorías y tipos. | `templates/catalogs/base.yaml` y `custom/*.yaml` |
| **Layout**   | Los directorios y archivos que `new` crea en cada proyecto.                             | `templates/layouts/base.yaml` y `custom/*.yaml`  |

Al crear un workspace, el CLI copia el catálogo elegido a `.workspace.yaml` y el layout base a
`.layouts/base.yaml`. Desde ese momento ambas copias evolucionan de forma independiente de sus
originales. No existen revisiones ni historial: cada archivo es la única versión de sí mismo.

Los proyectos existentes nunca se rompen cuando cambia el catálogo del workspace: un valor que
siguen usando no se elimina, se marca como obsoleto.

---

## Requisitos Funcionales (RF)

### RF-100: Gestión del Workspace

- **RF-101 (Creación del workspace):** El comando `workspace create` debe crear un workspace en el
  directorio indicado (por defecto, el directorio actual). Pedirá qué catálogo global usar (base o
  custom, con opción de crear uno nuevo antes) y qué patrón de rutas aplicar a los proyectos (por
  defecto `{category}/{type}/{slug}`). Después copiará el catálogo a `.workspace.yaml`, copiará el
  layout global base a `.layouts/base.yaml`, creará los directorios de categorías y tipos derivados
  del catálogo y del patrón, y generará los índices iniciales de `meta/` con todos los indicadores
  en cero. Si el directorio ya contiene un `.workspace.yaml`, el comando debe rechazar la operación
  sin modificar nada.

- **RF-102 (Reconstrucción):** El comando `workspace rebuild` debe escanear recursivamente todos
  los `.project.json` del workspace y regenerar de forma atómica `meta/global-index.md`, los
  `category-index.md` y las fichas de proyecto. Si no encuentra manifiestos, conserva o regenera los
  índices base con datos en cero y no crea fichas.

- **RF-103 (Apertura del workspace):** Todo comando que opere sobre un workspace debe aceptar su
  ruta absoluta mediante la opción `--workspace` y, en su ausencia, usar el directorio actual. Antes
  de operar debe comprobar que existe `.workspace.yaml` y que su contenido es válido; si no lo es,
  la ruta no se considera un workspace gestionado y el comando termina con un error explícito.

### RF-200: Ciclo de Vida de los Proyectos

- **RF-201 (Creación):** El comando `project new` debe ofrecer un asistente interactivo que:
  - (1) permita elegir un layout de `.layouts/`.
  - (2) recopile los campos obligatorios (`name`, `category`, `type` si la categoría define tipos, `status`, `priority`, `language`) y los opcionales (`description`, `tags`, `repo`, `depends_on`, resto del bloque `stack`).
  - (3) ofrezca únicamente los valores vigentes del catálogo del workspace.
  - (4) genere el `slug` a partir de `name`.
  - (5) valide el manifiesto.
  - (6) cree el directorio del proyecto en la ruta que resulte del patrón del workspace.
  - (7) escriba `.project.json` con `dates.created` en ISO 8601.
  - (8) materialice el layout elegido.
  - (9) ejecute un rebuild automático.

- **RF-202 (Edición):** El comando `project edit <slug>` debe permitir modificar `name`, `status`,
  `priority`, `description`, `tags`, `repo`, `notes`, `depends_on` y los campos del bloque `stack`.
  No permite cambiar `slug`, `category` ni `type` (RN-303). Si algún valor actual del proyecto está
  marcado como obsoleto, el asistente lo indicará y ofrecerá mantenerlo o cambiarlo por uno vigente.
  Al guardar actualiza `dates.modified` y ejecuta un rebuild automático.

- **RF-203 (Reubicación):** El comando `project move <slug>` debe permitir cambiar la `category` de
  un proyecto y, opcionalmente, su `type` dentro de la nueva categoría. Debe:
  - (1) recalcular la ruta con el patrón del workspace.
  - (2) mover el directorio físico.
  - (3) actualizar `category` y `type` en `.project.json`.
  - (4) ejecutar un rebuild automático.

- **RF-204 (Eliminación):** El comando `project delete <slug>` debe eliminar el directorio físico del
  proyecto, previa confirmación interactiva explícita (RN-302). No borra manualmente nada de
  `meta/`: el rebuild automático que dispara depura los documentos derivados.

- **RF-205 (Registro de proyecto existente):** El comando `project register <ruta>` debe permitir
  incorporar al workspace un proyecto ya iniciado. Recopila los mismos campos que RF-201, pero no
  materializa ningún layout, no inicializa ni clona Git y no modifica archivos existentes: solo
  escribe `.project.json` en la raíz indicada y ejecuta un rebuild. La ruta debe existir, ser un
  directorio dentro del workspace, coincidir con el patrón de rutas y no contener ya un
  `.project.json`.

### RF-300: Generación de Documentación y Reportes

- **RF-301 (Índice global):** El motor de reportes debe generar `meta/global-index.md` con conteos
  por `status`, conteos por `priority`, distribución por `category`, distribución por lenguaje y
  framework, y una tabla completa de todos los proyectos del workspace.

- **RF-302 (Índices por categoría):** Debe generar `meta/category/{category}/category-index.md` para
  cada categoría del catálogo del workspace, incluidas las marcadas como obsoletas mientras tengan
  proyectos, y aunque la categoría no tenga proyectos. En ausencia de proyectos, el archivo muestra
  métricas y listados en cero.

- **RF-303 (Fichas de proyecto):** Debe generar `meta/category/{category}/projects/{slug}.md` por
  cada proyecto, renderizando todos los campos del manifiesto en formato legible. Los valores
  obsoletos se marcan como tales en la ficha.

- **RF-304 (Materialización condicional de `projects/`):** Dentro de `meta/category/{category}/`, el
  subdirectorio `projects/` solo debe existir cuando la categoría tenga al menos un proyecto.

### RF-400: Consultas y Herramientas

- **RF-401 (Listado filtrado):** El comando `project list` debe mostrar una tabla con los proyectos
  del workspace, con filtros acumulables (AND) por `--category`, `--type`, `--status` y `--priority`.
  Los valores obsoletos se muestran marcados.

- **RF-402 (Búsqueda por texto):** El comando `project search <término>` debe buscar texto libre en
  `name`, `description` y `tags`. No admite filtros estructurados.

- **RF-403 (Sugerencias con obsoletos):** Toda entrada interactiva que ofrezca valores del catálogo
  debe sugerir por defecto solo los valores vigentes. Con la opción `--include-obsolete` incluirá
  también los obsoletos, marcados como tales.

- **RF-404 (Integración con Git):** Durante `project new`, si el layout elegido tiene `init_git`
  activo, el sistema ejecutará `git init` en el directorio creado. En cualquier caso ofrecerá la
  alternativa de clonar un repositorio remoto (`git clone <url>`) como directorio del proyecto; en
  ese caso el campo `repo` se completará con la URL y el layout no se materializará sobre el clon.

### RF-500: Configuración

- **RF-501 (Resolución de rutas):** El sistema debe resolver dos rutas de forma independiente y en
  tiempo de ejecución. La ruta del workspace: la opción `--workspace` si se indica, y si no el
  directorio actual. La ruta del almacén global: la variable de entorno `LPM_HOME` si está definida
  (instalaciones portables), y si no `~/.local_project_manager/`. La segunda no depende de que
  exista un workspace.

### RF-600: Gestión de Catálogos

- **RF-601 (Catálogo base):** El sistema debe garantizar la existencia de
  `templates/catalogs/base.yaml` en el almacén global, materializándolo desde el recurso empaquetado
  si no existe. Es inmutable y no puede eliminarse (RN-601).

- **RF-602 (Listado de catálogos globales):** El comando `catalog list` debe listar el catálogo base y
  los custom del almacén global, sin requerir un workspace.

- **RF-603 (Creación de catálogo custom):** El comando `catalog create` debe ofrecer un asistente
  para crear un catálogo custom desde cero o a partir de otro existente (base o custom), asignándole
  un `catalog_id` único. El resultado se valida completo antes de guardarse en `custom/`.

- **RF-604 (Modificación de catálogo custom):** El comando `catalog edit <catalog_id>` debe permitir
  añadir o quitar valores de un catálogo custom global. El catálogo resultante se valida completo y
  el archivo se sobrescribe solo si es válido. Ningún workspace existente cambia (RN-604).

- **RF-605 (Eliminación de catálogo custom):** El comando `catalog delete <catalog_id>` debe eliminar
  un catálogo custom global previa confirmación. `base.yaml` no puede eliminarse.

- **RF-606 (Catálogo del workspace: consulta y alta):** Los comandos `workspace catalog show` y
  `workspace catalog add` deben mostrar el catálogo activo, con los obsoletos marcados, y permitir
  añadir un valor a cualquiera de sus listas o mapas. El alta se valida y se guarda directamente en
  `.workspace.yaml`; si el valor es una categoría o un tipo, se crean sus directorios y se regeneran
  los índices.

- **RF-607 (Catálogo del workspace: baja):** El comando `workspace catalog remove` debe comprobar
  todos los `.project.json` del workspace antes de retirar un valor. Si ningún proyecto lo usa, lo
  elimina físicamente. Si alguno lo usa, lo conserva y lo añade a `obsolete_values` (RN-701).

- **RF-608 (Restauración de obsoletos):** El comando `workspace catalog restore` debe retirar un
  valor de `obsolete_values` para que vuelva a ofrecerse.

- **RF-609 (Revisión de obsoletos):** El comando `workspace catalog review` debe recorrer todos los
  `.project.json` y reconciliar `obsolete_values` con el uso real (RN-704), mostrando un resumen de
  los cambios antes de guardar `.workspace.yaml`.

### RF-700: Gestión de Layouts

- **RF-701 (Layout base):** El sistema debe garantizar la existencia de
  `templates/layouts/base.yaml` en el almacén global, materializándolo desde el recurso empaquetado
  si no existe. Es inmutable y no puede eliminarse (RN-806).

- **RF-702 (Listado de layouts):** `layout list` debe listar los layouts del almacén global sin
  requerir un workspace; `workspace layout list` debe listar los de `.layouts/`.

- **RF-703 (Creación de layout custom):** `layout create` y `workspace layout create` deben ofrecer
  un asistente para crear un layout desde cero o a partir de otro existente, con `layout_id` único
  en su ámbito. El resultado se valida antes de guardarse.

- **RF-704 (Modificación de layout):** `layout edit <layout_id>` y `workspace layout edit
<layout_id>` deben permitir modificar directorios, archivos, contenidos y `init_git`. El layout
  global base no se modifica; la copia `.layouts/base.yaml` del workspace sí. La modificación no
  reescribe ningún proyecto ya creado (RN-805).

- **RF-705 (Eliminación de layout custom):** `layout delete` y `workspace layout delete` deben
  eliminar un layout custom previa confirmación. Ni el global `base.yaml` ni `.layouts/base.yaml`
  pueden eliminarse.

- **RF-706 (Copia de layout global al workspace):** `workspace layout import <layout_id>` debe copiar
  un layout del almacén global a `.layouts/`, validándolo, sin sobrescribir uno existente con el
  mismo identificador.

---

## Reglas de Negocio (RN)

### RN-100: Integridad y Tipado de Datos

- **RN-101 (Única fuente de verdad):** `.project.json` en la raíz de cada proyecto contiene la
  información autoritativa. Los documentos de `meta/` son derivados de solo lectura y no deben
  editarse manualmente.

- **RN-102 (Restricciones del slug):** `slug` es único en el workspace, inmutable una vez creado y
  está formado exclusivamente por `[a-z0-9-]`. Coincide exactamente con el nombre del directorio
  físico del proyecto.

- **RN-103 (Categoría del catálogo):** `category` debe existir en `categories` del catálogo del
  workspace, o en su lista de obsoletos si el proyecto ya la tenía.

- **RN-104 (Tipo del catálogo):** `type` debe pertenecer a `types_by_category[category]`. Si la
  categoría no define tipos, `type` debe ser cadena vacía.

- **RN-105 (Consistencia de identificadores):** Todos los identificadores de un catálogo (estados,
  prioridades, categorías, tipos, lenguajes, frameworks y tags) son únicos dentro de su lista o
  mapa, están normalizados (`[a-z0-9-]`, más `+` en tags) y son estables: renombrar un
  identificador equivale a eliminar uno y añadir otro.

- **RN-106 (Lenguaje y framework):** `language` debe existir en `languages`. Si
  `frameworks_by_language` define una lista para ese lenguaje, `framework` debe ser uno de ellos o
  cadena vacía; si no la define, `framework` es texto libre. `language_version`,
  `framework_version` y `database` son texto libre.

- **RN-107 (Tags):** `tags` es texto libre normalizado. El catálogo solo sugiere tags según
  `default_tags_by_type`; no los restringe.

### RN-200: Taxonomía de Estados y Prioridades

- **RN-201 (Estados permitidos):** `status` debe pertenecer a `statuses` del catálogo del workspace,
  o a su lista de obsoletos si el proyecto ya lo tenía.

- **RN-202 (Prioridades permitidas):** `priority` debe pertenecer a `priorities` del catálogo del
  workspace, o a su lista de obsoletos si el proyecto ya la tenía.

### RN-300: Restricciones Arquitectónicas

- **RN-301 (Aislamiento de la metadata):** Los reportes generados residen siempre bajo `meta/`,
  nunca dentro de los directorios de proyectos.

- **RN-302 (Confirmación obligatoria en eliminación):** Todo borrado permanente de disco (proyectos,
  catálogos custom, layouts custom) requiere confirmación interactiva explícita.

- **RN-303 (Campos no editables):** `project edit` no puede modificar `slug`, `category` ni `type`.
  El `slug` es permanente (RN-102) y el cambio de clasificación es competencia exclusiva de
  `project move`.

- **RN-304 (Los modelos no tocan disco):** Las clases de dominio no conocen rutas absolutas, YAML ni
  JSON. La persistencia es responsabilidad exclusiva de la capa de almacenamiento.

### RN-400: Sincronización y Persistencia

- **RN-401 (Auto-reconstrucción obligatoria):** Toda operación que altere el número, ubicación o
  contenido de los proyectos (`new`, `register`, `edit`, `move`, `delete`) y toda alta o baja de
  categorías o tipos en el catálogo del workspace deben regenerar `meta/` al finalizar.

- **RN-402 (Escritura atómica):** Todo archivo persistido (`.project.json`, `.workspace.yaml`,
  catálogos, layouts y documentos de `meta/`) se escribe en un fichero temporal y se renombra al
  final. Ningún temporal forma parte del modelo ni sobrevive a la operación.

### RN-500: Integridad del Manifiesto

- **RN-501 (Campos obligatorios):** Son obligatorios `manifest_version`, `identity.name`,
  `identity.slug`, `classification.category`, `state.status`, `state.priority`,
  `classification.stack.language` y `dates.created`. `classification.type` es obligatorio cuando la
  categoría define tipos. El resto de campos son opcionales pero deben estar presentes con cadena
  vacía o lista vacía.

- **RN-502 (Formato de fechas):** `created`, `modified` y `archived_at` usan ISO 8601 (`YYYY-MM-DD`)
  o cadena vacía. `archived_at` solo tiene valor cuando `status` es `archived`. `modified` lo
  actualiza el CLI en cada edición.

- **RN-503 (Integridad de dependencias):** `relations.depends_on` es una lista de slugs. Cada slug
  debe existir en el workspace en el momento de la creación o edición. No se permiten referencias
  circulares ni a proyectos inexistentes.

- **RN-504 (Versión del manifiesto):** `manifest_version` identifica la versión del formato de
  `.project.json`, no la del software del proyecto. Su valor actual es `"1.0"`.

- **RN-505 (No destrucción al registrar):** `project register` no puede crear, borrar, mover ni
  sobrescribir archivos distintos de `.project.json`. Si ya existe uno, la operación se bloquea.

### RN-600: Gobernanza de Catálogos

- **RN-601 (Catálogo base inmutable):** `templates/catalogs/base.yaml` no se edita ni se elimina. Es
  la referencia inicial y el origen clonable para catálogos custom.

- **RN-602 (Catálogos custom sin revisiones):** Un catálogo custom global es un único archivo
  editable desde el CLI. Cada modificación se valida completa antes de sobrescribirlo. No tiene
  revisiones ni historial de auditoría. Su `catalog_id` coincide con el nombre del archivo y no
  representa una versión.

- **RN-603 (Copia independiente del workspace):** Al crear el workspace se copia el catálogo elegido
  a `.workspace.yaml`. Esa copia evoluciona sola: no genera revisiones y nunca modifica el catálogo
  global de origen.

- **RN-604 (Los cambios globales no se propagan):** Modificar un catálogo o un layout global no
  actualiza ningún workspace existente. El cambio solo se aprovecha en workspaces nuevos o mediante
  una copia explícita.

- **RN-605 (Origen informativo):** `.workspace.yaml` conserva `source_catalog_id` únicamente como
  información. El CLI nunca vuelve a leer el catálogo global para resolver ni validar un proyecto.

- **RN-606 (Patrón de rutas fijo):** `project_path_pattern` se elige al crear el workspace y no se
  modifica después, porque implicaría mover todos los proyectos. Debe contener `{slug}` y, si el
  catálogo define tipos, debe contener `{type}` siempre que contenga `{category}`.

### RN-700: Valores Obsoletos

- **RN-701 (Sin borrado físico si hay uso):** Un valor del catálogo del workspace solo se elimina
  físicamente si ningún `.project.json` lo usa. En caso contrario permanece en su lista o mapa y se
  añade a `obsolete_values`.

- **RN-702 (Obsoleto pero válido):** Un valor en `obsolete_values` no se ofrece en formularios para
  proyectos nuevos, pero sigue siendo válido para los proyectos que ya lo tienen, se muestra en sus
  fichas y entra en búsquedas y listados.

- **RN-703 (Restauración explícita):** Reactivar un valor obsoleto es una operación explícita que lo
  retira de `obsolete_values`.

- **RN-704 (Reconciliación):** La revisión de obsoletos recorre todos los `.project.json` y: (a)
  retira de `obsolete_values` y elimina físicamente los valores que ya no usa ningún proyecto; (b)
  restituye al catálogo y marca como obsoletos los valores que algún proyecto usa pero que faltan en
  el catálogo y en `obsolete_values`.

- **RN-705 (Los proyectos no se reescriben):** Ni añadir, ni eliminar, ni restaurar, ni revisar
  valores del catálogo modifica ningún `.project.json`. El estado de obsoleto de un valor de un
  proyecto se deriva al mostrarlo, nunca se escribe en el manifiesto.

### RN-800: Layouts

- **RN-801 (El layout no conoce su ubicación):** Un layout describe el interior del directorio de un
  proyecto. Dónde se ubica ese directorio lo decide `project_path_pattern` del workspace.

- **RN-802 (`.project.json` reservado):** Ningún layout puede declarar `.project.json`. Lo escribe
  siempre el CLI.

- **RN-803 (Rutas relativas seguras):** Las rutas de `directories` y `files` son relativas al
  directorio del proyecto, no contienen `..` ni son absolutas. Las carpetas intermedias se crean
  automáticamente.

- **RN-804 (Marcadores):** El contenido de `files` y las rutas de `directories` y `files` admiten los
  marcadores `{name}`, `{slug}`, `{description}`, `{category}`, `{type}` y `{language}`. Una llave
  literal se escribe doblada (`{{`, `}}`). Un marcador desconocido invalida el layout.

- **RN-805 (Materialización única):** Un layout se materializa una sola vez, al crear el proyecto.
  Modificar el layout después no reescribe ningún proyecto existente; editar un proyecto tampoco
  vuelve a materializarlo.

- **RN-806 (Layout base inmutable en el almacén global):** `templates/layouts/base.yaml` no se edita
  ni se elimina. La copia `.layouts/base.yaml` de cada workspace sí es editable, pero no se elimina.
