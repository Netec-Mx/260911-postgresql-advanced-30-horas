# Prácticas 8.1 Actualización y optimización de PostgreSQL

## Metadatos

| Elemento | Valor |
|---|---|
| Duración | 147 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Aplicar |

## Descripción general

En este laboratorio realizará una actualización mayor controlada mediante respaldo lógico desde PostgreSQL 15.6 hacia PostgreSQL 16.2. La base de origen, `ventas_seguras`, contiene roles, políticas RLS, extensiones PostGIS, índices GiST y datos comerciales construidos en laboratorios anteriores.

Después de restaurar y validar la base en PostgreSQL 16.2, habilitará `pg_stat_statements`, generará carga controlada, analizará estadísticas y planes de ejecución, y aplicará optimizaciones basadas en evidencia. La reversión se garantizará conservando el contenedor de origen `pgadv15` y los respaldos generados.

## Objetivos de aprendizaje

Al finalizar el laboratorio, podrá:

- [ ] Inventariar una instancia PostgreSQL 15.6 antes de una actualización mayor.
- [ ] Generar y verificar respaldos lógicos de roles y de la base `ventas_seguras`.
- [ ] Restaurar roles, extensiones, objetos y datos en PostgreSQL 16.2.
- [ ] Validar integridad funcional, privilegios, políticas RLS y objetos PostGIS después de la migración.
- [ ] Monitorizar carga, bloqueos, estadísticas y planes de consulta para aplicar ajustes básicos de rendimiento.

## Prerrequisitos

### Conocimientos requeridos

- Uso básico de `psql`, `pg_dump`, `pg_restore`, `pg_dumpall` y Docker Compose.
- Comprensión de roles, privilegios, políticas Row-Level Security (RLS), índices B-tree e índices GiST.
- Conocimientos básicos de `EXPLAIN`, `ANALYZE`, `VACUUM` y transacciones.
- Haber completado los laboratorios 06-00-01 y 07-00-01.
- Disponer de una base `ventas_seguras` funcional en el contenedor PostgreSQL 15.6 denominado `pgadv15`.

### Acceso requerido

- Acceso local al host Linux con Docker Engine y Docker Compose Plugin.
- Permisos para ejecutar comandos `docker`, `docker compose` y editar archivos locales.
- Al menos 16 GB de RAM recomendados para mantener los contenedores PostgreSQL 15.6 y PostgreSQL 16.2 en ejecución.
- Espacio libre suficiente para el volumen de PostgreSQL 16, respaldos lógicos y archivos de medición.

> **Importante:** Esta práctica utiliza restauración lógica, no `pg_upgrade`. Nunca inicie PostgreSQL 16 apuntando al volumen de datos creado por PostgreSQL 15.

## Entorno de laboratorio

### Componentes

| Componente | Valor esperado |
|---|---|
| Origen | Contenedor `pgadv15`, PostgreSQL Server 15.6 |
| Destino | Contenedor `pgadv16`, PostgreSQL Server 16.2 |
| Base de datos migrada | `ventas_seguras` |
| Extensión espacial | PostGIS 3.4.2 |
| Herramienta de carga | `pgbench` |
| Puerto origen sugerido | `55432` |
| Puerto destino | `5432` |
| Directorio de trabajo | `~/lab08-upgrade` |

### Convenciones usadas

| Variable | Valor |
|---|---|
| Usuario administrador | `postgres` |
| Base origen | `ventas_seguras` |
| Base destino | `ventas_seguras` |
| Directorio de respaldos local | `~/lab08-upgrade/backups` |
| Directorio de evidencias local | `~/lab08-upgrade/evidencias` |
| Contenedor origen | `pgadv15` |
| Contenedor destino | `pgadv16` |

### Preparación inicial

1. Cree el directorio de trabajo y los subdirectorios requeridos:

   ```bash
   mkdir -p ~/lab08-upgrade/{backups,evidencias,sql}
   cd ~/lab08-upgrade
   ```

2. Verifique que el contenedor de origen esté disponible:

   ```bash
   docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
   ```

3. Si el contenedor `pgadv15` no está iniciado, inícielo según el procedimiento utilizado en los laboratorios anteriores:

   ```bash
   docker start pgadv15
   ```

4. Verifique la versión real del servidor de origen:

   ```bash
   docker exec -u postgres pgadv15 \
     psql -d postgres -c "SELECT version();"
   ```

**Salida esperada**

La salida debe mostrar PostgreSQL 15.6 o la versión 15.6 establecida para el laboratorio.

**Verificación**

No continúe si el contenedor de origen no responde o si la base `ventas_seguras` no existe:

```bash
docker exec -u postgres pgadv15 \
  psql -d postgres -c "\l ventas_seguras"
```

## Procedimiento paso a paso

### Paso 1. Registrar el inventario previo a la actualización

**Objetivo**

Documentar el estado técnico y funcional de PostgreSQL 15.6 antes de generar respaldos o modificar el entorno.

**Instrucciones**

1. Registre la versión del servidor, la versión del cliente y el directorio de datos:

   ```bash
   docker exec -u postgres pgadv15 \
     psql -d postgres -c "SELECT version();" \
     | tee evidencias/01_version_origen.txt

   docker exec -u postgres pgadv15 \
     psql -d postgres -c "SHOW server_version;" \
     | tee evidencias/02_server_version_origen.txt

   docker exec -u postgres pgadv15 \
     psql -d postgres -c "SHOW data_directory;" \
     | tee evidencias/03_data_directory_origen.txt

   docker exec pgadv15 psql --version \
     | tee evidencias/04_psql_version_origen.txt
   ```

2. Liste las bases de datos no plantilla y sus tamaños:

   ```bash
   docker exec -u postgres pgadv15 \
     psql -d postgres -P pager=off -c "
     SELECT datname,
            pg_size_pretty(pg_database_size(datname)) AS tamaño
     FROM pg_database
     WHERE datistemplate = false
     ORDER BY pg_database_size(datname) DESC;" \
     | tee evidencias/05_bases_y_tamanos_origen.txt
   ```

3. Registre las extensiones de `ventas_seguras`:

   ```bash
   docker exec -u postgres pgadv15 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT extname, extversion
     FROM pg_extension
     ORDER BY extname;" \
     | tee evidencias/06_extensiones_origen.txt
   ```

4. Registre los esquemas y las tablas de usuario:

   ```bash
   docker exec -u postgres pgadv15 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT table_schema, table_name
     FROM information_schema.tables
     WHERE table_type = 'BASE TABLE'
       AND table_schema NOT IN ('pg_catalog', 'information_schema')
     ORDER BY table_schema, table_name;" \
     | tee evidencias/07_tablas_origen.txt
   ```

