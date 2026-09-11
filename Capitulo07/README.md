# Prácticas 7.1 Implementación de una base de datos Geoespacial con PostGIS

## Metadatos

| Elemento | Valor |
|---|---|
| Duración | 88 minutos |
| Complejidad | Media |
| Nivel de Bloom | Crear |

## Descripción general

En este laboratorio se habilitará PostGIS 3.4.2 en la base de datos `ventas_seguras` y se incorporará un modelo geoespacial asociado a los pedidos comerciales existentes. Se crearán capas espaciales para sucursales, zonas de reparto y puntos de entrega, usando los tipos `geometry` y `geography` con el sistema de referencia EPSG:4326.

También se implementarán índices espaciales GiST, se analizarán planes de ejecución y se resolverán consultas de proximidad, contención, intersección y distancia. Finalmente, se aplicará un modelo de privilegios que permita la consulta de las capas GIS sin otorgar privilegios administrativos sobre la extensión PostGIS.

## Objetivos de aprendizaje

Al finalizar esta práctica, podrá:

- [ ] Verificar la disponibilidad e instalar la extensión PostGIS 3.4.2 en `ventas_seguras`.
- [ ] Crear tablas espaciales con columnas `geometry(Point, 4326)`, `geometry(Polygon, 4326)` y `geography(Point, 4326)`.
- [ ] Validar geometrías y aplicar restricciones de tipo, SRID, unicidad y relaciones con pedidos.
- [ ] Crear índices GiST y analizar consultas espaciales mediante `EXPLAIN (ANALYZE, BUFFERS)`.
- [ ] Ejecutar consultas de proximidad, contención, intersección y ordenamiento por distancia.
- [ ] Delegar permisos de lectura GIS a `rol_gis_lectura` y al usuario `carla_gis`.

## Prerrequisitos

### Conocimientos requeridos

- Uso básico de `psql`, esquemas, tablas, claves primarias y claves foráneas.
- Comprensión de restricciones `NOT NULL`, `CHECK`, `UNIQUE` e índices.
- Conceptos básicos de latitud, longitud y coordenadas geográficas.
- Conocimiento de los conceptos de extensión, `CREATE EXTENSION`, `pg_extension` y `pg_available_extensions`.
- Laboratorio 06-00-01 completado, incluyendo la base de datos `ventas_seguras`, roles y políticas de seguridad existentes.

### Acceso requerido

- Acceso al host que ejecuta el contenedor `pgadv15`.
- Rol administrador de PostgreSQL con privilegio para ejecutar `CREATE EXTENSION postgis`.
- Acceso a la base de datos `ventas_seguras`.
- Disponibilidad de los roles `rol_gis_lectura` y `carla_gis`. Si no existen, deben crearse según las políticas definidas en el laboratorio anterior.

> **Supuesto de integración:** esta guía usa como tabla comercial de referencia `ventas.pedidos`, con las columnas `pedido_id` y `region`. Si en el laboratorio anterior se utilizó otro esquema o nombre de tabla, identifique el objeto equivalente y sustituya `ventas.pedidos` en los comandos.

## Entorno de laboratorio

### Componentes de software

| Componente | Versión esperada | Uso |
|---|---:|---|
| PostgreSQL | 15.6 | Motor de base de datos del contenedor `pgadv15` |
| PostGIS | 3.4.2 | Tipos, funciones e índices espaciales |
| Docker Engine | 26.0.0 o equivalente | Ejecución del contenedor de laboratorio |
| Cliente `psql` | Compatible con PostgreSQL 15 | Administración y ejecución SQL |

### Convenciones utilizadas

| Elemento | Valor |
|---|---|
| Contenedor PostgreSQL | `pgadv15` |
| Base de datos | `ventas_seguras` |
| Esquema espacial | `gis` |
| SRID geográfico | `4326` |
| Sistema de referencia | WGS 84 |
| Rol propietario GIS | `rol_gis_propietario` |
| Rol de lectura GIS | `rol_gis_lectura` |
| Usuario de consulta | `carla_gis` |

### Preparación de sesión

1. Compruebe que el contenedor está en ejecución:

   ```bash
   docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
   ```

2. Abra una sesión administrativa contra `ventas_seguras`:

   ```bash
   docker exec -it pgadv15 psql -U postgres -d ventas_seguras
   ```

3. Dentro de `psql`, active una salida legible y detenga la ejecución ante errores:

   ```sql
   \set ON_ERROR_STOP on
   \pset pager off
   \timing on
   ```

4. Verifique la base actual y el usuario conectado:

   ```sql
   SELECT
       current_database() AS base_datos,
       current_user AS usuario,
       version() AS version_postgresql;
   ```

**Salida esperada**

La consulta debe indicar la base `ventas_seguras`, un usuario administrativo y una versión de PostgreSQL 15.6.

**Verificación**

```sql
SELECT current_database() = 'ventas_seguras' AS conectado_a_base_correcta;
```

El resultado debe ser `t`.

## Procedimiento paso a paso

### Paso 1. Verificar la disponibilidad de PostGIS y la estructura comercial

**Objetivo:** confirmar que los archivos de PostGIS están disponibles en la instancia y validar la tabla de pedidos que será relacionada con los puntos de entrega.

**Instrucciones**

1. Consulte la disponibilidad de PostGIS en la instancia:

   ```sql
   SELECT
       name,
       default_version,
       installed_version,
       comment
   FROM pg_available_extensions
   WHERE name IN ('postgis', 'postgis_topology')
   ORDER BY name;
   ```

