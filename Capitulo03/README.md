# Prácticas 3.1 Migración de una base de datos a PostgreSQL

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 118 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica se ejecutará una migración controlada de la base de datos MariaDB `source_sales` hacia PostgreSQL 16.2, utilizando `pgloader` 3.6.9. El proceso incluye inventario del origen, evaluación de compatibilidad, preparación de credenciales restringidas, carga de estructura y datos, corrección de incidencias de tipos y validación funcional.

La estrategia aplicada será una **migración con interrupción planificada** en un entorno de laboratorio: se establece una línea base de datos en MariaDB, se carga PostgreSQL y se validan ambos lados antes de considerar el destino como conjunto de datos válido para prácticas posteriores de respaldo, PITR y replicación.

## Objetivos de aprendizaje

Al finalizar la práctica, podrá:

- [ ] Elaborar un inventario técnico de tablas, columnas, restricciones, índices y registros de una base MariaDB.
- [ ] Identificar incompatibilidades entre MariaDB y PostgreSQL relacionadas con tipos, valores automáticos, sintaxis e identificadores.
- [ ] Migrar la estructura y los datos de `source_sales` hacia `sales_pg` mediante `pgloader`.
- [ ] Ajustar propietarios, privilegios, secuencias y objetos migrados al modelo de seguridad del laboratorio.
- [ ] Validar conteos, agregados, integridad referencial, índices, secuencias y consultas funcionales de negocio.

## Prerrequisitos

### Conocimientos requeridos

- Uso básico de terminal Linux, `ssh`, `sudo`, `psql` y cliente `mysql`.
- Conocimiento de tablas, columnas, claves primarias, claves foráneas, índices y consultas `SELECT`.
- Comprensión de tipos SQL como `TINYINT`, `DATETIME`, `DECIMAL`, `AUTO_INCREMENT`, `VARCHAR` y `TEXT`.
- Conocimiento de los conceptos de RTO, RPO, corte y reversión estudiados en la lección 3.1.
- Prácticas 1.1 y 2.1 completadas.

### Accesos requeridos

- Acceso administrativo a MariaDB 10.11.7 que contiene la base `source_sales`.
- Acceso `sudo` al nodo `pg-primary`.
- Acceso a PostgreSQL 16.2 con un rol administrativo, por ejemplo `postgres`.
- Base de datos PostgreSQL `sales_pg`, esquema `sales` y roles `app_owner`, `app_rw` y `app_ro` creados previamente.
- `pgloader` 3.6.9 instalado y operativo.

> **Nota:** Los nombres de tablas usados en ejemplos funcionales son `customers`, `products`, `orders` y `order_items`. Si el conjunto suministrado por el instructor utiliza nombres distintos, sustituya esos nombres de forma consistente en los comandos de validación.

## Entorno de laboratorio

### Nodos y direcciones

| Nodo | Dirección IP | Función |
|---|---:|---|
| `pg-primary` | `192.168.56.10` | PostgreSQL destino, MariaDB origen local o servidor de migración |
| `pg-standby` | `192.168.56.11` | Réplica para prácticas posteriores; no se usa en esta práctica |

### Software y parámetros relevantes

| Componente | Versión o valor requerido |
|---|---|
| Sistema operativo | Ubuntu Server 22.04.4 LTS x86_64 |
| PostgreSQL Server | 16.2 |
| PostgreSQL Client Utilities | 16.2 |
| MariaDB Server | 10.11.7 |
| pgloader | 3.6.9 |
| Base origen | `source_sales` |
| Base destino | `sales_pg` |
| Esquema destino | `sales` |
| Puerto PostgreSQL | `5432/TCP` |
| Puerto MariaDB | `3306/TCP` |
| Zona horaria | UTC |

### Comprobación inicial del entorno

En `pg-primary`, compruebe versiones, conectividad y resolución de nombres.

```bash
hostnamectl --static
timedatectl status | grep "Time zone"

psql --version
pgloader --version
mysql --version

getent hosts pg-primary
getent hosts pg-standby

sudo -u postgres psql -d sales_pg -c "SELECT version();"
sudo -u postgres psql -d sales_pg -c "SHOW TimeZone;"
```

Resultado esperado:

- El nombre del nodo es `pg-primary`.
- PostgreSQL Server y cliente muestran versión `16.2`.
- `pgloader` muestra versión `3.6.9`.
- La zona horaria del sistema y PostgreSQL es `UTC`.
- La conexión a `sales_pg` es satisfactoria.

---

## Procedimiento paso a paso

### Paso 1. Definir el plan de migración, corte y reversión

**Objetivo:** Documentar la estrategia de migración de laboratorio, sus criterios de aceptación y el punto de reversión.

**Instrucciones:**

1. Cree un directorio protegido para la evidencia de migración.

   ```bash
   sudo install -d -m 0750 -o postgres -g postgres /var/lib/postgresql/migration-lab03
   sudo -u postgres mkdir -p /var/lib/postgresql/migration-lab03/{evidence,logs,config}
   ```

2. Cree un archivo de planificación en el directorio de evidencia.

   ```bash
   sudo -u postgres tee /var/lib/postgresql/migration-lab03/evidence/plan_migracion.yaml > /dev/null <<'EOF'
   migracion:
     laboratorio: "Lab 03-00-01"
     origen: "MariaDB 10.11.7 / source_sales"
     destino: "PostgreSQL 16.2 / sales_pg / esquema sales"
     estrategia: "Migración con interrupción planificada"
     rto_objetivo: "45 minutos"
     rpo_objetivo: "0 minutos desde el inicio de la ventana de corte"
     ventana_corte:
       inicio: "Después de capturar inventario y línea base"
       accion: "No permitir escrituras de aplicación en source_sales durante la carga final"
     criterios_aceptacion:
       - "Conteos por tabla coincidentes"
       - "Agregados monetarios coincidentes"
       - "Claves primarias y foráneas válidas"
       - "Índices válidos"
       - "Secuencias o identity columns alineadas con los valores máximos"
       - "Consultas funcionales críticas correctas"
     reversion:
       condicion: "Error crítico antes de aceptar escrituras de aplicación en PostgreSQL"
       accion: "Mantener o restaurar el acceso de aplicación a MariaDB"
       limite: "Una vez aceptadas escrituras productivas en PostgreSQL, no volver al origen sin plan de resincronización"
   EOF
   ```