5. Registre las políticas RLS definidas:

   ```bash
   docker exec -u postgres pgadv15 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT schemaname,
            tablename,
            policyname,
            permissive,
            roles,
            cmd,
            qual,
            with_check
     FROM pg_policies
     ORDER BY schemaname, tablename, policyname;" \
     | tee evidencias/08_politicas_rls_origen.txt
   ```

6. Registre índices de los esquemas de negocio, incluyendo sus definiciones:

   ```bash
   docker exec -u postgres pgadv15 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT schemaname,
            tablename,
            indexname,
            indexdef
     FROM pg_indexes
     WHERE schemaname IN ('ventas', 'gis')
     ORDER BY schemaname, tablename, indexname;" \
     | tee evidencias/09_indices_origen.txt
   ```

7. Registre restricciones primarias, únicas y foráneas:

   ```bash
   docker exec -u postgres pgadv15 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT n.nspname AS esquema,
            c.relname AS tabla,
            con.conname AS restriccion,
            pg_get_constraintdef(con.oid) AS definicion
     FROM pg_constraint con
     JOIN pg_class c ON c.oid = con.conrelid
     JOIN pg_namespace n ON n.oid = c.relnamespace
     WHERE n.nspname IN ('ventas', 'gis')
     ORDER BY n.nspname, c.relname, con.conname;" \
     | tee evidencias/10_restricciones_origen.txt
   ```

8. Registre conteos de filas para todas las tablas de usuario. Este resultado será la línea base de integridad de datos:

   ```bash
   docker exec -u postgres pgadv15 \
     psql -d ventas_seguras -At -c "
     SELECT format(
       'SELECT %L AS tabla, count(*) AS filas FROM %I.%I;',
       table_schema || '.' || table_name,
       table_schema,
       table_name
     )
     FROM information_schema.tables
     WHERE table_type = 'BASE TABLE'
       AND table_schema IN ('ventas', 'gis')
     ORDER BY table_schema, table_name;" \
     > sql/conteos_origen.sql

   docker cp sql/conteos_origen.sql pgadv15:/tmp/conteos_origen.sql

   docker exec -u postgres pgadv15 \
     psql -d ventas_seguras -f /tmp/conteos_origen.sql \
     | tee evidencias/11_conteos_origen.txt
   ```

**Salida esperada**

Debe disponer de archivos de evidencia con versión, extensiones, objetos, índices, restricciones, políticas RLS y conteos de datos del entorno PostgreSQL 15.6.

**Verificación**

Compruebe que existen los archivos principales:

```bash
ls -lh evidencias/
```

Revise especialmente que `postgis` aparezca en `06_extensiones_origen.txt` y que existan políticas en `08_politicas_rls_origen.txt`, si fueron configuradas en el laboratorio anterior.

---

### Paso 2. Ejecutar comprobaciones funcionales y espaciales en el origen

**Objetivo**

Conservar resultados funcionales de referencia para comparar la operación de la base migrada.

**Instrucciones**

1. Verifique la versión de PostGIS:

   ```bash
   docker exec -u postgres pgadv15 \
     psql -d ventas_seguras -c "SELECT PostGIS_Full_Version();" \
     | tee evidencias/12_postgis_origen.txt
   ```

2. Inspeccione la estructura de las tablas principales:

   ```bash
   docker exec -u postgres pgadv15 \
     psql -d ventas_seguras -c "\d+ ventas.pedidos" \
     | tee evidencias/13_estructura_pedidos_origen.txt

   docker exec -u postgres pgadv15 \
     psql -d ventas_seguras -c "\d+ gis.puntos_entrega" \
     | tee evidencias/14_estructura_puntos_entrega_origen.txt
   ```

3. Ejecute una consulta de comprobación sobre pedidos. Ajuste los nombres de columna únicamente si su modelo usa nombres diferentes:

   ```bash
   docker exec -u postgres pgadv15 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT count(*) AS pedidos_totales,
            min(fecha_pedido) AS fecha_minima,
            max(fecha_pedido) AS fecha_maxima
     FROM ventas.pedidos;" \
     | tee evidencias/15_resumen_pedidos_origen.txt
   ```

4. Ejecute una consulta espacial representativa. El siguiente ejemplo cuenta puntos dentro de un radio de 5 km respecto de un punto de referencia en SRID 4326:

   ```bash
   docker exec -u postgres pgadv15 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT count(*) AS entregas_en_radio
     FROM gis.puntos_entrega
     WHERE ST_DWithin(
       geom::geography,
       ST_SetSRID(ST_MakePoint(-3.7038, 40.4168), 4326)::geography,
       5000
     );" \
     | tee evidencias/16_consulta_espacial_origen.txt
   ```

5. Identifique un rol no superusuario utilizado por las políticas RLS:

   ```bash
   docker exec -u postgres pgadv15 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT rolname,
            rolsuper,
            rolcanlogin
     FROM pg_roles
     WHERE rolname NOT LIKE 'pg_%'
     ORDER BY rolname;" \
     | tee evidencias/17_roles_origen.txt
   ```

6. Seleccione un rol funcional del resultado anterior y exporte su nombre como variable local. En este ejemplo se utilizará `analista_ventas`; sustitúyalo por el rol existente en su entorno:

   ```bash
   export RLS_ROLE=analista_ventas
   echo "$RLS_ROLE"
   ```

7. Pruebe una consulta bajo el rol seleccionado. Si el rol requiere contraseña o autenticación externa, realice la prueba usando un usuario de laboratorio autorizado:

   ```bash
   docker exec -u postgres pgadv15 \
     psql -d ventas_seguras -P pager=off -c "
     SET ROLE ${RLS_ROLE};
     SELECT current_user, session_user;
     SELECT count(*) AS pedidos_visibles
     FROM ventas.pedidos;
     RESET ROLE;" \
     | tee evidencias/18_prueba_rls_origen.txt
   ```

**Salida esperada**

Debe obtener una versión de PostGIS, un conteo de pedidos, un conteo espacial y un resultado de visibilidad bajo un rol RLS.

**Verificación**

Los valores registrados en los archivos `15`, `16` y `18` se utilizarán posteriormente. No es obligatorio que el rol RLS vea todas las filas; lo importante es que el resultado sea coherente antes y después de la migración.

---

### Paso 3. Generar y verificar los respaldos lógicos