2. Compruebe si PostGIS ya está instalada en la base actual:

   ```sql
   SELECT
       extname AS extension,
       extversion AS version,
       extnamespace::regnamespace AS schema
   FROM pg_extension
   WHERE extname = 'postgis';
   ```

3. Localice la tabla de pedidos disponible en la base:

   ```sql
   SELECT
       table_schema,
       table_name
   FROM information_schema.tables
   WHERE table_type = 'BASE TABLE'
     AND table_name = 'pedidos'
   ORDER BY table_schema;
   ```

4. Verifique las columnas de la tabla comercial prevista:

   ```sql
   SELECT
       column_name,
       data_type,
       is_nullable
   FROM information_schema.columns
   WHERE table_schema = 'ventas'
     AND table_name = 'pedidos'
   ORDER BY ordinal_position;
   ```

5. Revise una muestra de pedidos por región:

   ```sql
   SELECT
       pedido_id,
       region
   FROM ventas.pedidos
   WHERE region IN ('NORTE', 'CENTRO', 'SUR')
   ORDER BY pedido_id
   LIMIT 10;
   ```

**Salida esperada**

- `pg_available_extensions` debe mostrar `postgis` con una versión disponible 3.4.2.
- La tabla `ventas.pedidos` debe contener al menos `pedido_id` y `region`.
- Debe existir al menos un pedido para cada región: `NORTE`, `CENTRO` y `SUR`.

**Verificación**

```sql
SELECT
    region,
    count(*) AS cantidad_pedidos
FROM ventas.pedidos
WHERE region IN ('NORTE', 'CENTRO', 'SUR')
GROUP BY region
ORDER BY region;
```

Debe obtener tres filas, una por región. Si una región no tiene pedidos, use una región existente para las pruebas o incorpore pedidos de prueba conforme al procedimiento autorizado del laboratorio 06-00-01.

---

### Paso 2. Instalar y verificar la extensión PostGIS

**Objetivo:** crear la extensión PostGIS en la base `ventas_seguras` y comprobar que la versión instalada es la requerida.

**Instrucciones**

1. Cree la extensión como administrador:

   ```sql
   CREATE EXTENSION IF NOT EXISTS postgis;
   ```

2. Consulte la información completa de PostGIS:

   ```sql
   SELECT postgis_full_version();
   ```

3. Verifique específicamente la versión de la biblioteca PostGIS:

   ```sql
   SELECT
       postgis_lib_version() AS version_postgis,
       postgis_geos_version() AS version_geos,
       postgis_proj_version() AS version_proj;
   ```

4. Consulte el catálogo de extensiones instaladas:

   ```sql
   SELECT
       extname AS extension,
       extversion AS version,
       extnamespace::regnamespace AS schema
   FROM pg_extension
   WHERE extname = 'postgis';
   ```

5. Compruebe que los tipos espaciales ya están disponibles:

   ```sql
   SELECT
       typname AS tipo,
       typnamespace::regnamespace AS esquema
   FROM pg_type
   WHERE typname IN ('geometry', 'geography')
   ORDER BY typname;
   ```

**Salida esperada**

- `CREATE EXTENSION` debe finalizar correctamente o indicar que la extensión ya existe.
- `postgis_full_version()` debe contener una referencia a `POSTGIS="3.4.2"`.
- Deben existir los tipos `geometry` y `geography`.

**Verificación**

```sql
SELECT
    extname,
    extversion,
    extversion = '3.4.2' AS version_requerida
FROM pg_extension
WHERE extname = 'postgis';
```

El valor de `version_requerida` debe ser `t`.

> **Nota operativa:** la disponibilidad del paquete PostGIS en el sistema operativo es distinta de la instalación de la extensión en una base de datos. El paquete permite que la instancia conozca PostGIS; `CREATE EXTENSION postgis` crea los objetos en `ventas_seguras`.

---

### Paso 3. Crear el esquema GIS y el modelo espacial

**Objetivo:** crear el esquema `gis`, establecer un propietario controlado y definir las tablas espaciales relacionadas con los pedidos.

**Instrucciones**

1. Cree el rol propietario GIS si no existe:

   ```sql
   DO $$
   BEGIN
       IF NOT EXISTS (
           SELECT 1
           FROM pg_roles
           WHERE rolname = 'rol_gis_propietario'
       ) THEN
           CREATE ROLE rol_gis_propietario NOLOGIN;
       END IF;
   END;
   $$;
   ```

2. Cree el esquema espacial y asigne el propietario:

   ```sql
   CREATE SCHEMA IF NOT EXISTS gis AUTHORIZATION rol_gis_propietario;

   ALTER SCHEMA gis OWNER TO rol_gis_propietario;

   REVOKE ALL ON SCHEMA gis FROM PUBLIC;
   ```

3. Cambie temporalmente al rol propietario para crear los objetos:

   ```sql
   GRANT rol_gis_propietario TO CURRENT_USER;
   SET ROLE rol_gis_propietario;
   ```

4. Cree la tabla de sucursales. La ubicación se almacena como un punto geométrico WGS 84:

   ```sql
   CREATE TABLE IF NOT EXISTS gis.sucursales (
       sucursal_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
       nombre text NOT NULL UNIQUE,
       region varchar(20) NOT NULL
           CHECK (region IN ('NORTE', 'CENTRO', 'SUR')),
       ubicacion geometry(Point, 4326) NOT NULL,
       creada_en timestamptz NOT NULL DEFAULT current_timestamp,
       CONSTRAINT sucursales_ubicacion_valida_ck
           CHECK (ST_IsValid(ubicacion))
   );
   ```

