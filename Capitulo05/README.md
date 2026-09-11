# Prácticas 5.1 Replicación asíncrona y síncrona

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 118 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Crear |

## Descripción general

En esta práctica se construirá una réplica física en streaming desde `pg-primary` hacia `pg-standby` usando PostgreSQL 16.2. Se validará inicialmente la replicación asíncrona, se observarán las posiciones WAL y el retraso de la réplica, y posteriormente se configurará replicación síncrona para comprobar su impacto sobre la confirmación de transacciones.

La práctica utiliza una ranura de replicación física para evitar la eliminación prematura de WAL requerido por la réplica. Al finalizar, se analizará el comportamiento ante una interrupción controlada de `pg-standby` y se distinguirán los usos de la replicación física y lógica.

## Objetivos de aprendizaje

Al completar esta práctica, podrá:

- [ ] Configurar el primario PostgreSQL para ofrecer replicación física mediante streaming WAL.
- [ ] Crear un rol de replicación, una regla de acceso restringida y una ranura física de replicación.
- [ ] Inicializar y conectar un servidor standby con `pg_basebackup` y `standby.signal`.
- [ ] Validar la replicación asíncrona mediante LSN, vistas de catálogo y cambios de datos.
- [ ] Configurar replicación síncrona y evaluar su efecto cuando la réplica está disponible o desconectada.

## Prerrequisitos

### Conocimientos previos

El estudiante debe comprender los siguientes conceptos:

- Funcionamiento básico de PostgreSQL, `psql`, roles y archivos de configuración.
- Propósito del WAL, los LSN y el archivado WAL.
- Diferencias entre respaldo físico, replicación física y replicación lógica.
- Conceptos de replicación asíncrona y síncrona.
- Uso básico de `systemctl`, permisos Linux y edición de archivos con privilegios `sudo`.

### Acceso y estado previo requerido

Antes de comenzar, confirme que dispone de:

- Nodo `pg-primary` con Ubuntu Server 22.04.4 LTS y PostgreSQL Server/Client Utilities 16.2.
- Nodo `pg-standby` con Ubuntu Server 22.04.4 LTS y PostgreSQL Server/Client Utilities 16.2.
- Base de datos `sales_pg` disponible en `pg-primary`.
- PGDATA del primario: `/var/lib/postgresql/16/main`.
- PGDATA de la réplica: `/var/lib/postgresql/16/main`.
- Directorio de configuración: `/etc/postgresql/16/main`.
- Dirección primaria: `192.168.56.10`.
- Dirección de réplica: `192.168.56.11`.
- Conectividad TCP/IP en el puerto `5432`.
- Acceso `sudo` en ambos nodos.
- Acceso administrativo PostgreSQL mediante el usuario del sistema `postgres`.
- Al menos 20 GB libres en `pg-standby`.
- Archivado WAL activo y respaldo físico disponible en `pg-primary`.
- Snapshot o punto de restauración de las máquinas virtuales antes de modificar la topología.

> **Advertencia:** esta práctica reinicia PostgreSQL en el nodo primario durante la configuración inicial. En un entorno productivo, este cambio debe planificarse dentro de una ventana de mantenimiento.

## Entorno de laboratorio

### Topología

| Elemento | pg-primary | pg-standby |
|---|---:|---:|
| Nombre lógico | `lab16-primary` | `lab16-standby` |
| Dirección IP | `192.168.56.10` | `192.168.56.11` |
| Puerto PostgreSQL | `5432` | `5432` |
| Versión PostgreSQL | 16.2 | 16.2 |
| PGDATA | `/var/lib/postgresql/16/main` | `/var/lib/postgresql/16/main` |
| Rol principal | Primario de escritura | Réplica física de solo lectura |
| Base de datos validada | `sales_pg` | Copia física de `sales_pg` |

### Resolución local de nombres

Ejecute el siguiente bloque en **ambos nodos**. Si existe un DNS local, confirme que proporciona la misma resolución y no duplique entradas innecesariamente.

```bash
sudo tee -a /etc/hosts >/dev/null <<'EOF'
192.168.56.10 pg-primary
192.168.56.11 pg-standby
EOF

getent hosts pg-primary pg-standby
```

Salida esperada:

```text
192.168.56.10  pg-primary
192.168.56.11  pg-standby
```

### Comprobaciones iniciales

Ejecute estas comprobaciones en ambos nodos.

```bash
hostnamectl --static
psql --version
sudo systemctl status postgresql --no-pager
df -h /var/lib/postgresql
```

La versión debe indicar PostgreSQL 16.2:

```text
psql (PostgreSQL) 16.2
```

Compruebe la conectividad desde `pg-standby` al primario:

```bash
ping -c 2 pg-primary
nc -vz pg-primary 5432
```

La comprobación TCP debe indicar que el puerto `5432` está accesible. Si `nc` no está instalado, puede instalarse con:

```bash
sudo apt update
sudo apt install -y netcat-openbsd
```

---

## Procedimiento paso a paso

## Paso 1. Verificar el estado inicial y la configuración del primario

**Objetivo:** confirmar que `pg-primary` está operativo, que el archivado WAL se encuentra activo y que el clúster puede configurarse para replicación física.

