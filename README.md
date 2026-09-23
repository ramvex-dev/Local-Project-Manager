# Local Project Manager

> **Estado: Pre-Alpha / en desarrollo.** 
Local Project Manager (`lpm`) es una herramienta de línea de comandos, escrita en Python, para mantener ordenados todos los proyectos de desarrollo que una persona tiene en su máquina.

## Para qué sirve

Quien programa acumula con el tiempo decenas de proyectos repartidos por el disco: pruebas, cursos, herramientas propias, trabajos para clientes, repositorios clonados. Al cabo de unos meses aparecen siempre los mismos problemas:

- No se sabe dónde está cada cosa ni en qué estado quedó.
- Cada proyecto empieza de una forma distinta: unos con README, otros sin `.gitignore`, otros sin carpeta de tests.
- No hay una vista de conjunto: qué proyectos hay, cuáles están activos, con qué tecnologías, cuáles dependen de otros.
- Cuando se quiere poner orden, cambiar la clasificación rompe lo que ya existía.

`lpm` resuelve esto con un **workspace**: una carpeta raíz donde todos los proyectos siguen la misma clasificación, nacen con la misma estructura inicial y llevan una ficha de metadatos que la herramienta mantiene al día.

## Cómo lo resuelve

Tres ideas, cada una con su archivo:

| Idea           | Qué es                                                                                                                                       | Dónde vive                              |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------- |
| **Catálogo**   | El vocabulario permitido: estados, prioridades, categorías, tipos, lenguajes, frameworks y tags. Decide qué valores puede tener un proyecto. | `.workspace.yaml` del workspace         |
| **Layout**     | El esqueleto de un proyecto nuevo: carpetas y archivos iniciales (`README.md`, `.gitignore`, `src/`...). Decide con qué nace un proyecto.    | `.layouts/*.yaml` del workspace         |
| **Manifiesto** | La ficha de cada proyecto: nombre, categoría, tipo, estado, prioridad, stack, tags, fechas y dependencias. Es la única fuente de verdad.     | `.project.json` en la raíz del proyecto |

A partir de los manifiestos, `lpm` genera y regenera automáticamente una carpeta `meta/` con un índice global, un índice por categoría y una ficha Markdown por proyecto. Nunca se editan a mano: si un dato cambia, cambia en el manifiesto y `meta/` se reconstruye.

Los catálogos y layouts se guardan además como **plantillas reutilizables** en un almacén global del CLI (`~/.local_project_manager/`), fuera de cualquier workspace. Un workspace nuevo se crea copiando una plantilla, y a partir de ahí cada uno evoluciona por su cuenta. Se pueden crear catálogos y layouts propios para distintos tipos de workspace (trabajo, personal, formación) y reutilizarlos en varias máquinas copiando esa carpeta.

Y una regla que protege lo ya hecho: **el catálogo puede cambiar sin romper proyectos**. Si se retira un valor que algún proyecto sigue usando, no se borra, se marca como obsoleto. Deja de ofrecerse a proyectos nuevos, pero los existentes siguen siendo válidos y se pueden migrar cuando convenga.

## Cómo funciona

1. **Se crea el workspace** una vez, eligiendo el catálogo y el patrón de rutas (por defecto `{category}/{type}/{slug}`). `lpm` copia el catálogo a `.workspace.yaml`, el layout base a `.layouts/`, crea las carpetas de categorías y tipos y genera `meta/` vacío.
2. **Se crean proyectos** con un asistente que solo ofrece los valores del catálogo. `lpm` genera el `slug`, calcula la ruta con el patrón, materializa el layout elegido, escribe `.project.json` y opcionalmente ejecuta `git init` o clona un repositorio. Un proyecto que ya existía se puede registrar sin tocar sus archivos.
3. **Se gestiona el día a día**: editar la ficha, mover un proyecto a otra categoría, eliminarlo con confirmación, listar con filtros o buscar por texto. Cada cambio regenera `meta/`.
4. **Se adapta el catálogo** cuando hace falta: añadir un lenguaje nuevo, retirar un tipo que ya no se usa, restaurar uno obsoleto o reconciliar el catálogo con lo que los proyectos realmente usan.

Todo está disponible como subcomandos (`lpm project new`, `lpm workspace catalog add`...) y como menú interactivo al ejecutar `lpm` sin argumentos.

---

## Estructura del workspace

Al crear un workspace se elige un catálogo global, se copia a `.workspace.yaml` junto con el layout base, y se fija un **patrón de rutas** que decide dónde vive cada proyecto (por defecto `{category}/{type}/{slug}`). Con el catálogo base:

```sh
~/dev/                                 # workspace
├── .workspace.yaml                    # identidad, patrón de rutas, catálogo propio y valores obsoletos
├── .layouts/
│   ├── base.yaml                      # copia editable del layout global base
│   └── python-package.yaml            # layouts propios de este workspace
│
├── web/
│   ├── backend/
│   │   └── auth-service/              # proyecto: {category}/{type}/{slug}
│   ├── frontend/
│   └── fullstack/
├── mobile/
├── devops/
├── ...                                # una carpeta por categoría y tipo del catálogo
│
└── meta/                              # derivado; se regenera en cada operación
    ├── global-index.md                # dashboard e índice de todos los proyectos
    └── category/
        └── web/
            ├── category-index.md      # índice de la categoría
            └── projects/
                └── auth-service.md    # ficha del proyecto
```

`.project.json` es la única fuente de verdad de cada proyecto. `meta/` es un caché legible que nunca se edita a mano.

## Estructura de un proyecto

`project new` materializa el layout elegido en la ruta que resulta del patrón del workspace y escribe el manifiesto. Con el layout base:

```sh
web/backend/auth-service/
│                       # layout base
├── .project.json       # manifiesto del proyecto
├── README.md            
├── CHANGELOG.md
├── .gitignore
├── src/
├── docs/
└── test/
```

Un layout propio puede omitir cualquiera de estos elementos, añadir archivos como `pyproject.toml` o `LICENSE` con el contenido que se quiera, declarar rutas con subdirectorios como `src/{slug}/__init__.py` y activar `git init`.

## Almacén global del CLI

Las plantillas no viven dentro de ningún workspace. Residen en `~/.local_project_manager/`, o en la ruta indicada por `LPM_HOME` para instalaciones portables, y se transportan entre máquinas copiando esa carpeta:

```sh
~/.local_project_manager/
└── templates/
    ├── catalogs/
    │   ├── base.yaml              # catálogo base, inmutable
    │   └── custom/
    │       └── <catalog_id>.yaml  # catálogos propios, editables desde el CLI
    └── layouts/
        ├── base.yaml              # layout base, inmutable
        └── custom/
            └── <layout_id>.yaml   # layouts propios, editables desde el CLI
```

El identificador de cada plantilla es el nombre de su archivo. No representa una versión: `web-projects.yaml` y `personal.yaml` son dos catálogos distintos.

## Licencia

MPL-2.0 — ver [LICENSE](LICENSE). Historial de cambios en [CHANGELOG.md](CHANGELOG.md).
