# 07. Plantillas Base

Contenido de las dos semillas que el CLI empaqueta en `resources/` y materializa en el almacén
global la primera vez que arranca: `templates/catalogs/base.yaml` y `templates/layouts/base.yaml`.
Ambas son inmutables en el almacén global (RN-601, RN-806) y sirven como origen para catálogos y
layouts custom y como copia inicial de cada workspace.

Los campos y sus invariantes se definen en [06-modelo-de-dominio.md](06-modelo-de-dominio.md).
Este documento solo fija los valores.

## Catálogo base (`templates/catalogs/base.yaml`)

```yaml
schema_version: "1.0"
catalog_id: "base"
name: "Catálogo base"
description: "Vocabulario inicial de Local Project Manager"

statuses:
    todo: "Pendiente de iniciar"
    in_progress: "En desarrollo activo"
    done: "Completado / en producción"
    paused: "Pausado temporalmente"
    archived: "Archivado / sin mantenimiento"

priorities:
    critical: "Bloquea otros proyectos"
    high: "Importante, próximo a completar"
    medium: "Progreso normal"
    low: "Baja prioridad, puede esperar"
    none: "Sin prioridad asignada"

categories:
    - web
    - mobile
    - desktop
    - automation
    - scraping
    - devops
    - shared
    - lab
    - archive

types_by_category:
    web: [backend, frontend, fullstack]
    mobile: [native, flutter, react-native]
    desktop: [gui, cli]
    automation: [script, bot, workflow]
    scraping: [static, crawler]
    devops: [docker, kubernetes, terraform]
    shared: [library, ui-component, snippet, template]
    lab: [course, kata, poc, experiment]
    archive: []

languages:
    - python
    - javascript
    - typescript
    - java
    - csharp
    - cpp
    - c
    - go
    - rust
    - php

frameworks_by_language:
    python: [django, fastapi, flask, tkinter, pyside6, celery, apache-airflow, scrapy, pytes, unittest]
    javascript: [react, vue, nodejs]
    typescript: [angular, react, nextjs, nestjs]
    java: [spring-boot, quarkus, micronaut]
    csharp: [aspnet-core, blazor, dotnet-maui]
    cpp: [qt, juce]
    c: [gtk, sdl]
    go: [gin, echo, fiber]
    rust: [axum, actix-web, tauri]
    php: [laravel, symfony]

default_tags_by_type:
    backend:
        [
            api,
            rest,
            graphql,
            auth,
            crud,
            middleware,
            jwt,
            oauth,
            microservice,
            websocket,
            grpc,
            cache,
            queue,
            migration,
            logging,
            rate-limit,
        ]
    frontend:
        [
            spa,
            pwa,
            ssr,
            ssg,
            component-library,
            design-system,
            dashboard,
            admin-panel,
            ecommerce,
            landing,
            form,
            table,
            chart,
            i18n,
            responsive,
            accessible,
        ]
    fullstack:
        [api+ui, ssr, monorepo, crud-app, saas, cms, blog, ecommerce, auth-flow, multi-tenant, realtime, bff, dashboard]
    native: [kotlin, android, jetpack, ios, maps, camera, push-notification, offline-first, auth, biometric]
    flutter: [android, ios, maps, camera, push-notification, offline-first, auth, biometric]
    react-native: [android, ios, maps, camera, push-notification, offline-first, auth, biometric]
    gui: [gui, electron, tauri, qt, desktop, tray-app, editor, file-manager, plugin, config-file]
    cli: [cli, interactive, tui, plugin, config-file, auto-complete]
    script: [script, one-off, scheduled, batch, backup, file-ops, transform, migration, cleanup]
    bot: [bot, notifier, chatbot, scheduled]
    workflow: [scheduled, batch, transform, monitoring, health-check, sync, scheduler]
    static: [parser, html, csv, json, xml, rss, sitemap, pdf]
    crawler: [crawler, headless, playwright, pagination, login-required, javascript-rendered, rate-limit]
    docker: [docker, compose, multi-stage, devcontainer, reverse-proxy, ssl, monitoring, logging]
    kubernetes: [kubernetes, helm, deployment, ingress, monitoring, logging]
    terraform: [terraform, aws, gcp, azure, ci-pipeline, infrastructure-as-code]
    library:
        [library, orm, http-client, validation, testing, utils, parser, serializer, auth, crypto, storage, date-time]
    ui-component: [ui-component, form, table, chart, component-library, design-system, headless]
    snippet: [utils]
    template: [template]
    course: [course, tutorial, workshop, exercises, learning]
    poc: [poc, prototype, proof-of-concept]
    experiment: [experiment, benchmark, comparison, research]
    kata: [kata, tdd, refactoring, exercises]
```