5. Cree la tabla de zonas de reparto. Cada zona se almacena como un polígono:

   ```sql
   CREATE TABLE IF NOT EXISTS gis.zonas_reparto (
       zona_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
       nombre text NOT NULL UNIQUE,
       region varchar(20) NOT NULL
           CHECK (region IN ('NORTE', 'CENTRO', 'SUR')),
       area geometry(Polygon, 4326) NOT NULL,
       creada_en timestamptz NOT NULL DEFAULT current_timestamp,
       CONSTRAINT zonas_reparto_area_valida_ck
           CHECK (ST_IsValid(area))
   );
   ```

6. Cree la tabla de puntos de entrega. La misma ubicación se conserva en dos representaciones:

   - `ubicacion_geom`: para relaciones geométricas y operaciones topológicas.
   - `ubicacion_geog`: para cálculos de distancia geodésica en metros.

   ```sql
   CREATE TABLE IF NOT EXISTS gis.puntos_entrega (
       punto_entrega_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
       pedido_id bigint NOT NULL UNIQUE,
       region varchar(20) NOT NULL
           CHECK (region IN ('NORTE', 'CENTRO', 'SUR')),
       direccion_referencia text NOT NULL,
       ubicacion_geom geometry(Point, 4326) NOT NULL,
       ubicacion_geog geography(Point, 4326) NOT NULL,
       creada_en timestamptz NOT NULL DEFAULT current_timestamp,

       CONSTRAINT puntos_entrega_pedido_fk
           FOREIGN KEY (pedido_id)
           REFERENCES ventas.pedidos(pedido_id)
           ON UPDATE CASCADE
           ON DELETE RESTRICT,

       CONSTRAINT puntos_entrega_geom_valida_ck
           CHECK (ST_IsValid(ubicacion_geom)),

       CONSTRAINT puntos_entrega_srid_coincidente_ck
           CHECK (ST_SRID(ubicacion_geom) = 4326)
   );
   ```

7. Restablezca su rol administrativo:

   ```sql
   RESET ROLE;
   ```

**Salida esperada**

Las tres tablas deben crearse en el esquema `gis`. Las definiciones de columna deben reflejar los tipos espaciales y el SRID 4326.

**Verificación**

```sql
SELECT
    table_name,
    column_name,
    udt_name
FROM information_schema.columns
WHERE table_schema = 'gis'
  AND table_name IN ('sucursales', 'zonas_reparto', 'puntos_entrega')
ORDER BY table_name, ordinal_position;
```

Además, confirme las restricciones:

```sql
SELECT
    conrelid::regclass AS tabla,
    conname AS restriccion,
    contype AS tipo
FROM pg_constraint
WHERE connamespace = 'gis'::regnamespace
ORDER BY conrelid::regclass::text, conname;
```

---

### Paso 4. Cargar datos espaciales de sucursales, zonas y entregas

**Objetivo:** incorporar datos geoespaciales de muestra para las regiones NORTE, CENTRO y SUR y relacionar pedidos existentes con ubicaciones de entrega.

**Instrucciones**

1. Inserte tres sucursales de referencia. Recuerde que `ST_MakePoint(longitud, latitud)` recibe primero la longitud y después la latitud.

   ```sql
   INSERT INTO gis.sucursales (nombre, region, ubicacion)
   VALUES
       (
           'Sucursal Norte Bilbao',
           'NORTE',
           ST_SetSRID(ST_MakePoint(-2.9350, 43.2630), 4326)
       ),
       (
           'Sucursal Centro Madrid',
           'CENTRO',
           ST_SetSRID(ST_MakePoint(-3.7038, 40.4168), 4326)
       ),
       (
           'Sucursal Sur Sevilla',
           'SUR',
           ST_SetSRID(ST_MakePoint(-5.9845, 37.3891), 4326)
       )
   ON CONFLICT (nombre) DO UPDATE
   SET
       region = EXCLUDED.region,
       ubicacion = EXCLUDED.ubicacion;
   ```

2. Inserte zonas de reparto rectangulares simplificadas alrededor de cada sucursal:

   ```sql
   INSERT INTO gis.zonas_reparto (nombre, region, area)
   VALUES
       (
           'Zona Norte Bilbao',
           'NORTE',
           ST_GeomFromText(
               'POLYGON((
                   -3.0200 43.2200,
                   -2.8500 43.2200,
                   -2.8500 43.3100,
                   -3.0200 43.3100,
                   -3.0200 43.2200
               ))',
               4326
           )
       ),
       (
           'Zona Centro Madrid',
           'CENTRO',
           ST_GeomFromText(
               'POLYGON((
                   -3.7900 40.3600,
                   -3.6200 40.3600,
                   -3.6200 40.4800,
                   -3.7900 40.4800,
                   -3.7900 40.3600
               ))',
               4326
           )
       ),
       (
           'Zona Sur Sevilla',
           'SUR',
           ST_GeomFromText(
               'POLYGON((
                   -6.0800 37.3300,
                   -5.8800 37.3300,
                   -5.8800 37.4500,
                   -6.0800 37.4500,
                   -6.0800 37.3300
               ))',
               4326
           )
       )
   ON CONFLICT (nombre) DO UPDATE
   SET
       region = EXCLUDED.region,
       area = EXCLUDED.area;
   ```