3. Revise el plan.

   ```bash
   sudo -u postgres cat /var/lib/postgresql/migration-lab03/evidence/plan_migracion.yaml
   ```

4. Registre la hora de inicio de la ventana de trabajo.

   ```bash
   date -u +"%Y-%m-%dT%H:%M:%SZ" | \
     sudo -u postgres tee /var/lib/postgresql/migration-lab03/evidence/inicio_migracion_utc.txt
   ```

**Resultado esperado:**

Existe un archivo `plan_migracion.yaml` que identifica:

- La estrategia seleccionada.
- El RTO y RPO de laboratorio.
- Los criterios de aceptación.
- La condición explícita de reversión.

**Verificación:**

```bash
sudo -u postgres grep -E "estrategia|rto_objetivo|rpo_objetivo|reversion" \
  /var/lib/postgresql/migration-lab03/evidence/plan_migracion.yaml
```

La estrategia debe ser una migración con interrupción planificada. En un escenario real de alta disponibilidad, una carga inicial más captura de cambios podría ser más adecuada, pero no forma parte del alcance de esta práctica.

---

### Paso 2. Inventariar la base de datos MariaDB de origen

**Objetivo:** Obtener una línea base de estructura, objetos y volumen de datos antes de migrar.

**Instrucciones:**

1. Defina variables para la conexión MariaDB. Ajuste `MARIADB_HOST` si la base fuente está en otro servidor.

   ```bash
   export MARIADB_HOST="127.0.0.1"
   export MARIADB_PORT="3306"
   export MARIADB_DB="source_sales"
   export MARIADB_ADMIN="root"
   ```

2. Compruebe la base de origen y su codificación.

   ```bash
   mysql -h "${MARIADB_HOST}" -P "${MARIADB_PORT}" -u "${MARIADB_ADMIN}" -p \
     -e "
     SELECT
       SCHEMA_NAME,
       DEFAULT_CHARACTER_SET_NAME,
       DEFAULT_COLLATION_NAME
     FROM information_schema.SCHEMATA
     WHERE SCHEMA_NAME = '${MARIADB_DB}';
     "
   ```

3. Liste las tablas base y las vistas. Guarde el resultado como evidencia.

   ```bash
   mysql -h "${MARIADB_HOST}" -P "${MARIADB_PORT}" -u "${MARIADB_ADMIN}" -p -N \
     -e "
     SELECT TABLE_TYPE, TABLE_NAME
     FROM information_schema.TABLES
     WHERE TABLE_SCHEMA = '${MARIADB_DB}'
     ORDER BY TABLE_TYPE, TABLE_NAME;
     " | tee /tmp/source_objects.tsv

   sudo install -m 0640 -o postgres -g postgres \
     /tmp/source_objects.tsv \
     /var/lib/postgresql/migration-lab03/evidence/source_objects.tsv
   ```

4. Inventaríe columnas, tipos, nulabilidad, valores por defecto y atributos automáticos.

   ```bash
   mysql -h "${MARIADB_HOST}" -P "${MARIADB_PORT}" -u "${MARIADB_ADMIN}" -p \
     -e "
     SELECT
       TABLE_NAME,
       ORDINAL_POSITION,
       COLUMN_NAME,
       COLUMN_TYPE,
       IS_NULLABLE,
       COLUMN_DEFAULT,
       EXTRA
     FROM information_schema.COLUMNS
     WHERE TABLE_SCHEMA = '${MARIADB_DB}'
     ORDER BY TABLE_NAME, ORDINAL_POSITION;
     " | tee /tmp/source_columns.txt

   sudo install -m 0640 -o postgres -g postgres \
     /tmp/source_columns.txt \
     /var/lib/postgresql/migration-lab03/evidence/source_columns.txt
   ```

5. Identifique claves primarias, claves foráneas y restricciones.

   ```bash
   mysql -h "${MARIADB_HOST}" -P "${MARIADB_PORT}" -u "${MARIADB_ADMIN}" -p \
     -e "
     SELECT
       tc.TABLE_NAME,
       tc.CONSTRAINT_NAME,
       tc.CONSTRAINT_TYPE,
       kcu.COLUMN_NAME,
       kcu.REFERENCED_TABLE_NAME,
       kcu.REFERENCED_COLUMN_NAME
     FROM information_schema.TABLE_CONSTRAINTS AS tc
     LEFT JOIN information_schema.KEY_COLUMN_USAGE AS kcu
       ON tc.CONSTRAINT_SCHEMA = kcu.CONSTRAINT_SCHEMA
      AND tc.TABLE_NAME = kcu.TABLE_NAME
      AND tc.CONSTRAINT_NAME = kcu.CONSTRAINT_NAME
     WHERE tc.CONSTRAINT_SCHEMA = '${MARIADB_DB}'
     ORDER BY tc.TABLE_NAME, tc.CONSTRAINT_TYPE, tc.CONSTRAINT_NAME, kcu.ORDINAL_POSITION;
     " | tee /tmp/source_constraints.txt

   sudo install -m 0640 -o postgres -g postgres \
     /tmp/source_constraints.txt \
     /var/lib/postgresql/migration-lab03/evidence/source_constraints.txt
   ```