`archive` es una categoría sin tipos: sus proyectos se ubican directamente bajo `archive/` y su
campo `type` es cadena vacía (RN-104).

### Frameworks: referencia rápida

| Lenguaje   | Frameworks                                                                                                                                                                                                                                                                    |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| python     | django (web completa), fastapi (APIs), flask (minimalista), tkinter (GUI-basica), pyside6(), celery (Tareas asíncronas y distribuidas), apache airflow (Orquestación de workflows), scrapy (Testing general), pytes (Testing general), unittest (Testing incluido en Python), |
| javascript | react, vue (frontend), nodejs (backend y herramientas)                                                                                                                                                                                                                        |
| typescript | react, nextjs (full-stack), angular, nestjs (backend)                                                                                                                                                                                                                         |
| java       | spring-boot, quarkus, micronaut                                                                                                                                                                                                                                               |
| csharp     | aspnet-core, blazor, dotnet-maui                                                                                                                                                                                                                                              |
| cpp        | qt (desktop), juce (audio)                                                                                                                                                                                                                                                    |
| c          | gtk (GUI), sdl (multimedia)                                                                                                                                                                                                                                                   |
| go         | gin, echo, fiber                                                                                                                                                                                                                                                              |
| rust       | axum, actix-web (APIs), tauri (desktop)                                                                                                                                                                                                                                       |
| php        | laravel, symfony                                                                                                                                                                                                                                                              |

### Vocabulario de tags recomendado

`tags` es texto libre (RN-107). Las tablas siguientes son el vocabulario recomendado del que se
extraen las sugerencias de `default_tags_by_type`. Un workspace puede ampliar las sugerencias con
`workspace catalog add`.

**_web/backend_**

| Tag            | Descripción                                        |
| -------------- | -------------------------------------------------- |
| `api`          | Servicio accesible por API.                        |
| `rest`         | API basada en recursos HTTP.                       |
| `graphql`      | API con consultas tipadas.                         |
| `auth`         | Autenticación y autorización.                      |
| `crud`         | Operaciones de crear, leer, actualizar y eliminar. |
| `middleware`   | Intercepta peticiones o respuestas.                |
| `jwt`          | Autenticación mediante tokens JWT.                 |
| `oauth`        | Delegación de acceso mediante OAuth.               |
| `microservice` | Servicio independiente y desplegable.              |
| `websocket`    | Comunicación bidireccional en tiempo real.         |
| `grpc`         | Comunicación RPC eficiente.                        |
| `cache`        | Almacenamiento temporal para acelerar accesos.     |
| `queue`        | Procesamiento asíncrono mediante colas.            |
| `migration`    | Cambio versionado del esquema de datos.            |
| `logging`      | Registro de eventos de la aplicación.              |
| `rate-limit`   | Limita la frecuencia de peticiones.                |

**_web/frontend_**

