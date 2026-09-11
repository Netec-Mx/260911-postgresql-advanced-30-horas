# Prácticas 2.1 Administración y configuración de una instancia PostgreSQL

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 88 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica administrará la base de datos `sales_pg` y configurará controles básicos de operación y seguridad en la instancia PostgreSQL 16.2 del nodo `pg-primary`. Creará roles con privilegios mínimos, configurará el esquema `sales`, aplicará reglas de autenticación mediante `pg_hba.conf` y modificará parámetros de red, memoria, conexiones y registro de eventos.

La configuración resultante será reutilizada en prácticas posteriores de migración con pgloader, respaldo con pgBackRest y replicación física con `pg-standby`.

## Objetivos de aprendizaje

Al finalizar la práctica, podrá:

- [ ] Administrar roles, bases de datos, esquemas y privilegios aplicando el principio de mínimo privilegio.
- [ ] Configurar parámetros básicos de red, memoria, conexiones y registro en PostgreSQL 16.2.
- [ ] Definir reglas explícitas de autenticación local y remota en `pg_hba.conf`.
- [ ] Diferenciar cambios aplicables mediante `pg_reload_conf()`, `ALTER SYSTEM` y reinicio con `systemctl`.
- [ ] Validar conexiones, permisos, parámetros activos y disponibilidad del servidor mediante herramientas administrativas.

## Prerrequisitos

### Conocimientos requeridos

- Uso básico de Linux y comandos con `sudo`.
- Manejo básico de `psql`.
- Fundamentos de SQL DDL: `CREATE`, `ALTER`, `GRANT`, `REVOKE` y `DROP`.
- Conceptos de roles, privilegios, conexiones cliente-servidor y autenticación.

### Acceso requerido

- Nodo `pg-primary` operativo.
- Acceso `sudo` en `pg-primary`.
- Acceso al rol PostgreSQL `postgres` o a un rol administrativo equivalente.
- Práctica 1.1 completada.
- Rol `lab_admin` y base de datos `sales_pg` disponibles.

## Entorno de laboratorio

### Nodos y red

| Nodo | Dirección IPv4 | Función |
|---|---:|---|
| `pg-primary` | `192.168.56.10` | Nodo PostgreSQL primario |
| `pg-standby` | `192.168.56.11` | Nodo que se utilizará posteriormente como réplica |

### Componentes relevantes

| Componente | Valor esperado |
|---|---|
| Sistema operativo | Ubuntu Server 22.04.4 LTS x86_64 |
| PostgreSQL Server | 16.2 |
| PostgreSQL Client Utilities | 16.2 |
| Puerto PostgreSQL | TCP 5432 |
| Clúster | `16/main` |
| Nombre lógico del clúster | `lab16-primary` |
| Directorio de datos | `/var/lib/postgresql/16/main` |
| Directorio de configuración | `/etc/postgresql/16/main` |
| Directorio de logs | `/var/log/postgresql/` |
| Zona horaria | UTC |

### Preparación inicial

1. Conéctese al nodo `pg-primary`.

2. Verifique el nombre del host, la dirección IP y la zona horaria:

   ```bash
   hostnamectl
   hostname -I
   timedatectl
   ```

3. Confirme que ambos nodos se resuelven localmente. Si no existe un DNS de laboratorio, agregue las entradas necesarias en `/etc/hosts`:

   ```bash
   sudoedit /etc/hosts
   ```

   Asegúrese de incluir:

   ```text
   192.168.56.10  pg-primary
   192.168.56.11  pg-standby
   ```

4. Verifique las versiones instaladas:

   ```bash
   psql --version
   pg_config --version
   sudo -u postgres psql -d postgres -c "SELECT version();"
   ```

**Resultado esperado**

La salida debe indicar PostgreSQL 16.2 tanto para las utilidades cliente como para el servidor.

**Verificación**

Ejecute:

```bash
sudo -u postgres psql -d postgres -c "
SELECT current_setting('server_version') AS version_servidor,
       current_setting('TimeZone') AS zona_horaria;"
```

El resultado debe mostrar:

```text
 version_servidor | zona_horaria
------------------+--------------
 16.2             | UTC
```

---

## Procedimiento paso a paso

### Paso 1. Verificar el estado inicial del clúster y de la base `sales_pg`

**Objetivo:** Confirmar que la instancia PostgreSQL está disponible y que la base de datos creada en la práctica anterior existe antes de modificar su configuración.

**Instrucciones**

1. Consulte el estado del servicio del clúster:

   ```bash
   sudo systemctl status postgresql@16-main --no-pager
   ```

2. Verifique la disponibilidad en el puerto local:

   ```bash
   pg_isready -h 127.0.0.1 -p 5432 -d postgres
   ```

