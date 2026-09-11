# Prácticas 6.1 Gestión de usuarios y autenticación, acceso remoto y permisos

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 118 minutos |
| Complejidad | Alta |
| Nivel de Bloom | Aplicar |

## Descripción general

En este laboratorio se desplegará un contenedor PostgreSQL 15.6 denominado `pgadv15` y se creará la base de datos `ventas_seguras`, que será reutilizada en los laboratorios posteriores de PostGIS, respaldo y recuperación. Se aplicará un modelo de seguridad basado en roles de grupo, cuentas personales, privilegios mínimos, autenticación SCRAM-SHA-256, acceso TCP/IP restringido y conexiones TLS.

También se implementará seguridad a nivel de filas (RLS) sobre los pedidos comerciales. Cada operador solo podrá consultar, insertar o modificar pedidos correspondientes a la región que tenga asignada.

> **Importante:** las contraseñas incluidas en esta guía son exclusivamente contraseñas temporales de laboratorio. No reutilice este patrón ni almacene contraseñas reales en scripts, repositorios o historiales de comandos.

## Objetivos de aprendizaje

Al finalizar el laboratorio, podrá:

- [ ] Crear roles propietarios, roles de grupo y roles personales con atributos mínimos.
- [ ] Configurar autenticación `scram-sha-256` y reglas de acceso en `pg_hba.conf`.
- [ ] Restringir el acceso TCP/IP a la red Docker del laboratorio y habilitar TLS.
- [ ] Conceder, revocar y revisar privilegios sobre bases de datos, esquemas, tablas, secuencias y funciones.
- [ ] Implementar políticas RLS basadas en `current_user` para separar pedidos por región comercial.

## Requisitos previos

### Conocimientos necesarios

- Uso básico de `psql`.
- Sentencias `CREATE ROLE`, `CREATE TABLE`, `GRANT`, `REVOKE` y `ALTER ROLE`.
- Edición de archivos de texto en Linux.
- Conceptos básicos de Docker Compose y redes Docker.
- Diferencia entre rol con `LOGIN`, rol de grupo `NOLOGIN` y propietario de objetos.

### Acceso requerido

- Equipo Linux, Ubuntu Desktop o servidor Linux con Docker Engine y Docker Compose Plugin operativos.
- Acceso a Internet para descargar la imagen `postgres:15.6`.
- Usuario con permiso para ejecutar comandos Docker.
- Puerto TCP local `5432` disponible en el equipo anfitrión.
- OpenSSL instalado en el anfitrión.

Compruebe las herramientas:

```bash
docker --version
docker compose version
openssl version
```

## Entorno de laboratorio

| Componente | Valor |
|---|---|
| Motor PostgreSQL | PostgreSQL 15.6 |
| Contenedor | `pgadv15` |
| Base de datos | `ventas_seguras` |
| Red Docker | `pgadv-net` |
| Subred Docker | `172.28.0.0/16` |
| Dirección del contenedor | `172.28.0.10` |
| Puerto publicado | `127.0.0.1:5432` |
| Usuario administrador inicial | `postgres` |
| Esquemas de trabajo | `ventas`, `auditoria` |
| Autenticación | SCRAM-SHA-256 |
| Cifrado de transporte | TLS con certificado autofirmado |

> **Nota de compatibilidad:** este laboratorio usa PostgreSQL 15.6 porque representa el entorno de transición anterior a PostgreSQL 16.2. En los laboratorios de alta disponibilidad y respaldo con nodos `pg-primary` y `pg-standby`, mantenga la constante de versión PostgreSQL 16.2 indicada en el curso.

---

## Procedimiento paso a paso

### Paso 1. Crear e iniciar el entorno Docker

**Objetivo:** desplegar un servidor PostgreSQL 15.6 en una red privada Docker con dirección IP fija.

**Instrucciones:**

1. Cree un directorio de trabajo y acceda a él:

   ```bash
   mkdir -p ~/lab06-usuarios-seguridad
   cd ~/lab06-usuarios-seguridad
   ```

2. Cree el archivo `compose.yaml`:

   ```bash
   cat > compose.yaml <<'EOF'
   services:
     pgadv15:
       image: postgres:15.6
       container_name: pgadv15
       hostname: pgadv15
       environment:
         POSTGRES_USER: postgres
         POSTGRES_PASSWORD: PostgresAdminLab06!
         POSTGRES_DB: postgres
         TZ: UTC
         PGTZ: UTC
       ports:
         - "127.0.0.1:5432:5432"
       volumes:
         - pgadv15-data:/var/lib/postgresql/data
       networks:
         pgadv-net:
           ipv4_address: 172.28.0.10
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U postgres -d postgres"]
         interval: 5s
         timeout: 3s
         retries: 20

   volumes:
     pgadv15-data:

   networks:
     pgadv-net:
       name: pgadv-net
       ipam:
         config:
           - subnet: 172.28.0.0/16
   EOF
   ```

3. Inicie el contenedor:

   ```bash
   docker compose up -d
   ```

4. Espere hasta que el servicio tenga estado saludable:

   ```bash
   docker ps
   docker inspect --format='{{.State.Health.Status}}' pgadv15
   ```

5. Compruebe la versión del servidor y la conexión administrativa local:

   ```bash
   docker exec -u postgres pgadv15 psql -d postgres -c "SELECT version();"
   ```