| Tag                 | Descripción                                       |
| ------------------- | ------------------------------------------------- |
| `spa`               | Aplicación de una sola página.                    |
| `pwa`               | Aplicación web instalable.                        |
| `ssr`               | Renderizado del servidor.                         |
| `ssg`               | Sitio generado estáticamente.                     |
| `component-library` | Biblioteca reutilizable de componentes.           |
| `design-system`     | Reglas y componentes visuales compartidos.        |
| `dashboard`         | Interfaz de indicadores y métricas.               |
| `admin-panel`       | Panel para gestionar el sistema.                  |
| `ecommerce`         | Tienda o catálogo de comercio electrónico.        |
| `landing`           | Página orientada a conversión.                    |
| `form`              | Interfaz para capturar datos.                     |
| `table`             | Interfaz para mostrar datos tabulares.            |
| `chart`             | Visualización gráfica de datos.                   |
| `i18n`              | Soporte para varios idiomas.                      |
| `responsive`        | Adaptación a distintos tamaños de pantalla.       |
| `accessible`        | Diseñado para cumplir criterios de accesibilidad. |

**_web/fullstack_**

| Tag            | Descripción                              |
| -------------- | ---------------------------------------- |
| `api+ui`       | Backend y frontend integrados.           |
| `monorepo`     | Varios paquetes en un repositorio.       |
| `crud-app`     | Aplicación centrada en operaciones CRUD. |
| `saas`         | Software ofrecido como servicio.         |
| `cms`          | Sistema para gestionar contenidos.       |
| `blog`         | Sitio de publicaciones periódicas.       |
| `ecommerce`    | Solución de comercio electrónico.        |
| `auth-flow`    | Flujo completo de autenticación.         |
| `multi-tenant` | Aplicación con varios clientes aislados. |
| `realtime`     | Actualización de datos en tiempo real.   |
| `bff`          | Backend adaptado a un frontend.          |
| `dashboard`    | Interfaz de indicadores y métricas.      |

**_mobile_**

| Tag                 | Descripción                               |
| ------------------- | ----------------------------------------- |
| `kotlin`            | Desarrollo móvil con Kotlin.              |
| `android`           | Aplicación para Android.                  |
| `jetpack`           | Componentes modernos de Android.          |
| `ios`               | Aplicación para dispositivos Apple.       |
| `maps`              | Funcionalidad basada en mapas.            |
| `camera`            | Captura o procesamiento de cámara.        |
| `push-notification` | Notificaciones enviadas al dispositivo.   |
| `offline-first`     | Funciona priorizando el uso sin conexión. |
| `auth`              | Inicio de sesión y control de acceso.     |
| `biometric`         | Acceso mediante datos biométricos.        |

**_desktop_**

| Tag             | Descripción                                   |
| --------------- | --------------------------------------------- |
| `gui`           | Interfaz gráfica de usuario.                  |
| `electron`      | Aplicación desktop basada en tecnologías web. |
| `tauri`         | Aplicación desktop ligera multiplataforma.    |
| `qt`            | Aplicación multiplataforma con Qt.            |
| `desktop`       | Software ejecutado en escritorio.             |
| `tray-app`      | Aplicación residente en la bandeja.           |
| `editor`        | Herramienta para editar contenido o código.   |
| `file-manager`  | Gestión visual de archivos.                   |
| `cli`           | Interfaz ejecutada desde terminal.            |
| `interactive`   | Interacción continua con el usuario.          |
| `tui`           | Interfaz de usuario para terminal.            |
| `plugin`        | Extensión instalable del programa.            |
| `config-file`   | Configuración almacenada en archivo.          |
| `auto-complete` | Sugerencias automáticas de entrada.           |

**_automation_**