3. Inserte un punto de entrega por región a partir de los primeros pedidos disponibles. Las coordenadas están cerca de la sucursal correspondiente.

   ```sql
   WITH pedidos_por_region AS (
       SELECT
           pedido_id,
           region,
           row_number() OVER (
               PARTITION BY region
               ORDER BY pedido_id
           ) AS posicion
       FROM ventas.pedidos
       WHERE region IN ('NORTE', 'CENTRO', 'SUR')
   ),
   ubicaciones AS (
       SELECT
           'NORTE'::varchar(20) AS region,
           'Entrega Bilbao Centro'::text AS direccion_referencia,
           -2.9300::double precision AS longitud,
           43.2600::double precision AS latitud
       UNION ALL
       SELECT
           'CENTRO',
           'Entrega Madrid Centro',
           -3.7000,
           40.4200
       UNION ALL
       SELECT
           'SUR',
           'Entrega Sevilla Centro',
           -5.9800,
           37.3900
   )
   INSERT INTO gis.puntos_entrega (
       pedido_id,
       region,
       direccion_referencia,
       ubicacion_geom,
       ubicacion_geog
   )
   SELECT
       p.pedido_id,
       u.region,
       u.direccion_referencia,
       ST_SetSRID(ST_MakePoint(u.longitud, u.latitud), 4326),
       ST_SetSRID(ST_MakePoint(u.longitud, u.latitud), 4326)::geography
   FROM pedidos_por_region p
   JOIN ubicaciones u
     ON u.region = p.region
   WHERE p.posicion = 1
   ON CONFLICT (pedido_id) DO UPDATE
   SET
       region = EXCLUDED.region,
       direccion_referencia = EXCLUDED.direccion_referencia,
       ubicacion_geom = EXCLUDED.ubicacion_geom,
       ubicacion_geog = EXCLUDED.ubicacion_geog;
   ```

4. Revise los datos incorporados:

   ```sql
   SELECT
       pe.pedido_id,
       pe.region,
       pe.direccion_referencia,
       ST_AsText(pe.ubicacion_geom) AS geometria,
       ST_SRID(pe.ubicacion_geom) AS srid,
       GeometryType(pe.ubicacion_geom) AS tipo_geometria
   FROM gis.puntos_entrega pe
   ORDER BY pe.region, pe.pedido_id;
   ```

**Salida esperada**

- Deben existir tres sucursales y tres zonas de reparto.
- Debe existir al menos un punto de entrega para cada región con pedidos disponibles.
- Las geometrías deben mostrarse como `POINT(longitud latitud)`.
- El SRID debe ser `4326`.

**Verificación**

```sql
SELECT
    'sucursales' AS capa,
    count(*) AS filas
FROM gis.sucursales
UNION ALL
SELECT
    'zonas_reparto',
    count(*)
FROM gis.zonas_reparto
UNION ALL
SELECT
    'puntos_entrega',
    count(*)
FROM gis.puntos_entrega;
```

La salida debe indicar al menos 3 filas para cada capa en un entorno preparado con pedidos de las tres regiones.

---

### Paso 5. Validar y corregir geometrías

**Objetivo:** comprobar la validez espacial de las capas, identificar geometrías problemáticas y comprender el uso de `ST_MakeValid`.

**Instrucciones**

1. Valide todas las geometrías cargadas:

   ```sql
   SELECT
       'sucursales' AS capa,
       sucursal_id AS identificador,
       ST_IsValid(ubicacion) AS es_valida,
       ST_IsValidReason(ubicacion) AS detalle
   FROM gis.sucursales

   UNION ALL

   SELECT
       'zonas_reparto',
       zona_id,
       ST_IsValid(area),
       ST_IsValidReason(area)
   FROM gis.zonas_reparto

   UNION ALL

   SELECT
       'puntos_entrega',
       punto_entrega_id,
       ST_IsValid(ubicacion_geom),
       ST_IsValidReason(ubicacion_geom)
   FROM gis.puntos_entrega
   ORDER BY capa, identificador;
   ```

2. Compruebe que las zonas tienen tipo `POLYGON` y SRID 4326:

   ```sql
   SELECT
       zona_id,
       nombre,
       GeometryType(area) AS tipo,
       ST_SRID(area) AS srid,
       ST_IsValid(area) AS valida
   FROM gis.zonas_reparto
   ORDER BY zona_id;
   ```

3. Analice un ejemplo didáctico de polígono inválido con forma de “lazo”:

   ```sql
   WITH geometria_invalida AS (
       SELECT ST_GeomFromText(
           'POLYGON((
               -3.75 40.40,
               -3.65 40.45,
               -3.75 40.45,
               -3.65 40.40,
               -3.75 40.40
           ))',
           4326
       ) AS geom
   )
   SELECT
       ST_IsValid(geom) AS original_valida,
       ST_IsValidReason(geom) AS motivo,
       GeometryType(ST_MakeValid(geom)) AS tipo_corregido,
       ST_IsValid(ST_MakeValid(geom)) AS corregida_valida
   FROM geometria_invalida;
   ```