**Objetivo**

Crear copias de seguridad verificables de roles globales y de la base `ventas_seguras` antes de iniciar PostgreSQL 16.2.

**Instrucciones**

1. Genere el respaldo de roles, membresías y demás objetos globales:

   ```bash
   docker exec -u postgres pgadv15 \
     pg_dumpall --globals-only \
     --file=/tmp/globals_pre_pg16.sql
   ```

2. Copie el archivo de roles desde el contenedor origen al host:

   ```bash
   docker cp pgadv15:/tmp/globals_pre_pg16.sql backups/globals_pre_pg16.sql
   ```

3. Genere un respaldo personalizado de la base `ventas_seguras`:

   ```bash
   docker exec -u postgres pgadv15 \
     pg_dump \
     --format=custom \
     --verbose \
     --file=/tmp/ventas_seguras_pre_pg16.dump \
     --dbname=ventas_seguras
   ```

4. Copie el respaldo personalizado al host:

   ```bash
   docker cp pgadv15:/tmp/ventas_seguras_pre_pg16.dump \
     backups/ventas_seguras_pre_pg16.dump
   ```

5. Proteja los permisos del respaldo global, ya que puede contener hashes de contraseñas de roles:

   ```bash
   chmod 600 backups/globals_pre_pg16.sql
   chmod 600 backups/ventas_seguras_pre_pg16.dump
   ```

6. Verifique que el archivo de roles tiene contenido SQL:

   ```bash
   head -n 20 backups/globals_pre_pg16.sql
   ```

7. Inspeccione el contenido del respaldo personalizado sin restaurarlo:

   ```bash
   pg_restore --list backups/ventas_seguras_pre_pg16.dump \
     | tee evidencias/19_lista_respaldo_personalizado.txt
   ```

8. Confirme que el respaldo contiene objetos esperados:

   ```bash
   grep -E "ventas|gis|pedidos|puntos_entrega|EXTENSION" \
     evidencias/19_lista_respaldo_personalizado.txt \
     | head -n 30
   ```

9. Calcule sumas de verificación para registrar integridad de archivos:

   ```bash
   sha256sum backups/globals_pre_pg16.sql \
             backups/ventas_seguras_pre_pg16.dump \
     | tee evidencias/20_checksums_respaldos.txt
   ```

**Salida esperada**

Deben existir dos respaldos:

- `backups/globals_pre_pg16.sql`
- `backups/ventas_seguras_pre_pg16.dump`

El comando `pg_restore --list` debe completar sin errores.

**Verificación**

```bash
ls -lh backups/
cat evidencias/20_checksums_respaldos.txt
```

> **Punto de control:** No continúe si `pg_restore --list` informa que el archivo no es un respaldo válido o si el tamaño del archivo `.dump` es anormalmente pequeño.

---

### Paso 4. Iniciar el destino PostgreSQL 16.2 con PostGIS 3.4.2

**Objetivo**

Desplegar una instancia limpia de PostgreSQL 16.2 compatible con los objetos PostGIS del origen.

**Instrucciones**

1. Cree el archivo `compose.yaml` para el destino. Este laboratorio asume que la imagen local o entregada por el instructor se denomina `pgadv16:16.2-postgis3.4.2`.

   ```bash
   cat > compose.yaml <<'EOF'
   services:
     pgadv16:
       image: pgadv16:16.2-postgis3.4.2
       container_name: pgadv16
       restart: unless-stopped
       environment:
         POSTGRES_USER: postgres
         POSTGRES_PASSWORD: postgres_lab_16
         POSTGRES_DB: postgres
         TZ: UTC
         PGTZ: UTC
       ports:
         - "5432:5432"
       volumes:
         - pgadv16_data:/var/lib/postgresql/data
         - ./backups:/backups:ro
         - ./evidencias:/evidencias
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U postgres -d postgres"]
         interval: 10s
         timeout: 5s
         retries: 10

   volumes:
     pgadv16_data:
   EOF
   ```

2. Inicie el contenedor destino:

   ```bash
   docker compose up -d
   ```

3. Espere hasta que el servicio responda:

   ```bash
   docker exec pgadv16 pg_isready -U postgres -d postgres
   ```

4. Verifique la versión exacta del servidor:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d postgres -c "SELECT version();" \
     | tee evidencias/21_version_destino.txt
   ```

5. Verifique la disponibilidad de PostGIS en la imagen de destino:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d postgres -P pager=off -c "
     SELECT name, default_version, installed_version
     FROM pg_available_extensions
     WHERE name IN ('postgis', 'postgis_topology', 'pg_stat_statements')
     ORDER BY name;" \
     | tee evidencias/22_extensiones_disponibles_destino.txt
   ```

6. Configure la zona horaria UTC a nivel de instancia:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d postgres -c "ALTER SYSTEM SET timezone = 'UTC';"

   docker restart pgadv16
   ```

7. Confirme la zona horaria configurada:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d postgres -c "SHOW timezone;"
   ```

**Salida esperada**

- PostgreSQL debe reportar versión `16.2`.
- La extensión `postgis` debe aparecer como disponible.
- La zona horaria debe ser `UTC`.

**Verificación**

```bash
docker logs --tail 30 pgadv16
```

No deben existir errores de inicialización ni mensajes que indiquen incompatibilidad de bibliotecas de extensiones.

---

### Paso 5. Restaurar roles y la base de datos en PostgreSQL 16.2

**Objetivo**

Restaurar los objetos globales y la base de datos de negocio manteniendo propietarios, privilegios y dependencias.

**Instrucciones**

1. Inspeccione los roles ya existentes en PostgreSQL 16.2:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d postgres -P pager=off -c "
     SELECT rolname, rolsuper, rolcanlogin
     FROM pg_roles
     ORDER BY rolname;"
   ```

2. Restaure los objetos globales. La opción `ON_ERROR_STOP=1` obliga a detener el proceso ante un error real:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -v ON_ERROR_STOP=1 \
     -d postgres \
     -f /backups/globals_pre_pg16.sql \
     | tee evidencias/23_restauracion_roles.txt
   ```