3. Conéctese como el usuario administrativo del sistema `postgres`:

   ```bash
   sudo -u postgres psql -d postgres
   ```

4. Dentro de `psql`, liste las bases de datos existentes:

   ```sql
   \l
   ```

5. Consulte las propiedades de `sales_pg` desde el catálogo `pg_database`:

   ```sql
   SELECT datname,
          pg_get_userbyid(datdba) AS propietario,
          pg_encoding_to_char(encoding) AS codificacion,
          datcollate AS lc_collate,
          datctype AS lc_ctype,
          datconnlimit AS limite_conexiones,
          datallowconn AS permite_conexiones
   FROM pg_database
   WHERE datname = 'sales_pg';
   ```

6. Revise las sesiones activas contra la base de datos:

   ```sql
   SELECT pid,
          usename,
          application_name,
          client_addr,
          state
   FROM pg_stat_activity
   WHERE datname = 'sales_pg'
   ORDER BY pid;
   ```

7. Salga de `psql`:

   ```sql
   \q
   ```

**Resultado esperado**

- El servicio `postgresql@16-main` debe aparecer como `active (running)`.
- `pg_isready` debe informar que el servidor acepta conexiones.
- La base `sales_pg` debe existir.
- La codificación debe ser `UTF8`.
- No deben existir conexiones inesperadas o desconocidas contra `sales_pg`.

**Verificación**

```bash
sudo -u postgres psql -d postgres -c "
SELECT datname,
       pg_encoding_to_char(encoding) AS codificacion
FROM pg_database
WHERE datname = 'sales_pg';"
```

La salida debe mostrar la base `sales_pg` con codificación `UTF8`.

---

### Paso 2. Crear y configurar roles de aplicación con privilegios mínimos

**Objetivo:** Crear los roles requeridos para la aplicación y la migración, evitando atributos administrativos innecesarios.

**Instrucciones**

1. Abra una sesión administrativa:

   ```bash
   sudo -u postgres psql -d postgres
   ```

2. Verifique si los roles ya existen:

   ```sql
   SELECT rolname,
          rolcanlogin,
          rolsuper,
          rolcreatedb,
          rolcreaterole,
          rolreplication
   FROM pg_roles
   WHERE rolname IN ('lab_admin', 'app_owner', 'app_rw', 'app_ro', 'migration_user')
   ORDER BY rolname;
   ```

3. Cree los roles requeridos. Ejecute cada comando solamente si el rol correspondiente no existe:

   ```sql
   CREATE ROLE app_owner
       NOLOGIN
       NOSUPERUSER
       NOCREATEDB
       NOCREATEROLE
       NOREPLICATION
       NOINHERIT;

   CREATE ROLE app_rw
       LOGIN
       NOSUPERUSER
       NOCREATEDB
       NOCREATEROLE
       NOREPLICATION
       NOINHERIT;

   CREATE ROLE app_ro
       LOGIN
       NOSUPERUSER
       NOCREATEDB
       NOCREATEROLE
       NOREPLICATION
       NOINHERIT;

   CREATE ROLE migration_user
       LOGIN
       NOSUPERUSER
       NOCREATEDB
       NOCREATEROLE
       NOREPLICATION
       NOINHERIT;
   ```

4. Asegure que las contraseñas futuras se almacenen mediante SCRAM. Este cambio utiliza `ALTER SYSTEM`, por lo que PostgreSQL escribirá la configuración en `postgresql.auto.conf`.

   ```sql
   ALTER SYSTEM SET password_encryption = 'scram-sha-256';
   SELECT pg_reload_conf();
   ```

5. Confirme que el parámetro activo corresponde al valor esperado:

   ```sql
   SHOW password_encryption;
   ```

6. Establezca contraseñas seguras. Use el metacomando `\password`, que evita dejar la contraseña en el historial de comandos o en texto visible.

   ```sql
   \password app_rw
   ```

   Repita para los demás roles con inicio de sesión:

   ```sql
   \password app_ro
   \password migration_user
   ```

   Use contraseñas distintas, robustas y documentadas únicamente en el mecanismo autorizado de custodia de credenciales del laboratorio.

7. Revise los atributos finales:

   ```sql
   SELECT rolname,
          rolcanlogin AS puede_iniciar_sesion,
          rolsuper AS superusuario,
          rolcreatedb AS puede_crear_bd,
          rolcreaterole AS puede_crear_roles,
          rolreplication AS puede_replicar
   FROM pg_roles
   WHERE rolname IN ('app_owner', 'app_rw', 'app_ro', 'migration_user')
   ORDER BY rolname;
   ```

8. Salga de `psql`:

   ```sql
   \q
   ```

**Resultado esperado**