4. Si una zona real resultara inválida, revise el tipo devuelto por `ST_MakeValid` antes de actualizarla. Para un polígono que siga siendo compatible con la columna `geometry(Polygon,4326)`, el patrón de corrección es:

   ```sql
   -- Ejemplo: sustituya <zona_id> solo después de validar el resultado.
   UPDATE gis.zonas_reparto
   SET area = ST_CollectionExtract(ST_MakeValid(area), 3)::geometry(Polygon, 4326)
   WHERE zona_id = <zona_id>
     AND NOT ST_IsValid(area);
   ```

**Salida esperada**

Las geometrías cargadas deben ser válidas. El ejemplo didáctico debe mostrar que la geometría original es inválida y que `ST_MakeValid` genera una geometría válida, posiblemente de tipo `MULTIPOLYGON`.

**Verificación**

```sql
SELECT
    count(*) AS geometrías_invalidas
FROM (
    SELECT ubicacion AS geom FROM gis.sucursales
    UNION ALL
    SELECT area FROM gis.zonas_reparto
    UNION ALL
    SELECT ubicacion_geom FROM gis.puntos_entrega
) AS geometrías
WHERE NOT ST_IsValid(geom);
```

El resultado debe ser `0`.

> **Importante:** una corrección espacial puede transformar un `POLYGON` en `MULTIPOLYGON`. No aplique automáticamente una conversión sin revisar que no se pierdan componentes espaciales.

---

### Paso 6. Evaluar consultas antes de crear índices espaciales

**Objetivo:** capturar un plan de referencia para una búsqueda espacial antes de implementar índices GiST.

**Instrucciones**

1. Elimine índices espaciales previos si existen, únicamente en este entorno de laboratorio:

   ```sql
   DROP INDEX IF EXISTS gis.sucursales_ubicacion_gix;
   DROP INDEX IF EXISTS gis.zonas_reparto_area_gix;
   DROP INDEX IF EXISTS gis.puntos_entrega_geom_gix;
   DROP INDEX IF EXISTS gis.puntos_entrega_geog_gix;
   ```

2. Actualice las estadísticas disponibles:

   ```sql
   ANALYZE gis.sucursales;
   ANALYZE gis.zonas_reparto;
   ANALYZE gis.puntos_entrega;
   ```

3. Ejecute un plan para localizar entregas en un radio de 5 km de la sucursal Centro:

   ```sql
   EXPLAIN (ANALYZE, BUFFERS)
   SELECT
       s.nombre AS sucursal,
       pe.pedido_id,
       pe.direccion_referencia
   FROM gis.sucursales s
   JOIN gis.puntos_entrega pe
     ON ST_DWithin(
         pe.ubicacion_geog,
         s.ubicacion::geography,
         5000
     )
   WHERE s.region = 'CENTRO';
   ```

**Salida esperada**

En conjuntos pequeños es normal que PostgreSQL use `Seq Scan`, incluso si después se crean índices, porque recorrer pocas filas cuesta menos que usar una estructura de índice. Antes de crear los índices, el plan no debe contener un `Index Scan` sobre los índices GIS definidos en este laboratorio.

**Verificación**

Revise el plan y confirme que no existe una referencia a `puntos_entrega_geog_gix`. Registre el tiempo de ejecución mostrado por `Execution Time`.

---

### Paso 7. Crear índices GiST y comparar planes

**Objetivo:** implementar índices espaciales GiST sobre las columnas espaciales y observar su participación en el planificador.

**Instrucciones**

1. Cree índices GiST sobre las columnas `geometry`:

   ```sql
   CREATE INDEX sucursales_ubicacion_gix
   ON gis.sucursales
   USING gist (ubicacion);

   CREATE INDEX zonas_reparto_area_gix
   ON gis.zonas_reparto
   USING gist (area);

   CREATE INDEX puntos_entrega_geom_gix
   ON gis.puntos_entrega
   USING gist (ubicacion_geom);
   ```

2. Cree el índice GiST para cálculos geodésicos basados en `geography`:

   ```sql
   CREATE INDEX puntos_entrega_geog_gix
   ON gis.puntos_entrega
   USING gist (ubicacion_geog);
   ```

3. Actualice las estadísticas del optimizador:

   ```sql
   ANALYZE gis.sucursales;
   ANALYZE gis.zonas_reparto;
   ANALYZE gis.puntos_entrega;
   ```

4. Compruebe la definición de los índices:

   ```sql
   SELECT
       schemaname,
       tablename,
       indexname,
       indexdef
   FROM pg_indexes
   WHERE schemaname = 'gis'
   ORDER BY tablename, indexname;
   ```

5. Repita la consulta de proximidad:

   ```sql
   EXPLAIN (ANALYZE, BUFFERS)
   SELECT
       s.nombre AS sucursal,
       pe.pedido_id,
       pe.direccion_referencia
   FROM gis.sucursales s
   JOIN gis.puntos_entrega pe
     ON ST_DWithin(
         pe.ubicacion_geog,
         s.ubicacion::geography,
         5000
     )
   WHERE s.region = 'CENTRO';
   ```

6. Para fines diagnósticos, fuerce temporalmente la preferencia por índices y observe un plan candidato:

   ```sql
   SET enable_seqscan = off;

   EXPLAIN (ANALYZE, BUFFERS)
   SELECT
       pe.pedido_id,
       pe.direccion_referencia
   FROM gis.puntos_entrega pe
   WHERE ST_DWithin(
       pe.ubicacion_geog,
       ST_SetSRID(ST_MakePoint(-3.7038, 40.4168), 4326)::geography,
       5000
   );

   RESET enable_seqscan;
   ```

**Salida esperada**

