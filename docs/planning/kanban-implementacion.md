# Kanban de Implementación

Checklist operativo para convertir el [plan de implementación](plan-implementacion.md) en issues de
GitHub Projects.

## Cómo usarlo

- Crea una issue por cada fase y añade sus subtareas como checklist.
- Usa las columnas `Backlog`, `Ready`, `In progress`, `In review` y `Done`.
- Mueve una fase a `Done` solo cuando su criterio de cierre esté cumplido.
- Marca cada tarea al terminarla y enlaza la pull request o los tests relacionados.
- Todas las fases forman parte del diseño completo; no hay fases futuras.
- Consulta la documentación de arquitectura para entender el diseño; este archivo solo sirve como
  checklist verificable.

## Fase 0 - Fundación y modelos

**Issue sugerida:** `F0 - Crear esqueleto y modelos del paquete`

- [x] Crear el layout `src/local_project_manager/`.
- [x] Crear `__init__.py` con `__version__`.
- [ ] Crear los paquetes `storage/`, `services/`, `reports/templates/`, `commands/`, `ui/`, `utils/` y `resources/`.
- [ ] Eliminar el paquete `generator/`.
- [ ] Actualizar `constants.py`: `catalogs`, `layouts`, `.layouts`, `MANIFEST_VERSION`, `BASE_ID`, patrón por defecto y marcadores; eliminar `BASE_TEMPLATE_REVISION`.
- [ ] Crear `models.py` con `Catalog`, `Workspace`, `ProjectLayout` y `Project`.
- [ ] Añadir `BLOCKS`, `Project.to_dict()` y `Project.from_dict()`.
- [ ] Añadir `empty_obsolete_values()` y el alias `ObsoleteValues`.
- [ ] Crear `config.py` con `resolve_workspace()` y `resolve_home()`.
- [ ] Crear `tests/conftest.py` con fixtures de almacén global y workspace temporales.
- [ ] Añadir `tests/test_models.py` (construcción, `BLOCKS` completo, round-trip).
- [ ] Ejecutar `pytest -q`, Ruff y mypy.

**Cierre:** el paquete importa correctamente, no quedan clases del diseño anterior y los tests de
modelos pasan con la cobertura objetivo.

## Fase 1 - Persistencia (`storage/`)

**Issue sugerida:** `F1 - Implementar stores y escritura atómica`

- [ ] Añadir `resources/catalogs/base.yaml` con el contenido del 07.
- [ ] Añadir `resources/layouts/base.yaml` con el contenido del 07.
- [ ] Implementar `storage/atomic.py`.
- [ ] Implementar `CatalogStore` (list, load, save, delete, ensure_base).
- [ ] Proteger `base` frente a `save` y `delete`.
- [ ] Implementar `LayoutStore` en ámbito global y de workspace.
- [ ] Implementar `WorkspaceStore` (exists, load, save con `updated_at`).
- [ ] Implementar `ProjectStore` (load, save, delete).
- [ ] Implementar `ProjectStore.scan()` ignorando `meta/` y `.layouts/`.
- [ ] Implementar `ProjectStore.resolve_path()` con el patrón de rutas.
- [ ] Rechazar YAML y JSON inválidos con ruta y campo en el error.
- [ ] Añadir tests de round-trip de los cuatro stores.
- [ ] Añadir tests de materialización de semillas y protección de `base`.
- [ ] Añadir tests de `scan` y `resolve_path` con los tres patrones del 04.
- [ ] Ejecutar `pytest -q`, Ruff y mypy.

**Cierre:** el almacén global se materializa desde cero y los cuatro archivos se leen y escriben
sin pérdida.

## Fase 2 - Validadores

**Issue sugerida:** `F2 - Implementar validadores de catálogo, layout y proyecto`