- `app_owner` debe ser un rol sin inicio de sesión.
- `app_rw`, `app_ro` y `migration_user` deben poder iniciar sesión.
- Ninguno de estos roles debe ser superusuario ni debe poder crear bases de datos o roles.
- Las contraseñas deben usar SCRAM-SHA-256.

**Verificación**

Ejecute:

```bash
sudo -u postgres psql -d postgres -c "
SELECT rolname,
       rolcanlogin,
       rolsuper,
       rolcreatedb,
       rolcreaterole,
       rolreplication
FROM pg_roles
WHERE rolname IN ('app_owner', 'app_rw', 'app_ro', 'migration_user')
ORDER BY rolname;"
```

Los campos administrativos deben tener valor `f`.

---

### Paso 3. Configurar la base `sales_pg`, el esquema `sales` y los privilegios

**Objetivo:** Implementar una estructura de autorización donde `app_owner` sea propietario de los objetos, `app_rw` pueda leer y modificar datos, `app_ro` tenga acceso de solo lectura y `migration_user` tenga permisos limitados para la carga inicial de migración.

**Instrucciones**

1. Conéctese como `postgres` a la base administrativa:

   ```bash
   sudo -u postgres psql -d postgres
   ```

2. Establezca un límite controlado de conexiones y un esquema de búsqueda predeterminado para `sales_pg`:

   ```sql
   ALTER DATABASE sales_pg CONNECTION LIMIT 30;

   ALTER DATABASE sales_pg
       SET search_path TO sales, public;
   ```

3. Revise las propiedades actualizadas de la base:

   ```sql
   SELECT datname,
          pg_get_userbyid(datdba) AS propietario,
          datconnlimit AS limite_conexiones
   FROM pg_database
   WHERE datname = 'sales_pg';
   ```

4. Conéctese a `sales_pg`:

   ```sql
   \c sales_pg
   ```

5. Elimine privilegios implícitos sobre el esquema `public`. Esto evita que roles de aplicación creen objetos fuera del esquema controlado.

   ```sql
   REVOKE ALL ON SCHEMA public FROM PUBLIC;
   ```

6. Cree el esquema de negocio `sales` con `app_owner` como propietario:

   ```sql
   CREATE SCHEMA IF NOT EXISTS sales AUTHORIZATION app_owner;

   ALTER SCHEMA sales OWNER TO app_owner;
   ```

7. Elimine permisos implícitos sobre el esquema `sales`:

   ```sql
   REVOKE ALL ON SCHEMA sales FROM PUBLIC;
   ```

8. Retire privilegios generales concedidos por defecto sobre la base y asigne solamente los necesarios:

   ```sql
   REVOKE ALL ON DATABASE sales_pg FROM PUBLIC;

   GRANT CONNECT, TEMPORARY ON DATABASE sales_pg
       TO app_rw, app_ro, migration_user;
   ```

9. Asigne privilegios de uso del esquema:

   ```sql
   GRANT USAGE ON SCHEMA sales
       TO app_rw, app_ro, migration_user;
   ```

10. Otorgue a `migration_user` el permiso temporal de crear objetos en el esquema para la futura carga inicial con pgloader:

   ```sql
   GRANT CREATE ON SCHEMA sales TO migration_user;
   ```

11. Configure privilegios predeterminados para los objetos que cree `app_owner` en el esquema `sales`:

   ```sql
   ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA sales
       GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;

   ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA sales
       GRANT SELECT ON TABLES TO app_ro;

   ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA sales
       GRANT USAGE, SELECT ON SEQUENCES TO app_rw;
   ```

12. Cree una tabla de prueba como `app_owner`. El uso de `SET ROLE` permite crear el objeto con el propietario correcto:

   ```sql
   SET ROLE app_owner;

   CREATE TABLE sales.lab_access_test (
       id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
       customer_name text NOT NULL,
       amount numeric(12,2) NOT NULL CHECK (amount >= 0),
       created_at timestamptz NOT NULL DEFAULT now()
   );

   INSERT INTO sales.lab_access_test (customer_name, amount)
   VALUES ('Cliente de prueba', 150.00);

   RESET ROLE;
   ```

13. Revise la propiedad de la tabla y los privilegios otorgados:

   ```sql
   \dp sales.lab_access_test
   ```

14. Consulte los privilegios mediante el catálogo:

   ```sql
   SELECT grantee,
          privilege_type
   FROM information_schema.role_table_grants
   WHERE table_schema = 'sales'
     AND table_name = 'lab_access_test'
   ORDER BY grantee, privilege_type;
   ```

15. Salga de `psql`:

   ```sql
   \q
   ```

**Resultado esperado**