3. Revise los roles restaurados:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d postgres -P pager=off -c "
     SELECT rolname,
            rolsuper,
            rolinherit,
            rolcreaterole,
            rolcreatedb,
            rolcanlogin
     FROM pg_roles
     WHERE rolname NOT LIKE 'pg_%'
     ORDER BY rolname;" \
     | tee evidencias/24_roles_destino.txt
   ```

4. Cree una base de datos limpia de destino. Si el respaldo contiene extensiones, `pg_restore` las creará siempre que sus bibliotecas estén instaladas:

   ```bash
   docker exec -u postgres pgadv16 \
     createdb \
     --encoding=UTF8 \
     --template=template0 \
     --owner=postgres \
     ventas_seguras
   ```

5. Restaure el respaldo personalizado con salida detallada:

   ```bash
   docker exec -u postgres pgadv16 \
     pg_restore \
     --verbose \
     --exit-on-error \
     --dbname=ventas_seguras \
     /backups/ventas_seguras_pre_pg16.dump \
     | tee evidencias/25_restauracion_base.txt
   ```

6. Si el respaldo no incluía PostGIS, habilítelo explícitamente. Ejecute esta instrucción solo después de confirmar que no existe:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -c "CREATE EXTENSION IF NOT EXISTS postgis;"
   ```

7. Ejecute `ANALYZE` para actualizar estadísticas del planificador después de la restauración:

   ```bash
   docker exec -u postgres pgadv16 \
     vacuumdb \
     --analyze-in-stages \
     --dbname=ventas_seguras \
     --verbose \
     | tee evidencias/26_analyze_post_restauracion.txt
   ```

**Salida esperada**

La restauración debe finalizar sin errores y crear esquemas, tablas, datos, secuencias, restricciones, índices, políticas y extensiones.

**Verificación**

Compruebe que existen los esquemas esperados:

```bash
docker exec -u postgres pgadv16 \
  psql -d ventas_seguras -c "\dn"
```

Compruebe que existen las tablas principales:

```bash
docker exec -u postgres pgadv16 \
  psql -d ventas_seguras -c "\dt ventas.*"

docker exec -u postgres pgadv16 \
  psql -d ventas_seguras -c "\dt gis.*"
```

---

### Paso 6. Validar integridad funcional y de datos posterior a la migración

**Objetivo**

Confirmar que el destino PostgreSQL 16.2 contiene los datos y objetos requeridos, y que conserva el comportamiento funcional básico.

**Instrucciones**

1. Registre las extensiones instaladas en el destino:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT extname, extversion
     FROM pg_extension
     ORDER BY extname;" \
     | tee evidencias/27_extensiones_destino.txt
   ```

2. Confirme la versión de PostGIS:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -c "SELECT PostGIS_Full_Version();" \
     | tee evidencias/28_postgis_destino.txt
   ```

3. Copie el script de conteos al contenedor destino y ejecútelo:

   ```bash
   docker cp sql/conteos_origen.sql pgadv16:/tmp/conteos_destino.sql

   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -f /tmp/conteos_destino.sql \
     | tee evidencias/29_conteos_destino.txt
   ```

4. Compare los conteos de origen y destino:

   ```bash
   diff -u evidencias/11_conteos_origen.txt \
           evidencias/29_conteos_destino.txt \
     | tee evidencias/30_diferencia_conteos.txt
   ```

5. Registre restricciones en el destino:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT n.nspname AS esquema,
            c.relname AS tabla,
            con.conname AS restriccion,
            pg_get_constraintdef(con.oid) AS definicion
     FROM pg_constraint con
     JOIN pg_class c ON c.oid = con.conrelid
     JOIN pg_namespace n ON n.oid = c.relnamespace
     WHERE n.nspname IN ('ventas', 'gis')
     ORDER BY n.nspname, c.relname, con.conname;" \
     | tee evidencias/31_restricciones_destino.txt
   ```

6. Registre las políticas RLS en el destino:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT schemaname,
            tablename,
            policyname,
            permissive,
            roles,
            cmd,
            qual,
            with_check
     FROM pg_policies
     ORDER BY schemaname, tablename, policyname;" \
     | tee evidencias/32_politicas_rls_destino.txt
   ```

7. Registre las definiciones de índices, prestando atención a B-tree, GiST y posibles índices parciales:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT schemaname,
            tablename,
            indexname,
            indexdef
     FROM pg_indexes
     WHERE schemaname IN ('ventas', 'gis')
     ORDER BY schemaname, tablename, indexname;" \
     | tee evidencias/33_indices_destino.txt
   ```

8. Compare restricciones, políticas e índices con los resultados del origen:

   ```bash
   diff -u evidencias/10_restricciones_origen.txt \
           evidencias/31_restricciones_destino.txt \
     | tee evidencias/34_diferencia_restricciones.txt

   diff -u evidencias/08_politicas_rls_origen.txt \
           evidencias/32_politicas_rls_destino.txt \
     | tee evidencias/35_diferencia_politicas.txt

   diff -u evidencias/09_indices_origen.txt \
           evidencias/33_indices_destino.txt \
     | tee evidencias/36_diferencia_indices.txt
   ```

9. Ejecute la consulta funcional de pedidos en el destino:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT count(*) AS pedidos_totales,
            min(fecha_pedido) AS fecha_minima,
            max(fecha_pedido) AS fecha_maxima
     FROM ventas.pedidos;" \
     | tee evidencias/37_resumen_pedidos_destino.txt
   ```

10. Ejecute la consulta espacial de referencia:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT count(*) AS entregas_en_radio
     FROM gis.puntos_entrega
     WHERE ST_DWithin(
       geom::geography,
       ST_SetSRID(ST_MakePoint(-3.7038, 40.4168), 4326)::geography,
       5000
     );" \
     | tee evidencias/38_consulta_espacial_destino.txt
   ```

11. Ejecute la prueba RLS con el mismo rol usado en el origen:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SET ROLE ${RLS_ROLE};
     SELECT current_user, session_user;
     SELECT count(*) AS pedidos_visibles
     FROM ventas.pedidos;
     RESET ROLE;" \
     | tee evidencias/39_prueba_rls_destino.txt
   ```

**Salida esperada**

- Los conteos de filas deben coincidir.
- Las políticas RLS, restricciones e índices deben coincidir funcionalmente.
- La consulta espacial debe devolver el mismo resultado que en PostgreSQL 15.6.
- La visibilidad de filas bajo el rol RLS debe conservarse.

**Verificación**

Ejecute las comparaciones funcionales:

```bash
diff -u evidencias/15_resumen_pedidos_origen.txt \
        evidencias/37_resumen_pedidos_destino.txt

diff -u evidencias/16_consulta_espacial_origen.txt \
        evidencias/38_consulta_espacial_destino.txt

diff -u evidencias/18_prueba_rls_origen.txt \
        evidencias/39_prueba_rls_destino.txt
```

Cualquier diferencia debe investigarse antes de aceptar la actualización.