**Salida esperada:**

La consulta debe mostrar una versión que incluya `PostgreSQL 15.6`. El estado de salud debe ser `healthy`.

**Verificación:**

```bash
docker exec -u postgres pgadv15 psql -d postgres -c \
"SELECT current_database(), current_user, current_setting('TimeZone');"
```

Debe mostrar la base de datos `postgres`, el usuario `postgres` y la zona horaria `UTC`.

---

### Paso 2. Configurar SCRAM, `pg_hba.conf`, escucha TCP/IP y TLS

**Objetivo:** restringir las conexiones a métodos SCRAM y TLS desde la red Docker del laboratorio.

**Instrucciones:**

1. Consulte las rutas de configuración activas:

   ```bash
   docker exec -u postgres pgadv15 psql -d postgres -c \
   "SHOW config_file; SHOW hba_file; SHOW data_directory;"
   ```

2. Cree un directorio local para los certificados:

   ```bash
   mkdir -p tls
   ```

3. Genere un certificado autofirmado válido para el nombre Docker `pgadv15` y para la IP privada del contenedor:

   ```bash
   openssl req -x509 -newkey rsa:4096 -sha256 -nodes \
     -keyout tls/server.key \
     -out tls/server.crt \
     -days 365 \
     -subj "/CN=pgadv15" \
     -addext "subjectAltName=DNS:pgadv15,IP:172.28.0.10"
   ```

4. Cree el archivo de configuración adicional de PostgreSQL:

   ```bash
   cat > lab-postgresql.conf <<'EOF'
   # Configuración específica del laboratorio 06
   listen_addresses = '*'
   password_encryption = 'scram-sha-256'

   ssl = on
   ssl_cert_file = 'tls/server.crt'
   ssl_key_file = 'tls/server.key'
   EOF
   ```

5. Cree un archivo `pg_hba.conf` restrictivo. La primera regla permite al usuario interno `postgres` administrar el servicio por socket Unix. Las conexiones locales para otros usuarios requieren SCRAM; las conexiones TCP/IP únicamente se aceptan desde la red Docker y deben utilizar TLS.

   ```bash
   cat > pg_hba.conf <<'EOF'
   # TYPE  DATABASE        USER            ADDRESS             METHOD

   # Administración local dentro del contenedor.
   local   all             postgres                            peer

   # Usuarios locales mediante socket Unix.
   local   all             all                                 scram-sha-256

   # Acceso remoto permitido exclusivamente desde la red Docker y con TLS.
   hostssl ventas_seguras  all             172.28.0.0/16        scram-sha-256

   # Rechazar explícitamente cualquier otro acceso TCP/IP.
   host    all             all             0.0.0.0/0            reject
   host    all             all             ::0/0                reject
   EOF
   ```

6. Copie los certificados y archivos de configuración al contenedor:

   ```bash
   docker cp tls/server.crt pgadv15:/tmp/server.crt
   docker cp tls/server.key pgadv15:/tmp/server.key
   docker cp lab-postgresql.conf pgadv15:/tmp/lab-postgresql.conf
   docker cp pg_hba.conf pgadv15:/tmp/pg_hba.conf
   ```

7. Instale los archivos con propietario y permisos apropiados. La clave privada debe ser legible solamente por el usuario del servicio PostgreSQL.

   ```bash
   docker exec -u 0 pgadv15 sh -c '
     mkdir -p /var/lib/postgresql/data/tls &&
     mv /tmp/server.crt /var/lib/postgresql/data/tls/server.crt &&
     mv /tmp/server.key /var/lib/postgresql/data/tls/server.key &&
     mv /tmp/pg_hba.conf /var/lib/postgresql/data/pg_hba.conf &&
     mv /tmp/lab-postgresql.conf /var/lib/postgresql/data/lab-postgresql.conf &&
     chown -R postgres:postgres /var/lib/postgresql/data/tls &&
     chmod 700 /var/lib/postgresql/data/tls &&
     chmod 600 /var/lib/postgresql/data/tls/server.key &&
     chmod 644 /var/lib/postgresql/data/tls/server.crt &&
     chown postgres:postgres /var/lib/postgresql/data/pg_hba.conf \
                              /var/lib/postgresql/data/lab-postgresql.conf &&
     printf "\ninclude_if_exists = '\''/var/lib/postgresql/data/lab-postgresql.conf'\''\n" \
       >> /var/lib/postgresql/data/postgresql.conf
   '
   ```

8. Recargue la configuración de autenticación. La recarga aplica las reglas de `pg_hba.conf` sin reiniciar el servicio:

   ```bash
   docker exec -u postgres pgadv15 psql -d postgres -c "SELECT pg_reload_conf();"
   ```

9. Reinicie una única vez el contenedor para aplicar los parámetros que requieren reinicio, en particular `ssl` y `listen_addresses`:

   ```bash
   docker restart pgadv15
   ```

10. Espere a que PostgreSQL vuelva a estar disponible:

   ```bash
   docker exec -u postgres pgadv15 psql -d postgres -c \
   "SHOW password_encryption; SHOW ssl; SHOW listen_addresses;"
   ```

**Salida esperada:**

La salida debe contener valores equivalentes a los siguientes:

```text
 password_encryption
---------------------
 scram-sha-256

 ssl
-----
 on

 listen_addresses
------------------
 *
```

**Verificación:**