- [ ] Implementar `utils/slug.py` (`normalize_id`, `slugify`, `is_valid_id`).
- [ ] Implementar `utils/dates.py`.
- [ ] Implementar `catalog_validator` (normalización, unicidad, listas no vacías, claves coherentes).
- [ ] Implementar la validación de `project_path_pattern`.
- [ ] Implementar `layout_validator` (rutas relativas, sin `..`, sin `.project.json`, marcadores).
- [ ] Implementar `project_validator` (obligatorios, slug, tipo según categoría, vigentes u obsoletos asignados).
- [ ] Validar `framework` según RN-106 y `tags` como texto libre.
- [ ] Validar fechas ISO y `archived_at` solo con `archived`.
- [ ] Validar `depends_on` existentes y sin ciclos.
- [ ] Devolver todos los errores, no solo el primero.
- [ ] Añadir tests de casos válidos e inválidos por regla.
- [ ] Ejecutar `pytest -q`, Ruff y mypy.

**Cierre:** ningún catálogo, layout ni proyecto inválido supera su validador y cada error es
trazable a una RN.

## Fase 3 - Creación del workspace y rebuild

**Issue sugerida:** `F3 - Crear y reconstruir el workspace`

- [ ] Implementar `scaffold_service.create_category_dirs()` según el patrón.
- [ ] Implementar `rebuild_service.run()` con generación en temporal e intercambio atómico.
- [ ] Excluir manifiestos inválidos del rebuild con aviso.
- [ ] Implementar `open_workspace()` compartido (RF-103).
- [ ] Implementar `workspace create`: elegir catálogo global o crear uno.
- [ ] Implementar `workspace create`: elegir patrón de rutas.
- [ ] Copiar el catálogo a `.workspace.yaml` y el layout base a `.layouts/base.yaml`.
- [ ] Crear directorios de categorías y tipos.
- [ ] Generar índices iniciales en cero.
- [ ] Rechazar la creación si ya existe `.workspace.yaml`.
- [ ] Implementar `workspace rebuild`.
- [ ] Añadir tests de creación, rechazo y patrón custom.
- [ ] Añadir tests de rebuild en workspace vacío y con manifiesto corrupto.
- [ ] Ejecutar `pytest -q`, Ruff y mypy.

**Cierre:** `workspace create` deja un workspace válido y `workspace rebuild` deja `meta/`
consistente.

## Fase 4 - Motor de reportes

**Issue sugerida:** `F4 - Generar documentación Markdown`

- [ ] Crear `global-index.md.j2`, `category-index.md.j2` y `project-data.md.j2`.
- [ ] Configurar `PackageLoader` y `StrictUndefined`.
- [ ] Generar el índice global con conteos y tabla de proyectos.
- [ ] Generar un índice por cada categoría, incluidas las vacías y las obsoletas con proyectos.
- [ ] Generar una ficha por proyecto.
- [ ] Crear `projects/` solo cuando la categoría tenga proyectos.
- [ ] Traducir identificadores a etiquetas desde el catálogo del workspace.
- [ ] Marcar valores obsoletos en índices y fichas.
- [ ] Añadir tests de contenido Markdown con y sin proyectos.
- [ ] Ejecutar `pytest -q`, Ruff y mypy.

**Cierre:** todos los documentos derivados se generan desde los manifiestos y coinciden con los
mockups del 03.

## Fase 5 - Formularios, scaffold y Git

**Issue sugerida:** `F5 - Añadir interacción, materialización de layouts y Git`

- [ ] Implementar selección con valores vigentes y opción `include_obsolete`.
- [ ] Implementar el formulario de proyecto.
- [ ] Implementar el formulario de edición con aviso de obsoletos (mantener o actualizar).
- [ ] Mostrar listas completas de errores de validación.
- [ ] Comprobar terminal interactivo y fallo limpio en CI.
- [ ] Implementar `scaffold_service.materialize()` con marcadores en rutas y contenidos.
- [ ] Soportar llaves dobladas (`{{`, `}}`).
- [ ] Ejecutar `git init` cuando el layout lo indique.
- [ ] Implementar `utils/git.py` (`init`, `clone`) con `shutil.which` y sin `shell=True`.
- [ ] Añadir tests de prompts con entradas simuladas.
- [ ] Añadir tests de materialización (layout base, subdirectorios, TOML).
- [ ] Añadir tests de integración de Git.
- [ ] Ejecutar `pytest -q`, Ruff y mypy.