6. Liste índices no asociados a claves primarias.

   ```bash
   mysql -h "${MARIADB_HOST}" -P "${MARIADB_PORT}" -u "${MARIADB_ADMIN}" -p \
     -e "
     SELECT
       TABLE_NAME,
       INDEX_NAME,
       NON_UNIQUE,
       SEQ_IN_INDEX,
       COLUMN_NAME
     FROM information_schema.STATISTICS
     WHERE TABLE_SCHEMA = '${MARIADB_DB}'
     ORDER BY TABLE_NAME, INDEX_NAME, SEQ_IN_INDEX;
     " | tee /tmp/source_indexes.txt

   sudo install -m 0640 -o postgres -g postgres \
     /tmp/source_indexes.txt \
     /var/lib/postgresql/migration-lab03/evidence/source_indexes.txt
   ```

7. Genere la línea base de conteos de filas por tabla. Este archivo será comparado posteriormente con PostgreSQL.

   ```bash
   mysql -h "${MARIADB_HOST}" -P "${MARIADB_PORT}" -u "${MARIADB_ADMIN}" -p -N \
     -e "
     SELECT TABLE_NAME
     FROM information_schema.TABLES
     WHERE TABLE_SCHEMA = '${MARIADB_DB}'
       AND TABLE_TYPE = 'BASE TABLE'
     ORDER BY TABLE_NAME;
     " > /tmp/source_table_list.txt

   : > /tmp/source_counts.tsv

   while read -r table_name; do
     mysql -h "${MARIADB_HOST}" -P "${MARIADB_PORT}" -u "${MARIADB_ADMIN}" -p -N \
       -e "SELECT '${table_name}', COUNT(*) FROM \`${MARIADB_DB}\`.\`${table_name}\`;" \
       >> /tmp/source_counts.tsv
   done < /tmp/source_table_list.txt

   sort /tmp/source_counts.tsv | tee /tmp/source_counts_sorted.tsv

   sudo install -m 0640 -o postgres -g postgres \
     /tmp/source_counts_sorted.tsv \
     /var/lib/postgresql/migration-lab03/evidence/source_counts.tsv
   ```

8. Busque tipos potencialmente incompatibles o que requieren decisión de conversión.

   ```bash
   mysql -h "${MARIADB_HOST}" -P "${MARIADB_PORT}" -u "${MARIADB_ADMIN}" -p \
     -e "
     SELECT
       TABLE_NAME,
       COLUMN_NAME,
       COLUMN_TYPE,
       COLUMN_DEFAULT,
       EXTRA
     FROM information_schema.COLUMNS
     WHERE TABLE_SCHEMA = '${MARIADB_DB}'
       AND (
         DATA_TYPE IN ('tinyint', 'datetime', 'timestamp', 'enum', 'set', 'json', 'bit', 'year')
         OR EXTRA LIKE '%auto_increment%'
       )
     ORDER BY TABLE_NAME, ORDINAL_POSITION;
     "
   ```

**Resultado esperado:**

Se han creado evidencias de:

- Tablas y vistas.
- Columnas y tipos.
- Restricciones e índices.
- Conteos de línea base.
- Campos que requieren mapeo o revisión antes de la carga.

**Verificación:**

```bash
sudo -u postgres ls -lh /var/lib/postgresql/migration-lab03/evidence/
sudo -u postgres cat /var/lib/postgresql/migration-lab03/evidence/source_counts.tsv
```

---

### Paso 3. Analizar compatibilidad y definir decisiones de conversión

**Objetivo:** Decidir cómo se convertirán los tipos y objetos no portables antes de ejecutar `pgloader`.

**Instrucciones:**

1. Revise los resultados del inventario e identifique los objetos que requieren tratamiento.

2. Utilice la siguiente matriz como referencia para documentar la conversión.

   | MariaDB | PostgreSQL recomendado | Consideración |
   |---|---|---|
   | `INT AUTO_INCREMENT` | `integer` con secuencia o `GENERATED ... AS IDENTITY` | `pgloader` crea secuencias y puede reiniciarlas. |
   | `BIGINT AUTO_INCREMENT` | `bigint` con secuencia o identidad | Revisar que el máximo no exceda el rango. |
   | `TINYINT(1)` | `boolean` solo si contiene exclusivamente `0` y `1` | Si representa estados numéricos, usar `smallint`. |
   | `TINYINT(n)` | `smallint` | PostgreSQL no usa el ancho de visualización de MariaDB. |
   | `DATETIME` | `timestamp without time zone` | Usar UTC de forma consistente si representa fecha y hora UTC. |
   | `TIMESTAMP` | `timestamp` o `timestamptz` | Revisar semántica de zona horaria de la aplicación. |
   | `DECIMAL(p,s)` | `numeric(p,s)` | Adecuado para importes monetarios. |
   | `VARCHAR`, `TEXT` | `varchar`, `text` | Confirmar codificación UTF-8. |
   | `ENUM`, `SET` | `text`, tabla de referencia o tipo `ENUM` PostgreSQL | Requiere revisión funcional. |
   | Funciones MySQL | Funciones PostgreSQL equivalentes | No asumir compatibilidad automática. |

3. Para cada posible campo booleano, confirme que sus valores son binarios. Ejemplo para una columna `is_active`:

   ```bash
   mysql -h "${MARIADB_HOST}" -P "${MARIADB_PORT}" -u "${MARIADB_ADMIN}" -p \
     -e "
     SELECT is_active, COUNT(*) AS total
     FROM source_sales.customers
     GROUP BY is_active
     ORDER BY is_active;
     "
   ```

4. Si existe una columna `TINYINT(1)` con valores diferentes de `0` y `1`, **no** la convierta a `boolean`; manténgala como `smallint`.

