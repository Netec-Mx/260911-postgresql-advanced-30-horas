# 1 Implementación de estrategias de respaldo y recuperación

## Metadatos

| Elemento | Valor |
|---|---|
| Duración | 118 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica se implementa una estrategia combinada de respaldo lógico y físico para la base de datos migrada `sales_pg` en `pg-primary`. Se creará y validará un respaldo lógico en formato personalizado, se configurará `pgBackRest` para respaldos físicos y archivado continuo de WAL, y se ejecutará una recuperación Point-in-Time Recovery (PITR) en un directorio aislado.

La recuperación PITR no modificará el clúster operativo `lab16-primary`; se iniciará temporalmente una instancia recuperada en el puerto `5433`. La configuración de archivado WAL obtenida será un prerrequisito para la práctica posterior de replicación física.

## Objetivos de aprendizaje

Al finalizar la práctica, podrá:

- [ ] Crear un respaldo lógico consistente de `sales_pg` con `pg_dump` en formato personalizado.
- [ ] Inspeccionar un respaldo lógico y restaurarlo en la base de validación `sales_pg_restore`.
- [ ] Configurar `pgBackRest 2.51` con el stanza `lab16` y el repositorio local `/var/lib/pgbackrest`.
- [ ] Crear y verificar un respaldo físico completo, incluyendo una verificación con `pg_verifybackup`.
- [ ] Recuperar `sales_pg` a un instante anterior a un borrado lógico controlado mediante PITR y WAL archivados.

## Prerrequisitos

### Conocimientos necesarios

- Administración básica de Ubuntu Server 22.04.4 LTS.
- Uso de `sudo`, `systemctl`, `psql`, `pg_dump`, `pg_restore` y editores de texto.
- Conceptos de bases de datos PostgreSQL, transacciones, MVCC, WAL y roles.
- Prácticas 1.1, 2.1 y 3.1 completadas.
- Base de datos `sales_pg` migrada y validada.

### Acceso y condiciones técnicas

- Acceso administrativo mediante `sudo` en `pg-primary`.
- PostgreSQL Server y Client Utilities exactamente en versión 16.2.
- `pgBackRest` versión 2.51 instalado.
- Al menos 25 GB libres en `pg-primary`.
- Snapshot de la máquina virtual `pg-primary` creado antes de iniciar la fase PITR.
- El clúster principal debe estar activo y usar:
  - Nombre lógico: `lab16-primary`
  - `PGDATA`: `/var/lib/postgresql/16/main`
  - Configuración: `/etc/postgresql/16/main`
  - Puerto: `5432`
  - Zona horaria: UTC

> **Advertencia:** ejecute todos los comandos en `pg-primary`. No ejecute procedimientos PITR sobre `pg-standby` ni sobre un entorno productivo sin una ventana de mantenimiento, revisión de pares y respaldos validados.

## Entorno de laboratorio

### Servidor utilizado

| Componente | Valor |
|---|---|
| Nodo | `pg-primary` |
| Dirección IP | `192.168.56.10` |
| Sistema operativo | Ubuntu Server 22.04.4 LTS x86_64 |
| PostgreSQL Server | 16.2 |
| PostgreSQL Client Utilities | 16.2 |
| pgBackRest | 2.51 |
| Base de datos de origen | `sales_pg` |
| Base de validación lógica | `sales_pg_restore` |
| Puerto del clúster principal | `5432` |
| Puerto de recuperación aislada | `5433` |
| Repositorio pgBackRest | `/var/lib/pgbackrest` |
| Directorio PITR temporal | `/var/lib/postgresql/16/recovery` |

### Variables de trabajo

Ejecute las siguientes variables en la sesión de shell administrativa. Permanecerán disponibles mientras no cierre la terminal.

```bash
export PGDATA_MAIN=/var/lib/postgresql/16/main
export PGCONF=/etc/postgresql/16/main
export BACKUP_DIR=/var/backups/postgresql/lab04
export PGBACKREST_REPO=/var/lib/pgbackrest
export RECOVERY_DIR=/var/lib/postgresql/16/recovery
export STANZA=lab16
export DBNAME=sales_pg
```

### Comprobación inicial

```bash
hostnamectl --static
df -h /var/lib/postgresql /var/backups
sudo -u postgres psql --version
sudo -u postgres pg_dump --version
pgbackrest version
sudo systemctl status postgresql@16-main --no-pager
sudo -u postgres psql -d sales_pg -c "SELECT version();"
sudo -u postgres psql -d sales_pg -c "SHOW TimeZone;"
```