### Instrucciones

1. Inicie sesión en `pg-primary`.

2. Compruebe el estado del servicio PostgreSQL:

   ```bash
   sudo systemctl status postgresql --no-pager
   sudo pg_ctlcluster 16 main status
   ```

3. Compruebe la versión del servidor y el nombre lógico de la instancia:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT
       version(),
       current_setting('cluster_name', true) AS cluster_name,
       current_setting('data_directory') AS pgdata;"
   ```

4. Compruebe el estado de parámetros relevantes para WAL, archivado y replicación:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT
       name,
       setting,
       unit,
       context
   FROM pg_settings
   WHERE name IN (
       'wal_level',
       'archive_mode',
       'archive_command',
       'max_wal_senders',
       'max_replication_slots',
       'wal_keep_size',
       'max_slot_wal_keep_size',
       'listen_addresses',
       'port',
       'hot_standby',
       'synchronous_standby_names'
   )
   ORDER BY name;"
   ```

5. Verifique que la base `sales_pg` existe:

   ```bash
   sudo -u postgres psql -d postgres -c "\l sales_pg"
   ```

6. Compruebe que no exista aún una conexión de réplica activa:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT application_name, client_addr, state, sync_state
   FROM pg_stat_replication;"
   ```

### Salida esperada

- PostgreSQL está activo en `pg-primary`.
- La versión es PostgreSQL 16.2.
- `archive_mode` muestra `on`.
- `wal_level` puede mostrar ya `replica`; si no es así, se cambiará en el siguiente paso.
- Antes de crear la réplica, `pg_stat_replication` normalmente no devuelve filas.
- La base `sales_pg` aparece en el listado de bases.

### Verificación

Ejecute:

```bash
sudo -u postgres psql -d postgres -c "
SELECT
    pg_is_in_recovery() AS en_recuperacion,
    current_setting('archive_mode') AS archive_mode,
    current_setting('wal_level') AS wal_level;"
```

El resultado debe indicar:

```text
 en_recuperacion | archive_mode | wal_level
-----------------+--------------+-----------
 f               | on           | replica
```

El valor de `wal_level` podría no ser todavía `replica` antes del siguiente paso.

---

## Paso 2. Configurar parámetros de replicación y acceso en pg-primary

**Objetivo:** preparar el primario para enviar WAL a la réplica, conservar WAL suficiente y permitir exclusivamente la conexión del rol de replicación desde `pg-standby`.

### Instrucciones

1. En `pg-primary`, configure los parámetros que requieren reinicio. Los valores seleccionados permiten una réplica y dejan margen para futuras conexiones administrativas o réplicas adicionales.

   ```bash
   sudo -u postgres psql -d postgres -c "ALTER SYSTEM SET wal_level = 'replica';"
   sudo -u postgres psql -d postgres -c "ALTER SYSTEM SET max_wal_senders = '5';"
   sudo -u postgres psql -d postgres -c "ALTER SYSTEM SET max_replication_slots = '5';"
   sudo -u postgres psql -d postgres -c "ALTER SYSTEM SET wal_keep_size = '512MB';"
   sudo -u postgres psql -d postgres -c "ALTER SYSTEM SET max_slot_wal_keep_size = '10GB';"
   sudo -u postgres psql -d postgres -c "ALTER SYSTEM SET cluster_name = 'lab16-primary';"
   ```

2. Configure PostgreSQL para escuchar en la interfaz local y en la red privada del laboratorio.

   ```bash
   sudo -u postgres psql -d postgres -c \
   "ALTER SYSTEM SET listen_addresses = 'localhost,192.168.56.10';"
   ```

3. Revise el archivo generado por `ALTER SYSTEM`:

   ```bash
   sudo cat /var/lib/postgresql/16/main/postgresql.auto.conf
   ```

4. Reinicie PostgreSQL para aplicar los parámetros cuyo contexto es `postmaster`:

   ```bash
   sudo systemctl restart postgresql
   sudo systemctl status postgresql --no-pager
   ```

5. Cree el rol dedicado para la transmisión de WAL. Utilice una contraseña robusta y documentada únicamente para el laboratorio.

   ```bash
   sudo -u postgres psql -d postgres
   ```

   En la consola `psql`, ejecute:

   ```sql
   CREATE ROLE replication_user
       WITH LOGIN
       REPLICATION
       PASSWORD 'Cambiar_Esta_Clave_Replicacion_2026';
   ```

   Salga:

   ```sql
   \q
   ```

6. Localice el archivo `pg_hba.conf` efectivo:

   ```bash
   sudo -u postgres psql -d postgres -tAc "SHOW hba_file;"
   ```

7. Edite el archivo indicado, normalmente `/etc/postgresql/16/main/pg_hba.conf`, y agregue la siguiente regla **antes de reglas genéricas más permisivas**:

   ```conf
   # Replica física autorizada exclusivamente desde pg-standby
   host    replication     replication_user     192.168.56.11/32     scram-sha-256
   ```

   Puede usar el siguiente comando si conoce la ruta estándar:

   ```bash
   echo "host    replication    replication_user    192.168.56.11/32    scram-sha-256" | \
   sudo tee -a /etc/postgresql/16/main/pg_hba.conf
   ```

8. Recargue la configuración de autenticación:

   ```bash
   sudo -u postgres psql -d postgres -c "SELECT pg_reload_conf();"
   ```

9. Verifique que PostgreSQL no tenga errores de sintaxis o carga en `pg_hba.conf`:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT line_number, type, database, user_name, address, auth_method, error
   FROM pg_hba_file_rules
   WHERE error IS NOT NULL;"
   ```