---

### Paso 7. Habilitar monitorización con pg_stat_statements y generar carga

**Objetivo**

Activar la extensión `pg_stat_statements`, generar actividad controlada y capturar una línea base de monitorización en PostgreSQL 16.2.

**Instrucciones**

1. Configure la precarga de la biblioteca `pg_stat_statements`. Este parámetro requiere reiniciar la instancia:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d postgres -c "
     ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements';"

   docker exec -u postgres pgadv16 \
     psql -d postgres -c "
     ALTER SYSTEM SET pg_stat_statements.track = 'all';"

   docker exec -u postgres pgadv16 \
     psql -d postgres -c "
     ALTER SYSTEM SET pg_stat_statements.track_planning = 'on';"

   docker restart pgadv16
   ```

2. Cree la extensión en la base migrada:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -c "
     CREATE EXTENSION IF NOT EXISTS pg_stat_statements;"
   ```

3. Verifique su carga y configuración:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SHOW shared_preload_libraries;
     SHOW pg_stat_statements.track;
     SHOW pg_stat_statements.track_planning;" \
     | tee evidencias/40_configuracion_pgss.txt
   ```

4. Cree un conjunto de tablas de prueba para `pgbench`. No use tablas de negocio para inicializar `pgbench`:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -c "
     CREATE SCHEMA IF NOT EXISTS rendimiento;
     GRANT USAGE ON SCHEMA rendimiento TO PUBLIC;"
   ```

5. Inicialice una carga pequeña de `pgbench`. El tamaño de escala `10` es adecuado para un laboratorio y puede ajustarse según el hardware disponible:

   ```bash
   docker exec -u postgres pgadv16 \
     pgbench -i -s 10 -n -U postgres ventas_seguras \
     | tee evidencias/41_pgbench_inicializacion.txt
   ```

6. Ejecute una carga transaccional controlada durante 60 segundos con cuatro clientes:

   ```bash
   docker exec -u postgres pgadv16 \
     pgbench -c 4 -j 2 -T 60 -U postgres ventas_seguras \
     | tee evidencias/42_pgbench_carga.txt
   ```

7. Ejecute consultas representativas de negocio varias veces para que aparezcan en `pg_stat_statements`:

   ```bash
   for i in $(seq 1 30); do
     docker exec -u postgres pgadv16 \
       psql -d ventas_seguras -qAt -c "
       SELECT count(*)
       FROM ventas.pedidos
       WHERE fecha_pedido >= CURRENT_DATE - INTERVAL '30 days';" >/dev/null
   done
   ```

8. Genere carga espacial repetida:

   ```bash
   for i in $(seq 1 20); do
     docker exec -u postgres pgadv16 \
       psql -d ventas_seguras -qAt -c "
       SELECT count(*)
       FROM gis.puntos_entrega
       WHERE ST_DWithin(
         geom::geography,
         ST_SetSRID(ST_MakePoint(-3.7038, 40.4168), 4326)::geography,
         5000
       );" >/dev/null
   done
   ```

**Salida esperada**

- `shared_preload_libraries` debe incluir `pg_stat_statements`.
- `pgbench` debe informar transacciones procesadas y TPS.
- Las consultas de negocio y espaciales deben quedar registradas en las vistas de estadísticas.

**Verificación**

```bash
docker exec -u postgres pgadv16 \
  psql -d ventas_seguras -P pager=off -c "
  SELECT count(*) AS sentencias_registradas
  FROM pg_stat_statements;"
```

---

### Paso 8. Analizar actividad, bloqueos, estadísticas y uso de índices

**Objetivo**

Usar vistas del catálogo para identificar actividad, consultas costosas, tablas con mantenimiento pendiente e índices con bajo uso.

**Instrucciones**

1. Inspeccione las sesiones activas:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT pid,
            usename,
            application_name,
            state,
            wait_event_type,
            wait_event,
            now() - query_start AS duracion,
            left(query, 120) AS consulta
     FROM pg_stat_activity
     WHERE datname = 'ventas_seguras'
     ORDER BY query_start NULLS LAST;" \
     | tee evidencias/43_pg_stat_activity.txt
   ```

2. Revise bloqueos actuales:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT l.pid,
            a.usename,
            c.relname AS relacion,
            l.mode,
            l.granted,
            a.wait_event_type,
            a.wait_event,
            left(a.query, 100) AS consulta
     FROM pg_locks l
     LEFT JOIN pg_stat_activity a ON a.pid = l.pid
     LEFT JOIN pg_class c ON c.oid = l.relation
     WHERE a.datname = 'ventas_seguras'
     ORDER BY l.granted, l.pid;" \
     | tee evidencias/44_pg_locks.txt
   ```

3. Consulte estadísticas generales de la base:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT datname,
            numbackends,
            xact_commit,
            xact_rollback,
            blks_read,
            blks_hit,
            temp_files,
            temp_bytes,
            deadlocks
     FROM pg_stat_database
     WHERE datname = 'ventas_seguras';" \
     | tee evidencias/45_pg_stat_database.txt
   ```

4. Inspeccione tablas de negocio, accesos secuenciales, accesos por índice y tuplas muertas:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT schemaname,
            relname,
            n_live_tup,
            n_dead_tup,
            seq_scan,
            idx_scan,
            last_analyze,
            last_autoanalyze,
            last_vacuum,
            last_autovacuum
     FROM pg_stat_user_tables
     WHERE schemaname IN ('ventas', 'gis')
     ORDER BY n_dead_tup DESC, relname;" \
     | tee evidencias/46_pg_stat_user_tables.txt
   ```

5. Identifique índices con escaneos bajos o nulos. Un valor bajo no implica automáticamente que el índice sea inútil; puede ser necesario para restricciones o cargas poco frecuentes:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT s.schemaname,
            s.relname AS tabla,
            s.indexrelname AS indice,
            s.idx_scan,
            pg_size_pretty(pg_relation_size(s.indexrelid)) AS tamaño_indice
     FROM pg_stat_user_indexes s
     WHERE s.schemaname IN ('ventas', 'gis')
     ORDER BY s.idx_scan ASC, pg_relation_size(s.indexrelid) DESC;" \
     | tee evidencias/47_pg_stat_user_indexes.txt
   ```

6. Consulte las sentencias con mayor tiempo total acumulado:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT calls,
            round(total_exec_time::numeric, 2) AS tiempo_total_ms,
            round(mean_exec_time::numeric, 2) AS tiempo_medio_ms,
            rows,
            left(query, 180) AS consulta
     FROM pg_stat_statements
     WHERE dbid = (SELECT oid FROM pg_database WHERE datname = 'ventas_seguras')
     ORDER BY total_exec_time DESC
     LIMIT 10;" \
     | tee evidencias/48_pg_stat_statements_top10.txt
   ```