- `sales` debe pertenecer a `app_owner`.
- `app_rw` debe tener permisos `SELECT`, `INSERT`, `UPDATE` y `DELETE` sobre tablas futuras de `app_owner`.
- `app_ro` debe tener únicamente `SELECT`.
- `migration_user` debe tener `USAGE` y `CREATE` sobre `sales`, necesarios para una carga inicial controlada.
- El rol `PUBLIC` no debe conservar permisos generales sobre la base ni sobre el esquema `sales`.

**Verificación**

```bash
sudo -u postgres psql -d sales_pg -c "
SELECT n.nspname AS esquema,
       pg_get_userbyid(n.nspowner) AS propietario
FROM pg_namespace n
WHERE n.nspname IN ('public', 'sales')
ORDER BY n.nspname;"
```

El esquema `sales` debe indicar `app_owner` como propietario.

---

### Paso 4. Practicar la administración controlada de bases de datos

**Objetivo:** Aplicar los conceptos de creación, inspección, modificación y eliminación segura de una base de datos utilizando una base temporal de validación.

**Instrucciones**

1. Conéctese a la base administrativa:

   ```bash
   sudo -u postgres psql -d postgres
   ```

2. Cree una base temporal basada en `template0`. Esta plantilla permite una creación predecible y sin personalizaciones heredadas desde `template1`.

   ```sql
   CREATE DATABASE sales_pg_check
       WITH
       OWNER = app_owner
       ENCODING = 'UTF8'
       TEMPLATE = template0
       CONNECTION LIMIT = 2;
   ```

3. Inspeccione la base creada:

   ```sql
   \l sales_pg_check
   ```

4. Consulte el catálogo del sistema:

   ```sql
   SELECT datname,
          pg_get_userbyid(datdba) AS propietario,
          pg_encoding_to_char(encoding) AS codificacion,
          datconnlimit AS limite_conexiones,
          datallowconn AS permite_conexiones
   FROM pg_database
   WHERE datname = 'sales_pg_check';
   ```

5. Configure un parámetro de sesión predeterminado para la nueva base:

   ```sql
   ALTER DATABASE sales_pg_check
       SET search_path TO public;
   ```

6. Verifique las sesiones activas contra esta base:

   ```sql
   SELECT pid,
          usename,
          client_addr,
          state
   FROM pg_stat_activity
   WHERE datname = 'sales_pg_check';
   ```

7. Elimine la base temporal de forma controlada. El modificador `WITH (FORCE)` termina las conexiones existentes antes de eliminarla.

   ```sql
   DROP DATABASE sales_pg_check WITH (FORCE);
   ```

8. Salga de `psql`:

   ```sql
   \q
   ```

**Resultado esperado**

- `sales_pg_check` debe crearse con codificación `UTF8`.
- El propietario debe ser `app_owner`.
- La base debe eliminarse correctamente tras usar `DROP DATABASE ... WITH (FORCE)`.

**Verificación**

```bash
sudo -u postgres psql -d postgres -c "
SELECT datname
FROM pg_database
WHERE datname = 'sales_pg_check';"
```

La consulta no debe devolver filas.

> **Precaución:** No ejecute `DROP DATABASE` sobre `sales_pg`. En un entorno real, confirme siempre el nombre de la base, la existencia de respaldos válidos y las sesiones activas antes de una operación destructiva.

---

### Paso 5. Configurar parámetros básicos en `postgresql.conf`

**Objetivo:** Configurar escucha de red, puerto, conexiones, memoria y registro de eventos para el nodo `pg-primary`.

**Instrucciones**

1. Cree copias de seguridad de los archivos de configuración antes de modificarlos:

   ```bash
   sudo cp -a /etc/postgresql/16/main/postgresql.conf \
      /etc/postgresql/16/main/postgresql.conf.lab02.bak

   sudo cp -a /etc/postgresql/16/main/pg_hba.conf \
      /etc/postgresql/16/main/pg_hba.conf.lab02.bak

   sudo cp -a /etc/postgresql/16/main/pg_ident.conf \
      /etc/postgresql/16/main/pg_ident.conf.lab02.bak
   ```

2. Verifique el directorio de logs:

   ```bash
   sudo ls -ld /var/log/postgresql
   ```

3. Edite el archivo principal de configuración:

   ```bash
   sudoedit /etc/postgresql/16/main/postgresql.conf
   ```

4. Localice y establezca los siguientes parámetros. Si una directiva ya existe, modifique su valor; evite mantener varias definiciones activas del mismo parámetro.

   ```conf
   # Red y conexiones
   listen_addresses = 'localhost,192.168.56.10'
   port = 5432
   max_connections = 100

   # Memoria
   shared_buffers = 256MB

   # Registro de eventos
   logging_collector = on
   log_destination = 'stderr'
   log_directory = '/var/log/postgresql'
   log_filename = 'postgresql-16-main-%Y-%m-%d.log'
   log_line_prefix = '%m [%p] %q%u@%d/%a '
   log_connections = on
   log_disconnections = on
   log_checkpoints = on
   log_min_duration_statement = 1000
   ```