### Salida esperada

- PostgreSQL se reinicia correctamente.
- El rol `replication_user` posee los atributos `LOGIN` y `REPLICATION`.
- La regla HBA restringe el acceso al host `192.168.56.11/32`.
- La consulta sobre `pg_hba_file_rules` no muestra errores.

### Verificación

Compruebe los valores efectivos:

```bash
sudo -u postgres psql -d postgres -c "
SELECT name, setting, unit
FROM pg_settings
WHERE name IN (
    'wal_level',
    'max_wal_senders',
    'max_replication_slots',
    'wal_keep_size',
    'max_slot_wal_keep_size',
    'listen_addresses',
    'cluster_name'
)
ORDER BY name;"

sudo -u postgres psql -d postgres -c "\du replication_user"
```

Confirme especialmente:

- `wal_level = replica`
- `max_wal_senders >= 1`
- `max_replication_slots >= 1`
- `listen_addresses` incluye `192.168.56.10`
- El rol tiene el atributo `Replication`.

> **Nota de operación:** una ranura de replicación evita eliminar WAL que la réplica todavía necesita. Sin embargo, una réplica detenida durante demasiado tiempo puede causar acumulación de WAL. El límite `max_slot_wal_keep_size = 10GB` protege el almacenamiento del primario, pero una réplica que exceda ese límite podría requerir una nueva copia base.

---

## Paso 3. Crear y supervisar la ranura de replicación física

**Objetivo:** crear una ranura física que retenga los WAL requeridos por `pg-standby`.

### Instrucciones

1. En `pg-primary`, compruebe si existe una ranura con el nombre previsto:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT slot_name, slot_type, active, restart_lsn, wal_status
   FROM pg_replication_slots;"
   ```

2. Cree la ranura física `standby_slot`:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT * FROM pg_create_physical_replication_slot('standby_slot');"
   ```

3. Consulte el estado de la ranura:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT
       slot_name,
       slot_type,
       active,
       restart_lsn,
       confirmed_flush_lsn,
       wal_status,
       safe_wal_size
   FROM pg_replication_slots
   WHERE slot_name = 'standby_slot';"
   ```

4. Consulte el tamaño aproximado de WAL retenido por la ranura. Antes de que la réplica se conecte, puede ser bajo o nulo.

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT
       slot_name,
       active,
       pg_size_pretty(
           pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
       ) AS wal_retenido_aproximado
   FROM pg_replication_slots
   WHERE slot_name = 'standby_slot';"
   ```

### Salida esperada

La ranura debe aparecer con características similares a:

```text
  slot_name   | slot_type | active | restart_lsn | wal_status
--------------+-----------+--------+-------------+------------
 standby_slot | physical  | f      |             | reserved
```

El valor `active` es inicialmente `false` porque `pg-standby` aún no se ha conectado.

### Verificación

Ejecute:

```bash
sudo -u postgres psql -d postgres -c "
SELECT slot_name, slot_type, active
FROM pg_replication_slots
WHERE slot_name = 'standby_slot';"
```

Debe existir exactamente una fila para `standby_slot` de tipo `physical`.

---

## Paso 4. Preparar pg-standby e inicializar la copia base

**Objetivo:** eliminar el clúster local vacío de `pg-standby`, obtener una copia base coherente desde `pg-primary` y crear la señalización de standby.

### Instrucciones

1. Inicie sesión en `pg-standby`.

2. Verifique que la versión cliente y servidor sea PostgreSQL 16.2:

   ```bash
   psql --version
   sudo -u postgres postgres --version
   ```

3. Detenga el clúster local. Esto es obligatorio antes de eliminar el contenido inicializado por el paquete PostgreSQL.

   ```bash
   sudo systemctl stop postgresql
   sudo pg_ctlcluster 16 main status
   ```

   Se espera que el clúster esté detenido.

4. Confirme cuidadosamente el directorio de datos configurado para el clúster:

   ```bash
   sudo -u postgres psql -d postgres -c "SHOW data_directory;" 2>/dev/null || true
   sudo grep -E "^[[:space:]]*data_directory" /etc/postgresql/16/main/postgresql.conf
   ```

   Para este laboratorio debe ser:

   ```text
   /var/lib/postgresql/16/main
   ```

5. Elimine el contenido del directorio de datos local. **No ejecute este comando en `pg-primary`.**

   ```bash
   sudo rm -rf /var/lib/postgresql/16/main/*
   sudo rm -rf /var/lib/postgresql/16/main/.[!.]*
   ```

6. Restablezca propietario y permisos adecuados:

   ```bash
   sudo chown -R postgres:postgres /var/lib/postgresql/16/main
   sudo chmod 700 /var/lib/postgresql/16/main
   ```