7. Revise estadísticas de columnas relevantes para el planificador:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT schemaname,
            tablename,
            attname,
            null_frac,
            n_distinct,
            most_common_vals,
            histogram_bounds
     FROM pg_stats
     WHERE schemaname = 'ventas'
       AND tablename = 'pedidos'
     ORDER BY attname;" \
     | tee evidencias/49_pg_stats_pedidos.txt
   ```

**Salida esperada**

Debe poder identificar:

- Sesiones activas y eventos de espera.
- Bloqueos concedidos y no concedidos.
- Uso de caché y actividad transaccional.
- Tablas con tuplas muertas o estadísticas antiguas.
- Consultas con mayor coste acumulado.
- Índices poco utilizados que requieren análisis, no eliminación inmediata.

**Verificación**

La vista `pg_stat_statements` debe mostrar consultas normalizadas con parámetros representados normalmente como `$1`, `$2` u otros marcadores.

---

### Paso 9. Optimizar consultas con EXPLAIN ANALYZE BUFFERS e índices

**Objetivo**

Comparar planes antes y después de aplicar mejoras de consulta, índices y estadísticas.

**Instrucciones**

1. Evalúe una consulta no sargable basada en fecha. Una condición no sargable aplica una función sobre la columna indexable, lo cual puede impedir el uso eficiente de un índice B-tree:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     EXPLAIN (ANALYZE, BUFFERS)
     SELECT count(*)
     FROM ventas.pedidos
     WHERE date(fecha_pedido) = CURRENT_DATE;" \
     | tee evidencias/50_plan_no_sargable_antes.txt
   ```

2. Ejecute la versión sargable equivalente usando un rango de fechas:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     EXPLAIN (ANALYZE, BUFFERS)
     SELECT count(*)
     FROM ventas.pedidos
     WHERE fecha_pedido >= CURRENT_DATE
       AND fecha_pedido < CURRENT_DATE + INTERVAL '1 day';" \
     | tee evidencias/51_plan_sargable_antes.txt
   ```

3. Identifique las columnas existentes en `ventas.pedidos` antes de crear un índice compuesto:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT column_name, data_type
     FROM information_schema.columns
     WHERE table_schema = 'ventas'
       AND table_name = 'pedidos'
     ORDER BY ordinal_position;"
   ```

4. Si existen las columnas `cliente_id` y `fecha_pedido`, cree un índice B-tree compuesto. El orden elegido favorece filtros por cliente y rangos por fecha:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -c "
     CREATE INDEX IF NOT EXISTS idx_pedidos_cliente_fecha
     ON ventas.pedidos (cliente_id, fecha_pedido);"
   ```

5. Actualice estadísticas de la tabla modificada:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -c "ANALYZE VERBOSE ventas.pedidos;" \
     | tee evidencias/52_analyze_pedidos.txt
   ```

6. Pruebe una consulta que aproveche el índice compuesto. Sustituya `1` por un identificador de cliente existente si fuera necesario:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     EXPLAIN (ANALYZE, BUFFERS)
     SELECT *
     FROM ventas.pedidos
     WHERE cliente_id = 1
       AND fecha_pedido >= CURRENT_DATE - INTERVAL '90 days'
     ORDER BY fecha_pedido DESC;" \
     | tee evidencias/53_plan_indice_compuesto.txt
   ```

7. Inspeccione el índice espacial existente sobre la geometría:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT indexname, indexdef
     FROM pg_indexes
     WHERE schemaname = 'gis'
       AND tablename = 'puntos_entrega';"
   ```

8. Si no existe un índice GiST sobre `geom`, créelo. No cree un índice duplicado si ya existe uno equivalente:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -c "
     CREATE INDEX IF NOT EXISTS idx_puntos_entrega_geom_gist
     ON gis.puntos_entrega
     USING GIST (geom);"
   ```

9. Actualice las estadísticas espaciales:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -c "ANALYZE VERBOSE gis.puntos_entrega;" \
     | tee evidencias/54_analyze_puntos_entrega.txt
   ```

10. Analice el plan espacial. Para que un índice GiST de geometría sea aprovechable con mayor probabilidad, use primero el operador de caja delimitadora `&&` y después el predicado exacto:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     EXPLAIN (ANALYZE, BUFFERS)
     WITH referencia AS (
       SELECT ST_SetSRID(ST_MakePoint(-3.7038, 40.4168), 4326) AS g
     )
     SELECT p.*
     FROM gis.puntos_entrega p
     CROSS JOIN referencia r
     WHERE p.geom && ST_Expand(r.g, 0.05)
       AND ST_DWithin(p.geom::geography, r.g::geography, 5000);" \
     | tee evidencias/55_plan_espacial_gist.txt
   ```

11. Ejecute mantenimiento sobre las tablas de negocio:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -c "
     VACUUM (ANALYZE, VERBOSE) ventas.pedidos;" \
     | tee evidencias/56_vacuum_analyze_pedidos.txt

   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -c "
     VACUUM (ANALYZE, VERBOSE) gis.puntos_entrega;" \
     | tee evidencias/57_vacuum_analyze_puntos.txt
   ```

12. Revise nuevamente las estadísticas de tablas:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SELECT schemaname,
            relname,
            n_live_tup,
            n_dead_tup,
            last_analyze,
            last_autoanalyze,
            last_vacuum,
            last_autovacuum
     FROM pg_stat_user_tables
     WHERE schemaname IN ('ventas', 'gis')
     ORDER BY relname;" \
     | tee evidencias/58_estadisticas_post_mantenimiento.txt
   ```

**Salida esperada**

Los planes pueden mostrar `Index Scan`, `Bitmap Index Scan`, `Bitmap Heap Scan` o, para tablas pequeñas, todavía `Seq Scan`. Un escaneo secuencial no es necesariamente incorrecto: PostgreSQL elegirá el plan con menor coste estimado para el volumen de datos y las estadísticas disponibles.

**Verificación**

Compare:

- `50_plan_no_sargable_antes.txt` con `51_plan_sargable_antes.txt`.
- El tiempo de ejecución y los bloques leídos en `53_plan_indice_compuesto.txt`.
- La presencia de un índice GiST o bitmap scan en `55_plan_espacial_gist.txt`.
- La reducción o control de `n_dead_tup` en `58_estadisticas_post_mantenimiento.txt`.

---

### Paso 10. Aplicar ajustes básicos de parámetros con control de riesgo

**Objetivo**

Configurar parámetros razonables para el entorno de laboratorio, documentando su finalidad, riesgo y método de reversión.

**Instrucciones**

1. Registre los valores actuales de parámetros relevantes:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d postgres -P pager=off -c "
     SELECT name,
            setting,
            unit,
            context,
            source
     FROM pg_settings
     WHERE name IN (
       'shared_buffers',
       'work_mem',
       'maintenance_work_mem',
       'effective_cache_size',
       'log_min_duration_statement',
       'autovacuum',
       'autovacuum_naptime',
       'autovacuum_vacuum_scale_factor',
       'autovacuum_analyze_scale_factor'
     )
     ORDER BY name;" \
     | tee evidencias/59_parametros_antes.txt
   ```