5. Guarde el archivo y salga del editor.

6. Consulte, sin modificar, el archivo `pg_ident.conf`:

   ```bash
   sudo cat /etc/postgresql/16/main/pg_ident.conf
   ```

   Para esta práctica no se utilizará autenticación `ident` ni mapeo de usuarios del sistema operativo a roles PostgreSQL. Mantenga el archivo sin cambios.

7. Comprenda el mecanismo aplicado a cada tipo de cambio:

   | Mecanismo | Uso en esta práctica |
   |---|---|
   | Edición de `postgresql.conf` | Configuración documentada y administrada como archivo del clúster |
   | `ALTER SYSTEM` | Se utilizó para `password_encryption`; escribe en `postgresql.auto.conf` |
   | `SELECT pg_reload_conf()` | Aplica parámetros recargables, como opciones de logging |
   | `systemctl restart` | Requerido para parámetros que necesitan reinicio, como `listen_addresses`, `shared_buffers` y `logging_collector` |

8. Antes de reiniciar, compruebe la sintaxis y los archivos configurados:

   ```bash
   sudo -u postgres psql -d postgres -c "SHOW config_file;"
   sudo -u postgres psql -d postgres -c "SHOW hba_file;"
   ```

**Resultado esperado**

- `postgresql.conf` debe contener la dirección privada de `pg-primary`.
- PostgreSQL debe conservar el puerto TCP 5432.
- La instancia debe aceptar hasta 100 conexiones.
- El registro de conexiones, desconexiones, checkpoints y consultas lentas debe estar habilitado.
- Los cambios de `listen_addresses`, `shared_buffers` y `logging_collector` requerirán reinicio.

**Verificación**

Compruebe qué configuración procede de `postgresql.auto.conf`:

```bash
sudo -u postgres psql -d postgres -c "
SELECT name,
       setting,
       source,
       sourcefile
FROM pg_settings
WHERE name = 'password_encryption';"
```

El origen debe reflejar el uso de `ALTER SYSTEM`.

---

### Paso 6. Configurar reglas de autenticación en `pg_hba.conf`

**Objetivo:** Definir una política explícita de autenticación para conexiones locales, aplicaciones de la red privada, migración y replicación futura.

**Instrucciones**

1. Edite el archivo de reglas de acceso:

   ```bash
   sudoedit /etc/postgresql/16/main/pg_hba.conf
   ```

2. Conserve o adapte los comentarios de cabecera y establezca las siguientes reglas. El orden es importante: PostgreSQL usa la primera regla coincidente.

   ```conf
   # TYPE  DATABASE   USER                                  ADDRESS                 METHOD

   # Administración local del sistema operativo.
   local   all        postgres                                                      peer

   # Roles de aplicación mediante socket local con contraseña SCRAM.
   local   sales_pg   lab_admin,app_owner,app_rw,app_ro,migration_user              scram-sha-256

   # Otros usuarios locales no definidos explícitamente.
   local   all        all                                                           peer

   # Aplicaciones locales que utilizan TCP/IP.
   host    sales_pg   lab_admin,app_rw,app_ro,migration_user 127.0.0.1/32          scram-sha-256

   # Aplicaciones de la red privada de laboratorio.
   host    sales_pg   lab_admin,app_rw,app_ro,migration_user 192.168.56.0/24       scram-sha-256

   # Regla reservada para la futura réplica física en pg-standby.
   # El rol replicator se creará en la práctica de replicación.
   host    replication replicator                              192.168.56.11/32     scram-sha-256

   # Rechazo explícito de cualquier otro acceso IPv4 e IPv6.
   host    all        all                                     0.0.0.0/0             reject
   host    all        all                                     ::0/0                 reject
   ```

3. Guarde el archivo.

4. Recargue las reglas de autenticación sin reiniciar el servicio:

   ```bash
   sudo -u postgres psql -d postgres -c "SELECT pg_reload_conf();"
   ```

5. Revise la interpretación de las reglas por PostgreSQL:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT line_number,
          type,
          database,
          user_name,
          address,
          auth_method,
          error
   FROM pg_hba_file_rules
   ORDER BY line_number;"
   ```

6. Confirme que no existan errores de sintaxis:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT line_number, error
   FROM pg_hba_file_rules
   WHERE error IS NOT NULL;"
   ```

**Resultado esperado**