Revise que PostgreSQL haya cargado las reglas esperadas:

```bash
docker exec -u postgres pgadv15 psql -d postgres -c \
"SELECT type, database, user_name, address, auth_method, error
 FROM pg_hba_file_rules
 ORDER BY line_number;"
```

No debe haber errores en la columna `error`.

---

### Paso 3. Crear roles propietarios, grupos y cuentas de inicio de sesión

**Objetivo:** aplicar separación entre propiedad de objetos, permisos colectivos e identidades personales.

**Instrucciones:**

1. Cree el archivo SQL de roles. Los roles de grupo y el propietario no tienen `LOGIN`; las cuentas personales sí lo tienen.

   ```bash
   cat > 01_roles.sql <<'EOF'
   -- Rol técnico propietario de los objetos comerciales.
   CREATE ROLE rol_propietario_ventas
     NOLOGIN
     NOSUPERUSER
     NOCREATEDB
     NOCREATEROLE
     NOINHERIT
     NOBYPASSRLS;

   -- Roles de grupo.
   CREATE ROLE rol_ventas_lectura NOLOGIN;
   CREATE ROLE rol_ventas_operacion NOLOGIN;
   CREATE ROLE rol_gis_lectura NOLOGIN;
   CREATE ROLE rol_auditoria NOLOGIN;

   -- Roles personales con privilegios mínimos.
   CREATE ROLE ana_ventas
     LOGIN
     PASSWORD 'AnaVentasLab06!'
     NOSUPERUSER NOCREATEDB NOCREATEROLE
     INHERIT NOBYPASSRLS;

   CREATE ROLE bruno_operador
     LOGIN
     PASSWORD 'BrunoOperadorLab06!'
     NOSUPERUSER NOCREATEDB NOCREATEROLE
     INHERIT NOBYPASSRLS;

   CREATE ROLE carla_gis
     LOGIN
     PASSWORD 'CarlaGISLab06!'
     NOSUPERUSER NOCREATEDB NOCREATEROLE
     INHERIT NOBYPASSRLS;

   CREATE ROLE diana_auditora
     LOGIN
     PASSWORD 'DianaAuditoraLab06!'
     NOSUPERUSER NOCREATEDB NOCREATEROLE
     INHERIT NOBYPASSRLS;

   -- Membresías: autorización mediante grupos, no privilegios individuales.
   GRANT rol_ventas_lectura TO ana_ventas;
   GRANT rol_ventas_operacion TO bruno_operador;
   GRANT rol_gis_lectura TO carla_gis;
   GRANT rol_auditoria TO diana_auditora;
   EOF
   ```

2. Ejecute el script como administrador:

   ```bash
   docker cp 01_roles.sql pgadv15:/tmp/01_roles.sql

   docker exec -u postgres pgadv15 psql -v ON_ERROR_STOP=1 \
     -d postgres -f /tmp/01_roles.sql
   ```

3. Revise roles, atributos y membresías:

   ```bash
   docker exec -u postgres pgadv15 psql -d postgres -c "\du"
   ```

4. Verifique que las contraseñas de los usuarios de laboratorio se almacenaron con SCRAM. Esta consulta solo puede ejecutarla un superusuario:

   ```bash
   docker exec -u postgres pgadv15 psql -d postgres -c \
   "SELECT rolname,
           rolpassword LIKE 'SCRAM-SHA-256%' AS usa_scram
    FROM pg_authid
    WHERE rolname IN ('ana_ventas', 'bruno_operador', 'carla_gis', 'diana_auditora')
    ORDER BY rolname;"
   ```

**Salida esperada:**

Los cuatro usuarios personales deben mostrar `usa_scram = t`. Ningún rol de grupo debe tener el atributo `Login`.

**Verificación:**

```bash
docker exec -u postgres pgadv15 psql -d postgres -c \
"SELECT r.rolname AS miembro, g.rolname AS rol_grupo
 FROM pg_auth_members m
 JOIN pg_roles r ON r.oid = m.member
 JOIN pg_roles g ON g.oid = m.roleid
 WHERE r.rolname IN ('ana_ventas', 'bruno_operador', 'carla_gis', 'diana_auditora')
 ORDER BY r.rolname;"
```

Debe existir una membresía de grupo para cada cuenta personal.

---

### Paso 4. Crear la base de datos, esquemas, tablas y datos comerciales

**Objetivo:** construir la base de datos `ventas_seguras` con propiedad centralizada en un rol técnico.

**Instrucciones:**

1. Cree la base de datos con `rol_propietario_ventas` como propietario:

   ```bash
   docker exec -u postgres pgadv15 psql -v ON_ERROR_STOP=1 -d postgres -c \
   "CREATE DATABASE ventas_seguras OWNER rol_propietario_ventas;"
   ```