2. Aplique valores orientativos para un laboratorio con memoria suficiente. No use estos valores directamente en producción sin dimensionamiento, pruebas y revisión de concurrencia:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d postgres -c "ALTER SYSTEM SET shared_buffers = '512MB';"

   docker exec -u postgres pgadv16 \
     psql -d postgres -c "ALTER SYSTEM SET maintenance_work_mem = '256MB';"

   docker exec -u postgres pgadv16 \
     psql -d postgres -c "ALTER SYSTEM SET effective_cache_size = '2GB';"

   docker exec -u postgres pgadv16 \
     psql -d postgres -c "ALTER SYSTEM SET log_min_duration_statement = '500ms';"

   docker exec -u postgres pgadv16 \
     psql -d postgres -c "ALTER SYSTEM SET autovacuum_naptime = '30s';"

   docker exec -u postgres pgadv16 \
     psql -d postgres -c "ALTER SYSTEM SET autovacuum_vacuum_scale_factor = '0.05';"

   docker exec -u postgres pgadv16 \
     psql -d postgres -c "ALTER SYSTEM SET autovacuum_analyze_scale_factor = '0.05';"
   ```

3. Reinicie PostgreSQL para aplicar los parámetros que requieren reinicio, especialmente `shared_buffers`:

   ```bash
   docker restart pgadv16
   ```

4. Configure `work_mem` solamente para una sesión de prueba. Esto evita multiplicar excesivamente memoria por operaciones de ordenación, hash y sesiones concurrentes:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -P pager=off -c "
     SET work_mem = '32MB';
     SHOW work_mem;
     EXPLAIN (ANALYZE, BUFFERS)
     SELECT cliente_id, count(*)
     FROM ventas.pedidos
     GROUP BY cliente_id
     ORDER BY count(*) DESC;
     RESET work_mem;" \
     | tee evidencias/60_prueba_work_mem.txt
   ```

5. Registre los parámetros efectivos:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d postgres -P pager=off -c "
     SELECT name,
            setting,
            unit,
            context,
            source
     FROM pg_settings
     WHERE name IN (
       'shared_buffers',
       'work_mem',
       'maintenance_work_mem',
       'effective_cache_size',
       'log_min_duration_statement',
       'autovacuum_naptime',
       'autovacuum_vacuum_scale_factor',
       'autovacuum_analyze_scale_factor'
     )
     ORDER BY name;" \
     | tee evidencias/61_parametros_despues.txt
   ```

6. Documente los riesgos en un archivo de línea base:

   ```bash
   cat > evidencias/62_linea_base_y_riesgos.md <<'EOF'
   # Línea base y riesgos de ajuste

   ## Resultados observados
   - Versión origen: PostgreSQL 15.6.
   - Versión destino: PostgreSQL 16.2.
   - Respaldo global y respaldo personalizado verificados con SHA-256 y pg_restore --list.
   - Conteos de tablas comparados entre origen y destino.
   - Políticas RLS, restricciones, índices y consultas PostGIS validados.
   - Carga generada con pgbench y consultas de negocio observadas mediante pg_stat_statements.

   ## Riesgos de parámetros
   - shared_buffers: un valor excesivo puede provocar presión de memoria, intercambio y degradación general del host.
   - work_mem: se asigna por operación y por sesión; un valor global alto puede agotar la memoria con concurrencia.
   - maintenance_work_mem: acelera VACUUM, CREATE INDEX y mantenimiento, pero múltiples tareas simultáneas pueden consumir demasiada RAM.
   - effective_cache_size: es una estimación para el planificador; un valor irreal puede favorecer planes no adecuados.
   - log_min_duration_statement: mejora observabilidad, pero genera más E/S y puede exponer datos de consultas en registros.
   - autovacuum_*_scale_factor: valores bajos mejoran mantenimiento en tablas activas, pero incrementan actividad de autovacuum y E/S.

   ## Reversión
   1. Detener pgadv16 si la aceptación funcional falla.
   2. Mantener pgadv15 sin modificaciones y con su volumen original intacto.
   3. Reanudar conexiones contra pgadv15.
   4. Conservar backups/globals_pre_pg16.sql y backups/ventas_seguras_pre_pg16.dump.
   5. Para revertir parámetros en pgadv16, ejecutar ALTER SYSTEM RESET para cada parámetro y reiniciar el contenedor.
   EOF
   ```

**Salida esperada**

Los parámetros ajustados deben aparecer con `source = configuration file` o `source = override`, según la imagen y el mecanismo de configuración. La prueba de `work_mem` debe afectar solo a la sesión utilizada.

**Verificación**

Revise los cambios:

```bash
cat evidencias/59_parametros_antes.txt
cat evidencias/61_parametros_despues.txt
cat evidencias/62_linea_base_y_riesgos.md
```

## Validación y pruebas

Considere completado el laboratorio cuando todos los criterios siguientes se cumplan:

| Criterio | Evidencia requerida |
|---|---|
| Versión objetivo | `21_version_destino.txt` indica PostgreSQL 16.2 |
| Respaldo verificable | `19_lista_respaldo_personalizado.txt` y `20_checksums_respaldos.txt` |
| Roles restaurados | `24_roles_destino.txt` contiene los roles de negocio requeridos |
| Extensiones disponibles | `27_extensiones_destino.txt` y `28_postgis_destino.txt` |
| Integridad de datos | No hay diferencias no justificadas en `30_diferencia_conteos.txt` |
| Restricciones y privilegios | Comparación satisfactoria de restricciones, índices y políticas RLS |
| Funcionalidad RLS | Resultado equivalente en `18_prueba_rls_origen.txt` y `39_prueba_rls_destino.txt` |
| Funcionalidad espacial | Resultado equivalente en `16_consulta_espacial_origen.txt` y `38_consulta_espacial_destino.txt` |
| Observabilidad | `pg_stat_statements` contiene sentencias de carga y consultas funcionales |
| Optimización | Existen planes `EXPLAIN (ANALYZE, BUFFERS)` antes y después de la optimización |
| Mantenimiento | Se ejecutó `VACUUM (ANALYZE)` y se revisaron estadísticas de autovacuum |
| Reversión documentada | Archivo `62_linea_base_y_riesgos.md` completado |

Ejecute una comprobación final resumida:

```bash
docker exec -u postgres pgadv16 \
  psql -d ventas_seguras -P pager=off -c "
  SELECT version();
  SELECT extname, extversion FROM pg_extension ORDER BY extname;
  SELECT count(*) AS politicas_rls FROM pg_policies;
  SELECT count(*) AS sentencias_pgss FROM pg_stat_statements;"