- Deben existir cuatro índices GiST.
- El plan forzado debe poder mostrar un `Index Scan` o `Bitmap Index Scan` que utilice `puntos_entrega_geog_gix`.
- En tablas con muy pocas filas, el planificador puede preferir legítimamente un escaneo secuencial sin que esto constituya un error.

**Verificación**

```sql
SELECT
    indexrelid::regclass AS indice,
    pg_size_pretty(pg_relation_size(indexrelid)) AS tamaño
FROM pg_stat_user_indexes
WHERE schemaname = 'gis'
ORDER BY indexrelid::regclass::text;
```

Deben aparecer los cuatro índices espaciales creados.

---

### Paso 8. Ejecutar consultas geoespaciales

**Objetivo:** resolver consultas de proximidad, contención, intersección y distancia usando funciones PostGIS.

**Instrucciones**

1. Localice pedidos a menos de 5 km de una sucursal. `ST_DWithin` sobre valores `geography` recibe la distancia en metros:

   ```sql
   SELECT
       s.nombre AS sucursal,
       s.region,
       pe.pedido_id,
       pe.direccion_referencia,
       round(
           ST_Distance(
               s.ubicacion::geography,
               pe.ubicacion_geog
           )::numeric,
           2
       ) AS distancia_metros
   FROM gis.sucursales s
   JOIN gis.puntos_entrega pe
     ON ST_DWithin(
         s.ubicacion::geography,
         pe.ubicacion_geog,
         5000
     )
   ORDER BY distancia_metros, pe.pedido_id;
   ```

2. Determine qué zona de reparto contiene cada punto de entrega. `ST_Contains` devuelve verdadero cuando el primer objeto contiene completamente al segundo:

   ```sql
   SELECT
       pe.pedido_id,
       pe.region AS region_pedido,
       z.nombre AS zona_reparto,
       z.region AS region_zona
   FROM gis.puntos_entrega pe
   JOIN gis.zonas_reparto z
     ON ST_Contains(z.area, pe.ubicacion_geom)
   ORDER BY pe.pedido_id;
   ```

3. Identifique puntos de entrega que no están contenidos en ninguna zona:

   ```sql
   SELECT
       pe.pedido_id,
       pe.region,
       pe.direccion_referencia
   FROM gis.puntos_entrega pe
   LEFT JOIN gis.zonas_reparto z
     ON ST_Contains(z.area, pe.ubicacion_geom)
   WHERE z.zona_id IS NULL
   ORDER BY pe.pedido_id;
   ```

4. Determine qué zonas se intersectan con un área de servicio de 5 km alrededor de cada sucursal. Se crea el área de servicio mediante un búfer en metros sobre `geography`:

   ```sql
   SELECT
       s.nombre AS sucursal,
       z.nombre AS zona_reparto,
       ST_Intersects(
           ST_Buffer(s.ubicacion::geography, 5000)::geometry,
           z.area
       ) AS se_intersecta
   FROM gis.sucursales s
   JOIN gis.zonas_reparto z
     ON ST_Intersects(
         ST_Buffer(s.ubicacion::geography, 5000)::geometry,
         z.area
     )
   ORDER BY s.nombre, z.nombre;
   ```

5. Ordene las entregas por distancia respecto a la sucursal Centro:

   ```sql
   SELECT
       pe.pedido_id,
       pe.region,
       pe.direccion_referencia,
       round(
           ST_Distance(
               pe.ubicacion_geog,
               s.ubicacion::geography
           )::numeric,
           2
       ) AS distancia_metros
   FROM gis.puntos_entrega pe
   CROSS JOIN gis.sucursales s
   WHERE s.nombre = 'Sucursal Centro Madrid'
   ORDER BY
       ST_Distance(
           pe.ubicacion_geog,
           s.ubicacion::geography
       ),
       pe.pedido_id;
   ```

6. Revise la distancia entre cada pedido y la sucursal de su misma región:

   ```sql
   SELECT
       pe.pedido_id,
       pe.region,
       s.nombre AS sucursal,
       round(
           ST_Distance(
               pe.ubicacion_geog,
               s.ubicacion::geography
           )::numeric,
           2
       ) AS distancia_metros
   FROM gis.puntos_entrega pe
   JOIN gis.sucursales s
     ON s.region = pe.region
   ORDER BY pe.region, distancia_metros;
   ```

**Salida esperada**

- Cada punto de entrega de muestra debe encontrarse a menos de 5 km de la sucursal de su región.
- Cada punto de entrega debe estar contenido en una zona de reparto de la región correspondiente.
- Las consultas de distancia deben expresar resultados en metros.

**Verificación**

```sql
SELECT
    count(*) AS entregas_fuera_de_su_zona
FROM gis.puntos_entrega pe
LEFT JOIN gis.zonas_reparto z
  ON z.region = pe.region
 AND ST_Contains(z.area, pe.ubicacion_geom)
WHERE z.zona_id IS NULL;
```

El resultado esperado es `0`.

---

### Paso 9. Configurar privilegios sobre las capas espaciales

**Objetivo:** mantener el control del esquema GIS en el propietario definido y otorgar acceso de lectura autorizado sin conceder privilegios administrativos sobre PostGIS.

**Instrucciones**

1. Verifique la existencia de los roles requeridos:

   ```sql
   SELECT
       rolname,
       rolcanlogin,
       rolsuper
   FROM pg_roles
   WHERE rolname IN (
       'rol_gis_propietario',
       'rol_gis_lectura',
       'carla_gis'
   )
   ORDER BY rolname;
   ```