2. Cree el script de estructura y datos:

   ```bash
   cat > 02_estructura.sql <<'EOF'
   SET ROLE rol_propietario_ventas;

   CREATE SCHEMA ventas AUTHORIZATION rol_propietario_ventas;
   CREATE SCHEMA auditoria AUTHORIZATION rol_propietario_ventas;

   CREATE TABLE ventas.clientes (
     cliente_id bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
     nombre text NOT NULL,
     correo text NOT NULL UNIQUE,
     region text NOT NULL CHECK (region IN ('NORTE', 'CENTRO', 'SUR')),
     fecha_registro timestamptz NOT NULL DEFAULT now()
   );

   CREATE TABLE ventas.pedidos (
     pedido_id bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
     cliente_id bigint NOT NULL REFERENCES ventas.clientes(cliente_id),
     region text NOT NULL CHECK (region IN ('NORTE', 'CENTRO', 'SUR')),
     fecha_pedido timestamptz NOT NULL DEFAULT now(),
     estado text NOT NULL DEFAULT 'NUEVO'
       CHECK (estado IN ('NUEVO', 'EN_PREPARACION', 'ENVIADO', 'CANCELADO')),
     importe_total numeric(12,2) NOT NULL CHECK (importe_total >= 0)
   );

   CREATE TABLE ventas.pedido_detalle (
     pedido_detalle_id bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
     pedido_id bigint NOT NULL REFERENCES ventas.pedidos(pedido_id) ON DELETE CASCADE,
     producto text NOT NULL,
     cantidad integer NOT NULL CHECK (cantidad > 0),
     precio_unitario numeric(12,2) NOT NULL CHECK (precio_unitario >= 0)
   );

   CREATE TABLE auditoria.asignacion_region (
     rolname name PRIMARY KEY,
     region text NOT NULL CHECK (region IN ('NORTE', 'CENTRO', 'SUR')),
     asignado_en timestamptz NOT NULL DEFAULT now()
   );

   CREATE TABLE auditoria.eventos_laboratorio (
     evento_id bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
     evento text NOT NULL,
     registrado_en timestamptz NOT NULL DEFAULT now()
   );

   -- Función sin acceso a información sensible: se utiliza para practicar EXECUTE.
   CREATE FUNCTION ventas.version_esquema()
   RETURNS text
   LANGUAGE sql
   STABLE
   AS $$
     SELECT 'ventas_seguras: esquema comercial inicial';
   $$;

   INSERT INTO ventas.clientes (nombre, correo, region) VALUES
     ('Almacén Norte S.A.', 'contacto@norte.example', 'NORTE'),
     ('Comercial Centro Ltda.', 'ventas@centro.example', 'CENTRO'),
     ('Distribuidora Sur S.L.', 'pedidos@sur.example', 'SUR');

   INSERT INTO ventas.pedidos (cliente_id, region, estado, importe_total) VALUES
     (1, 'NORTE', 'NUEVO', 1200.00),
     (1, 'NORTE', 'EN_PREPARACION', 850.50),
     (2, 'CENTRO', 'NUEVO', 2300.00),
     (3, 'SUR', 'ENVIADO', 450.75);

   INSERT INTO ventas.pedido_detalle (pedido_id, producto, cantidad, precio_unitario) VALUES
     (1, 'Router industrial', 2, 600.00),
     (2, 'Switch administrable', 3, 283.50),
     (3, 'Servidor comercial', 1, 2300.00),
     (4, 'Punto de acceso', 3, 150.25);

   INSERT INTO auditoria.asignacion_region (rolname, region) VALUES
     ('bruno_operador', 'NORTE');

   INSERT INTO auditoria.eventos_laboratorio (evento) VALUES
     ('Base de datos ventas_seguras creada'),
     ('Asignación regional creada para bruno_operador');

   RESET ROLE;
   EOF
   ```

3. Ejecute el script:

   ```bash
   docker cp 02_estructura.sql pgadv15:/tmp/02_estructura.sql

   docker exec -u postgres pgadv15 psql -v ON_ERROR_STOP=1 \
     -d ventas_seguras -f /tmp/02_estructura.sql
   ```

4. Compruebe la propiedad de las tablas:

   ```bash
   docker exec -u postgres pgadv15 psql -d ventas_seguras -c \
   "SELECT schemaname, tablename, tableowner
    FROM pg_tables
    WHERE schemaname IN ('ventas', 'auditoria')
    ORDER BY schemaname, tablename;"
   ```

**Salida esperada:**

Las tablas de los esquemas `ventas` y `auditoria` deben pertenecer a `rol_propietario_ventas`.

**Verificación:**

```bash
docker exec -u postgres pgadv15 psql -d ventas_seguras -c \
"SELECT region, count(*) AS pedidos
 FROM ventas.pedidos
 GROUP BY region
 ORDER BY region;"
```

Debe mostrar pedidos para las regiones `NORTE`, `CENTRO` y `SUR`.

---

### Paso 5. Aplicar privilegios mínimos y privilegios predeterminados

**Objetivo:** conceder permisos mediante roles de grupo, eliminar privilegios públicos innecesarios y definir permisos para objetos futuros.

**Instrucciones:**