```

## Resolución de problemas

### Problema 1: `pg_restore` falla indicando que PostGIS o una biblioteca de extensión no están disponibles

**Síntomas**

Durante la restauración aparece un error similar a:

```text
ERROR: could not access file "$libdir/postgis-3": No such file or directory
```

o:

```text
ERROR: extension "postgis" is not available
```

**Causa**

La imagen PostgreSQL 16.2 no contiene los paquetes o bibliotecas de PostGIS compatibles con la versión de origen. Instalar solamente PostgreSQL no instala automáticamente las extensiones de terceros.

**Solución**

1. Detenga y elimine únicamente el contenedor destino fallido, conservando el origen y los respaldos:

   ```bash
   docker compose down -v
   ```

2. Verifique que la imagen usada sea la imagen de laboratorio compatible:

   ```bash
   docker image inspect pgadv16:16.2-postgis3.4.2
   ```

3. Inicie nuevamente el destino con una imagen que incluya PostgreSQL 16.2 y PostGIS 3.4.2.

4. Confirme la disponibilidad antes de restaurar:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d postgres -c "
     SELECT name, default_version
     FROM pg_available_extensions
     WHERE name = 'postgis';"
   ```

5. Repita la restauración desde los archivos conservados en `backups/`.

### Problema 2: `pg_stat_statements` informa que debe cargarse mediante `shared_preload_libraries`

**Síntomas**

Al ejecutar `CREATE EXTENSION pg_stat_statements` o consultar la vista aparece un mensaje similar a:

```text
ERROR: pg_stat_statements must be loaded via shared_preload_libraries
```

**Causa**

La biblioteca no fue incluida en `shared_preload_libraries` antes de iniciar PostgreSQL, o el contenedor no se reinició después de ejecutar `ALTER SYSTEM`.

**Solución**

1. Configure la precarga:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d postgres -c "
     ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements';"
   ```

2. Reinicie completamente el contenedor:

   ```bash
   docker restart pgadv16
   ```

3. Cree o valide la extensión en la base correcta:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -c "
     CREATE EXTENSION IF NOT EXISTS pg_stat_statements;"
   ```

4. Verifique:

   ```bash
   docker exec -u postgres pgadv16 \
     psql -d ventas_seguras -c "
     SHOW shared_preload_libraries;
     SELECT count(*) FROM pg_stat_statements;"
   ```

## Limpieza

> **Advertencia:** No elimine `pgadv15` ni los archivos de respaldo si todavía necesita capacidad de reversión. La limpieza completa debe realizarse solamente después de aprobar formalmente la actualización.

1. Detenga el contenedor PostgreSQL 16.2 manteniendo el volumen para futuras pruebas:

   ```bash
   docker compose stop
   ```

2. Si el instructor autoriza eliminar únicamente la instancia de destino y su volumen, ejecute:

   ```bash
   docker compose down -v
   ```

3. Conserve los respaldos y evidencias:

   ```bash
   ls -lh backups/
   ls -lh evidencias/
   ```

4. Para revertir los ajustes de parámetros sin eliminar la instancia, ejecute:

   ```bash
   docker start pgadv16

   docker exec -u postgres pgadv16 \
     psql -d postgres -c "ALTER SYSTEM RESET shared_buffers;"

   docker exec -u postgres pgadv16 \
     psql -d postgres -c "ALTER SYSTEM RESET maintenance_work_mem;"

   docker exec -u postgres pgadv16 \
     psql -d postgres -c "ALTER SYSTEM RESET effective_cache_size;"

   docker exec -u postgres pgadv16 \
     psql -d postgres -c "ALTER SYSTEM RESET log_min_duration_statement;"

   docker exec -u postgres pgadv16 \
     psql -d postgres -c "ALTER SYSTEM RESET autovacuum_naptime;"

   docker exec -u postgres pgadv16 \
     psql -d postgres -c "ALTER SYSTEM RESET autovacuum_vacuum_scale_factor;"

   docker exec -u postgres pgadv16 \
     psql -d postgres -c "ALTER SYSTEM RESET autovacuum_analyze_scale_factor;"

   docker restart pgadv16
   ```

## Resumen

En este laboratorio realizó una actualización mayor controlada de PostgreSQL 15.6 a PostgreSQL 16.2 mediante respaldo lógico y restauración. Antes de migrar, creó un inventario de versiones, extensiones, objetos, restricciones, índices, políticas RLS y conteos de datos.

Posteriormente restauró roles y datos en PostgreSQL 16.2, validó PostGIS, RLS, integridad de filas y consultas espaciales. Finalmente, habilitó `pg_stat_statements`, generó carga con `pgbench`, analizó estadísticas del catálogo, revisó planes con `EXPLAIN (ANALYZE, BUFFERS)`, creó índices selectivos, ejecutó mantenimiento y documentó ajustes con sus riesgos y procedimiento de reversión.

Como práctica administrativa, conserve el contenedor `pgadv15` y los respaldos hasta completar el período de aceptación definido para la actualización.

### Recursos opcionales

- Documentación oficial de PostgreSQL: https://www.postgresql.org/docs/16/
- Documentación de `pg_dump`: https://www.postgresql.org/docs/16/app-pgdump.html
- Documentación de `pg_restore`: https://www.postgresql.org/docs/16/app-pgrestore.html
- Documentación de `pg_stat_statements`: https://www.postgresql.org/docs/16/pgstatstatements.html
- Manual de PostGIS: https://postgis.net/documentation/