2. Si `rol_gis_lectura` no existe, créelo como rol sin inicio de sesión:

   ```sql
   DO $$
   BEGIN
       IF NOT EXISTS (
           SELECT 1
           FROM pg_roles
           WHERE rolname = 'rol_gis_lectura'
       ) THEN
           CREATE ROLE rol_gis_lectura NOLOGIN;
       END IF;
   END;
   $$;
   ```

3. Asigne al rol de lectura el acceso mínimo necesario al esquema y a las tablas GIS:

   ```sql
   GRANT USAGE ON SCHEMA gis TO rol_gis_lectura;

   GRANT SELECT ON TABLE
       gis.sucursales,
       gis.zonas_reparto,
       gis.puntos_entrega
   TO rol_gis_lectura;
   ```

4. Conceda el rol de lectura a `carla_gis`:

   ```sql
   GRANT rol_gis_lectura TO carla_gis;
   ```

5. Evite privilegios de creación en el esquema GIS para los roles de consulta:

   ```sql
   REVOKE CREATE ON SCHEMA gis
   FROM rol_gis_lectura, carla_gis;
   ```

6. Compruebe los privilegios efectivos sobre las tablas:

   ```sql
   SELECT
       grantee,
       table_schema,
       table_name,
       privilege_type
   FROM information_schema.role_table_grants
   WHERE table_schema = 'gis'
     AND grantee IN ('rol_gis_lectura', 'carla_gis')
   ORDER BY grantee, table_name, privilege_type;
   ```

7. Compruebe que `carla_gis` no es superusuario ni propietario de la extensión:

   ```sql
   SELECT
       r.rolname,
       r.rolsuper,
       e.extname,
       e.extowner::regrole AS propietario_extension
   FROM pg_roles r
   CROSS JOIN pg_extension e
   WHERE r.rolname = 'carla_gis'
     AND e.extname = 'postgis';
   ```

**Salida esperada**

- `rol_gis_lectura` debe tener `USAGE` sobre el esquema `gis` y `SELECT` sobre las tres tablas espaciales.
- `carla_gis` debe heredar el rol de lectura.
- `carla_gis` no debe ser superusuario ni propietario de PostGIS.

**Verificación**

Como administrador, ejecute:

```sql
SET ROLE carla_gis;

SELECT
    pedido_id,
    region,
    direccion_referencia
FROM gis.puntos_entrega
ORDER BY pedido_id;

RESET ROLE;
```

La consulta debe ejecutarse correctamente. El siguiente comando debe fallar para `carla_gis` y no debe probarse en un entorno productivo:

```sql
-- SET ROLE carla_gis;
-- CREATE EXTENSION postgis;
-- RESET ROLE;
```

> **Nota de seguridad:** los permisos sobre `gis.puntos_entrega` no otorgan automáticamente acceso a `ventas.pedidos`. Si `carla_gis` necesita consultar datos comerciales adicionales, debe hacerlo mediante las vistas autorizadas y las políticas RLS definidas en el laboratorio 06-00-01.

## Validación y pruebas

Ejecute el siguiente bloque como administrador para validar el estado final del laboratorio:

```sql
SELECT
    'PostGIS instalada' AS prueba,
    EXISTS (
        SELECT 1
        FROM pg_extension
        WHERE extname = 'postgis'
          AND extversion = '3.4.2'
    ) AS resultado

UNION ALL

SELECT
    'Esquema gis existente',
    EXISTS (
        SELECT 1
        FROM pg_namespace
        WHERE nspname = 'gis'
    )

UNION ALL

SELECT
    'Tres capas espaciales existentes',
    (
        SELECT count(*) = 3
        FROM information_schema.tables
        WHERE table_schema = 'gis'
          AND table_name IN (
              'sucursales',
              'zonas_reparto',
              'puntos_entrega'
          )
    )

UNION ALL

SELECT
    'Sin geometrías inválidas',
    NOT EXISTS (
        SELECT 1
        FROM (
            SELECT ubicacion AS geom FROM gis.sucursales
            UNION ALL
            SELECT area FROM gis.zonas_reparto
            UNION ALL
            SELECT ubicacion_geom FROM gis.puntos_entrega
        ) AS todas_las_geometrias
        WHERE NOT ST_IsValid(geom)
    )

UNION ALL

SELECT
    'Índices GiST creados',
    (
        SELECT count(*) = 4
        FROM pg_indexes
        WHERE schemaname = 'gis'
          AND indexname IN (
              'sucursales_ubicacion_gix',
              'zonas_reparto_area_gix',
              'puntos_entrega_geom_gix',
              'puntos_entrega_geog_gix'
          )
    )

UNION ALL

SELECT
    'rol_gis_lectura puede consultar puntos',
    has_table_privilege(
        'rol_gis_lectura',
        'gis.puntos_entrega',
        'SELECT'
    );
```

**Resultado esperado**

Todas las filas de la columna `resultado` deben mostrar `t`.

Realice además esta prueba funcional de distancia:

```sql
SELECT
    count(*) AS entregas_cercanas_a_alguna_sucursal
FROM gis.puntos_entrega pe
JOIN gis.sucursales s
  ON ST_DWithin(
      pe.ubicacion_geog,
      s.ubicacion::geography,
      5000
  );
```

El resultado debe ser mayor que cero.

## Solución de problemas