- La vista `pg_hba_file_rules` no debe mostrar errores.
- `postgres` debe autenticarse localmente mediante `peer`.
- Los roles de aplicación deben autenticarse mediante `scram-sha-256`.
- Las conexiones desde la red `192.168.56.0/24` deben estar permitidas solo para los roles y la base definidos.
- El acceso de replicación futuro debe limitarse a `pg-standby` (`192.168.56.11`).
- Las conexiones no coincidentes deben rechazarse explícitamente.

**Verificación**

```bash
sudo -u postgres psql -d postgres -c "
SELECT line_number,
       type,
       database,
       user_name,
       address,
       auth_method
FROM pg_hba_file_rules
WHERE error IS NULL
ORDER BY line_number;"
```

Revise que el orden de las reglas sea igual o equivalente al definido en el procedimiento.

---

### Paso 7. Reiniciar el clúster y comprobar parámetros activos

**Objetivo:** Aplicar los parámetros que requieren reinicio y verificar su origen, contexto y valor activo mediante `pg_settings`.

**Instrucciones**

1. Reinicie el clúster de forma controlada:

   ```bash
   sudo systemctl restart postgresql@16-main
   ```

2. Revise el estado del servicio:

   ```bash
   sudo systemctl status postgresql@16-main --no-pager
   ```

3. Compruebe la disponibilidad mediante socket y TCP/IP:

   ```bash
   pg_isready -h 127.0.0.1 -p 5432 -d postgres
   pg_isready -h 192.168.56.10 -p 5432 -d postgres
   ```

4. Consulte los parámetros activos:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT name,
          setting,
          unit,
          context,
          source,
          sourcefile
   FROM pg_settings
   WHERE name IN (
       'listen_addresses',
       'port',
       'max_connections',
       'shared_buffers',
       'logging_collector',
       'log_line_prefix',
       'log_connections',
       'log_disconnections',
       'log_checkpoints',
       'log_min_duration_statement',
       'password_encryption'
   )
   ORDER BY name;"
   ```

5. Revise las direcciones de escucha reales:

   ```bash
   sudo ss -lntp | grep 5432
   ```

6. Consulte mensajes recientes del servicio:

   ```bash
   sudo journalctl -u postgresql@16-main -n 30 --no-pager
   ```

7. Revise los archivos de log generados:

   ```bash
   sudo ls -ltr /var/log/postgresql/ | tail
   ```

**Resultado esperado**

- El servicio debe reiniciarse sin errores.
- PostgreSQL debe escuchar en `127.0.0.1:5432` y `192.168.56.10:5432`.
- `shared_buffers` debe mostrar `256MB`.
- `max_connections` debe mostrar `100`.
- `logging_collector` debe mostrar `on`.
- Los parámetros deben mostrar como origen el archivo de configuración correspondiente o `configuration file`.

**Verificación**

```bash
sudo -u postgres psql -d postgres -c "
SELECT name, setting
FROM pg_settings
WHERE name IN ('listen_addresses', 'port', 'max_connections', 'shared_buffers', 'logging_collector')
ORDER BY name;"
```

---

### Paso 8. Validar conexiones y privilegios con usuarios de aplicación

**Objetivo:** Confirmar que las reglas de autenticación y los privilegios de los roles se aplican correctamente.

**Instrucciones**

1. Pruebe una conexión TCP/IP como `app_rw`. Se solicitará la contraseña definida anteriormente:

   ```bash
   psql -h 192.168.56.10 -p 5432 -U app_rw -d sales_pg
   ```

2. Dentro de la sesión de `app_rw`, compruebe el contexto de conexión:

   ```sql
   SELECT current_user,
          current_database(),
          current_setting('search_path') AS search_path;
   ```

3. Consulte los datos de prueba:

   ```sql
   SELECT * FROM sales.lab_access_test;
   ```

4. Inserte una fila como usuario de lectura/escritura:

   ```sql
   INSERT INTO sales.lab_access_test (customer_name, amount)
   VALUES ('Cliente app_rw', 250.00);
   ```

5. Confirme el resultado:

   ```sql
   SELECT id, customer_name, amount
   FROM sales.lab_access_test
   ORDER BY id;
   ```

6. Salga de la sesión:

   ```sql
   \q
   ```

7. Conéctese como `app_ro`:

   ```bash
   psql -h 192.168.56.10 -p 5432 -U app_ro -d sales_pg
   ```

8. Compruebe que puede consultar:

   ```sql
   SELECT id, customer_name, amount
   FROM sales.lab_access_test
   ORDER BY id;
   ```

9. Intente insertar una fila. Este comando debe fallar:

   ```sql
   INSERT INTO sales.lab_access_test (customer_name, amount)
   VALUES ('Operación no permitida', 1.00);
   ```

10. Salga de la sesión:

   ```sql
   \q
   ```

11. Pruebe un acceso no autorizado a una base distinta. El siguiente comando debe ser rechazado por las reglas HBA:

   ```bash
   psql -h 192.168.56.10 -p 5432 -U app_ro -d postgres
   ```

12. Como administrador, consulte las conexiones y revise el registro de eventos:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT datname,
          usename,
          client_addr,
          application_name,
          state
   FROM pg_stat_activity
   WHERE usename IN ('app_rw', 'app_ro', 'migration_user')
   ORDER BY usename;"
   ```

   ```bash
   sudo tail -n 30 /var/log/postgresql/postgresql-16-main-*.log
   ```