La zona horaria de PostgreSQL debe ser `UTC`.

---

## Procedimiento paso a paso

### Paso 1. Preparar directorios protegidos y comprobar la base de origen

**Objetivo:** preparar ubicaciones seguras para respaldos y confirmar que la base de datos de origen está disponible antes de iniciar operaciones de respaldo.

**Instrucciones:**

1. Cree el directorio para respaldos lógicos y asígnelo al usuario del sistema `postgres`.

   ```bash
   sudo install -d -o postgres -g postgres -m 700 "$BACKUP_DIR"
   sudo install -d -o postgres -g postgres -m 750 "$PGBACKREST_REPO"
   sudo install -d -o postgres -g postgres -m 750 /var/log/pgbackrest
   ```

2. Compruebe el propietario y los permisos.

   ```bash
   ls -ld "$BACKUP_DIR" "$PGBACKREST_REPO" /var/log/pgbackrest
   ```

3. Valide la existencia de la base y obtenga una referencia básica de su tamaño.

   ```bash
   sudo -u postgres psql -d postgres -c "\l+ sales_pg"
   sudo -u postgres psql -d sales_pg -c \
     "SELECT pg_size_pretty(pg_database_size(current_database())) AS tamano_sales_pg;"
   ```

4. Registre algunas métricas funcionales que utilizará para comparar la restauración lógica. Revise primero los esquemas y tablas disponibles.

   ```bash
   sudo -u postgres psql -d sales_pg -c "\dn"
   sudo -u postgres psql -d sales_pg -c "\dt *.*"
   ```

5. Si la práctica anterior creó tablas de ventas, ejecute una consulta representativa. Ajuste el nombre de tabla si el modelo migrado utiliza otro esquema.

   ```bash
   sudo -u postgres psql -d sales_pg -c \
     "SELECT schemaname, relname, n_live_tup
      FROM pg_stat_user_tables
      ORDER BY schemaname, relname;"
   ```

**Resultado esperado:**

- Los directorios existen con permisos restrictivos.
- `sales_pg` aparece en el catálogo de bases de datos.
- PostgreSQL responde en el puerto `5432`.
- Se muestran tablas de usuario y estimaciones de filas.

**Verificación:**

```bash
sudo -u postgres psql -d sales_pg -c \
  "SELECT current_database(), current_user, now() AT TIME ZONE 'UTC' AS hora_utc;"
```

La consulta debe devolver `sales_pg`, el usuario conectado y una hora expresada en UTC.

---

### Paso 2. Crear e inspeccionar el respaldo lógico de `sales_pg`

**Objetivo:** generar un respaldo lógico consistente en formato personalizado y comprobar que su catálogo interno puede ser leído.

**Instrucciones:**

1. Defina una marca temporal UTC para identificar los archivos de la práctica.

   ```bash
   export LAB_TS=$(date -u +%Y%m%dT%H%M%SZ)
   echo "$LAB_TS"
   ```

2. Exporte los objetos globales del clúster. Este archivo puede contener roles y hashes de contraseñas; por ello, debe permanecer protegido.

   ```bash
   sudo -u postgres bash -c \
     "pg_dumpall --globals-only > '$BACKUP_DIR/globals_${LAB_TS}.sql'"
   ```

3. Aplique permisos restrictivos al archivo de globales.

   ```bash
   sudo chmod 600 "$BACKUP_DIR/globals_${LAB_TS}.sql"
   ```

4. Cree un respaldo lógico de `sales_pg` en formato personalizado. El formato `-Fc` permite inspección, restauración selectiva y restauración paralela.

   ```bash
   sudo -u postgres pg_dump \
     -d "$DBNAME" \
     -F c \
     -Z 6 \
     -v \
     -f "$BACKUP_DIR/${DBNAME}_${LAB_TS}.dump"
   ```

5. Compruebe el tamaño y los permisos de los archivos creados.

   ```bash
   ls -lh "$BACKUP_DIR"
   ```

6. Inspeccione las primeras entradas del catálogo interno del respaldo.

   ```bash
   sudo -u postgres pg_restore --list \
     "$BACKUP_DIR/${DBNAME}_${LAB_TS}.dump" | head -n 40
   ```

7. Localice entradas relacionadas con tablas, datos, índices y restricciones.

   ```bash
   sudo -u postgres pg_restore --list \
     "$BACKUP_DIR/${DBNAME}_${LAB_TS}.dump" \
     | grep -E "TABLE|TABLE DATA|INDEX|CONSTRAINT" \
     | head -n 30
   ```

