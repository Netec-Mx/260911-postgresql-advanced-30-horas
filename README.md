# PostgreSQL Advanced 30 Horas

Este curso proporciona los conocimientos y habilidades necesarios para administrar, optimizar y mantener bases de datos PostgreSQL en entornos empresariales. A lo largo del curso, los participantes conocerán la arquitectura de PostgreSQL, la administración y configuración del servidor, las estrategias de migración desde otros sistemas gestores de bases de datos, la implementación de mecanismos de respaldo y recuperación, la configuración de replicación y alta disponibilidad, así como las mejores prácticas de seguridad, monitoreo y optimización del rendimiento.

Asimismo, se abordará el uso de extensiones especializadas como PostGIS para el manejo de información geoespacial, permitiendo a los participantes conocer las capacidades de PostgreSQL en escenarios de Sistemas de Información Geográfica (SIG).

El curso combina sesiones teóricas con laboratorios prácticos que permiten aplicar los conocimientos adquiridos en un entorno controlado, simulando actividades propias de la administración de una base de datos PostgreSQL en ambientes empresariales.

## Estructura

- `CapituloXX/README.md`: guía de laboratorio por capítulo.

## Lista de laboratorios

### Capítulo 1

- [1 Instalación de PostgreSQL en Linux](Capitulo01/README.md#1-instalación-de-postgresql-en-linux)
  - Descripción: Actividad práctica orientada a la instalación de PostgreSQL en Linux, aplicando los conceptos de introducción, arquitectura e interacción básica con el servidor mediante psql.
  - Duración estimada: 88 min

### Capítulo 2

- [Prácticas 2.1 Administración y configuración de una instancia PostgreSQL](Capitulo02/README.md#prácticas-21-administración-y-configuración-de-una-instancia-postgresql)
  - Descripción: Actividad práctica orientada a la administración y configuración de una instancia PostgreSQL, incluyendo la gestión de bases de datos, la configuración del servidor, el control de acceso y la operación del clúster.
  - Duración estimada: 88 min

### Capítulo 3

- [Prácticas 3.1 Migración de una base de datos a PostgreSQL](Capitulo03/README.md#prácticas-31-migración-de-una-base-de-datos-a-postgresql)
  - Descripción: Actividad práctica orientada a la migración de una base de datos a PostgreSQL, aplicando estrategias y herramientas de migración y realizando la validación posterior de los resultados.
  - Duración estimada: 118 min

### Capítulo 4

- [1 Implementación de estrategias de respaldo y recuperación](Capitulo04/README.md#1-implementación-de-estrategias-de-respaldo-y-recuperación)
  - Descripción: Actividad práctica orientada a la implementación de estrategias de respaldo y recuperación en PostgreSQL mediante copias de seguridad lógicas y físicas, archivado del Write-Ahead Log y recuperación Point-in-Time.
  - Duración estimada: 118 min

### Capítulo 5

- [Prácticas 5.1 Replicación asíncrona y síncrona](Capitulo05/README.md#prácticas-51-replicación-asíncrona-y-síncrona)
  - Descripción: Actividad práctica orientada a configurar esquemas de replicación asíncrona y síncrona en PostgreSQL para incrementar la disponibilidad de los datos y la continuidad de la operación.
  - Duración estimada: 118 min

### Capítulo 6

- [Prácticas 6.1 Gestión de usuarios y autenticación, acceso remoto y permisos](Capitulo06/README.md#prácticas-61-gestión-de-usuarios-y-autenticación-acceso-remoto-y-permisos)
  - Descripción: Actividad práctica orientada a la gestión de usuarios y autenticación, el acceso remoto y la administración de permisos en PostgreSQL, aplicando mecanismos de seguridad del capítulo.
  - Duración estimada: 118 min

### Capítulo 7

- [Prácticas 7.1 Implementación de una base de datos Geoespacial con PostGIS](Capitulo07/README.md#prácticas-71-implementación-de-una-base-de-datos-geoespacial-con-postgis)
  - Descripción: Actividad práctica orientada a la implementación de una base de datos geoespacial con PostGIS, utilizando extensiones, tipos de datos espaciales y consultas espaciales.
  - Duración estimada: 88 min

### Capítulo 8

- [Prácticas 8.1 Actualización y optimización de PostgreSQL](Capitulo08/README.md#prácticas-81-actualización-y-optimización-de-postgresql)
  - Descripción: Actividad práctica orientada a la actualización y optimización de PostgreSQL mediante monitoreo, estadísticas de rendimiento, optimización de sentencias SQL, uso de estadísticas y ajuste de parámetros de configuración.
  - Duración estimada: 147 min

## Flujo de colaboración

- Trabajar en `changes_course`.
- Crear Pull Request hacia `main`.
- Merge por `Squash and merge`.