1. Cree el script de privilegios:

   ```bash
   cat > 03_privilegios.sql <<'EOF'
   -- El propietario ejecuta los privilegios por defecto de sus objetos futuros.
   SET ROLE rol_propietario_ventas;

   -- Reducir permisos implícitos o públicos no requeridos.
   REVOKE ALL ON DATABASE ventas_seguras FROM PUBLIC;
   REVOKE ALL ON SCHEMA public FROM PUBLIC;
   REVOKE ALL ON SCHEMA ventas FROM PUBLIC;
   REVOKE ALL ON SCHEMA auditoria FROM PUBLIC;
   REVOKE ALL ON ALL TABLES IN SCHEMA ventas FROM PUBLIC;
   REVOKE ALL ON ALL TABLES IN SCHEMA auditoria FROM PUBLIC;
   REVOKE ALL ON ALL SEQUENCES IN SCHEMA ventas FROM PUBLIC;
   REVOKE ALL ON ALL SEQUENCES IN SCHEMA auditoria FROM PUBLIC;
   REVOKE ALL ON ALL FUNCTIONS IN SCHEMA ventas FROM PUBLIC;
   REVOKE ALL ON ALL FUNCTIONS IN SCHEMA auditoria FROM PUBLIC;

   -- Permiso de conexión concedido solo a las identidades requeridas.
   GRANT CONNECT ON DATABASE ventas_seguras
     TO ana_ventas, bruno_operador, carla_gis, diana_auditora;

   -- Lectura comercial limitada: clientes y función informativa.
   GRANT USAGE ON SCHEMA ventas TO rol_ventas_lectura;
   GRANT SELECT ON ventas.clientes TO rol_ventas_lectura;
   GRANT EXECUTE ON FUNCTION ventas.version_esquema() TO rol_ventas_lectura;

   -- Operación de pedidos. SELECT es necesario para UPDATE y RETURNING.
   GRANT USAGE ON SCHEMA ventas TO rol_ventas_operacion;
   GRANT SELECT ON ventas.clientes TO rol_ventas_operacion;
   GRANT SELECT, INSERT, UPDATE ON ventas.pedidos TO rol_ventas_operacion;
   GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA ventas TO rol_ventas_operacion;

   -- Rol preparado para la práctica posterior de GIS: acceso de solo lectura a clientes.
   GRANT USAGE ON SCHEMA ventas TO rol_gis_lectura;
   GRANT SELECT ON ventas.clientes TO rol_gis_lectura;

   -- Auditoría: solo lectura del esquema y sus tablas.
   GRANT USAGE ON SCHEMA auditoria TO rol_auditoria;
   GRANT SELECT ON ALL TABLES IN SCHEMA auditoria TO rol_auditoria;

   -- Vista controlada para que la política RLS consulte la región del usuario.
   -- La vista accede a la tabla base como propietario; los operadores no reciben
   -- permisos directos sobre auditoria.asignacion_region.
   CREATE VIEW ventas.v_region_actual
   WITH (security_barrier = true)
   AS
   SELECT region
   FROM auditoria.asignacion_region
   WHERE rolname = current_user;

   GRANT SELECT ON ventas.v_region_actual TO rol_ventas_operacion;

   -- Privilegios para objetos futuros creados por rol_propietario_ventas.
   ALTER DEFAULT PRIVILEGES FOR ROLE rol_propietario_ventas
   IN SCHEMA ventas
   GRANT SELECT ON TABLES TO rol_ventas_lectura;

   ALTER DEFAULT PRIVILEGES FOR ROLE rol_propietario_ventas
   IN SCHEMA ventas
   GRANT SELECT, INSERT, UPDATE ON TABLES TO rol_ventas_operacion;

   ALTER DEFAULT PRIVILEGES FOR ROLE rol_propietario_ventas
   IN SCHEMA ventas
   GRANT USAGE, SELECT ON SEQUENCES TO rol_ventas_operacion;

   ALTER DEFAULT PRIVILEGES FOR ROLE rol_propietario_ventas
   IN SCHEMA ventas
   GRANT EXECUTE ON FUNCTIONS TO rol_ventas_lectura;

   RESET ROLE;
   EOF
   ```

2. Ejecute el script:

   ```bash
   docker cp 03_privilegios.sql pgadv15:/tmp/03_privilegios.sql

   docker exec -u postgres pgadv15 psql -v ON_ERROR_STOP=1 \
     -d ventas_seguras -f /tmp/03_privilegios.sql
   ```

3. Revise los privilegios efectivos sobre objetos del esquema `ventas`:

   ```bash
   docker exec -u postgres pgadv15 psql -d ventas_seguras -c "\dp ventas.*"
   ```

4. Revise los privilegios predeterminados configurados:

   ```bash
   docker exec -u postgres pgadv15 psql -d ventas_seguras -c \
   "\ddp"
   ```

**Salida esperada:**

- `rol_ventas_lectura` debe tener `SELECT` sobre `ventas.clientes`.
- `rol_ventas_operacion` debe tener permisos `SELECT`, `INSERT` y `UPDATE` sobre `ventas.pedidos`.
- `rol_ventas_operacion` debe tener permisos de uso y lectura sobre secuencias del esquema `ventas`.
- `PUBLIC` no debe tener privilegios de creación ni acceso innecesario en los esquemas del laboratorio.

**Verificación:**

Compruebe explícitamente que `PUBLIC` no posee permisos sobre `ventas.pedidos`:

```bash
docker exec -u postgres pgadv15 psql -d ventas_seguras -c \
"SELECT has_table_privilege('public', 'ventas.pedidos', 'SELECT') AS public_select,
        has_table_privilege('public', 'ventas.pedidos', 'INSERT') AS public_insert;"
```

El resultado esperado es `f` para ambos privilegios.

---

### Paso 6. Implementar Row-Level Security por región

**Objetivo:** asegurar que `bruno_operador` solo pueda operar sobre los pedidos de su región asignada.

**Instrucciones:**