**Resultado esperado:**

- Se crean los archivos `globals_<marca>.sql` y `sales_pg_<marca>.dump`.
- `pg_restore --list` muestra objetos como `SCHEMA`, `TABLE`, `TABLE DATA`, `SEQUENCE`, `INDEX` y `CONSTRAINT`.
- No deben aparecer errores como `input file appears to be a text format dump`.

**Verificación:**

```bash
sudo -u postgres pg_restore --list \
  "$BACKUP_DIR/${DBNAME}_${LAB_TS}.dump" >/dev/null \
  && echo "Respaldo lógico legible por pg_restore"
```

Debe mostrarse el mensaje `Respaldo lógico legible por pg_restore`.

---

### Paso 3. Restaurar el respaldo lógico en una base aislada

**Objetivo:** demostrar que el respaldo lógico permite reconstruir la base de datos sin afectar a `sales_pg`.

**Instrucciones:**

1. Elimine una base de validación previa si existe.

   ```bash
   sudo -u postgres dropdb --if-exists sales_pg_restore
   ```

2. Cree una base vacía para la restauración.

   ```bash
   sudo -u postgres createdb sales_pg_restore
   ```

3. Restaure el archivo personalizado. Se utiliza `-j 2` para permitir restauración paralela moderada; reduzca a `-j 1` si el laboratorio dispone de pocos recursos.

   ```bash
   sudo -u postgres pg_restore \
     -d sales_pg_restore \
     -j 2 \
     -v \
     "$BACKUP_DIR/${DBNAME}_${LAB_TS}.dump"
   ```

4. Compruebe la existencia de esquemas y tablas restauradas.

   ```bash
   sudo -u postgres psql -d sales_pg_restore -c "\dn"
   sudo -u postgres psql -d sales_pg_restore -c "\dt *.*"
   ```

5. Compare el número de tablas de usuario entre origen y destino.

   ```bash
   sudo -u postgres psql -d sales_pg -Atc \
     "SELECT count(*) FROM pg_tables
      WHERE schemaname NOT IN ('pg_catalog', 'information_schema');"

   sudo -u postgres psql -d sales_pg_restore -Atc \
     "SELECT count(*) FROM pg_tables
      WHERE schemaname NOT IN ('pg_catalog', 'information_schema');"
   ```

6. Compare métricas de tablas utilizando estadísticas del catálogo.

   ```bash
   sudo -u postgres psql -d sales_pg -c \
     "SELECT schemaname, relname, n_live_tup
      FROM pg_stat_user_tables
      ORDER BY schemaname, relname;"

   sudo -u postgres psql -d sales_pg_restore -c \
     "SELECT schemaname, relname, n_live_tup
      FROM pg_stat_user_tables
      ORDER BY schemaname, relname;"
   ```

**Resultado esperado:**

- `sales_pg_restore` se crea y recibe los objetos del respaldo.
- Las tablas y esquemas de usuario son equivalentes a los de `sales_pg`.
- La restauración finaliza sin errores de objetos faltantes, extensiones no disponibles o permisos insuficientes.

**Verificación:**

```bash
sudo -u postgres psql -d sales_pg_restore -c \
  "SELECT current_database(),
          pg_size_pretty(pg_database_size(current_database())) AS tamano_restaurado;"
```

El resultado debe indicar la base `sales_pg_restore` y un tamaño razonablemente próximo al de la base original.

---

### Paso 4. Configurar pgBackRest y el archivado continuo de WAL

**Objetivo:** configurar el repositorio físico, crear el stanza `lab16` y habilitar los parámetros requeridos para archivado WAL y futuras prácticas de replicación.

**Instrucciones:**

1. Confirme la versión instalada de pgBackRest.

   ```bash
   pgbackrest version
   ```

   Debe informar la versión `2.51`.

2. Cree el archivo de configuración `/etc/pgbackrest/pgbackrest.conf`.

   ```bash
   sudo install -d -o root -g postgres -m 750 /etc/pgbackrest

   sudo tee /etc/pgbackrest/pgbackrest.conf >/dev/null <<'EOF'
   [global]
   repo1-path=/var/lib/pgbackrest
   repo1-retention-full=2
   log-level-console=info
   log-level-file=detail
   log-path=/var/log/pgbackrest
   process-max=2
   start-fast=y

   [lab16]
   pg1-path=/var/lib/postgresql/16/main
   pg1-port=5432
   EOF

   sudo chown root:postgres /etc/pgbackrest/pgbackrest.conf
   sudo chmod 640 /etc/pgbackrest/pgbackrest.conf
   ```