7. Cree un archivo `.pgpass` para el usuario del sistema `postgres`. Esto permite que `pg_basebackup` se autentique sin mostrar la contraseña en la línea de comandos.

   ```bash
   sudo -u postgres bash -c "cat > ~/.pgpass <<'EOF'
pg-primary:5432:*:replication_user:Cambiar_Esta_Clave_Replicacion_2026
EOF
chmod 600 ~/.pgpass"
   ```

8. Ejecute `pg_basebackup` como usuario `postgres`. La opción `-R` crea `standby.signal` y registra `primary_conninfo`; la opción `-S` indica que la réplica utilizará la ranura física existente.

   ```bash
   sudo -u postgres pg_basebackup \
     -h pg-primary \
     -p 5432 \
     -U replication_user \
     -D /var/lib/postgresql/16/main \
     -Fp \
     -Xs \
     -P \
     -R \
     -S standby_slot
   ```

9. Compruebe que se hayan creado los archivos de señalización y configuración automática:

   ```bash
   sudo ls -l /var/lib/postgresql/16/main/standby.signal
   sudo -u postgres grep -E "primary_conninfo|primary_slot_name" \
     /var/lib/postgresql/16/main/postgresql.auto.conf
   ```

10. Configure un nombre lógico para identificar la réplica en `pg_stat_replication`. Añada esta directiva al archivo de configuración de `pg-standby`:

   ```bash
   echo "cluster_name = 'lab16-standby'" | \
   sudo tee -a /etc/postgresql/16/main/postgresql.conf
   ```

11. Asegure que la réplica permita consultas de solo lectura durante la recuperación:

   ```bash
   echo "hot_standby = on" | \
   sudo tee -a /etc/postgresql/16/main/postgresql.conf
   ```

12. Inicie PostgreSQL en `pg-standby`:

   ```bash
   sudo systemctl start postgresql
   sudo systemctl status postgresql --no-pager
   ```

### Salida esperada

`pg_basebackup` debe mostrar el progreso de la copia, por ejemplo:

```text
waiting for checkpoint
...
30958/30958 kB (100%), 1/1 tablespace
```

El archivo `standby.signal` debe existir. El archivo `postgresql.auto.conf` debe contener parámetros similares a:

```conf
primary_conninfo = 'user=replication_user passfile=''/var/lib/postgresql/.pgpass'' channel_binding=prefer host=pg-primary port=5432 sslmode=prefer ...'
primary_slot_name = 'standby_slot'
```

### Verificación

En `pg-standby`, ejecute:

```bash
sudo -u postgres psql -d postgres -c "
SELECT
    pg_is_in_recovery() AS en_recuperacion,
    current_setting('cluster_name', true) AS cluster_name,
    current_setting('hot_standby') AS hot_standby;"
```

Resultado esperado:

```text
 en_recuperacion |  cluster_name  | hot_standby
-----------------+----------------+-------------
 t               | lab16-standby  | on
```

El valor `t` confirma que el servidor está funcionando como réplica física.

---

## Paso 5. Validar la conexión de streaming y el estado WAL

**Objetivo:** comprobar que `pg-primary` transmite WAL y que `pg-standby` lo recibe y reproduce.

### Instrucciones

1. En `pg-primary`, consulte la vista `pg_stat_replication`:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT
       application_name,
       client_addr,
       state,
       sync_state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       write_lag,
       flush_lag,
       replay_lag
   FROM pg_stat_replication;"
   ```

2. En `pg-primary`, valide que la ranura se encuentra activa:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT
       slot_name,
       slot_type,
       active,
       restart_lsn,
       wal_status,
       safe_wal_size
   FROM pg_replication_slots
   WHERE slot_name = 'standby_slot';"
   ```

3. En `pg-standby`, consulte el proceso receptor WAL:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT
       status,
       sender_host,
       sender_port,
       slot_name,
       written_lsn,
       flushed_lsn,
       latest_end_lsn,
       latest_end_time
   FROM pg_stat_wal_receiver;"
   ```

4. En `pg-standby`, consulte el avance de recepción y reproducción:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT
       pg_last_wal_receive_lsn() AS ultimo_wal_recibido,
       pg_last_wal_replay_lsn() AS ultimo_wal_reproducido,
       now() - pg_last_xact_replay_timestamp() AS retraso_aproximado;"
   ```

5. En ambos nodos, obtenga el LSN actual para comparación:

   En `pg-primary`:

   ```bash
   sudo -u postgres psql -d postgres -c "SELECT pg_current_wal_lsn() AS lsn_primario;"
   ```

   En `pg-standby`:

   ```bash
   sudo -u postgres psql -d postgres -c "SELECT pg_last_wal_replay_lsn() AS lsn_reproducido;"
   ```

### Salida esperada

En el primario, `pg_stat_replication` debe contener una fila similar a:

```text
 application_name |  client_addr   |   state   | sync_state
------------------+----------------+-----------+------------
 lab16-standby    | 192.168.56.11  | streaming | async
```

En la réplica, `pg_stat_wal_receiver` debe mostrar:

```text
 status   | sender_host | sender_port |  slot_name
----------+-------------+-------------+--------------
 streaming| pg-primary  |        5432 | standby_slot
```

### Verificación