5. Cree un registro de decisiones. Ajuste las entradas de acuerdo con el inventario real.

   ```bash
   sudo -u postgres tee /var/lib/postgresql/migration-lab03/evidence/matriz_conversion.md > /dev/null <<'EOF'
   # Matriz de conversión

   | Objeto origen | Tipo/objeto MariaDB | Decisión PostgreSQL | Justificación |
   |---|---|---|---|
   | Claves automáticas | AUTO_INCREMENT | Secuencia generada por pgloader; revisar reinicio | Evita colisiones en nuevas inserciones |
   | Fechas de negocio | DATETIME | timestamp without time zone | Los valores se interpretan en UTC |
   | Importes | DECIMAL(p,s) | numeric(p,s) | Conserva precisión decimal |
   | Flags binarios validados | TINYINT(1) | boolean, si solo usa 0/1 | Semántica lógica |
   | Estados numéricos | TINYINT | smallint | Puede tener más de dos valores |
   | Vistas y rutinas | SQL específico de MariaDB | Recreación manual tras revisión | Dialecto SQL no portable |
   EOF
   ```

6. Identifique procedimientos, funciones, triggers y eventos de MariaDB que deben ser recreados manualmente o excluidos del alcance.

   ```bash
   mysql -h "${MARIADB_HOST}" -P "${MARIADB_PORT}" -u "${MARIADB_ADMIN}" -p \
     -e "
     SELECT ROUTINE_TYPE, ROUTINE_NAME
     FROM information_schema.ROUTINES
     WHERE ROUTINE_SCHEMA = '${MARIADB_DB}'
     UNION ALL
     SELECT 'TRIGGER', TRIGGER_NAME
     FROM information_schema.TRIGGERS
     WHERE TRIGGER_SCHEMA = '${MARIADB_DB}'
     UNION ALL
     SELECT 'EVENT', EVENT_NAME
     FROM information_schema.EVENTS
     WHERE EVENT_SCHEMA = '${MARIADB_DB}'
     ORDER BY 1, 2;
     " | tee /tmp/source_nonportable_objects.txt

   sudo install -m 0640 -o postgres -g postgres \
     /tmp/source_nonportable_objects.txt \
     /var/lib/postgresql/migration-lab03/evidence/source_nonportable_objects.txt
   ```

**Resultado esperado:**

Existe una decisión documentada para los tipos no portables y los objetos que no deben asumirse compatibles automáticamente.

**Verificación:**

```bash
sudo -u postgres cat /var/lib/postgresql/migration-lab03/evidence/matriz_conversion.md
sudo -u postgres cat /var/lib/postgresql/migration-lab03/evidence/source_nonportable_objects.txt
```

---

### Paso 4. Crear credenciales restringidas para la migración

**Objetivo:** Ejecutar la migración con cuentas dedicadas y privilegios mínimos necesarios.

**Instrucciones:**

1. En MariaDB, cree un usuario de solo lectura. Si `pgloader` se ejecuta desde el mismo host que MariaDB, use `localhost`; si se ejecuta desde otro host, sustituya el origen permitido por la dirección IP correspondiente.

   ```sql
   CREATE USER IF NOT EXISTS 'migrate_reader'@'localhost'
     IDENTIFIED BY 'Cambiar_MariaDB_2026!';

   GRANT SELECT ON source_sales.* TO 'migrate_reader'@'localhost';

   FLUSH PRIVILEGES;

   SHOW GRANTS FOR 'migrate_reader'@'localhost';
   ```

   Ejecute estas sentencias desde el cliente MariaDB administrativo:

   ```bash
   mysql -h "${MARIADB_HOST}" -P "${MARIADB_PORT}" -u "${MARIADB_ADMIN}" -p
   ```

2. Compruebe que el usuario restringido puede consultar datos, pero no modificar la base.

   ```bash
   mysql -h "${MARIADB_HOST}" -P "${MARIADB_PORT}" -u migrate_reader -p \
     -e "SELECT COUNT(*) AS total_tablas FROM information_schema.TABLES WHERE TABLE_SCHEMA='source_sales';"
   ```

3. En PostgreSQL, cree un rol dedicado para la carga. Use una contraseña distinta de la usada en MariaDB.

   ```bash
   sudo -u postgres psql -d sales_pg <<'SQL'
   CREATE ROLE migrate_loader
     LOGIN
     PASSWORD 'Cambiar_PostgreSQL_2026!'
     NOSUPERUSER
     NOCREATEDB
     NOCREATEROLE
     NOINHERIT;

   GRANT CONNECT, TEMPORARY, CREATE ON DATABASE sales_pg TO migrate_loader;
   SQL
   ```

4. Verifique el rol PostgreSQL.

   ```bash
   sudo -u postgres psql -d sales_pg -c "\du migrate_loader"
   sudo -u postgres psql -d sales_pg -c "
   SELECT
     has_database_privilege('migrate_loader', 'sales_pg', 'CONNECT') AS puede_conectar,
     has_database_privilege('migrate_loader', 'sales_pg', 'CREATE') AS puede_crear_esquemas;
   "
   ```

5. Cree un archivo de configuración con permisos restrictivos. Sustituya las contraseñas y codifique caracteres especiales si se usan dentro de una URI.

   ```bash
   sudo -u postgres tee /var/lib/postgresql/migration-lab03/config/pgloader_sales.load > /dev/null <<'EOF'
   LOAD DATABASE
        FROM mysql://migrate_reader:Cambiar_MariaDB_2026!@127.0.0.1:3306/source_sales
        INTO postgresql://migrate_loader:Cambiar_PostgreSQL_2026!@127.0.0.1:5432/sales_pg

   WITH include drop,
        create tables,
        create indexes,
        reset sequences,
        foreign keys

   SET work_mem to '64MB',
       maintenance_work_mem to '256MB',
       client_encoding to 'utf8'

   CAST
        type tinyint to smallint,
        type datetime to timestamp without time zone

   ALTER SCHEMA 'source_sales' RENAME TO 'sales'
   ;
   EOF

   sudo chmod 0600 /var/lib/postgresql/migration-lab03/config/pgloader_sales.load
   ```