3. Cree una configuración adicional de PostgreSQL para WAL y respaldo. El parámetro `archive_mode` requiere reinicio; `wal_level=replica` y `max_wal_senders=5` también quedan preparados para la práctica de replicación.

   ```bash
   sudo tee "$PGCONF/conf.d/20-lab04-backup.conf" >/dev/null <<'EOF'
   wal_level = replica
   max_wal_senders = 5
   archive_mode = on
   archive_command = 'pgbackrest --stanza=lab16 archive-push %p'
   archive_timeout = 60s
   EOF
   ```

4. Revise la configuración antes de reiniciar.

   ```bash
   sudo cat "$PGCONF/conf.d/20-lab04-backup.conf"
   ```

5. Reinicie PostgreSQL para aplicar `archive_mode`.

   ```bash
   sudo systemctl restart postgresql@16-main
   sudo systemctl status postgresql@16-main --no-pager
   ```

6. Compruebe los parámetros efectivos desde SQL.

   ```bash
   sudo -u postgres psql -d postgres -c \
     "SHOW wal_level;
      SHOW max_wal_senders;
      SHOW archive_mode;
      SHOW archive_command;
      SHOW archive_timeout;"
   ```

7. Cree el stanza de pgBackRest.

   ```bash
   sudo -u postgres pgbackrest --stanza="$STANZA" stanza-create
   ```

8. Valide el stanza.

   ```bash
   sudo -u postgres pgbackrest --stanza="$STANZA" check
   ```

**Resultado esperado:**

- PostgreSQL reinicia correctamente.
- `archive_mode` muestra `on`.
- `archive_command` muestra el uso de `pgbackrest --stanza=lab16 archive-push %p`.
- `pgbackrest check` finaliza con código de retorno cero.

**Verificación:**

Fuerce un cambio de segmento WAL y compruebe de nuevo el stanza:

```bash
sudo -u postgres psql -d postgres -c "SELECT pg_switch_wal();"
sleep 10
sudo -u postgres pgbackrest --stanza="$STANZA" check
```

El comando `check` debe finalizar correctamente y no informar errores de archivado.

---

### Paso 5. Crear y verificar un respaldo físico completo

**Objetivo:** crear un respaldo físico completo con pgBackRest, revisar su estado y validar una copia física con el manifiesto de `pg_basebackup`.

**Instrucciones:**

1. Cree una tabla de marcadores para la prueba PITR. Esta tabla debe existir antes del respaldo físico completo para que forme parte de la base de referencia.

   ```bash
   sudo -u postgres psql -d sales_pg <<'SQL'
   CREATE TABLE IF NOT EXISTS public.pitr_lab_marker (
       marker_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
       event_name text NOT NULL,
       created_at timestamptz NOT NULL DEFAULT clock_timestamp(),
       details text
   );
   SQL
   ```

2. Cree el respaldo físico completo con pgBackRest.

   ```bash
   sudo -u postgres pgbackrest \
     --stanza="$STANZA" \
     --type=full \
     backup
   ```

3. Consulte la información del respaldo.

   ```bash
   sudo -u postgres pgbackrest --stanza="$STANZA" info
   ```

4. Revise los archivos almacenados en el repositorio.

   ```bash
   sudo find "$PGBACKREST_REPO" -maxdepth 3 -type d | sort
   sudo du -sh "$PGBACKREST_REPO"
   ```

5. Cree además una copia física de verificación con `pg_basebackup`. Esta copia se utiliza exclusivamente para demostrar `pg_verifybackup`, ya que `pg_verifybackup` valida el archivo `backup_manifest` generado por `pg_basebackup`, no el formato interno de pgBackRest.

   ```bash
   sudo rm -rf "$PGBACKREST_REPO/verifybackup"
   sudo install -d -o postgres -g postgres -m 700 "$PGBACKREST_REPO/verifybackup"

   sudo -u postgres pg_basebackup \
     -D "$PGBACKREST_REPO/verifybackup/basebackup" \
     -F p \
     -X stream \
     --manifest-checksums=SHA256 \
     -P
   ```

6. Verifique la copia física basada en manifiesto.

   ```bash
   sudo -u postgres pg_verifybackup \
     "$PGBACKREST_REPO/verifybackup/basebackup"
   ```

**Resultado esperado:**

- `pgbackrest info` informa al menos un respaldo de tipo `full` con estado válido.
- El repositorio contiene estructura de respaldo y archivos WAL archivados.
- `pg_verifybackup` informa que el manifiesto y los archivos fueron verificados correctamente.