### Problema 1. `CREATE EXTENSION postgis` falla indicando que el archivo de control no existe

**Síntoma**

Al ejecutar:

```sql
CREATE EXTENSION postgis;
```

se recibe un error similar a:

```text
ERROR: extension "postgis" is not available
DETAIL: Could not open extension control file ".../postgis.control": No such file or directory.
```

**Causa**

Los archivos de PostGIS no están instalados en el sistema operativo o no corresponden a la versión de PostgreSQL utilizada por el contenedor. Tener PostgreSQL operativo no implica que el paquete PostGIS esté disponible.

**Corrección**

1. Salga de `psql`:

   ```sql
   \q
   ```

2. Ingrese al contenedor como usuario con privilegios administrativos:

   ```bash
   docker exec -it --user root pgadv15 bash
   ```

3. Instale el paquete de PostGIS correspondiente a PostgreSQL 15, si la imagen y el repositorio del laboratorio lo permiten:

   ```bash
   apt-get update
   apt-get install -y postgresql-15-postgis-3 postgresql-15-postgis-3-scripts
   ```

4. Salga del contenedor y reinícielo si es necesario:

   ```bash
   exit
   docker restart pgadv15
   ```

5. Vuelva a conectarse y compruebe la disponibilidad:

   ```bash
   docker exec -it pgadv15 psql -U postgres -d ventas_seguras -c \
   "SELECT name, default_version FROM pg_available_extensions WHERE name = 'postgis';"
   ```

No continúe hasta que `default_version` corresponda a `3.4.2` según el estándar del laboratorio.

### Problema 2. La consulta con `ST_DWithin` no usa el índice GiST o devuelve distancias inesperadas

**Síntoma**

La consulta muestra `Seq Scan` en vez de `Index Scan`, o las distancias parecen estar expresadas en grados y no en metros.

**Causa**

Existen dos causas frecuentes:

1. La tabla contiene muy pocas filas y el planificador determina correctamente que un escaneo secuencial es más barato.
2. Se está usando una columna `geometry` con SRID 4326 para calcular distancia directa. En ese caso, las unidades son grados, no metros.

**Corrección**

1. Confirme que el índice de `geography` existe:

   ```sql
   SELECT indexname, indexdef
   FROM pg_indexes
   WHERE schemaname = 'gis'
     AND tablename = 'puntos_entrega';
   ```

2. Actualice estadísticas:

   ```sql
   ANALYZE gis.puntos_entrega;
   ```

3. Use `geography` para radios y distancias en metros:

   ```sql
   SELECT ST_Distance(
       pe.ubicacion_geog,
       s.ubicacion::geography
   ) AS distancia_metros
   FROM gis.puntos_entrega pe
   CROSS JOIN gis.sucursales s
   WHERE s.nombre = 'Sucursal Centro Madrid';
   ```

4. Para revisar el uso potencial del índice en un conjunto pequeño, aplique solo de forma diagnóstica:

   ```sql
   SET enable_seqscan = off;

   EXPLAIN (ANALYZE, BUFFERS)
   SELECT *
   FROM gis.puntos_entrega
   WHERE ST_DWithin(
       ubicacion_geog,
       ST_SetSRID(ST_MakePoint(-3.7038, 40.4168), 4326)::geography,
       5000
   );

   RESET enable_seqscan;
   ```

No mantenga `enable_seqscan = off` como parámetro operativo permanente.

## Limpieza

Las tablas espaciales y sus índices deben permanecer en `ventas_seguras`, ya que serán utilizados en el laboratorio 08-00-01 para pruebas de actualización y rendimiento.

Para cerrar la sesión sin eliminar objetos:

```sql
RESET ROLE;
\q
```

Si necesita repetir exclusivamente la carga de datos de prueba, elimine los datos respetando el orden de claves foráneas:

```sql
DELETE FROM gis.puntos_entrega;
DELETE FROM gis.zonas_reparto;
DELETE FROM gis.sucursales;
```

> **No ejecute** `DROP EXTENSION postgis CASCADE` en este laboratorio. Esa operación puede eliminar objetos dependientes y dejar la base `ventas_seguras` sin las capacidades geoespaciales requeridas para prácticas posteriores.

## Resumen

En esta práctica se instaló y verificó PostGIS 3.4.2 como extensión de la base de datos `ventas_seguras`. Se creó el esquema `gis` con capas para sucursales, zonas de reparto y puntos de entrega, aplicando tipos espaciales con SRID 4326, restricciones de validez y una relación de clave foránea con los pedidos comerciales.

También se crearon índices GiST para columnas `geometry` y `geography`, se compararon planes de ejecución y se utilizaron funciones PostGIS como `ST_IsValid`, `ST_MakeValid`, `ST_DWithin`, `ST_Contains`, `ST_Intersects` y `ST_Distance`. Finalmente, se delegó acceso de consulta mediante `rol_gis_lectura` y `carla_gis`, sin conceder control administrativo sobre la extensión PostGIS.

### Recursos opcionales

- [Documentación oficial de PostGIS](https://postgis.net/documentation/)
- [Referencia de ST_DWithin](https://postgis.net/docs/ST_DWithin.html)
- [Referencia de ST_Contains](https://postgis.net/docs/ST_Contains.html)
- [Referencia de ST_Intersects](https://postgis.net/docs/ST_Intersects.html)
- [Documentación de extensiones de PostgreSQL](https://www.postgresql.org/docs/current/extend-extensions.html)