> **Importante:** La regla `type tinyint to smallint` es segura para todos los `TINYINT`. Si el análisis del paso 3 confirma que todos los campos `TINYINT(1)` relevantes son estrictamente binarios, se puede cambiar la regla por una conversión más específica a `boolean`. No aplique una conversión global a `boolean` sin validar los valores existentes.

**Resultado esperado:**

- MariaDB dispone de un usuario de solo lectura.
- PostgreSQL dispone de un rol limitado para la carga.
- El archivo de `pgloader` solo es legible por el usuario `postgres`.

**Verificación:**

```bash
sudo ls -l /var/lib/postgresql/migration-lab03/config/pgloader_sales.load
sudo -u postgres psql -d sales_pg -c "\du migrate_loader"
```

---

### Paso 5. Preparar el destino PostgreSQL y ejecutar la carga

**Objetivo:** Limpiar el esquema de destino, ejecutar `pgloader` y registrar la salida de la migración.

**Instrucciones:**

1. Realice un respaldo lógico previo del estado actual de `sales_pg`. Este respaldo permite volver al estado anterior de laboratorio si fuera necesario.

   ```bash
   sudo -u postgres pg_dump \
     --format=custom \
     --file=/var/lib/postgresql/migration-lab03/evidence/sales_pg_pre_migration.dump \
     sales_pg
   ```

2. Compruebe que el respaldo se generó correctamente.

   ```bash
   sudo -u postgres pg_restore --list \
     /var/lib/postgresql/migration-lab03/evidence/sales_pg_pre_migration.dump | head
   ```

3. Elimine y recree el esquema de destino. Esta acción asegura que la carga se realiza sobre un destino limpio.

   ```bash
   sudo -u postgres psql -d sales_pg <<'SQL'
   DROP SCHEMA IF EXISTS sales CASCADE;
   CREATE SCHEMA sales AUTHORIZATION migrate_loader;
   REVOKE ALL ON SCHEMA sales FROM PUBLIC;
   SQL
   ```

4. Inicie la carga con `pgloader`, guardando toda la salida en un archivo de log.

   ```bash
   sudo -u postgres bash -c '
   pgloader /var/lib/postgresql/migration-lab03/config/pgloader_sales.load \
     2>&1 | tee /var/lib/postgresql/migration-lab03/logs/pgloader_sales.log
   '
   ```

5. Revise el resumen final de `pgloader`.

   ```bash
   sudo -u postgres tail -n 80 \
     /var/lib/postgresql/migration-lab03/logs/pgloader_sales.log
   ```

6. Busque errores, advertencias y tablas rechazadas.

   ```bash
   sudo -u postgres grep -Ei "error|fatal|reject|failed|warning" \
     /var/lib/postgresql/migration-lab03/logs/pgloader_sales.log || true
   ```

7. Compruebe que el esquema `sales` y sus tablas existen en PostgreSQL.

   ```bash
   sudo -u postgres psql -d sales_pg -c "\dn+ sales"
   sudo -u postgres psql -d sales_pg -c "\dt sales.*"
   ```

**Resultado esperado:**

- `pgloader` termina con un resumen de carga.
- Las tablas del origen aparecen bajo el esquema `sales`.
- El log no contiene errores fatales ni tablas rechazadas sin resolver.
- Se han creado tablas, índices, claves foráneas y secuencias según la configuración.

**Verificación:**

```bash
sudo -u postgres psql -d sales_pg -c "
SELECT
  table_schema,
  table_name
FROM information_schema.tables
WHERE table_schema = 'sales'
  AND table_type = 'BASE TABLE'
ORDER BY table_name;
"
```

---

### Paso 6. Ajustar propietarios, privilegios y objetos posteriores a la carga

**Objetivo:** Transferir la propiedad de los objetos migrados a `app_owner` y restaurar el modelo de privilegios de aplicación.

**Instrucciones:**

1. Transfiera la propiedad del esquema a `app_owner`.

   ```bash
   sudo -u postgres psql -d sales_pg -c \
     "ALTER SCHEMA sales OWNER TO app_owner;"
   ```

2. Cambie el propietario de todas las tablas, vistas materializadas, secuencias y vistas del esquema.

   ```bash
   sudo -u postgres psql -d sales_pg <<'SQL'
   DO $$
   DECLARE
     objeto record;
   BEGIN
     FOR objeto IN
       SELECT c.relkind, n.nspname, c.relname
       FROM pg_class AS c
       JOIN pg_namespace AS n ON n.oid = c.relnamespace
       WHERE n.nspname = 'sales'
         AND c.relkind IN ('r', 'p', 'S', 'v', 'm', 'f')
   LOOP
     EXECUTE format(
       'ALTER %s %I.%I OWNER TO app_owner',
       CASE objeto.relkind
         WHEN 'S' THEN 'SEQUENCE'
         WHEN 'v' THEN 'VIEW'
         WHEN 'm' THEN 'MATERIALIZED VIEW'
         WHEN 'f' THEN 'FOREIGN TABLE'
         ELSE 'TABLE'
       END,
       objeto.nspname,
       objeto.relname
     );
   END LOOP;
   END $$;
   SQL
   ```

3. Aplique privilegios de acceso al esquema, tablas y secuencias.

   ```bash
   sudo -u postgres psql -d sales_pg <<'SQL'
   REVOKE ALL ON SCHEMA sales FROM PUBLIC;
   GRANT USAGE ON SCHEMA sales TO app_rw, app_ro;

   GRANT SELECT, INSERT, UPDATE, DELETE
     ON ALL TABLES IN SCHEMA sales
     TO app_rw;

   GRANT SELECT
     ON ALL TABLES IN SCHEMA sales
     TO app_ro;

   GRANT USAGE, SELECT
     ON ALL SEQUENCES IN SCHEMA sales
     TO app_rw;

   REVOKE ALL
     ON ALL TABLES IN SCHEMA sales
     FROM PUBLIC;

   REVOKE ALL
     ON ALL SEQUENCES IN SCHEMA sales
     FROM PUBLIC;

   ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA sales
     GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;

   ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA sales
     GRANT SELECT ON TABLES TO app_ro;

   ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA sales
     GRANT USAGE, SELECT ON SEQUENCES TO app_rw;
   SQL
   ```