**Verificación:**

```bash
sudo -u postgres pgbackrest --stanza="$STANZA" info | grep -E "status|full|timestamp"
sudo -u postgres pg_verifybackup "$PGBACKREST_REPO/verifybackup/basebackup"
```

> **Nota técnica:** pgBackRest realiza validación mediante checksums y metadatos propios durante las operaciones de respaldo y restauración. `pg_verifybackup` es una utilidad específica para respaldos físicos que contienen el archivo `backup_manifest` de PostgreSQL, como los creados por `pg_basebackup`.

---

### Paso 6. Generar transacciones identificables y simular un error lógico

**Objetivo:** crear un punto temporal de recuperación, registrar una transacción que debe conservarse y simular posteriormente un borrado lógico controlado.

**Instrucciones:**

1. Inserte el marcador que debe existir después de la recuperación.

   ```bash
   sudo -u postgres psql -d sales_pg -c \
     "INSERT INTO public.pitr_lab_marker(event_name, details)
      VALUES ('ANTES_DEL_ERROR', 'Registro que debe existir tras PITR');"
   ```

2. Capture un instante UTC inmediatamente posterior a la transacción protegida. Guarde este valor; será el objetivo de recuperación.

   ```bash
   export TARGET_TIME=$(sudo -u postgres psql -d sales_pg -Atc \
     "SELECT to_char(
        clock_timestamp() AT TIME ZONE 'UTC',
        'YYYY-MM-DD HH24:MI:SS.US+00'
      );")

   echo "TARGET_TIME=$TARGET_TIME"
   ```

3. Espere unos segundos para asegurar una separación temporal clara entre el objetivo y el error lógico.

   ```bash
   sleep 3
   ```

4. Simule el error lógico eliminando el marcador protegido.

   ```bash
   sudo -u postgres psql -d sales_pg -c \
     "DELETE FROM public.pitr_lab_marker
      WHERE event_name = 'ANTES_DEL_ERROR';"
   ```

5. Inserte un segundo marcador que represente una operación posterior no deseada.

   ```bash
   sudo -u postgres psql -d sales_pg -c \
     "INSERT INTO public.pitr_lab_marker(event_name, details)
      VALUES ('DESPUES_DEL_ERROR', 'Registro que no debe existir tras PITR');"
   ```

6. Compruebe el estado actual, que representa el estado erróneo.

   ```bash
   sudo -u postgres psql -d sales_pg -c \
     "SELECT marker_id, event_name, created_at, details
      FROM public.pitr_lab_marker
      ORDER BY marker_id;"
   ```

7. Fuerce el archivado del WAL que contiene las transacciones de prueba.

   ```bash
   sudo -u postgres psql -d postgres -c "SELECT pg_switch_wal();"
   sleep 10
   sudo -u postgres pgbackrest --stanza="$STANZA" check
   ```

**Resultado esperado:**

- El valor `TARGET_TIME` se muestra en formato UTC.
- La consulta final sobre el clúster principal no muestra `ANTES_DEL_ERROR`.
- La consulta final sí muestra `DESPUES_DEL_ERROR`.
- El `check` de pgBackRest confirma que el archivado WAL puede continuar.

**Verificación:**

```bash
echo "Objetivo de recuperación: $TARGET_TIME"

sudo -u postgres psql -d sales_pg -c \
  "SELECT event_name, created_at
   FROM public.pitr_lab_marker
   ORDER BY created_at;"
```

Confirme que el estado actual del clúster principal es posterior al instante objetivo y contiene el error simulado.

---

### Paso 7. Ejecutar una recuperación PITR en una instancia aislada

**Objetivo:** restaurar el respaldo físico y reproducir WAL hasta `TARGET_TIME` sin modificar el clúster principal.

**Instrucciones:**

1. Confirme que el clúster principal continúa activo en el puerto `5432`.

   ```bash
   sudo -u postgres pg_isready -h 127.0.0.1 -p 5432 -d sales_pg
   ```

2. Elimine cualquier directorio de recuperación previo. Esta operación afecta solamente a la instancia aislada.

   ```bash
   sudo rm -rf "$RECOVERY_DIR"
   ```

3. Restaure el último respaldo completo de pgBackRest en el directorio aislado.

   ```bash
   sudo -u postgres pgbackrest \
     --stanza="$STANZA" \
     --pg1-path="$RECOVERY_DIR" \
     restore
   ```