1. Cree y aplique la configuración RLS:

   ```bash
   cat > 04_rls.sql <<'EOF'
   SET ROLE rol_propietario_ventas;

   ALTER TABLE ventas.pedidos ENABLE ROW LEVEL SECURITY;
   ALTER TABLE ventas.pedidos FORCE ROW LEVEL SECURITY;

   CREATE POLICY pedidos_por_region_operador
   ON ventas.pedidos
   FOR ALL
   TO rol_ventas_operacion
   USING (
     region IN (SELECT region FROM ventas.v_region_actual)
   )
   WITH CHECK (
     region IN (SELECT region FROM ventas.v_region_actual)
   );

   RESET ROLE;
   EOF
   ```

2. Ejecute el script:

   ```bash
   docker cp 04_rls.sql pgadv15:/tmp/04_rls.sql

   docker exec -u postgres pgadv15 psql -v ON_ERROR_STOP=1 \
     -d ventas_seguras -f /tmp/04_rls.sql
   ```

3. Revise las políticas definidas:

   ```bash
   docker exec -u postgres pgadv15 psql -d ventas_seguras -c \
   "\d+ ventas.pedidos"
   ```

4. Consulte el catálogo de políticas:

   ```bash
   docker exec -u postgres pgadv15 psql -d ventas_seguras -c \
   "SELECT schemaname, tablename, policyname, roles, cmd, qual, with_check
    FROM pg_policies
    WHERE schemaname = 'ventas'
      AND tablename = 'pedidos';"
   ```

**Salida esperada:**

Debe existir una política denominada `pedidos_por_region_operador`, aplicable al rol `rol_ventas_operacion`, con comandos `ALL` y expresiones `USING` y `WITH CHECK` basadas en la región del usuario.

**Verificación:**

Compruebe como administrador que la asignación regional existe:

```bash
docker exec -u postgres pgadv15 psql -d ventas_seguras -c \
"SELECT rolname, region FROM auditoria.asignacion_region;"
```

Debe mostrar que `bruno_operador` tiene asignada la región `NORTE`.

---

### Paso 7. Validar autenticación TLS, privilegios y aislamiento RLS

**Objetivo:** probar el comportamiento del sistema como usuarios no administrativos y comprobar que el acceso se cifra mediante TLS.

**Instrucciones:**

1. Pruebe una conexión TLS con validación completa del certificado desde un contenedor cliente de la misma red Docker:

   ```bash
   docker run --rm \
     --network pgadv-net \
     -v "$PWD/tls/server.crt:/certs/root.crt:ro" \
     -e PGPASSWORD='AnaVentasLab06!' \
     postgres:15.6 \
     psql "host=pgadv15 port=5432 dbname=ventas_seguras user=ana_ventas sslmode=verify-full sslrootcert=/certs/root.crt" \
     -c "SELECT current_user, current_setting('ssl') AS ssl_servidor;"
   ```

2. Compruebe que la sesión utiliza TLS:

   ```bash
   docker run --rm \
     --network pgadv-net \
     -v "$PWD/tls/server.crt:/certs/root.crt:ro" \
     -e PGPASSWORD='AnaVentasLab06!' \
     postgres:15.6 \
     psql "host=pgadv15 port=5432 dbname=ventas_seguras user=ana_ventas sslmode=verify-full sslrootcert=/certs/root.crt" \
     -c "SELECT ssl, version, cipher
         FROM pg_stat_ssl
         WHERE pid = pg_backend_pid();"
   ```

3. Valide que `ana_ventas` puede consultar clientes, pero no puede insertar pedidos:

   ```bash
   docker run --rm \
     --network pgadv-net \
     -v "$PWD/tls/server.crt:/certs/root.crt:ro" \
     -e PGPASSWORD='AnaVentasLab06!' \
     postgres:15.6 \
     psql "host=pgadv15 dbname=ventas_seguras user=ana_ventas sslmode=verify-full sslrootcert=/certs/root.crt" \
     -c "SELECT cliente_id, nombre, region FROM ventas.clientes ORDER BY cliente_id;"
   ```

   ```bash
   docker run --rm \
     --network pgadv-net \
     -v "$PWD/tls/server.crt:/certs/root.crt:ro" \
     -e PGPASSWORD='AnaVentasLab06!' \
     postgres:15.6 \
     psql "host=pgadv15 dbname=ventas_seguras user=ana_ventas sslmode=verify-full sslrootcert=/certs/root.crt" \
     -c "INSERT INTO ventas.pedidos (cliente_id, region, importe_total)
         VALUES (1, 'NORTE', 100.00);"
   ```

4. Pruebe la visibilidad de pedidos como `bruno_operador`. Solo deben aparecer pedidos de la región `NORTE`:

   ```bash
   docker run --rm \
     --network pgadv-net \
     -v "$PWD/tls/server.crt:/certs/root.crt:ro" \
     -e PGPASSWORD='BrunoOperadorLab06!' \
     postgres:15.6 \
     psql "host=pgadv15 dbname=ventas_seguras user=bruno_operador sslmode=verify-full sslrootcert=/certs/root.crt" \
     -c "SELECT pedido_id, region, estado, importe_total
         FROM ventas.pedidos
         ORDER BY pedido_id;"
   ```