| Tag            | Descripción                                  |
| -------------- | -------------------------------------------- |
| `script`       | Automatización ejecutada como script.        |
| `one-off`      | Ejecución puntual sin recurrencia.           |
| `scheduled`    | Ejecución programada.                        |
| `batch`        | Procesamiento de un conjunto de datos.       |
| `backup`       | Copia de seguridad de información.           |
| `file-ops`     | Operaciones automatizadas sobre archivos.    |
| `transform`    | Conversión o modificación de datos.          |
| `migration`    | Traslado o actualización de datos.           |
| `monitoring`   | Observación continua de un sistema.          |
| `health-check` | Comprobación del estado de un servicio.      |
| `cleanup`      | Eliminación de datos temporales o sobrantes. |
| `sync`         | Sincronización entre fuentes.                |
| `bot`          | Automatización que interactúa con usuarios.  |
| `notifier`     | Envío automático de avisos.                  |
| `scheduler`    | Componente que coordina tareas programadas.  |
| `chatbot`      | Bot que conversa con usuarios.               |

**_scraping_**

| Tag                   | Descripción                                |
| --------------------- | ------------------------------------------ |
| `parser`              | Analiza y extrae datos estructurados.      |
| `html`                | Procesamiento de documentos HTML.          |
| `csv`                 | Entrada o salida en formato CSV.           |
| `json`                | Entrada o salida en formato JSON.          |
| `xml`                 | Entrada o salida en formato XML.           |
| `rss`                 | Consumo de fuentes RSS.                    |
| `sitemap`             | Descubrimiento mediante mapas del sitio.   |
| `pdf`                 | Extracción de datos desde PDF.             |
| `crawler`             | Recorre múltiples páginas automáticamente. |
| `headless`            | Navegador ejecutado sin interfaz visual.   |
| `playwright`          | Automatización con Playwright.             |
| `pagination`          | Procesamiento de páginas consecutivas.     |
| `login-required`      | Requiere autenticación para acceder.       |
| `javascript-rendered` | Procesa contenido generado con JavaScript. |
| `rate-limit`          | Respeta o controla límites de peticiones.  |

**_devops_**

| Tag                      | Descripción                                    |
| ------------------------ | ---------------------------------------------- |
| `docker`                 | Contenedorización con Docker.                  |
| `compose`                | Orquestación local de contenedores.            |
| `multi-stage`            | Imagen construida en varias etapas.            |
| `devcontainer`           | Entorno de desarrollo reproducible.            |
| `reverse-proxy`          | Proxy que recibe y enruta peticiones.          |
| `ssl`                    | Cifrado de comunicaciones HTTPS.               |
| `monitoring`             | Observabilidad de infraestructura o servicios. |
| `logging`                | Centralización o gestión de logs.              |
| `kubernetes`             | Orquestación de contenedores con Kubernetes.   |
| `helm`                   | Gestión de paquetes de Kubernetes.             |
| `deployment`             | Definición de despliegue automatizado.         |
| `ingress`                | Entrada de tráfico al clúster.                 |
| `terraform`              | Infraestructura definida como código.          |
| `aws`                    | Despliegue en Amazon Web Services.             |
| `gcp`                    | Despliegue en Google Cloud.                    |
| `azure`                  | Despliegue en Microsoft Azure.                 |
| `ci-pipeline`            | Integración continua automatizada.             |
| `infrastructure-as-code` | Infraestructura versionada como código.        |

**_shared_**

| Tag                 | Descripción                              |
| ------------------- | ---------------------------------------- |
| `library`           | Código reutilizable distribuible.        |
| `orm`               | Mapeo entre objetos y base de datos.     |
| `http-client`       | Cliente para consumir servicios HTTP.    |
| `validation`        | Comprobación de datos de entrada.        |
| `testing`           | Código orientado a pruebas.              |
| `utils`             | Utilidades generales reutilizables.      |
| `parser`            | Analizador de datos o texto.             |
| `serializer`        | Conversión entre objetos y formatos.     |
| `auth`              | Funcionalidad de autenticación y acceso. |
| `crypto`            | Operaciones criptográficas.              |
| `storage`           | Abstracción para almacenar datos.        |
| `date-time`         | Manejo de fechas y horas.                |
| `ui-component`      | Componente visual reutilizable.          |
| `form`              | Componente para entrada de datos.        |
| `table`             | Componente para datos tabulares.         |
| `chart`             | Componente para gráficos.                |
| `component-library` | Colección de componentes reutilizables.  |
| `design-system`     | Sistema visual y de interacción.         |
| `headless`          | Lógica sin estilos visuales impuestos.   |
| `template`          | Plantilla o esqueleto reutilizable.      |