4. Copie archivos de autenticación mínimos para la instancia aislada.

   ```bash
   sudo cp "$PGCONF/pg_hba.conf" "$RECOVERY_DIR/pg_hba.conf"
   sudo cp "$PGCONF/pg_ident.conf" "$RECOVERY_DIR/pg_ident.conf"

   sudo chown postgres:postgres \
     "$RECOVERY_DIR/pg_hba.conf" \
     "$RECOVERY_DIR/pg_ident.conf"
   ```

5. Cree un archivo de configuración independiente para la instancia recuperada. La instancia escuchará solamente en `127.0.0.1:5433`.

   ```bash
   sudo tee "$RECOVERY_DIR/postgresql.conf" >/dev/null <<EOF
   data_directory = '$RECOVERY_DIR'
   hba_file = '$RECOVERY_DIR/pg_hba.conf'
   ident_file = '$RECOVERY_DIR/pg_ident.conf'

   port = 5433
   listen_addresses = '127.0.0.1'
   unix_socket_directories = '/var/run/postgresql'
   external_pid_file = '/var/run/postgresql/16-recovery.pid'

   logging_collector = off
   archive_mode = off

   restore_command = 'pgbackrest --stanza=$STANZA archive-get %f "%p"'
   recovery_target_time = '$TARGET_TIME'
   recovery_target_inclusive = true
   recovery_target_action = 'pause'
   EOF

   sudo chown postgres:postgres "$RECOVERY_DIR/postgresql.conf"
   sudo chmod 600 "$RECOVERY_DIR/postgresql.conf"
   ```

6. Cree el archivo señal requerido para que PostgreSQL entre en modo recuperación.

   ```bash
   sudo -u postgres touch "$RECOVERY_DIR/recovery.signal"
   ```

7. Inicie la instancia aislada usando el directorio restaurado.

   ```bash
   sudo -u postgres pg_ctl \
     -D "$RECOVERY_DIR" \
     -l /var/log/postgresql/lab16-recovery.log \
     start
   ```

8. Espere a que la instancia quede disponible.

   ```bash
   for i in {1..30}; do
     if sudo -u postgres pg_isready -h 127.0.0.1 -p 5433 -d sales_pg; then
       break
     fi
     sleep 2
   done
   ```

9. Revise el final del log de recuperación.

   ```bash
   sudo tail -n 40 /var/log/postgresql/lab16-recovery.log
   ```

**Resultado esperado:**

- El clúster principal continúa disponible en `5432`.
- La instancia de recuperación inicia en `5433`.
- El log contiene mensajes de restauración WAL y una referencia al objetivo de recuperación.
- La instancia queda pausada al alcanzar `recovery_target_time`.

**Verificación:**

```bash
sudo -u postgres psql -h 127.0.0.1 -p 5433 -d sales_pg -c \
  "SELECT pg_is_in_recovery() AS en_recuperacion,
          now() AT TIME ZONE 'UTC' AS hora_instancia_recuperada;"
```

La columna `en_recuperacion` debe devolver `t`, porque `recovery_target_action = 'pause'` mantiene la instancia pausada al llegar al objetivo.

---

### Paso 8. Validar el estado recuperado y documentar el retorno seguro

**Objetivo:** demostrar que el PITR reconstruyó el estado previo al borrado lógico y definir cómo retornar de forma segura al entorno normal.

**Instrucciones:**

1. Consulte los marcadores de la instancia recuperada.

   ```bash
   sudo -u postgres psql -h 127.0.0.1 -p 5433 -d sales_pg -c \
     "SELECT marker_id, event_name, created_at, details
      FROM public.pitr_lab_marker
      ORDER BY marker_id;"
   ```

2. Compruebe específicamente que el registro anterior al error existe.

   ```bash
   sudo -u postgres psql -h 127.0.0.1 -p 5433 -d sales_pg -c \
     "SELECT count(*) AS antes_del_error
      FROM public.pitr_lab_marker
      WHERE event_name = 'ANTES_DEL_ERROR';"
   ```

3. Compruebe que no se recuperó el registro posterior al error.

   ```bash
   sudo -u postgres psql -h 127.0.0.1 -p 5433 -d sales_pg -c \
     "SELECT count(*) AS despues_del_error
      FROM public.pitr_lab_marker
      WHERE event_name = 'DESPUES_DEL_ERROR';"
   ```