5. Modifique un pedido de la región `NORTE` como `bruno_operador`:

   ```bash
   docker run --rm \
     --network pgadv-net \
     -v "$PWD/tls/server.crt:/certs/root.crt:ro" \
     -e PGPASSWORD='BrunoOperadorLab06!' \
     postgres:15.6 \
     psql "host=pgadv15 dbname=ventas_seguras user=bruno_operador sslmode=verify-full sslrootcert=/certs/root.crt" \
     -c "UPDATE ventas.pedidos
         SET estado = 'EN_PREPARACION'
         WHERE pedido_id = 1
         RETURNING pedido_id, region, estado;"
   ```

6. Intente modificar un pedido de la región `SUR`. La sentencia no debe modificar filas porque la política RLS impide su visibilidad:

   ```bash
   docker run --rm \
     --network pgadv-net \
     -v "$PWD/tls/server.crt:/certs/root.crt:ro" \
     -e PGPASSWORD='BrunoOperadorLab06!' \
     postgres:15.6 \
     psql "host=pgadv15 dbname=ventas_seguras user=bruno_operador sslmode=verify-full sslrootcert=/certs/root.crt" \
     -c "UPDATE ventas.pedidos
         SET estado = 'CANCELADO'
         WHERE pedido_id = 4
         RETURNING pedido_id, region, estado;"
   ```

7. Intente insertar un pedido en una región no asignada a `bruno_operador`. Esta operación debe fallar por la condición `WITH CHECK` de la política:

   ```bash
   docker run --rm \
     --network pgadv-net \
     -v "$PWD/tls/server.crt:/certs/root.crt:ro" \
     -e PGPASSWORD='BrunoOperadorLab06!' \
     postgres:15.6 \
     psql "host=pgadv15 dbname=ventas_seguras user=bruno_operador sslmode=verify-full sslrootcert=/certs/root.crt" \
     -c "INSERT INTO ventas.pedidos (cliente_id, region, importe_total)
         VALUES (3, 'SUR', 999.99);"
   ```

8. Verifique el acceso de auditoría como `diana_auditora`:

   ```bash
   docker run --rm \
     --network pgadv-net \
     -v "$PWD/tls/server.crt:/certs/root.crt:ro" \
     -e PGPASSWORD='DianaAuditoraLab06!' \
     postgres:15.6 \
     psql "host=pgadv15 dbname=ventas_seguras user=diana_auditora sslmode=verify-full sslrootcert=/certs/root.crt" \
     -c "SELECT evento_id, evento, registrado_en
         FROM auditoria.eventos_laboratorio
         ORDER BY evento_id;"
   ```

**Salida esperada:**

- La consulta de `pg_stat_ssl` debe mostrar `ssl = t` y un cifrado TLS.
- `ana_ventas` puede consultar `ventas.clientes`, pero su intento de `INSERT` debe producir `ERROR: permission denied for table pedidos`.
- `bruno_operador` solo debe ver pedidos `NORTE`.
- La actualización del pedido `1` debe devolver una fila.
- La actualización del pedido `4` debe devolver `UPDATE 0`.
- El intento de insertar un pedido `SUR` debe fallar con un error relacionado con la política RLS.
- `diana_auditora` debe poder consultar las tablas del esquema `auditoria`.

**Verificación:**

Ejecute esta consulta administrativa final:

```bash
docker exec -u postgres pgadv15 psql -d ventas_seguras -c \
"SELECT
   c.relname AS tabla,
   c.relrowsecurity AS rls_habilitada,
   c.relforcerowsecurity AS rls_forzada
 FROM pg_class c
 JOIN pg_namespace n ON n.oid = c.relnamespace
 WHERE n.nspname = 'ventas'
   AND c.relname = 'pedidos';"
```

El resultado debe indicar `rls_habilitada = t` y `rls_forzada = t`.

---

## Validación y pruebas

Ejecute las siguientes comprobaciones como lista de aceptación final del laboratorio.

| Prueba | Comando o criterio | Resultado esperado |
|---|---|---|
| Versiones | `SELECT version();` | PostgreSQL 15.6 |
| Roles de grupo | `\du` | Cuatro roles `NOLOGIN` de grupo |
| Cuentas personales | `\du` | Cuatro roles con `LOGIN`, sin privilegios administrativos |
| Contraseñas | Consulta sobre `pg_authid` | Hashes SCRAM para las cuatro cuentas |
| Base de datos | `\l ventas_seguras` | Propietario `rol_propietario_ventas` |
| Esquemas | `\dn+` | Esquemas `ventas` y `auditoria` con propietario técnico |
| Privilegios PUBLIC | `has_table_privilege('public', ...)` | Sin `SELECT` ni `INSERT` sobre pedidos |
| TLS | `pg_stat_ssl` | `ssl = true` |
| HBA | `pg_hba_file_rules` | Solo Docker `172.28.0.0/16` con `scram-sha-256` |
| RLS | `pg_policies` | Política regional sobre `ventas.pedidos` |
| Aislamiento | Consulta como `bruno_operador` | Solo pedidos de región `NORTE` |

Como comprobación adicional de atributos de seguridad, ejecute:

```bash
docker exec -u postgres pgadv15 psql -d postgres -c \
"SELECT rolname,
        rolcanlogin,
        rolsuper,
        rolcreaterole,
        rolcreatedb,
        rolreplication,
        rolbypassrls
 FROM pg_roles
 WHERE rolname IN (
   'rol_propietario_ventas',
   'rol_ventas_lectura',
   'rol_ventas_operacion',
   'rol_gis_lectura',
   'rol_auditoria',
   'ana_ventas',
   'bruno_operador',
   'carla_gis',
   'diana_auditora'
 )
 ORDER BY rolname;"
```