4. Si una columna fue identificada como un indicador booleano confirmado, conviértala después de validar que contiene únicamente `0` y `1`. Ejemplo para `sales.customers.is_active`:

   ```bash
   sudo -u postgres psql -d sales_pg <<'SQL'
   ALTER TABLE sales.customers
     ALTER COLUMN is_active DROP DEFAULT;

   ALTER TABLE sales.customers
     ALTER COLUMN is_active TYPE boolean
     USING CASE
       WHEN is_active = 1 THEN true
       WHEN is_active = 0 THEN false
       ELSE NULL
     END;

   ALTER TABLE sales.customers
     ALTER COLUMN is_active SET DEFAULT true;
   SQL
   ```

5. Analice las tablas migradas para actualizar estadísticas del planificador.

   ```bash
   sudo -u postgres vacuumdb \
     --dbname=sales_pg \
     --analyze-in-stages \
     --schema=sales
   ```

**Resultado esperado:**

- `app_owner` es propietario de los objetos de aplicación.
- `app_rw` puede leer y modificar datos.
- `app_ro` tiene acceso de solo lectura.
- El rol `PUBLIC` no dispone de permisos sobre tablas ni secuencias del esquema.
- Las estadísticas están actualizadas.

**Verificación:**

```bash
sudo -u postgres psql -d sales_pg -c "\dn+ sales"
sudo -u postgres psql -d sales_pg -c "\dp sales.*"
sudo -u postgres psql -d sales_pg -c "\ds sales.*"
```

---

### Paso 7. Generar conteos del destino y comparar con MariaDB

**Objetivo:** Confirmar que el número de filas por tabla coincide entre origen y destino.

**Instrucciones:**

1. Obtenga la lista de tablas migradas en PostgreSQL.

   ```bash
   sudo -u postgres psql -d sales_pg -At \
     -c "
     SELECT table_name
     FROM information_schema.tables
     WHERE table_schema = 'sales'
       AND table_type = 'BASE TABLE'
     ORDER BY table_name;
     " > /tmp/target_table_list.txt
   ```

2. Genere los conteos de filas del destino.

   ```bash
   : > /tmp/target_counts.tsv

   while read -r table_name; do
     sudo -u postgres psql -d sales_pg -At \
       -c "SELECT '${table_name}', COUNT(*) FROM sales.\"${table_name}\";" \
       >> /tmp/target_counts.tsv
   done < /tmp/target_table_list.txt

   sort /tmp/target_counts.tsv | tee /tmp/target_counts_sorted.tsv

   sudo install -m 0640 -o postgres -g postgres \
     /tmp/target_counts_sorted.tsv \
     /var/lib/postgresql/migration-lab03/evidence/target_counts.tsv
   ```

3. Compare la línea base de MariaDB con el resultado PostgreSQL.

   ```bash
   diff -u \
     /var/lib/postgresql/migration-lab03/evidence/source_counts.tsv \
     /var/lib/postgresql/migration-lab03/evidence/target_counts.tsv
   ```

4. Si `diff` no muestra salida, guarde una confirmación.

   ```bash
   echo "Conteos coincidentes: $(date -u +"%Y-%m-%dT%H:%M:%SZ")" | \
     sudo -u postgres tee \
     /var/lib/postgresql/migration-lab03/evidence/validacion_conteos.txt
   ```

**Resultado esperado:**

El comando `diff -u` no muestra diferencias. Cada tabla base de MariaDB tiene el mismo número de filas en PostgreSQL.

**Verificación:**

```bash
sudo -u postgres cat /var/lib/postgresql/migration-lab03/evidence/validacion_conteos.txt
```

> Si existen tablas técnicas que deliberadamente no se migraron, documente su exclusión y compare únicamente las tablas incluidas en el alcance.

---

## Validación y pruebas

### Validación de estructura

Ejecute las siguientes consultas en PostgreSQL para comprobar que existen las relaciones esperadas.

```bash
sudo -u postgres psql -d sales_pg -c "\d sales.customers"
sudo -u postgres psql -d sales_pg -c "\d sales.products"
sudo -u postgres psql -d sales_pg -c "\d sales.orders"
sudo -u postgres psql -d sales_pg -c "\d sales.order_items"
```

Compruebe columnas, tipos, claves primarias, claves foráneas e índices. Ajuste los nombres de tabla si el conjunto de datos proporcionado usa otra nomenclatura.

### Validación de restricciones

Liste las restricciones PostgreSQL y confirme que están validadas.

```bash
sudo -u postgres psql -d sales_pg -c "
SELECT
  conrelid::regclass AS tabla,
  conname AS restriccion,
  contype AS tipo,
  convalidated AS validada
FROM pg_constraint
WHERE connamespace = 'sales'::regnamespace
ORDER BY tabla::text, tipo, restriccion;
"
```

Resultado esperado:

- Las restricciones de tipo `p` representan claves primarias.
- Las restricciones de tipo `f` representan claves foráneas.
- La columna `validada` muestra `t` para todas las restricciones existentes.

Compruebe específicamente que no existan claves foráneas no validadas:

```bash
sudo -u postgres psql -d sales_pg -c "
SELECT conname, conrelid::regclass
FROM pg_constraint
WHERE connamespace = 'sales'::regnamespace
  AND contype = 'f'
  AND NOT convalidated;
"
```

La consulta debe devolver cero filas.

### Validación de índices

```bash
sudo -u postgres psql -d sales_pg -c "
SELECT
  schemaname,
  tablename,
  indexname,
  indexdef
FROM pg_indexes
WHERE schemaname = 'sales'
ORDER BY tablename, indexname;
"
```

Compruebe además que no haya índices inválidos:

```bash
sudo -u postgres psql -d sales_pg -c "
SELECT
  c.relname AS indice,
  i.indisvalid,
  i.indisready
FROM pg_index AS i
JOIN pg_class AS c ON c.oid = i.indexrelid
JOIN pg_namespace AS n ON n.oid = c.relnamespace
WHERE n.nspname = 'sales'
  AND (NOT i.indisvalid OR NOT i.indisready);
"
```

Resultado esperado: cero filas.

### Validación de secuencias e identidades

Liste las columnas con valores automáticos o secuencias asociadas.

```bash
sudo -u postgres psql -d sales_pg -c "
SELECT
  table_name,
  column_name,
  column_default,
  is_identity,
  identity_generation,
  pg_get_serial_sequence(format('%I.%I', table_schema, table_name), column_name) AS secuencia
FROM information_schema.columns
WHERE table_schema = 'sales'
  AND (
    is_identity = 'YES'
    OR column_default LIKE 'nextval%'
  )
ORDER BY table_name, ordinal_position;
"
```

Para cada tabla con identificador numérico, compare el máximo de la columna con el valor de la secuencia. Ejemplo para `orders.id`:

```bash
sudo -u postgres psql -d sales_pg -c "
SELECT MAX(id) AS maximo_id
FROM sales.orders;
"

sudo -u postgres psql -d sales_pg -c "
SELECT last_value, is_called
FROM sales.orders_id_seq;
"
```

El `last_value` debe ser igual o superior al máximo identificador cargado. Si fuese inferior, reinicie la secuencia con el valor correcto:

```bash
sudo -u postgres psql -d sales_pg -c "
SELECT setval(
  pg_get_serial_sequence('sales.orders', 'id'),
  COALESCE((SELECT MAX(id) FROM sales.orders), 1),
  true
);
"
```

### Comparación de agregados monetarios

Ejecute una consulta equivalente en MariaDB y PostgreSQL. Ajuste los nombres de columnas según el modelo real.

En MariaDB:

```bash
mysql -h "${MARIADB_HOST}" -P "${MARIADB_PORT}" -u "${MARIADB_ADMIN}" -p \
  -e "
  SELECT
    COUNT(*) AS total_pedidos,
    COALESCE(SUM(total_amount), 0) AS importe_total,
    COALESCE(AVG(total_amount), 0) AS importe_promedio
  FROM source_sales.orders;
  "
```

En PostgreSQL:

```bash
sudo -u postgres psql -d sales_pg -c "
SELECT
  COUNT(*) AS total_pedidos,
  COALESCE(SUM(total_amount), 0) AS importe_total,
  COALESCE(AVG(total_amount), 0) AS importe_promedio
FROM sales.orders;
"
```

Los valores deben coincidir dentro de la precisión decimal definida para los importes.

Para validar líneas de pedido:

En MariaDB:

```bash
mysql -h "${MARIADB_HOST}" -P "${MARIADB_PORT}" -u "${MARIADB_ADMIN}" -p \
  -e "
  SELECT
    COUNT(*) AS total_lineas,
    COALESCE(SUM(quantity), 0) AS unidades,
    COALESCE(SUM(quantity * unit_price), 0) AS importe_lineas
  FROM source_sales.order_items;
  "
```

En PostgreSQL:

```bash
sudo -u postgres psql -d sales_pg -c "
SELECT
  COUNT(*) AS total_lineas,
  COALESCE(SUM(quantity), 0) AS unidades,
  COALESCE(SUM(quantity * unit_price), 0) AS importe_lineas
FROM sales.order_items;
"
```

### Pruebas funcionales de negocio

Ejecute una consulta de pedidos por cliente:

```bash
sudo -u postgres psql -d sales_pg -c "
SELECT
  c.id AS customer_id,
  c.name AS customer_name,
  COUNT(o.id) AS total_orders,
  COALESCE(SUM(o.total_amount), 0) AS total_spent
FROM sales.customers AS c
LEFT JOIN sales.orders AS o
  ON o.customer_id = c.id
GROUP BY c.id, c.name
ORDER BY total_spent DESC, customer_id
LIMIT 10;
"
```

Ejecute una consulta de ventas por producto:

```bash
sudo -u postgres psql -d sales_pg -c "
SELECT
  p.id AS product_id,
  p.name AS product_name,
  SUM(oi.quantity) AS units_sold,
  SUM(oi.quantity * oi.unit_price) AS sales_amount
FROM sales.order_items AS oi
JOIN sales.products AS p
  ON p.id = oi.product_id
GROUP BY p.id, p.name
ORDER BY sales_amount DESC
LIMIT 10;
"
```

Verifique permisos como rol de lectura:

```bash
sudo -u postgres psql -d sales_pg <<'SQL'
SET ROLE app_ro;
SELECT COUNT(*) AS total_pedidos FROM sales.orders;
RESET ROLE;
SQL
```

Verifique que el rol de lectura no puede modificar datos:

```bash
sudo -u postgres psql -d sales_pg <<'SQL'
SET ROLE app_ro;
BEGIN;
DELETE FROM sales.orders WHERE false;
ROLLBACK;
RESET ROLE;
SQL
```

Resultado esperado: PostgreSQL debe rechazar el `DELETE` con un error de permisos.

### Criterios de aceptación de la migración

Considere la migración aceptada únicamente si se cumple todo lo siguiente:

- [ ] Los conteos por tabla coinciden entre MariaDB y PostgreSQL.
- [ ] Los agregados monetarios y cantidades de negocio coinciden.
- [ ] No hay claves foráneas no validadas.
- [ ] No hay índices inválidos.
- [ ] Las secuencias o identidades están alineadas con los valores máximos cargados.
- [ ] Las consultas funcionales críticas devuelven resultados coherentes.
- [ ] `app_rw` y `app_ro` tienen únicamente los permisos esperados.
- [ ] El log de `pgloader` no contiene errores sin resolver.
- [ ] Se documentaron objetos no migrados automáticamente, como vistas, triggers, funciones o eventos MariaDB.