4. Compare el estado con el clúster principal, que no se ha modificado y debe conservar el estado erróneo.

   ```bash
   echo "Estado del clúster principal:"
   sudo -u postgres psql -h 127.0.0.1 -p 5432 -d sales_pg -c \
     "SELECT event_name, created_at
      FROM public.pitr_lab_marker
      ORDER BY created_at;"

   echo "Estado de la recuperación PITR:"
   sudo -u postgres psql -h 127.0.0.1 -p 5433 -d sales_pg -c \
     "SELECT event_name, created_at
      FROM public.pitr_lab_marker
      ORDER BY created_at;"
   ```

5. Documente en sus evidencias de laboratorio:
   - Marca temporal usada en `TARGET_TIME`.
   - Identificador y hora del respaldo full mostrado por `pgbackrest info`.
   - Resultado de `pgbackrest check`.
   - Resultado de `pg_verifybackup`.
   - Evidencia de que `ANTES_DEL_ERROR` existe en el PITR.
   - Evidencia de que `DESPUES_DEL_ERROR` no existe en el PITR.
   - Confirmación de que el clúster original en `5432` nunca fue detenido ni sobrescrito.

6. Registre el procedimiento de retorno seguro que se aplicaría en una recuperación real:

   1. Mantener el clúster recuperado aislado y en modo de solo lectura mientras se valida.
   2. Obtener aprobación formal sobre el instante de recuperación.
   3. Decidir si se extraen solamente datos desde la instancia recuperada o si se sustituirá el clúster principal.
   4. Si se requiere sustituir el clúster principal, detener las aplicaciones, preservar una copia del `PGDATA` actual y realizar un cambio controlado de directorio o infraestructura.
   5. No sobrescribir `PGDATA_MAIN` directamente sin un plan de reversión, respaldo adicional y validación funcional.
   6. Reconfigurar respaldos, archivado WAL y, posteriormente, réplicas físicas si la recuperación se convierte en el nuevo primario.

**Resultado esperado:**

- En la instancia recuperada:
  - `ANTES_DEL_ERROR` tiene recuento `1`.
  - `DESPUES_DEL_ERROR` tiene recuento `0`.
- En el clúster principal:
  - `ANTES_DEL_ERROR` permanece eliminado.
  - `DESPUES_DEL_ERROR` existe.
- Se demuestra que PITR restaura el estado al instante seleccionado.

**Verificación:**

```bash
sudo -u postgres psql -h 127.0.0.1 -p 5433 -d sales_pg -c \
  "SELECT
     EXISTS (
       SELECT 1 FROM public.pitr_lab_marker
       WHERE event_name = 'ANTES_DEL_ERROR'
     ) AS marcador_protegido_presente,
     EXISTS (
       SELECT 1 FROM public.pitr_lab_marker
       WHERE event_name = 'DESPUES_DEL_ERROR'
     ) AS marcador_posterior_presente;"
```

El resultado esperado es:

| marcador_protegido_presente | marcador_posterior_presente |
|---|---|
| `t` | `f` |

---

## Validación y pruebas

Complete la siguiente lista antes de considerar finalizada la práctica:

| Prueba | Comando o evidencia | Resultado esperado |
|---|---|---|
| Versión de cliente PostgreSQL | `pg_dump --version` | PostgreSQL 16.2 |
| Versión de pgBackRest | `pgbackrest version` | pgBackRest 2.51 |
| Respaldo lógico legible | `pg_restore --list archivo.dump` | Sin errores |
| Restauración lógica | Consulta en `sales_pg_restore` | Tablas y datos disponibles |
| Archivado WAL | `pgbackrest --stanza=lab16 check` | Finaliza correctamente |
| Respaldo físico pgBackRest | `pgbackrest --stanza=lab16 info` | Respaldo `full` válido |
| Manifiesto físico PostgreSQL | `pg_verifybackup directorio` | Verificación correcta |
| Clúster principal | `pg_isready -p 5432` | Acepta conexiones |
| Instancia PITR | `pg_is_in_recovery()` en `5433` | Devuelve `t` |
| Marcador recuperado | Consulta `ANTES_DEL_ERROR` en `5433` | Existe |
| Cambio posterior excluido | Consulta `DESPUES_DEL_ERROR` en `5433` | No existe |

## Solución de problemas

### Problema 1: `pgbackrest check` informa que no puede archivar WAL o muestra `archive command failed`

**Síntomas:**

- `pgbackrest --stanza=lab16 check` termina con error.
- En `/var/log/postgresql/postgresql-16-main.log` aparecen mensajes similares a `archive command failed`.
- PostgreSQL incrementa el valor de `failed_count` en `pg_stat_archiver`.

**Causa:**