Ninguno de los roles de laboratorio debe tener `rolsuper`, `rolcreaterole`, `rolcreatedb`, `rolreplication` o `rolbypassrls` en valor verdadero.

## Resolución de problemas

### Problema 1: PostgreSQL no inicia después de habilitar TLS

**Síntoma:**

```text
FATAL: could not load server certificate file
```

o:

```text
FATAL: private key file ".../server.key" has group or world access
```

**Causa:** el certificado no está en la ruta configurada, pertenece a un usuario incorrecto o la clave privada tiene permisos demasiado abiertos.

**Solución:**

Compruebe los archivos y corrija propietario y permisos:

```bash
docker exec -u 0 pgadv15 sh -c '
  ls -l /var/lib/postgresql/data/tls &&
  chown -R postgres:postgres /var/lib/postgresql/data/tls &&
  chmod 700 /var/lib/postgresql/data/tls &&
  chmod 600 /var/lib/postgresql/data/tls/server.key &&
  chmod 644 /var/lib/postgresql/data/tls/server.crt
'

docker restart pgadv15
docker logs pgadv15 --tail 50
```

### Problema 2: `bruno_operador` recibe “permission denied” o no ve pedidos esperados

**Síntoma:**

```text
ERROR: permission denied for table pedidos
```

o la consulta devuelve cero filas aunque existen pedidos `NORTE`.

**Causa:** la membresía de `rol_ventas_operacion`, los privilegios sobre `ventas.pedidos`, la asignación regional o la política RLS no se aplicaron correctamente.

**Solución:**

Ejecute las comprobaciones administrativas:

```bash
docker exec -u postgres pgadv15 psql -d ventas_seguras -c \
"SELECT rolname, region
 FROM auditoria.asignacion_region
 WHERE rolname = 'bruno_operador';"

docker exec -u postgres pgadv15 psql -d ventas_seguras -c \
"SELECT policyname, roles, cmd
 FROM pg_policies
 WHERE schemaname = 'ventas'
   AND tablename = 'pedidos';"

docker exec -u postgres pgadv15 psql -d ventas_seguras -c \
"SELECT has_table_privilege(
   'rol_ventas_operacion',
   'ventas.pedidos',
   'SELECT,INSERT,UPDATE'
 ) AS permisos_operacion;"
```

Si falta la asignación regional, restáurela como propietario:

```bash
docker exec -u postgres pgadv15 psql -d ventas_seguras -c \
"INSERT INTO auditoria.asignacion_region (rolname, region)
 VALUES ('bruno_operador', 'NORTE')
 ON CONFLICT (rolname) DO UPDATE SET region = EXCLUDED.region;"
```

## Limpieza

El estado final de `ventas_seguras`, los roles, las políticas RLS, los certificados y la configuración de acceso es una entrada requerida para los laboratorios posteriores. Por tanto, **no elimine el volumen Docker ni destruya la base de datos al finalizar esta práctica**.

Para detener temporalmente el entorno y conservar todos los datos:

```bash
cd ~/lab06-usuarios-seguridad
docker compose stop
```

Para volver a iniciarlo posteriormente:

```bash
cd ~/lab06-usuarios-seguridad
docker compose start
```

> **Acción destructiva opcional:** ejecute lo siguiente solo si necesita reiniciar completamente el laboratorio y acepta perder la base de datos, los roles y toda la configuración almacenada en el volumen.

```bash
cd ~/lab06-usuarios-seguridad
docker compose down -v
docker network rm pgadv-net 2>/dev/null || true
rm -rf tls
```

## Resumen

En este laboratorio se implementó un modelo de seguridad de PostgreSQL basado en el principio de mínimo privilegio:

- Se crearon roles propietarios sin inicio de sesión para evitar que los objetos dependan de cuentas personales.
- Se definieron roles de grupo `NOLOGIN` y cuentas individuales con permisos administrativos deshabilitados.
- Se configuró el almacenamiento de contraseñas con `scram-sha-256`.
- Se restringieron las conexiones mediante `pg_hba.conf` a la red Docker privada del laboratorio.
- Se habilitó TLS con certificado autofirmado y se validó una conexión cifrada mediante `sslmode=verify-full`.
- Se revocaron privilegios innecesarios de `PUBLIC`.
- Se concedieron permisos explícitos sobre esquemas, tablas, secuencias y funciones.
- Se definieron privilegios predeterminados para futuros objetos del propietario técnico.
- Se habilitó Row-Level Security sobre `ventas.pedidos` para limitar a `bruno_operador` a su región comercial asignada.

### Recursos opcionales

- [Documentación PostgreSQL: Roles de base de datos](https://www.postgresql.org/docs/15/database-roles.html)
- [Documentación PostgreSQL: Autenticación de clientes](https://www.postgresql.org/docs/15/client-authentication.html)
- [Documentación PostgreSQL: SSL/TLS](https://www.postgresql.org/docs/15/ssl-tcp.html)
- [Documentación PostgreSQL: Privilegios predeterminados](https://www.postgresql.org/docs/15/sql-alterdefaultprivileges.html)
- [Documentación PostgreSQL: Seguridad a nivel de filas](https://www.postgresql.org/docs/15/ddl-rowsecurity.html)