**Resultado esperado**

- `app_rw` debe conectarse y poder consultar e insertar datos.
- `app_ro` debe conectarse y consultar datos.
- El intento de inserción de `app_ro` debe devolver un error de permisos, por ejemplo:

   ```text
   ERROR: permission denied for table lab_access_test
   ```

- La conexión de `app_ro` hacia `postgres` debe ser rechazada por `pg_hba.conf`.
- Los logs deben contener eventos de conexión y desconexión.

**Verificación**

Ejecute como administrador:

```bash
sudo -u postgres psql -d sales_pg -c "
SELECT grantee,
       privilege_type
FROM information_schema.role_table_grants
WHERE table_schema = 'sales'
  AND table_name = 'lab_access_test'
ORDER BY grantee, privilege_type;"
```

Confirme que:

- `app_rw` tiene privilegios de lectura y escritura.
- `app_ro` tiene solo `SELECT`.
- `migration_user` no tiene permisos directos sobre esta tabla de prueba, salvo aquellos que se otorguen específicamente durante una migración.

---

## Validación y pruebas

Ejecute la siguiente lista de comprobación final.

| Validación | Comando o evidencia esperada |
|---|---|
| Servicio activo | `systemctl status postgresql@16-main` muestra `active (running)` |
| Disponibilidad local | `pg_isready -h 127.0.0.1 -p 5432` acepta conexiones |
| Disponibilidad en red privada | `pg_isready -h 192.168.56.10 -p 5432` acepta conexiones |
| Versión del servidor | `SELECT version();` indica PostgreSQL 16.2 |
| Base disponible | `sales_pg` existe y utiliza UTF8 |
| Esquema correcto | `sales` pertenece a `app_owner` |
| Autenticación | Reglas válidas en `pg_hba_file_rules` sin errores |
| Acceso RW | `app_rw` puede insertar en `sales.lab_access_test` |
| Acceso RO | `app_ro` puede hacer `SELECT`, pero no `INSERT` |
| Parámetros | `pg_settings` refleja los valores configurados |
| Logs | Existen archivos o eventos recientes en `/var/log/postgresql/` |
| Acceso remoto futuro | Existe una regla `replication` limitada a `192.168.56.11/32` |

Ejecute también este bloque de consulta administrativa:

```bash
sudo -u postgres psql -d postgres -c "
SELECT
    current_setting('server_version') AS version,
    current_setting('listen_addresses') AS escucha,
    current_setting('port') AS puerto,
    current_setting('max_connections') AS max_conexiones,
    current_setting('shared_buffers') AS buffers_compartidos,
    current_setting('logging_collector') AS colector_logs,
    current_setting('password_encryption') AS cifrado_password;"
```

La configuración debe ser coherente con los valores definidos durante la práctica.

## Solución de problemas

### Problema 1: PostgreSQL no inicia después de modificar `postgresql.conf`

**Síntomas**

- `sudo systemctl restart postgresql@16-main` falla.
- El estado muestra `failed`.
- `pg_isready` informa que no hay respuesta.
- El registro muestra mensajes como `FATAL`, `invalid value`, `syntax error` o `could not load`.

**Causa**

Existe un error de sintaxis, una unidad inválida, una directiva mal escrita o un valor no compatible en `postgresql.conf`. También puede existir una definición duplicada con un valor incorrecto al final del archivo.

**Corrección**

1. Revise los eventos del servicio:

   ```bash
   sudo journalctl -u postgresql@16-main -n 50 --no-pager
   ```

2. Compare el archivo actual con la copia de seguridad:

   ```bash
   sudo diff -u \
      /etc/postgresql/16/main/postgresql.conf.lab02.bak \
      /etc/postgresql/16/main/postgresql.conf
   ```

3. Corrija la línea indicada por el log o restaure temporalmente la copia:

   ```bash
   sudo cp -a \
      /etc/postgresql/16/main/postgresql.conf.lab02.bak \
      /etc/postgresql/16/main/postgresql.conf
   ```

4. Reinicie nuevamente:

   ```bash
   sudo systemctl restart postgresql@16-main
   ```