La topología es correcta si se cumplen simultáneamente estas condiciones:

- `pg_is_in_recovery()` devuelve `true` en `pg-standby`.
- `pg_stat_replication.state` es `streaming` en `pg-primary`.
- `pg_stat_wal_receiver.status` es `streaming` en `pg-standby`.
- La ranura `standby_slot` muestra `active = true`.
- `sync_state` es `async` antes de habilitar replicación síncrona.

---

## Paso 6. Generar transacciones y validar replicación asíncrona

**Objetivo:** generar cambios en `sales_pg`, confirmar que llegan a la réplica y observar los LSN y retrasos en modo asíncrono.

### Instrucciones

1. En `pg-primary`, cree una tabla específica para la práctica:

   ```bash
   sudo -u postgres psql -d sales_pg -c "
   CREATE TABLE IF NOT EXISTS replication_lab_events (
       event_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
       event_time timestamptz NOT NULL DEFAULT clock_timestamp(),
       source_node text NOT NULL,
       description text NOT NULL
   );"
   ```

2. Inserte un primer conjunto de transacciones. En modo asíncrono, el cliente no espera la confirmación remota de la réplica.

   ```bash
   sudo -u postgres psql -d sales_pg -c "
   INSERT INTO replication_lab_events (source_node, description)
   SELECT
       'pg-primary',
       'Evento asincrono ' || generate_series
   FROM generate_series(1, 100);"
   ```

3. Registre el LSN generado en el primario:

   ```bash
   sudo -u postgres psql -d sales_pg -c "
   SELECT
       pg_current_wal_lsn() AS lsn_actual_primario,
       count(*) AS eventos_primario
   FROM replication_lab_events;"
   ```

4. En `pg-standby`, consulte los datos replicados. Las consultas de lectura están permitidas porque `hot_standby` está activo.

   ```bash
   sudo -u postgres psql -d sales_pg -c "
   SELECT
       count(*) AS eventos_replicados,
       min(event_time) AS primer_evento,
       max(event_time) AS ultimo_evento
   FROM replication_lab_events;"
   ```

5. En `pg-primary`, consulte la diferencia entre el LSN actual y el LSN reproducido por la réplica:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT
       application_name,
       state,
       sync_state,
       pg_current_wal_lsn() AS lsn_primario_actual,
       replay_lsn AS lsn_reproducido_standby,
       pg_size_pretty(
           pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)
       ) AS diferencia_wal
   FROM pg_stat_replication;"
   ```

6. Compruebe que la réplica rechaza operaciones de escritura:

   ```bash
   sudo -u postgres psql -d sales_pg -c "
   INSERT INTO replication_lab_events (source_node, description)
   VALUES ('pg-standby', 'Esta escritura debe fallar');"
   ```

### Salida esperada

- En el primario se insertan 100 filas.
- En la réplica, el conteo alcanza 100 filas después de un tiempo corto.
- La diferencia WAL disminuye hasta ser pequeña o nula cuando la réplica alcanza al primario.
- El intento de escritura en la réplica genera un error similar a:

```text
ERROR: cannot execute INSERT in a read-only transaction
```

### Verificación

En `pg-primary`:

```bash
sudo -u postgres psql -d sales_pg -c "
SELECT count(*) AS eventos_en_primario
FROM replication_lab_events;"
```

En `pg-standby`:

```bash
sudo -u postgres psql -d sales_pg -c "
SELECT count(*) AS eventos_en_standby
FROM replication_lab_events;"
```

Ambos conteos deben ser iguales antes de continuar. Además, en el primario ejecute:

```bash
sudo -u postgres psql -d postgres -c "
SELECT application_name, state, sync_state, replay_lag
FROM pg_stat_replication;"
```

El valor `sync_state` debe continuar como `async`.

> **Interpretación:** una columna de retraso nula no implica necesariamente un problema. Cuando no hay actividad reciente, PostgreSQL puede no disponer de información temporal suficiente para calcular el retraso. Compare LSN y genere una transacción adicional si necesita una referencia actual.

---

## Paso 7. Configurar y validar replicación síncrona

**Objetivo:** exigir que una réplica denominada `lab16-standby` confirme la recepción WAL antes de que el primario confirme las transacciones al cliente.

### Instrucciones

1. En `pg-primary`, confirme que la réplica está conectada y usa el nombre de aplicación esperado:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT application_name, state, sync_state
   FROM pg_stat_replication;"
   ```

   Debe existir una fila con `application_name = lab16-standby` y `state = streaming`.

2. Configure la política de sincronización. La expresión `FIRST 1` exige confirmación de la primera réplica elegible de la lista.

   ```bash
   sudo -u postgres psql -d postgres -c "
   ALTER SYSTEM SET synchronous_standby_names = 'FIRST 1 (lab16-standby)';"
   ```

3. Recargue la configuración:

   ```bash
   sudo -u postgres psql -d postgres -c "SELECT pg_reload_conf();"
   ```

4. Compruebe el valor efectivo:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SHOW synchronous_standby_names;"
   ```

5. Consulte de nuevo el estado de replicación:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT
       application_name,
       state,
       sync_state,
       write_lsn,
       flush_lsn,
       replay_lsn
   FROM pg_stat_replication;"
   ```