**Cierre:** los formularios solo ofrecen valores vigentes por defecto y un layout se materializa
como describe el 06.

## Fase 6 - Ciclo de vida de proyectos

**Issue sugerida:** `F6 - Implementar creación y gestión de proyectos`

- [ ] Implementar `project new`: elegir layout de `.layouts/`.
- [ ] Implementar `project new`: formulario, slug desde `name`, validación.
- [ ] Implementar `project new`: resolver ruta, clonar o materializar, escribir manifiesto.
- [ ] Implementar `project register <ruta>` escribiendo solo `.project.json`.
- [ ] Bloquear `register` si ya existe `.project.json` o la ruta no respeta el patrón.
- [ ] Implementar `project edit` sin permitir cambiar `slug`, `category` ni `type`.
- [ ] Actualizar `dates.modified` en `edit`.
- [ ] Implementar `project move` recalculando la ruta con el patrón.
- [ ] Implementar `project delete` con confirmación obligatoria.
- [ ] Implementar `project show`.
- [ ] Ejecutar rebuild tras cada mutación.
- [ ] Añadir tests de creación, registro, edición, movimiento, borrado y ficha.
- [ ] Añadir tests de no destrucción en `register`.
- [ ] Ejecutar `pytest -q`, Ruff y mypy.

**Cierre:** las mutaciones validan antes de escribir y mantienen `meta/` actualizado.

## Fase 7 - Catálogo del workspace y valores obsoletos

**Issue sugerida:** `F7 - Editar el catálogo del workspace con obsoletos`

- [ ] Implementar `obsolete_service.remove()` con comprobación de uso por campo.
- [ ] Implementar `obsolete_service.restore()`.
- [ ] Implementar `obsolete_service.review()` con informe previo.
- [ ] Implementar `workspace catalog show [--include-obsolete]`.
- [ ] Implementar `workspace catalog add` (listas y mapas con clave padre; etiqueta para estados y prioridades).
- [ ] Implementar `workspace catalog remove`.
- [ ] Implementar `workspace catalog restore`.
- [ ] Implementar `workspace catalog review` con resumen antes de guardar.
- [ ] Crear directorios y disparar rebuild al añadir categorías o tipos.
- [ ] Validar el catálogo resultante antes de guardar.
- [ ] Añadir tests de eliminar sin uso, con uso, restaurar y revisar en ambos sentidos.
- [ ] Añadir tests de que ningún `.project.json` cambia.
- [ ] Ejecutar `pytest -q`, Ruff y mypy.

**Cierre:** un valor en uso nunca desaparece, uno sin uso desaparece físicamente y ningún
manifiesto se reescribe.

## Fase 8 - Consultas y búsqueda

**Issue sugerida:** `F8 - Implementar project list y project search`

- [ ] Implementar `project list` sin filtros.
- [ ] Añadir filtros `--category`, `--type`, `--status` y `--priority` en AND.
- [ ] Implementar `project search` sobre `name`, `description` y `tags`.
- [ ] Mostrar resultados con Rich Table y obsoletos marcados.
- [ ] Mostrar mensaje claro cuando no haya resultados.
- [ ] Añadir tests de filtros combinados y de búsquedas con y sin coincidencias.
- [ ] Ejecutar `pytest -q`, Ruff y mypy.

**Cierre:** `list` y `search` permiten consultar todos los proyectos sin modificar datos.

## Fase 9 - Catálogos y layouts globales, layouts del workspace

**Issue sugerida:** `F9 - Gestionar plantillas globales y layouts del workspace`

