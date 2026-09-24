# 08. Interfaz CLI

La aplicación se invoca como `lpm` (o `python -m local_project_manager`). Tiene dos modos de uso
equivalentes sobre las mismas funciones de `commands/`:

- **Subcomandos** para uso directo y scripting: `lpm project new`, `lpm workspace rebuild`, etc.
- **Menú interactivo** cuando se ejecuta sin argumentos: navega por las mismas operaciones con
  asistentes paso a paso.

Toda operación sobre un workspace acepta `--workspace <ruta absoluta>`; sin ella se usa el
directorio actual (RF-103, RF-501). Los comandos de catálogos y layouts globales no necesitan
workspace.

## Subcomandos

### Workspace

| Subcomando                                          | Acción                                                                   | RF     |
| --------------------------------------------------- | ------------------------------------------------------------------------ | ------ |
| `workspace create [--path] [--catalog] [--pattern]` | Crea un workspace: catálogo, layout base, directorios e índices en cero. | RF-101 |
| `workspace rebuild`                                 | Regenera `meta/` completo.                                               | RF-102 |
| `workspace catalog show [--include-obsolete]`       | Muestra el catálogo activo con los obsoletos marcados.                   | RF-606 |
| `workspace catalog add <lista> <valor>`             | Añade un valor a una lista o mapa del catálogo.                          | RF-606 |
| `workspace catalog remove <lista> <valor>`          | Elimina físicamente o marca obsoleto según el uso.                       | RF-607 |
| `workspace catalog restore <lista> <valor>`         | Retira un valor de `obsolete_values`.                                    | RF-608 |
| `workspace catalog review`                          | Reconcilia `obsolete_values` con el uso real y muestra el resumen.       | RF-609 |
| `workspace layout list`                             | Lista los layouts de `.layouts/`.                                        | RF-702 |
| `workspace layout show <layout_id>`                 | Muestra un layout del workspace.                                         | RF-702 |
| `workspace layout create [--from <id>]`             | Crea un layout custom desde cero o a partir de otro.                     | RF-703 |
| `workspace layout edit <layout_id>`                 | Modifica un layout del workspace, incluido `base`.                       | RF-704 |
| `workspace layout delete <layout_id>`               | Elimina un layout custom del workspace. `base` no se elimina.            | RF-705 |
| `workspace layout import <layout_id>`               | Copia un layout del almacén global a `.layouts/`.                        | RF-706 |

Para `add`, `remove` y `restore`, `<lista>` es una de `statuses`, `priorities`, `categories`,
`types`, `languages`, `frameworks` o `tags`. Los mapas necesitan la clave padre: `types web
fullstack`, `frameworks python litestar`, `tags backend grpc`. `statuses` y `priorities` piden
además la etiqueta legible.

### Proyectos

| Subcomando                                                   | Acción                                                           | RF     |
| ------------------------------------------------------------ | ---------------------------------------------------------------- | ------ |
| `project new [--layout <id>]`                                | Asistente de creación, materialización del layout y rebuild.     | RF-201 |
| `project register <ruta>`                                    | Registra un proyecto existente escribiendo solo `.project.json`. | RF-205 |
| `project edit <slug>`                                        | Asistente de edición con aviso de valores obsoletos.             | RF-202 |
| `project move <slug> --category <c> [--type <t>]`            | Reubica el proyecto según el patrón del workspace.               | RF-203 |
| `project delete <slug>`                                      | Elimina el directorio previa confirmación.                       | RF-204 |
| `project list [--category] [--type] [--status] [--priority]` | Tabla filtrable de proyectos.                                    | RF-401 |
| `project search <término>`                                   | Búsqueda de texto en `name`, `description` y `tags`.             | RF-402 |
| `project show <slug>`                                        | Muestra la ficha del proyecto en terminal.                       | RF-303 |

### Catálogos globales