6. Inserte una transacción mientras la réplica está disponible:

   ```bash
   time sudo -u postgres psql -d sales_pg -c "
   INSERT INTO replication_lab_events (source_node, description)
   VALUES ('pg-primary', 'Evento confirmado con replica sincrona disponible');"
   ```

7. En `pg-standby`, confirme la llegada del evento:

   ```bash
   sudo -u postgres psql -d sales_pg -c "
   SELECT event_id, event_time, source_node, description
   FROM replication_lab_events
   ORDER BY event_id DESC
   LIMIT 3;"
   ```

### Salida esperada

En `pg-primary`, la réplica debe cambiar a un estado similar a:

```text
 application_name |   state   | sync_state
------------------+-----------+------------
 lab16-standby    | streaming | sync
```

La inserción se confirma correctamente, aunque puede presentar mayor latencia que en modo asíncrono, porque el primario espera la confirmación remota requerida.

### Verificación

Ejecute en `pg-primary`:

```bash
sudo -u postgres psql -d postgres -c "
SELECT
    application_name,
    state,
    sync_state,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;"
```

La configuración síncrona se considera correcta si:

- `state = streaming`
- `sync_state = sync`
- La transacción insertada se observa desde `pg-standby`.

> **Nota técnica:** con la configuración predeterminada `synchronous_commit = on`, PostgreSQL espera una confirmación remota de nivel `flush`: la réplica debe haber vaciado el WAL recibido a almacenamiento persistente. Otros niveles, como `remote_write` o `remote_apply`, representan compromisos distintos entre latencia y protección.

---

## Paso 8. Simular una interrupción controlada y restaurar la operación

**Objetivo:** observar cómo la replicación síncrona afecta la confirmación de transacciones cuando la réplica requerida deja de estar disponible.

### Instrucciones

1. En `pg-standby`, detenga PostgreSQL de forma controlada:

   ```bash
   sudo systemctl stop postgresql
   sudo systemctl status postgresql --no-pager
   ```

2. En `pg-primary`, confirme que ya no existe una réplica conectada:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT application_name, state, sync_state
   FROM pg_stat_replication;"
   ```

   Es posible que la consulta no devuelva filas.

3. Abra una primera terminal en `pg-primary` y ejecute una inserción con un límite de tiempo de 10 segundos:

   ```bash
   sudo -u postgres psql -d sales_pg
   ```

   Dentro de `psql`, ejecute:

   ```sql
   SET statement_timeout = '10s';

   INSERT INTO replication_lab_events (source_node, description)
   VALUES ('pg-primary', 'Evento que no debe confirmar sin standby sincronico');
   ```

4. Mientras la inserción está esperando, abra una segunda terminal en `pg-primary` y observe la espera de sincronización:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT
       pid,
       usename,
       datname,
       state,
       wait_event_type,
       wait_event,
       query
   FROM pg_stat_activity
   WHERE datname = 'sales_pg'
     AND state <> 'idle';"
   ```

5. Espere a que venza el tiempo configurado en la primera terminal. El comando debe cancelarse por `statement_timeout`.

6. Compruebe que la fila no fue confirmada:

   ```bash
   sudo -u postgres psql -d sales_pg -c "
   SELECT count(*) AS eventos_bloqueados_confirmados
   FROM replication_lab_events
   WHERE description = 'Evento que no debe confirmar sin standby sincronico';"
   ```

7. En `pg-standby`, reinicie PostgreSQL:

   ```bash
   sudo systemctl start postgresql
   sudo systemctl status postgresql --no-pager
   ```

8. Espere unos segundos y valide la reconexión desde `pg-primary`:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT
       application_name,
       client_addr,
       state,
       sync_state,
       replay_lsn
   FROM pg_stat_replication;"
   ```

9. Cuando `sync_state` vuelva a ser `sync`, genere una nueva inserción válida:

   ```bash
   sudo -u postgres psql -d sales_pg -c "
   INSERT INTO replication_lab_events (source_node, description)
   VALUES ('pg-primary', 'Evento confirmado despues de restaurar standby sincronico');"
   ```

10. En `pg-standby`, confirme la llegada del evento:

   ```bash
   sudo -u postgres psql -d sales_pg -c "
   SELECT event_id, event_time, description
   FROM replication_lab_events
   ORDER BY event_id DESC
   LIMIT 5;"
   ```

### Salida esperada

Mientras la réplica está detenida y se exige una réplica síncrona, la inserción espera la confirmación remota. En la primera terminal se espera un error similar a:

```text
ERROR: canceling statement due to statement timeout
```

En la segunda terminal puede aparecer:

```text
 wait_event_type | wait_event
-----------------+------------
 IPC             | SyncRep
```

Después de reiniciar `pg-standby`:

- La conexión vuelve a `streaming`.
- `sync_state` vuelve a `sync`.
- Las nuevas transacciones se confirman y llegan a la réplica.

### Verificación

Ejecute en `pg-primary`:

```bash
sudo -u postgres psql -d postgres -c "
SELECT
    application_name,
    state,
    sync_state,
    pg_current_wal_lsn() AS lsn_primario,
    replay_lsn AS lsn_replicado