5. Reaplique los cambios de forma gradual, verificando el estado después de cada modificación importante.

### Problema 2: Un usuario de aplicación recibe `no pg_hba.conf entry` o falla la autenticación SCRAM

**Síntomas**

- La conexión remota falla con un mensaje similar a:

   ```text
   FATAL: no pg_hba.conf entry for host ...
   ```

- O bien aparece:

   ```text
   FATAL: password authentication failed for user "app_rw"
   ```

**Causa**

La dirección del cliente no coincide con una regla de `pg_hba.conf`, las reglas están en un orden incorrecto, no se recargó la configuración, la contraseña es incorrecta o fue creada antes de habilitar `password_encryption = 'scram-sha-256'`.

**Corrección**

1. Compruebe la dirección IP usada por el cliente:

   ```bash
   ip addr
   ```

2. Revise las reglas interpretadas por PostgreSQL:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT line_number, type, database, user_name, address, auth_method, error
   FROM pg_hba_file_rules
   ORDER BY line_number;"
   ```

3. Corrija la red, el usuario, la base o el orden de las reglas en:

   ```text
   /etc/postgresql/16/main/pg_hba.conf
   ```

4. Recargue la configuración:

   ```bash
   sudo -u postgres psql -d postgres -c "SELECT pg_reload_conf();"
   ```

5. Si es necesario, restablezca la contraseña del rol mediante una sesión administrativa:

   ```bash
   sudo -u postgres psql -d postgres
   ```

   ```sql
   SHOW password_encryption;
   \password app_rw
   ```

6. Repita la prueba TCP/IP:

   ```bash
   psql -h 192.168.56.10 -p 5432 -U app_rw -d sales_pg
   ```

## Limpieza

La configuración, roles y esquema creados en esta práctica deben conservarse porque serán utilizados en las prácticas de migración, respaldo y replicación.

Elimine únicamente los datos temporales de validación.

1. Elimine la tabla de prueba como administrador:

   ```bash
   sudo -u postgres psql -d sales_pg -c "
   DROP TABLE IF EXISTS sales.lab_access_test;"
   ```

2. Verifique que la base temporal de administración no exista:

   ```bash
   sudo -u postgres psql -d postgres -c "
   DROP DATABASE IF EXISTS sales_pg_check WITH (FORCE);"
   ```

3. Confirme que permanecen los roles requeridos:

   ```bash
   sudo -u postgres psql -d postgres -c "
   SELECT rolname, rolcanlogin
   FROM pg_roles
   WHERE rolname IN ('lab_admin', 'app_owner', 'app_rw', 'app_ro', 'migration_user')
   ORDER BY rolname;"
   ```

4. Confirme que el esquema `sales` permanece disponible:

   ```bash
   sudo -u postgres psql -d sales_pg -c "\dn sales"
   ```

> No elimine `sales_pg`, el esquema `sales`, los roles de aplicación ni las modificaciones de `postgresql.conf` y `pg_hba.conf`. Estos elementos constituyen la línea base para las siguientes prácticas.

## Resumen

En esta práctica administró la instancia PostgreSQL 16.2 del nodo `pg-primary` y aplicó controles operativos y de seguridad básicos. Creó roles de aplicación con privilegios mínimos, configuró el esquema `sales` y estableció permisos diferenciados para propietario, lectura/escritura, solo lectura y migración.

También configuró la escucha TCP/IP, límites de conexiones, memoria y registro de eventos mediante `postgresql.conf`; aplicó autenticación SCRAM y restricciones de red mediante `pg_hba.conf`; y verificó el estado del servicio con `systemctl`, `pg_isready`, `pg_settings`, `pg_hba_file_rules` y `pg_stat_activity`.

Como resultado, la instancia queda preparada para:

- Recibir datos migrados en la práctica de migración con pgloader.
- Integrarse en procedimientos de respaldo lógico y físico.
- Permitir la conexión futura del nodo `pg-standby` para replicación física.
- Mantener una política inicial de acceso basada en roles, autenticación SCRAM y privilegios mínimos.

### Recursos opcionales

- [Documentación oficial de CREATE DATABASE](https://www.postgresql.org/docs/current/sql-createdatabase.html)
- [Documentación oficial de ALTER DATABASE](https://www.postgresql.org/docs/current/sql-alterdatabase.html)
- [Documentación oficial de roles y privilegios](https://www.postgresql.org/docs/current/user-manag.html)
- [Documentación oficial de pg_hba.conf](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)
- [Documentación oficial de pg_settings](https://www.postgresql.org/docs/current/view-pg-settings.html)
- [Documentación oficial de pg_isready](https://www.postgresql.org/docs/current/app-pg-isready.html)