**_lab_**

| Tag                | Descripción                              |
| ------------------ | ---------------------------------------- |
| `course`           | Material estructurado de aprendizaje.    |
| `tutorial`         | Guía práctica paso a paso.               |
| `workshop`         | Ejercicio práctico guiado.               |
| `exercises`        | Conjunto de ejercicios.                  |
| `poc`              | Prueba inicial de concepto.              |
| `prototype`        | Implementación preliminar de una idea.   |
| `proof-of-concept` | Valida la viabilidad técnica.            |
| `experiment`       | Exploración controlada de una hipótesis. |
| `benchmark`        | Medición comparativa de rendimiento.     |
| `comparison`       | Evaluación entre alternativas.           |
| `research`         | Investigación técnica o exploratoria.    |
| `kata`             | Ejercicio corto repetido para practicar. |
| `tdd`              | Desarrollo guiado por pruebas.           |
| `refactoring`      | Mejora del diseño sin cambiar conducta.  |

**_archive_**

| Tag          | Descripción                        |
| ------------ | ---------------------------------- |
| `deprecated` | Recomendado para no usar.          |
| `legacy`     | Código antiguo aún conservado.     |
| `completed`  | Proyecto finalizado.               |
| `abandoned`  | Proyecto sin continuidad prevista. |

**\*Transversales** (aplican a cualquier tipo)\*

| Tag            | Descripción                              |
| -------------- | ---------------------------------------- |
| `open-source`  | Código con licencia abierta.             |
| `personal`     | Proyecto de uso personal.                |
| `work`         | Proyecto relacionado con el trabajo.     |
| `freelance`    | Proyecto para un cliente independiente.  |
| `production`   | En uso real o productivo.                |
| `staging`      | Entorno previo a producción.             |
| `development`  | En desarrollo activo.                    |
| `learning`     | Creado con finalidad de aprendizaje.     |
| `dockerized`   | Ejecutable mediante contenedores Docker. |
| `documented`   | Cuenta con documentación suficiente.     |
| `test-covered` | Incluye cobertura de pruebas.            |
| `free`         | Disponible sin coste.                    |
| `paid`         | Requiere pago o licencia comercial.      |
| `spanish`      | Contenido principal en español.          |
| `english`      | Contenido principal en inglés.           |

---

## Layout base (`templates/layouts/base.yaml`)

```yaml
schema_version: "1.0"
layout_id: "base"
name: "Layout base"
description: "Estructura mínima común a cualquier proyecto"
directories: [src, docs, test]
files:
    README.md: |
        # {name}

        {description}

        ## Clasificación

        - Categoría: {category}
        - Tipo: {type}
        - Lenguaje: {language}

        ## Inicio rápido

        Pendiente de documentar.
    CHANGELOG.md: |
        # Changelog

        Formato basado en [Keep a Changelog](https://keepachangelog.com/es/1.1.0/).

        ## [Unreleased]

        ### Added

        - Estructura inicial del proyecto.
    .gitignore: |
        # Entornos y caché
        .venv/
        venv/
        __pycache__/
        *.pyc
        node_modules/

        # IDEs
        .idea/
        .vscode/
        *.swp

        # Build y distribución
        dist/
        build/
        *.egg-info/

        # Sistema
        .DS_Store
        Thumbs.db
init_git: false
```

Materializa, junto con el `.project.json` que escribe el CLI:

```text
<proyecto>/
├── .project.json
├── README.md
├── CHANGELOG.md
├── .gitignore
├── src/
├── docs/
└── test/
```

La copia `.layouts/base.yaml` de cada workspace puede editarse libremente; el original del almacén
global no.