---

## Solución de problemas

### Incidencia 1: `pgloader` falla al conectarse a MariaDB o PostgreSQL

**Síntomas:**

- El log contiene mensajes como `Access denied for user`, `Connection refused`, `authentication failed` o `Failed to connect`.
- La carga termina antes de crear tablas en PostgreSQL.

**Causa:**

Las credenciales son incorrectas, el usuario no tiene permisos suficientes, el servidor no escucha en la dirección indicada o existe un error en la URI de conexión. Las contraseñas con caracteres especiales pueden requerir codificación URL.

**Solución:**

1. Pruebe cada conexión por separado.

   ```bash
   mysql -h 127.0.0.1 -P 3306 -u migrate_reader -p -e "SELECT 1;"
   psql "postgresql://migrate_loader@127.0.0.1:5432/sales_pg" -c "SELECT 1;"
   ```

2. Confirme privilegios MariaDB:

   ```sql
   SHOW GRANTS FOR 'migrate_reader'@'localhost';
   ```

3. Confirme privilegios PostgreSQL:

   ```bash
   sudo -u postgres psql -d sales_pg -c "\du migrate_loader"
   ```

4. Si la contraseña contiene caracteres como `@`, `:`, `/`, `?` o `#`, reemplácelos por su equivalente codificado en URL dentro de la URI, o use una contraseña temporal de laboratorio sin dichos caracteres.

5. Corrija el archivo `.load`, mantenga permisos `0600` y ejecute de nuevo el paso 5.

---

### Incidencia 2: La carga termina, pero los conteos o agregados no coinciden

**Síntomas:**

- `diff -u` muestra diferencias entre `source_counts.tsv` y `target_counts.tsv`.
- Los importes de pedidos o líneas de pedido son distintos.
- El log de `pgloader` contiene advertencias de filas rechazadas o conversiones fallidas.

**Causa:**

La fuente recibió escrituras después de capturar la línea base, se produjo una conversión de tipos incorrecta, una tabla fue excluida, hubo filas rechazadas o una columna `TINYINT` fue interpretada con una semántica equivocada.

**Solución:**

1. Revise filas rechazadas, errores y advertencias.

   ```bash
   sudo -u postgres grep -Ei "reject|error|warning|failed" \
     /var/lib/postgresql/migration-lab03/logs/pgloader_sales.log
   ```

2. Compruebe que MariaDB no recibió cambios durante la ventana de migración. Si recibió cambios, vuelva a capturar la línea base o repita la carga desde un punto consistente.

3. Compare los conteos solo de la tabla afectada:

   ```bash
   mysql -h "${MARIADB_HOST}" -P "${MARIADB_PORT}" -u "${MARIADB_ADMIN}" -p \
     -e "SELECT COUNT(*) FROM source_sales.orders;"

   sudo -u postgres psql -d sales_pg -c \
     "SELECT COUNT(*) FROM sales.orders;"
   ```

4. Revise tipos de columnas y valores anómalos, especialmente `TINYINT`, fechas, importes `DECIMAL`, valores nulos y valores por defecto.

5. Corrija la regla `CAST` o la estructura de destino, elimine y recree el esquema `sales`, y repita la carga completa. No intente declarar la migración válida corrigiendo manualmente filas aisladas sin documentar la causa.

---

## Limpieza

> Ejecute esta sección solo después de que el instructor confirme que la migración fue validada y la evidencia fue revisada.

1. Elimine las credenciales de carga si no serán utilizadas nuevamente.

   ```bash
   sudo -u postgres psql -d sales_pg -c "DROP ROLE IF EXISTS migrate_loader;"
   ```

2. Elimine el usuario de solo lectura en MariaDB si fue creado exclusivamente para la práctica.

   ```sql
   DROP USER IF EXISTS 'migrate_reader'@'localhost';
   FLUSH PRIVILEGES;
   ```

3. Elimine el archivo de configuración que contiene contraseñas.

   ```bash
   sudo shred -u /var/lib/postgresql/migration-lab03/config/pgloader_sales.load
   ```

4. Conserve las evidencias, el log y el respaldo previo hasta completar las prácticas de respaldo y recuperación.

   ```bash
   sudo -u postgres find /var/lib/postgresql/migration-lab03 \
     -maxdepth 2 -type f -printf '%TY-%Tm-%Td %TT %p\n' | sort
   ```

5. Registre la hora de finalización.

   ```bash
   date -u +"%Y-%m-%dT%H:%M:%SZ" | \
     sudo -u postgres tee /var/lib/postgresql/migration-lab03/evidence/fin_migracion_utc.txt
   ```

## Resumen

En esta práctica se aplicó una estrategia de migración con interrupción planificada desde MariaDB hacia PostgreSQL. Se construyó un inventario técnico del origen, se analizaron incompatibilidades de tipos y objetos, se utilizaron credenciales restringidas y se ejecutó una carga con `pgloader`.

La aceptación de la migración se basó en evidencia verificable: conteos por tabla, agregados monetarios, restricciones, índices, secuencias, permisos y consultas de negocio. La base `sales_pg` validada queda preparada para ser protegida mediante respaldos lógicos y físicos en prácticas posteriores, y para su replicación hacia `pg-standby`.

### Recursos opcionales

- [Documentación oficial de PostgreSQL: Migración](https://www.postgresql.org/docs/current/migration.html)
- [Documentación oficial de PostgreSQL: Roles y privilegios](https://www.postgresql.org/docs/current/user-manag.html)
- [Documentación oficial de PostgreSQL: Secuencias](https://www.postgresql.org/docs/current/functions-sequence.html)
- [Documentación de pgloader](https://pgloader.readthedocs.io/)
- [Documentación MariaDB: Information Schema](https://mariadb.com/kb/en/information-schema/)