FROM pg_stat_replication;"
```

Ejecute en `pg-standby`:

```bash
sudo -u postgres psql -d postgres -c "
SELECT
    pg_is_in_recovery() AS en_recuperacion,
    pg_last_wal_receive_lsn() AS wal_recibido,
    pg_last_wal_replay_lsn() AS wal_reproducido;"
```

La topología se ha restaurado correctamente cuando el primario muestra `streaming` y `sync`, y la réplica sigue en recuperación con LSN de recepción y reproducción avanzando.

---

## Validación y pruebas

Complete las siguientes validaciones finales.

| Prueba | Comando o criterio | Resultado esperado |
|---|---|---|
| Primario operativo | `SELECT pg_is_in_recovery();` en `pg-primary` | `false` |
| Réplica en recuperación | `SELECT pg_is_in_recovery();` en `pg-standby` | `true` |
| Streaming activo | `pg_stat_replication.state` | `streaming` |
| Receptor WAL activo | `pg_stat_wal_receiver.status` | `streaming` |
| Ranura activa | `pg_replication_slots.active` | `true` |
| Replicación asíncrona validada | `sync_state` antes de configurar sincronía | `async` |
| Replicación síncrona validada | `sync_state` después de configurar sincronía | `sync` |
| Consistencia de tabla de práctica | Conteo de filas en ambos nodos | Mismo conteo tras alcanzar el LSN |
| Escrituras bloqueadas en standby | `INSERT` en `pg-standby` | Error de solo lectura |
| Efecto de caída síncrona | Inserción con standby detenido | Espera `SyncRep` o timeout |
| Recuperación tras reinicio | Reiniciar `pg-standby` | Retorno a `streaming` y `sync` |

Ejecute esta consulta consolidada en `pg-primary`:

```bash
sudo -u postgres psql -d postgres -c "
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;"
```

Ejecute esta consulta consolidada en `pg-standby`:

```bash
sudo -u postgres psql -d postgres -c "
SELECT
    pg_is_in_recovery() AS en_recuperacion,
    pg_last_wal_receive_lsn() AS ultimo_wal_recibido,
    pg_last_wal_replay_lsn() AS ultimo_wal_reproducido,
    now() - pg_last_xact_replay_timestamp() AS retraso_aproximado;"
```

### Diferenciación entre replicación física y lógica

Documente las siguientes conclusiones en el informe de práctica:

| Aspecto | Replicación física | Replicación lógica |
|---|---|---|
| Unidad transmitida | Registros WAL y cambios físicos del clúster | Operaciones lógicas de tablas publicadas |
| Alcance | Clúster completo | Tablas y objetos seleccionados |
| Destino | Réplica PostgreSQL físicamente compatible | PostgreSQL u otro consumidor compatible |
| Uso principal | Alta disponibilidad, recuperación rápida, réplicas de lectura | Integración, migración selectiva, consolidación de datos |
| Cambios de estructura | La réplica reproduce el clúster físico | Requieren planificación y no todos se propagan automáticamente |
| Roles y configuración | Forman parte del clúster físico replicado | No se replican automáticamente mediante publicación |
| Capacidad de promoción | Sí, mediante procedimiento controlado | No reemplaza directamente un primario físico completo |

Para esta práctica, la replicación física es apropiada porque `pg-standby` mantiene una copia completa de `sales_pg` y del clúster, preparada para uso de lectura o para una futura promoción controlada. La replicación lógica sería apropiada si fuera necesario enviar solo determinadas tablas de ventas a una plataforma de análisis, a otra versión de PostgreSQL o a un sistema de integración.

## Solución de problemas

### Problema 1: `pg_basebackup` falla con error de autenticación o no puede conectarse al primario

**Síntomas:**

```text
pg_basebackup: error: connection to server at "pg-primary" failed
FATAL: no pg_hba.conf entry for replication connection
```

O bien:

```text
FATAL: password authentication failed for user "replication_user"
```

**Causa probable:** la regla de `pg_hba.conf` no existe, está debajo de una regla incompatible, utiliza una dirección IP incorrecta, PostgreSQL no fue recargado, la contraseña en `.pgpass` no coincide o el archivo tiene permisos inseguros.

**Corrección:**

1. En `pg-primary`, confirme la regla:

   ```bash
   sudo grep -n "replication_user" /etc/postgresql/16/main/pg_hba.conf
   ```

2. Compruebe errores de sintaxis:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT line_number, error
   FROM pg_hba_file_rules
   WHERE error IS NOT NULL;"
   ```

3. Recargue la configuración:

   ```bash
   sudo -u postgres psql -d postgres -c "SELECT pg_reload_conf();"
   ```

4. En `pg-standby`, valide permisos de `.pgpass`:

   ```bash
   sudo -u postgres ls -l /var/lib/postgresql/.pgpass
   ```

   Debe ser propiedad de `postgres` y tener permisos `0600`.

5. Compruebe conectividad de red:

   ```bash
   nc -vz pg-primary 5432
   ```

6. Si cambió la contraseña, actualice `.pgpass` y vuelva a ejecutar `pg_basebackup` sobre un directorio de datos vacío.

### Problema 2: la réplica no aparece como `streaming` o la ranura acumula WAL