| Subcomando                     | Acción                                                                   | RF     |
| ------------------------------ | ------------------------------------------------------------------------ | ------ |
| `catalog list`                 | Lista `base` y los catálogos custom del almacén global.                  | RF-602 |
| `catalog show <catalog_id>`    | Muestra un catálogo global.                                              | RF-602 |
| `catalog create [--from <id>]` | Crea un catálogo custom desde cero o a partir de otro.                   | RF-603 |
| `catalog edit <catalog_id>`    | Modifica un catálogo custom; valida completo y sobrescribe si es válido. | RF-604 |
| `catalog delete <catalog_id>`  | Elimina un catálogo custom previa confirmación. `base` no se elimina.    | RF-605 |

### Layouts globales

| Subcomando                    | Acción                                                              | RF     |
| ----------------------------- | ------------------------------------------------------------------- | ------ |
| `layout list`                 | Lista `base` y los layouts custom del almacén global.               | RF-702 |
| `layout show <layout_id>`     | Muestra un layout global.                                           | RF-702 |
| `layout create [--from <id>]` | Crea un layout custom desde cero o a partir de otro.                | RF-703 |
| `layout edit <layout_id>`     | Modifica un layout custom global. `base` no se modifica.            | RF-704 |
| `layout delete <layout_id>`   | Elimina un layout custom previa confirmación. `base` no se elimina. | RF-705 |

## Menú interactivo

Al ejecutar `lpm` sin argumentos:

```text
1. Workspaces
   1. Crear workspace
      - Directorio destino (por defecto, el actual)
      - Catálogo global a usar, o crear uno nuevo antes
      - Patrón de rutas (por defecto {category}/{type}/{slug})
   2. Gestionar workspace
      - Ruta absoluta (por defecto, el actual); se valida .workspace.yaml
      1. Proyectos
         1. Listar (con filtros)
         2. Buscar
         3. Ver ficha (por slug)
         4. Nuevo proyecto
            - Elegir layout de .layouts/
            - Formulario guiado con valores vigentes
            - Validación, materialización y rebuild
         5. Registrar proyecto existente
         6. Modificar proyecto (por slug)
            - Campos a cambiar; aviso de obsoletos con mantener o actualizar
            - Guardar o cancelar
         7. Reubicar proyecto (por slug): nueva categoría y tipo
         8. Eliminar proyecto (por slug): confirmación explícita
         9. Volver
      2. Catálogo del workspace
         1. Ver catálogo activo
         2. Añadir valor
         3. Eliminar valor (borrado físico u obsoleto según uso)
         4. Restaurar valor obsoleto
         5. Revisar valores obsoletos (resumen antes de guardar)
         6. Volver
      3. Layouts del workspace
         1. Listar
         2. Ver layout
         3. Crear layout (desde cero o desde otro)
         4. Modificar layout
         5. Eliminar layout custom
         6. Importar layout del almacén global
         7. Volver
      4. Reconstruir meta/
      5. Volver
   3. Volver
2. Catálogos globales
   1. Listar
   2. Ver catálogo
   3. Crear catálogo custom (desde cero o desde otro)
   4. Modificar catálogo custom
   5. Eliminar catálogo custom
   6. Volver
3. Layouts globales
   1. Listar
   2. Ver layout
   3. Crear layout custom (desde cero o desde otro)
   4. Modificar layout custom
   5. Eliminar layout custom
   6. Volver
4. Salir
```

## Comportamiento común

- **Valores vigentes por defecto.** Toda selección de valores del catálogo ofrece solo los vigentes;
  `--include-obsolete` o la opción equivalente del menú añade los obsoletos marcados (RF-403).
- **Errores de validación completos.** Los formularios muestran todos los errores de una vez, no
  el primero, y permiten corregir sin reiniciar el asistente.
- **Confirmación en borrados.** Proyectos, catálogos custom y layouts custom piden confirmación
  explícita (RN-302).
- **Rebuild automático.** Las mutaciones de proyectos y las altas y bajas de categorías o tipos
  terminan regenerando `meta/` (RN-401).
- **Códigos de salida.** `0` éxito; `1` error de validación o de uso; `2` ruta no gestionada o
  archivo corrupto.