- [ ] Implementar `catalog list` y `catalog show`.
- [ ] Implementar `catalog create` desde cero y con `--from`.
- [ ] Implementar `catalog edit` con validación completa y sobrescritura atómica.
- [ ] Implementar `catalog delete` con confirmación; proteger `base`.
- [ ] Implementar `layout list`, `show`, `create`, `edit` y `delete` globales; proteger `base`.
- [ ] Implementar `workspace layout list`, `show`, `create` y `edit` (incluido `base` del workspace).
- [ ] Implementar `workspace layout delete` protegiendo `base`.
- [ ] Implementar `workspace layout import` sin sobrescribir.
- [ ] Añadir editores guiados de catálogo y de layout en `ui/prompts.py`.
- [ ] Comprobar que los comandos globales funcionan sin workspace.
- [ ] Añadir tests de creación, edición, borrado, protección de `base` e `import`.
- [ ] Añadir test de que editar un global no cambia un workspace existente.
- [ ] Ejecutar `pytest -q`, Ruff y mypy.

**Cierre:** el ciclo completo de catálogos y layouts custom funciona con y sin workspace y `base`
es intocable en el almacén global.

## Fase 10 - CLI, menú y punto de entrada

**Issue sugerida:** `F10 - Conectar la CLI, el menú y los comandos`

- [ ] Crear la aplicación Typer con los grupos `workspace`, `project`, `catalog` y `layout`.
- [ ] Añadir la opción global `--workspace`.
- [ ] Añadir `--help` descriptivo en cada grupo y subcomando.
- [ ] Implementar códigos de salida `0`, `1` y `2`.
- [ ] Implementar `ui/menu.py` según el 08.
- [ ] Lanzar el menú cuando no hay argumentos.
- [ ] Crear `__main__.py` y comprobar `python -m local_project_manager` y `lpm`.
- [ ] Añadir tests con `CliRunner` (ayuda, salida, delegación, `--workspace` inválido).
- [ ] Añadir tests de navegación del menú con entradas simuladas.
- [ ] Ejecutar `pytest -q`, Ruff y mypy.

**Cierre:** todos los subcomandos aparecen en la ayuda, delegan correctamente y el menú llega a
cada uno de ellos.

## Fase 11 - Integración final

**Issue sugerida:** `F11 - Validar el flujo completo y publicar documentación`

- [ ] Crear un catálogo custom con `catalog create --from base`.
- [ ] Crear un workspace con ese catálogo y un patrón de rutas.
- [ ] Crear un layout custom del workspace con `workspace layout create --from base`.
- [ ] Crear varios proyectos con `project new` usando dos layouts.
- [ ] Registrar un proyecto existente con `project register`.
- [ ] Consultar con `project list` y `project search`.
- [ ] Editar y mover un proyecto.
- [ ] Eliminar del catálogo un valor en uso y comprobar que queda obsoleto.
- [ ] Ejecutar `workspace catalog review` y `workspace rebuild`.
- [ ] Eliminar el proyecto que usaba el valor y comprobar que `review` lo elimina.
- [ ] Verificar que `meta/` coincide con los manifiestos en cada paso.
- [ ] Verificar que editar el catálogo global no cambia el workspace.
- [ ] Actualizar README y CHANGELOG.
- [ ] Ejecutar la suite completa con cobertura.
- [ ] Ejecutar Ruff, mypy y pre-commit.
- [ ] Comprobar la instalación desde un entorno limpio.
- [ ] Marcar las fases completadas en la tabla del plan principal.

**Cierre:** el flujo completo pasa en un workspace temporal real y la documentación refleja el
estado final.

## Plantilla para una issue de GitHub

```markdown
## Objetivo

Implementar la Fase N - Título.

## Checklist

- [ ] Tarea 1
- [ ] Tarea 2
- [ ] Tarea 3

## Criterio de cierre

Descripción verificable del resultado esperado.

## Evidencias

- Pull request:
- Tests:
- Documentación:
```