El usuario `postgres` no puede escribir en el repositorio o en el directorio de logs de pgBackRest, la configuración del stanza es incorrecta o el servicio no se reinició después de activar `archive_mode`.

**Corrección:**

```bash
sudo chown -R postgres:postgres /var/lib/pgbackrest /var/log/pgbackrest
sudo chmod 750 /var/lib/pgbackrest /var/log/pgbackrest

sudo -u postgres pgbackrest --stanza=lab16 check

sudo -u postgres psql -d postgres -c \
  "SELECT archived_count, failed_count, last_archived_wal, last_failed_wal
   FROM pg_stat_archiver;"

sudo systemctl restart postgresql@16-main
sudo -u postgres psql -d postgres -c "SELECT pg_switch_wal();"
```

Compruebe también que `/etc/pgbackrest/pgbackrest.conf` contiene `pg1-path=/var/lib/postgresql/16/main` y que puede ser leído por el grupo `postgres`.

### Problema 2: la instancia PITR no inicia o no alcanza el instante de recuperación

**Síntomas:**

- `pg_isready -p 5433` no responde.
- El log `/var/log/postgresql/lab16-recovery.log` muestra errores de `restore_command`.
- La instancia inicia, pero no contiene el marcador `ANTES_DEL_ERROR`.

**Causa:**

El directorio de recuperación no procede de un respaldo físico válido, `recovery.signal` no existe, el valor de `TARGET_TIME` es incorrecto o el WAL necesario no fue archivado antes de iniciar la recuperación.

**Corrección:**

1. Detenga y elimine la recuperación incompleta:

   ```bash
   sudo -u postgres pg_ctl -D "$RECOVERY_DIR" stop -m fast || true
   sudo rm -rf "$RECOVERY_DIR"
   ```

2. Compruebe que pgBackRest reconoce el respaldo y los WAL:

   ```bash
   sudo -u postgres pgbackrest --stanza=lab16 info
   sudo -u postgres pgbackrest --stanza=lab16 check
   ```

3. Asegúrese de que el clúster principal haya archivado WAL después de las transacciones:

   ```bash
   sudo -u postgres psql -d postgres -c "SELECT pg_switch_wal();"
   sleep 10
   sudo -u postgres pgbackrest --stanza=lab16 check
   ```

4. Repita el Paso 7 usando un `TARGET_TIME` situado después de insertar `ANTES_DEL_ERROR` y antes del `DELETE`.

## Limpieza

> **Importante:** no elimine el repositorio de pgBackRest ni desactive el archivado WAL si la práctica 5.1 utilizará este nodo. La configuración creada es un prerrequisito para replicación física.

1. Detenga la instancia PITR aislada.

   ```bash
   sudo -u postgres pg_ctl -D "$RECOVERY_DIR" stop -m fast
   ```

2. Confirme que el clúster principal sigue disponible.

   ```bash
   sudo -u postgres pg_isready -h 127.0.0.1 -p 5432 -d sales_pg
   ```

3. Elimine el directorio temporal de recuperación aislada.

   ```bash
   sudo rm -rf "$RECOVERY_DIR"
   ```

4. Elimine la base de validación lógica si no se utilizará en prácticas posteriores.

   ```bash
   sudo -u postgres dropdb --if-exists sales_pg_restore
   ```

5. Opcionalmente, elimine la copia creada solo para `pg_verifybackup` si necesita recuperar espacio. No elimine los respaldos de pgBackRest requeridos por la práctica siguiente.

   ```bash
   sudo rm -rf "$PGBACKREST_REPO/verifybackup"
   ```

6. Conserve:
   - `/etc/pgbackrest/pgbackrest.conf`
   - `20-lab04-backup.conf`
   - El stanza `lab16`
   - El repositorio `/var/lib/pgbackrest`
   - El respaldo full creado
   - Los WAL archivados

## Resumen

En esta práctica se creó un respaldo lógico de `sales_pg` con `pg_dump -Fc`, se inspeccionó con `pg_restore --list` y se restauró de forma aislada en `sales_pg_restore`. También se configuró pgBackRest con el stanza `lab16`, se habilitó archivado WAL continuo y se creó un respaldo físico completo.

Finalmente, se utilizó un respaldo físico y WAL archivados para recuperar la base a un instante anterior a un borrado lógico. La recuperación se realizó en `/var/lib/postgresql/16/recovery` y en el puerto `5433`, manteniendo intacto el clúster operativo en el puerto `5432`. La infraestructura de WAL y pgBackRest queda preparada para la construcción de la réplica física en la práctica 5.1.