**Síntomas:**

- `pg_stat_replication` no devuelve filas o muestra un estado distinto de `streaming`.
- `pg_stat_wal_receiver` está vacío en la réplica.
- `pg_replication_slots.active` es `false`.
- El tamaño de WAL retenido por `standby_slot` aumenta continuamente.

**Causa probable:** el servicio de `pg-standby` está detenido, falta `standby.signal`, la configuración `primary_conninfo` o `primary_slot_name` es incorrecta, existe un problema de red, o la réplica no puede recuperar WAL requerido.

**Corrección:**

1. En `pg-standby`, compruebe el servicio y los registros:

   ```bash
   sudo systemctl status postgresql --no-pager
   sudo journalctl -u postgresql -n 80 --no-pager
   sudo tail -n 80 /var/log/postgresql/postgresql-16-main.log
   ```

2. Confirme los archivos de señalización y configuración:

   ```bash
   sudo ls -l /var/lib/postgresql/16/main/standby.signal
   sudo -u postgres grep -E "primary_conninfo|primary_slot_name" \
     /var/lib/postgresql/16/main/postgresql.auto.conf
   ```

3. En `pg-primary`, revise el estado de la ranura:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT
       slot_name,
       active,
       restart_lsn,
       wal_status,
       safe_wal_size,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
         AS wal_retenido
   FROM pg_replication_slots
   WHERE slot_name = 'standby_slot';"
   ```

4. Reinicie la réplica si su configuración es correcta:

   ```bash
   sudo systemctl restart postgresql
   ```

5. Si el registro indica que falta WAL que ya no está disponible, no elimine arbitrariamente la ranura. Detenga la réplica, reconstruya su directorio de datos mediante `pg_basebackup` y vuelva a asociarla a la ranura según el procedimiento del Paso 4.

## Limpieza

La limpieza recomendada conserva la topología física, pero devuelve el comportamiento a replicación asíncrona para evitar que una futura detención de `pg-standby` bloquee confirmaciones en `pg-primary`.

1. En `pg-primary`, desactive el requisito de standby síncrono:

   ```bash
   sudo -u postgres psql -d postgres -c "
   ALTER SYSTEM RESET synchronous_standby_names;"
   sudo -u postgres psql -d postgres -c "SELECT pg_reload_conf();"
   ```

2. Verifique que la política ha sido eliminada:

   ```bash
   sudo -u postgres psql -d postgres -c "SHOW synchronous_standby_names;"
   ```

3. Confirme que la réplica conectada vuelve a mostrar estado asíncrono:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT application_name, state, sync_state
   FROM pg_stat_replication;"
   ```

4. Si desea eliminar únicamente los datos de prueba, ejecute en `pg-primary`:

   ```bash
   sudo -u postgres psql -d sales_pg -c "
   DROP TABLE IF EXISTS replication_lab_events;"
   ```

   Espere a que el cambio llegue a `pg-standby`.

5. Mantenga la ranura `standby_slot` mientras la réplica continúe usándola. No elimine la ranura activa.

6. Si el instructor solicita desmontar completamente la réplica, realice estas acciones solo después de detener `pg-standby` y de confirmar que ya no se necesita la topología:

   ```bash
   # En pg-standby
   sudo systemctl stop postgresql

   # En pg-primary
   sudo -u postgres psql -d postgres -c "
   SELECT pg_drop_replication_slot('standby_slot');"
   ```

   Posteriormente, el directorio de datos de la réplica puede eliminarse únicamente si se va a reconstruir desde cero.

## Resumen

En esta práctica se configuró una topología de replicación física PostgreSQL basada en streaming WAL. El nodo `pg-primary` fue preparado con `wal_level = replica`, procesos `walsender`, una ranura física y una regla HBA restringida. El nodo `pg-standby` fue inicializado desde una copia base obtenida con `pg_basebackup`, configurado mediante `standby.signal` y conectado al primario a través de `primary_conninfo`.

Se validó que:

- La replicación asíncrona confirma transacciones sin esperar a la réplica.
- Los LSN permiten comparar la posición WAL del primario con la recepción y reproducción del standby.
- `pg_stat_replication`, `pg_stat_wal_receiver` y `pg_replication_slots` proporcionan visibilidad operativa sobre la topología.
- La replicación síncrona aumenta la protección frente a pérdida de transacciones confirmadas, pero puede bloquear confirmaciones si no existe una réplica síncrona disponible.
- La ranura de replicación protege la continuidad de una réplica desconectada temporalmente, aunque exige supervisar el espacio ocupado por WAL retenido.
- La replicación física es apropiada para alta disponibilidad de un clúster completo; la replicación lógica es más adecuada para distribución selectiva de tablas, integración o migración de datos.

### Recursos recomendados

- Documentación PostgreSQL 16: Streaming Replication.
- Documentación PostgreSQL 16: High Availability, Load Balancing, and Replication.
- Documentación PostgreSQL 16: `pg_basebackup`.
- Documentación PostgreSQL 16: Vistas `pg_stat_replication`, `pg_stat_wal_receiver` y `pg_replication_slots`.
- Documentación PostgreSQL 16: Replication Slots y `synchronous_standby_names`.
