# 1 Instalación de PostgreSQL en Linux

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 88 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica se prepara el nodo `pg-primary` con Ubuntu Server 22.04.4 LTS para instalar PostgreSQL Server 16.2 desde el repositorio oficial PostgreSQL Global Development Group (PGDG). Se verifican las versiones del cliente y del servidor, se revisan los componentes principales de la instalación y se realiza una conexión administrativa local mediante `psql`.

Al finalizar, el nodo dispondrá del clúster PostgreSQL `16/main`, identificado lógicamente como `lab16-primary`, del rol administrativo `lab_admin` y de la base de datos de trabajo `sales_pg`. Estos recursos se utilizarán en prácticas posteriores de configuración, migración, respaldo, recuperación y replicación.

## Objetivos de aprendizaje

- [ ] Preparar el servidor Ubuntu con nombre de host, IP estática, resolución local de nombres y acceso SSH.
- [ ] Agregar el repositorio PGDG e instalar PostgreSQL Server 16.2 y PostgreSQL Client Utilities 16.2.
- [ ] Identificar el servicio PostgreSQL, el clúster, el directorio de datos, los archivos de configuración y los roles iniciales.
- [ ] Conectarse localmente con `psql` usando el rol administrativo `postgres`.
- [ ] Crear el rol administrativo `lab_admin` y la base de datos `sales_pg`.

## Requisitos previos

### Conocimientos

- Fundamentos de SQL: bases de datos, tablas, usuarios y sentencias `CREATE`.
- Uso básico de GNU/Linux: terminal Bash, `sudo`, permisos, edición de archivos y administración de servicios.
- Conceptos básicos de redes IPv4, nombre de host y resolución de nombres mediante `/etc/hosts`.
- Comprensión inicial de PostgreSQL como sistema objeto-relacional de código abierto y de la diferencia entre versiones mayores y menores.

### Acceso requerido

- Máquina virtual o servidor Linux con Ubuntu Server 22.04.4 LTS de 64 bits.
- Cuenta de usuario con privilegios `sudo`.
- Conectividad temporal a Internet para descargar paquetes desde PGDG.
- Acceso de consola o SSH al nodo que se configurará como `pg-primary`.
- Dirección IP disponible `192.168.56.10/24` en la red privada de laboratorio.
- Conocimiento del nombre de la interfaz de red del sistema, por ejemplo `enp0s8`.

> **Importante:** esta práctica exige PostgreSQL Server **16.2** y PostgreSQL Client Utilities **16.2**. No continúe si `apt` propone una versión distinta. En un entorno académico, el repositorio o espejo de paquetes debe conservar la versión requerida.

## Entorno de laboratorio

### Topología prevista

| Elemento | Valor |
|---|---|
| Nodo configurado en esta práctica | `pg-primary` |
| Nombre lógico del clúster | `lab16-primary` |
| Sistema operativo | Ubuntu Server 22.04.4 LTS x86_64 |
| Dirección IPv4 de `pg-primary` | `192.168.56.10/24` |
| Dirección IPv4 futura de `pg-standby` | `192.168.56.11/24` |
| Red privada | `192.168.56.0/24` |
| Puerto PostgreSQL | TCP 5432 |
| Puerto SSH | TCP 22 |
| Versión de PostgreSQL requerida | 16.2 |
| Directorio de datos (`PGDATA`) | `/var/lib/postgresql/16/main` |
| Directorio de configuración | `/etc/postgresql/16/main` |
| Directorio de logs | `/var/log/postgresql/` |

### Recursos mínimos recomendados

| Recurso | Mínimo para el nodo | Recomendado |
|---|---:|---:|
| CPU | 2 vCPU | 4 vCPU |
| Memoria RAM | 4 GB | 8 GB |
| Disco libre | 20 GB | 40 GB o más |
| Red | Adaptador host-only o privada | Red privada con IP estática |

### Convenciones de comandos

Ejecute los comandos como el usuario administrativo de Ubuntu, salvo que se indique lo contrario.

- El símbolo `$` representa comandos ejecutados por un usuario normal.
- El símbolo `#` representa comandos ejecutados como `root` mediante `sudo`.
- El símbolo `postgres=#` representa comandos SQL dentro de `psql`.

No copie los símbolos `$`, `#` o `postgres=#` como parte del comando.

---

## Procedimiento paso a paso

### Paso 1. Verificar el sistema operativo y preparar la identificación del nodo

**Objetivo:** confirmar que el equipo utiliza Ubuntu Server 22.04.4 LTS, establecer el nombre de host `pg-primary`, configurar la zona horaria UTC y registrar la resolución local de los nodos del laboratorio.

#### Instrucciones

1. Inicie sesión en el servidor y confirme el nombre de host actual:

   ```bash
   hostnamectl
   ```

2. Verifique la versión del sistema operativo:

   ```bash
   cat /etc/os-release
   ```

3. Configure el nombre de host persistente del nodo:

   ```bash
   sudo hostnamectl set-hostname pg-primary
   ```

4. Establezca la zona horaria UTC, que será la referencia temporal común para todos los nodos PostgreSQL del laboratorio:

   ```bash
   sudo timedatectl set-timezone UTC
   timedatectl
   ```

5. Edite el archivo `/etc/hosts`:

   ```bash
   sudo nano /etc/hosts
   ```

6. Agregue o ajuste las siguientes líneas. Mantenga las entradas estándar de `localhost` existentes:

   ```text
   192.168.56.10  pg-primary
   192.168.56.11  pg-standby
   ```

7. Guarde el archivo y compruebe la resolución local:

   ```bash
   getent hosts pg-primary
   getent hosts pg-standby
   ```

8. Compruebe la fecha, el nombre de host y la zona horaria configurados:

   ```bash
   hostname
   date -u
   timedatectl show --property=Timezone --value
   ```

#### Salida esperada

- `hostname` debe mostrar:

  ```text
  pg-primary
  ```

- `timedatectl` debe indicar:

  ```text
  Time zone: UTC (UTC, +0000)
  ```

- `getent hosts pg-primary` debe resolver a:

  ```text
  192.168.56.10  pg-primary
  ```

- `getent hosts pg-standby` debe resolver a:

  ```text
  192.168.56.11  pg-standby
  ```

#### Verificación

Ejecute:

```bash
hostnamectl --static
getent hosts pg-primary pg-standby
```

El resultado debe identificar correctamente el nodo local como `pg-primary` y mostrar ambas direcciones IP de laboratorio.

---

### Paso 2. Configurar y verificar la dirección IPv4 estática

**Objetivo:** asignar o confirmar la dirección IPv4 estática `192.168.56.10/24` en la interfaz de red privada del nodo `pg-primary`.

> **Advertencia:** la configuración de red puede cortar una sesión SSH si se modifica la interfaz equivocada. Si trabaja remotamente, mantenga una consola de la máquina virtual disponible.

#### Instrucciones

1. Identifique las interfaces de red disponibles:

   ```bash
   ip -br address
   ```

2. Identifique los archivos de configuración Netplan:

   ```bash
   ls -l /etc/netplan/
   ```

3. Abra el archivo YAML existente. Sustituya el nombre mostrado por el archivo real de su sistema:

   ```bash
   sudo nano /etc/netplan/00-installer-config.yaml
   ```

4. Configure la interfaz privada. Sustituya `enp0s8` por el nombre real de su adaptador host-only o privado.

   Ejemplo de configuración para una red privada sin puerta de enlace:

   ```yaml
   network:
     version: 2
     ethernets:
       enp0s8:
         dhcp4: false
         addresses:
           - 192.168.56.10/24
   ```

5. Valide sintácticamente la configuración sin aplicarla de forma permanente:

   ```bash
   sudo netplan try
   ```

6. Si la red funciona correctamente, confirme la aplicación antes de que venza el temporizador de `netplan try`. Después aplique la configuración:

   ```bash
   sudo netplan apply
   ```

7. Compruebe la dirección asignada:

   ```bash
   ip -4 addr show enp0s8
   ```

8. Pruebe la resolución y conectividad local. El nodo `pg-standby` puede no estar disponible todavía; en ese caso, solo confirme que el nombre se resuelve:

   ```bash
   getent hosts pg-standby
   ping -c 2 192.168.56.10
   ```

9. Compruebe el servicio SSH para asegurar que estará disponible para tareas posteriores:

   ```bash
   sudo systemctl enable --now ssh
   sudo systemctl status ssh --no-pager
   ```

#### Salida esperada

La interfaz privada debe mostrar una dirección similar a:

```text
inet 192.168.56.10/24
```

El servicio SSH debe aparecer como:

```text
Active: active (running)
```

#### Verificación

Ejecute:

```bash
ip -br -4 address
ss -ltn | grep ':22'
```

Debe observar `192.168.56.10/24` en la interfaz privada y un proceso escuchando en el puerto TCP 22.

---

### Paso 3. Actualizar el sistema e instalar dependencias del repositorio PGDG

**Objetivo:** actualizar el índice local de paquetes e instalar las herramientas necesarias para registrar de forma segura la clave y el repositorio PGDG.

#### Instrucciones

1. Actualice el índice de paquetes del sistema:

   ```bash
   sudo apt update
   ```

2. Instale actualizaciones disponibles para Ubuntu antes de instalar PostgreSQL:

   ```bash
   sudo apt -y upgrade
   ```

3. Instale las dependencias requeridas para descargar y registrar la clave del repositorio:

   ```bash
   sudo apt install -y curl ca-certificates gnupg lsb-release
   ```

4. Compruebe la arquitectura y la distribución detectada:

   ```bash
   dpkg --print-architecture
   . /etc/os-release && echo "$VERSION_CODENAME"
   ```

5. Verifique que los valores sean compatibles con el laboratorio:

   ```text
   amd64
   jammy
   ```

#### Salida esperada

La instalación de dependencias debe finalizar sin errores. El comando de distribución debe mostrar:

```text
jammy
```

#### Verificación

Ejecute:

```bash
command -v curl
command -v gpg
lsb_release -cs
```

Debe obtener rutas para `curl` y `gpg`, y el resultado `jammy`.

---

### Paso 4. Agregar el repositorio oficial PGDG

**Objetivo:** configurar el repositorio PostgreSQL Global Development Group para Ubuntu Jammy y validar que `apt` puede consultar paquetes PostgreSQL 16.

#### Instrucciones

1. Cree el directorio destinado a claves APT si no existe:

   ```bash
   sudo install -d -m 0755 /etc/apt/keyrings
   ```

2. Descargue e instale la clave de firma del repositorio PGDG:

   ```bash
   curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | \
     sudo gpg --dearmor -o /etc/apt/keyrings/postgresql-pgdg.gpg
   ```

3. Asigne permisos de lectura a la clave:

   ```bash
   sudo chmod 0644 /etc/apt/keyrings/postgresql-pgdg.gpg
   ```

4. Registre el repositorio PGDG para Ubuntu 22.04 Jammy:

   ```bash
   echo "deb [signed-by=/etc/apt/keyrings/postgresql-pgdg.gpg] https://apt.postgresql.org/pub/repos/apt jammy-pgdg main" | \
     sudo tee /etc/apt/sources.list.d/pgdg.list
   ```

5. Actualice nuevamente el índice de paquetes:

   ```bash
   sudo apt update
   ```

6. Consulte las versiones disponibles del paquete del servidor:

   ```bash
   apt-cache madison postgresql-16
   ```

7. Consulte las versiones disponibles de las utilidades cliente:

   ```bash
   apt-cache madison postgresql-client-16
   ```

8. Confirme que el repositorio contiene la versión exacta `16.2`. En el momento de creación de un repositorio histórico de laboratorio, una versión de paquete habitual puede tener una forma semejante a:

   ```text
   16.2-1.pgdg22.04+1
   ```

   Si la versión exacta no aparece, no sustituya automáticamente `16.2` por una versión más reciente. Informe al instructor o configure el espejo histórico proporcionado para el curso.

#### Salida esperada

`apt-cache madison` debe mostrar una entrada de PostgreSQL 16 procedente de:

```text
apt.postgresql.org
```

En un entorno preparado para esta práctica debe estar disponible un paquete cuya versión comience por:

```text
16.2-
```

#### Verificación

Ejecute:

```bash
apt-cache policy postgresql-16 postgresql-client-16
```

Compruebe que el origen candidato sea el repositorio PGDG y que exista una versión 16.2 seleccionable.

---

### Paso 5. Instalar PostgreSQL Server 16.2 y PostgreSQL Client Utilities 16.2

**Objetivo:** instalar de forma controlada el servidor PostgreSQL y las utilidades cliente de la versión 16.2.

#### Instrucciones

1. Defina la versión de paquete exacta que haya identificado en el paso anterior. Ajuste el valor solo si su repositorio académico utiliza un sufijo distinto para PostgreSQL 16.2:

   ```bash
   PG_VERSION_PACKAGE="16.2-1.pgdg22.04+1"
   ```

2. Muestre la versión seleccionada antes de instalar:

   ```bash
   echo "$PG_VERSION_PACKAGE"
   ```

3. Instale el servidor y el cliente indicando explícitamente la versión:

   ```bash
   sudo apt install -y \
     "postgresql-16=${PG_VERSION_PACKAGE}" \
     "postgresql-client-16=${PG_VERSION_PACKAGE}"
   ```

4. Instale también el paquete común de cliente si no fue instalado automáticamente:

   ```bash
   sudo apt install -y postgresql-client-common postgresql-common
   ```

5. Compruebe la versión del cliente `psql`:

   ```bash
   psql --version
   ```

6. Compruebe el binario del servidor:

   ```bash
   /usr/lib/postgresql/16/bin/postgres --version
   ```

7. Liste los clústeres PostgreSQL administrados por las herramientas de Ubuntu:

   ```bash
   pg_lsclusters
   ```

8. Revise los paquetes instalados:

   ```bash
   dpkg -l | grep -E '^ii\s+(postgresql-16|postgresql-client-16)'
   ```

#### Salida esperada

Los comandos de versión deben mostrar exactamente PostgreSQL 16.2, por ejemplo:

```text
psql (PostgreSQL) 16.2
```

```text
postgres (PostgreSQL) 16.2
```

El comando `pg_lsclusters` debe mostrar un clúster denominado `16 main`, con puerto `5432`. Una salida típica es:

```text
Ver Cluster Port Status Owner    Data directory                     Log file
16  main    5432 online postgres /var/lib/postgresql/16/main       /var/log/postgresql/postgresql-16-main.log
```

#### Verificación

Ejecute:

```bash
psql --version
/usr/lib/postgresql/16/bin/postgres --version
pg_lsclusters
```

La versión de cliente y servidor debe ser `16.2`, y el estado del clúster debe ser `online`.

> **Concepto aplicado:** PostgreSQL utiliza números de versión mayor y menor. En `16.2`, el número `16` identifica la versión mayor y `2` la actualización menor. Las actualizaciones menores corrigen defectos y vulnerabilidades dentro de la misma versión mayor; no deben confundirse con una migración mayor, como pasar de PostgreSQL 15 a PostgreSQL 16.

---

### Paso 6. Revisar el servicio systemd y la arquitectura básica de la instalación

**Objetivo:** identificar la relación entre el servicio PostgreSQL, el clúster `16/main`, el proceso servidor y los directorios principales de la instalación.

#### Instrucciones

1. Revise el estado del servicio global de PostgreSQL:

   ```bash
   sudo systemctl status postgresql --no-pager
   ```

2. Revise el servicio específico del clúster PostgreSQL 16:

   ```bash
   sudo systemctl status postgresql@16-main --no-pager
   ```

3. Compruebe que el servicio está habilitado para arrancar con el sistema:

   ```bash
   sudo systemctl is-enabled postgresql
   ```

4. Liste los procesos PostgreSQL en ejecución:

   ```bash
   ps -ef | grep '[p]ostgres'
   ```

5. Revise el directorio de datos del clúster:

   ```bash
   sudo ls -ld /var/lib/postgresql/16/main
   sudo ls -la /var/lib/postgresql/16/main | head -n 25
   ```

6. Revise el directorio de configuración administrado por la distribución Ubuntu:

   ```bash
   sudo ls -la /etc/postgresql/16/main
   ```

7. Localice los archivos principales de configuración:

   ```bash
   sudo ls -l \
     /etc/postgresql/16/main/postgresql.conf \
     /etc/postgresql/16/main/pg_hba.conf \
     /etc/postgresql/16/main/pg_ident.conf
   ```

8. Revise, sin modificar todavía, algunos parámetros relevantes:

   ```bash
   sudo grep -E '^(data_directory|listen_addresses|port|timezone|log_timezone)' \
     /etc/postgresql/16/main/postgresql.conf
   ```

9. Revise las reglas activas de autenticación local:

   ```bash
   sudo grep -vE '^\s*#|^\s*$' /etc/postgresql/16/main/pg_hba.conf
   ```

10. Revise el archivo de log generado para el clúster:

   ```bash
   sudo tail -n 20 /var/log/postgresql/postgresql-16-main.log
   ```

#### Salida esperada

Debe observar los siguientes elementos:

- El clúster `16/main` está gestionado por `postgresql@16-main.service`.
- El propietario del directorio de datos es el usuario del sistema `postgres`.
- El directorio de datos es:

  ```text
  /var/lib/postgresql/16/main
  ```

- La configuración principal se encuentra en:

  ```text
  /etc/postgresql/16/main/postgresql.conf
  ```

- Las reglas de autenticación se encuentran en:

  ```text
  /etc/postgresql/16/main/pg_hba.conf
  ```

- El puerto configurado es normalmente `5432`.

#### Verificación

Ejecute:

```bash
sudo systemctl is-active postgresql@16-main
sudo -u postgres pg_ctlcluster 16 main status
sudo ss -ltnp | grep ':5432'
```

Debe obtener:

```text
active
```

Además, el puerto TCP 5432 debe estar en escucha local.

> **Arquitectura observada**
>
> | Concepto | Identificación en el laboratorio |
> |---|---|
> | Proceso servidor | Procesos `postgres` visibles con `ps -ef` |
> | Clúster PostgreSQL | `16/main`, listado por `pg_lsclusters` |
> | Instancia lógica | `lab16-primary` |
> | Directorio de datos | `/var/lib/postgresql/16/main` |
> | Configuración | `/etc/postgresql/16/main/` |
> | Base administrativa inicial | `postgres` |
> | Rol superusuario inicial | `postgres` |
> | Cliente de línea de comandos | `psql` |
>
> En PostgreSQL, un **clúster** es una colección de bases de datos administrada por una misma instancia del servidor. No equivale a un grupo de servidores de alta disponibilidad; ese uso del término se estudiará posteriormente en replicación.

---

### Paso 7. Conectarse localmente mediante psql y verificar el servidor

**Objetivo:** establecer una conexión local mediante el rol administrativo del sistema y consultar la versión exacta del servidor PostgreSQL.

#### Instrucciones

1. Inicie una sesión `psql` como el usuario del sistema `postgres`:

   ```bash
   sudo -u postgres psql
   ```

2. Dentro de `psql`, consulte la versión completa del servidor:

   ```sql
   SELECT version();
   ```

3. Consulte la versión concisa del servidor:

   ```sql
   SHOW server_version;
   ```

4. Consulte la zona horaria configurada en PostgreSQL:

   ```sql
   SHOW TimeZone;
   ```

5. Consulte el nombre de la base de datos actual y el usuario de sesión:

   ```sql
   SELECT current_database(), current_user, session_user;
   ```

6. Consulte la dirección del archivo de configuración realmente cargado por el servidor:

   ```sql
   SHOW config_file;
   ```

7. Consulte la ubicación del directorio de datos:

   ```sql
   SHOW data_directory;
   ```

8. Consulte la dirección del archivo de autenticación:

   ```sql
   SHOW hba_file;
   ```

9. Muestre las bases de datos iniciales utilizando un metacomando de `psql`:

   ```text
   \l
   ```

10. Muestre los roles existentes:

    ```text
    \du
    ```

11. Salga de `psql`:

    ```text
    \q
    ```

#### Salida esperada

La consulta `SHOW server_version;` debe mostrar:

```text
16.2
```

La consulta de zona horaria debe mostrar:

```text
UTC
```

La salida de `SHOW config_file;` debe indicar:

```text
/etc/postgresql/16/main/postgresql.conf
```

La salida de `SHOW data_directory;` debe indicar:

```text
/var/lib/postgresql/16/main
```

Las bases iniciales deben incluir, como mínimo:

- `postgres`
- `template0`
- `template1`

El rol `postgres` debe disponer de privilegios de superusuario.

#### Verificación

Ejecute el siguiente comando no interactivo:

```bash
sudo -u postgres psql -d postgres -c \
"SELECT current_database(), current_user, version(), current_setting('TimeZone') AS timezone;"
```

Verifique que:

- La base sea `postgres`.
- El usuario sea `postgres`.
- El servidor sea PostgreSQL 16.2.
- La zona horaria sea `UTC`.

> **Relación con el proyecto PostgreSQL:** la consulta `version()` permite registrar la versión concreta de servidor, la plataforma y la compilación. Mantener este registro es una práctica operativa importante para identificar actualizaciones de mantenimiento y planificar futuras actualizaciones mayores.

---

### Paso 8. Crear el rol administrativo lab_admin y la base de datos sales_pg

**Objetivo:** crear una cuenta administrativa dedicada para el laboratorio y una base de datos de trabajo que será utilizada como destino de migraciones posteriores.

#### Instrucciones

1. Conéctese nuevamente como el superusuario local `postgres`:

   ```bash
   sudo -u postgres psql
   ```

2. Compruebe la configuración de cifrado de contraseñas:

   ```sql
   SHOW password_encryption;
   ```

3. Cree el rol administrativo `lab_admin` con capacidad de inicio de sesión, creación de bases de datos y creación de roles:

   ```sql
   CREATE ROLE lab_admin
     WITH LOGIN
     CREATEDB
     CREATEROLE
     NOSUPERUSER
     INHERIT;
   ```

4. Asigne una contraseña al rol utilizando el comando interactivo de `psql`:

   ```text
   \password lab_admin
   ```

5. Introduzca una contraseña robusta cuando se solicite. Utilice una contraseña exclusiva del laboratorio y no reutilice credenciales personales.

6. Cree la base de datos `sales_pg` con `lab_admin` como propietario:

   ```sql
   CREATE DATABASE sales_pg
     OWNER lab_admin
     ENCODING 'UTF8'
     TEMPLATE template0;
   ```

7. Revise los atributos del nuevo rol:

   ```text
   \du lab_admin
   ```

8. Revise la base creada:

   ```text
   \l sales_pg
   ```

9. Consulte el propietario de la base de datos mediante SQL:

   ```sql
   SELECT datname AS base_de_datos,
          pg_get_userbyid(datdba) AS propietario,
          pg_encoding_to_char(encoding) AS codificacion
   FROM pg_database
   WHERE datname = 'sales_pg';
   ```

10. Salga de `psql`:

    ```text
    \q
    ```

#### Salida esperada

La consulta de la base de datos debe mostrar valores equivalentes a:

```text
 base_de_datos | propietario | codificacion
---------------+-------------+--------------
 sales_pg      | lab_admin   | UTF8
```

El rol `lab_admin` debe tener los atributos:

```text
Create role, Create DB
```

No debe tener el atributo `Superuser`.

#### Verificación

Ejecute:

```bash
sudo -u postgres psql -d postgres -c "\du lab_admin"
sudo -u postgres psql -d postgres -c "\l sales_pg"
```

Confirme que:

- `lab_admin` existe y puede crear roles y bases de datos.
- `lab_admin` no es superusuario.
- `sales_pg` existe.
- `lab_admin` es propietario de `sales_pg`.

> **Buenas prácticas de seguridad:** se utiliza `postgres` solo como cuenta inicial de administración local. Para las tareas posteriores del laboratorio se empleará `lab_admin`, evitando usar el superusuario de forma rutinaria. La configuración de autenticación remota y los privilegios detallados se abordarán en prácticas posteriores.

---

### Paso 9. Probar la conexión local con lab_admin y crear una tabla de validación

**Objetivo:** confirmar que el rol `lab_admin` puede conectarse a la base `sales_pg`, administrar objetos de su propia base y confirmar el comportamiento transaccional básico.

#### Instrucciones

1. Conéctese localmente a la base de datos `sales_pg` como el usuario del sistema `postgres` y cambie al rol `lab_admin` para validar sus privilegios:

   ```bash
   sudo -u postgres psql -d sales_pg
   ```

2. Dentro de `psql`, cambie temporalmente al rol administrativo creado:

   ```sql
   SET ROLE lab_admin;
   ```

3. Compruebe el usuario efectivo y la base actual:

   ```sql
   SELECT current_database(), current_user, session_user;
   ```

4. Cree una tabla sencilla de validación:

   ```sql
   CREATE TABLE installation_check (
     check_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
     checked_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP,
     check_note text NOT NULL
   );
   ```

5. Inserte un registro dentro de una transacción explícita:

   ```sql
   BEGIN;

   INSERT INTO installation_check (check_note)
   VALUES ('Instalación inicial de PostgreSQL 16.2 validada.');

   COMMIT;
   ```

6. Consulte el registro insertado:

   ```sql
   SELECT check_id, checked_at, check_note
   FROM installation_check;
   ```

7. Consulte la definición de la tabla:

   ```text
   \d installation_check
   ```

8. Restablezca el rol y salga:

   ```sql
   RESET ROLE;
   ```

   ```text
   \q
   ```

#### Salida esperada

La consulta debe devolver una fila con la nota de validación. La columna `checked_at` debe mostrar una marca de tiempo con zona UTC, por ejemplo:

```text
 check_id |       checked_at       |                check_note
----------+------------------------+--------------------------------------------
        1 | 2026-...+00             | Instalación inicial de PostgreSQL 16.2...
```

#### Verificación

Ejecute:

```bash
sudo -u postgres psql -d sales_pg -c \
"SELECT count(*) AS registros_validacion FROM installation_check;"
```

El resultado esperado es:

```text
 registros_validacion
----------------------
                    1
```

> **Concepto aplicado:** el bloque `BEGIN` y `COMMIT` demuestra el uso básico de transacciones. PostgreSQL prioriza las propiedades ACID: una vez ejecutado `COMMIT`, el registro queda confirmado y forma parte del estado persistente de la base de datos.

---

## Validación y pruebas

Complete las siguientes verificaciones finales. Todas deben finalizar correctamente antes de considerar terminada la práctica.

### Lista de comprobación técnica

| Validación | Comando | Resultado esperado |
|---|---|---|
| Nombre del nodo | `hostname` | `pg-primary` |
| Zona horaria del sistema | `timedatectl show --property=Timezone --value` | `UTC` |
| Resolución de nombres | `getent hosts pg-primary pg-standby` | Direcciones `192.168.56.10` y `192.168.56.11` |
| Cliente PostgreSQL | `psql --version` | `psql (PostgreSQL) 16.2` |
| Servidor PostgreSQL | `/usr/lib/postgresql/16/bin/postgres --version` | `postgres (PostgreSQL) 16.2` |
| Estado del clúster | `pg_lsclusters` | `16 main 5432 online` |
| Servicio específico | `sudo systemctl is-active postgresql@16-main` | `active` |
| Puerto PostgreSQL | `sudo ss -ltnp \| grep ':5432'` | Puerto TCP 5432 en escucha |
| Directorio de datos | `sudo -u postgres psql -tAc "SHOW data_directory"` | `/var/lib/postgresql/16/main` |
| Archivo de configuración | `sudo -u postgres psql -tAc "SHOW config_file"` | `/etc/postgresql/16/main/postgresql.conf` |
| Archivo HBA | `sudo -u postgres psql -tAc "SHOW hba_file"` | `/etc/postgresql/16/main/pg_hba.conf` |
| Rol administrativo | `sudo -u postgres psql -c "\du lab_admin"` | Rol con `Create role, Create DB` |
| Base de trabajo | `sudo -u postgres psql -c "\l sales_pg"` | Propietario `lab_admin` |
| Tabla de validación | `sudo -u postgres psql -d sales_pg -c "\dt"` | Tabla `installation_check` |

### Prueba consolidada

Ejecute la siguiente prueba consolidada:

```bash
sudo -u postgres psql -d sales_pg -v ON_ERROR_STOP=1 <<'SQL'
SELECT version() AS version_servidor;
SHOW server_version;
SHOW TimeZone;
SHOW data_directory;
SELECT current_database() AS base_actual;
SELECT rolname,
       rolcanlogin,
       rolcreatedb,
       rolcreaterole,
       rolsuper
FROM pg_roles
WHERE rolname = 'lab_admin';

SELECT datname,
       pg_get_userbyid(datdba) AS propietario
FROM pg_database
WHERE datname = 'sales_pg';

SELECT count(*) AS registros_en_installation_check
FROM installation_check;
SQL
```

La prueba será satisfactoria si confirma lo siguiente:

1. El servidor informa versión `16.2`.
2. PostgreSQL opera con zona horaria `UTC`.
3. El directorio de datos es `/var/lib/postgresql/16/main`.
4. La base actual es `sales_pg`.
5. El rol `lab_admin` tiene inicio de sesión, `CREATEDB` y `CREATEROLE`, pero no es superusuario.
6. La base `sales_pg` pertenece a `lab_admin`.
7. La tabla `installation_check` contiene un registro.

---

## Solución de problemas

### Problema 1: APT no encuentra PostgreSQL 16.2 o propone una versión diferente

**Síntomas**

Al ejecutar la instalación se muestra un mensaje similar a:

```text
E: Version '16.2-1.pgdg22.04+1' for 'postgresql-16' was not found
```

O bien:

```bash
apt-cache madison postgresql-16
```

solo muestra versiones posteriores a 16.2.

**Causa**

El repositorio PGDG público mantiene normalmente las versiones recientes y puede dejar de ofrecer paquetes de una actualización menor antigua. También puede existir una configuración incorrecta del repositorio, una distribución Ubuntu distinta de `jammy`, una clave no válida o un proxy que impide actualizar el índice APT.

**Corrección**

1. Compruebe la versión de Ubuntu:

   ```bash
   . /etc/os-release && echo "$VERSION_CODENAME"
   ```

   Debe mostrar `jammy`.

2. Revise el repositorio configurado:

   ```bash
   cat /etc/apt/sources.list.d/pgdg.list
   ```

3. Actualice el índice y examine las versiones disponibles:

   ```bash
   sudo apt update
   apt-cache madison postgresql-16
   ```

4. Si 16.2 no está disponible, no instale otra versión para esta práctica. Solicite al instructor la URL del espejo o repositorio histórico del curso que contenga PostgreSQL 16.2.

5. Una vez configurado el espejo correcto, repita:

   ```bash
   apt-cache madison postgresql-16
   sudo apt install -y \
     "postgresql-16=16.2-1.pgdg22.04+1" \
     "postgresql-client-16=16.2-1.pgdg22.04+1"
   ```

---

### Problema 2: El clúster PostgreSQL aparece como `down` o psql no conecta

**Síntomas**

`pg_lsclusters` muestra un estado similar a:

```text
16  main  5432  down
```

O la conexión falla con un error similar a:

```text
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed
```

**Causa**

El servicio del clúster no se inició después de la instalación, falló por una configuración inválida, existe un conflicto en el puerto 5432 o los permisos del directorio de datos fueron modificados incorrectamente.

**Corrección**

1. Consulte el estado detallado del servicio:

   ```bash
   sudo systemctl status postgresql@16-main --no-pager
   ```

2. Revise los mensajes recientes del servicio:

   ```bash
   sudo journalctl -u postgresql@16-main -n 50 --no-pager
   ```

3. Revise el log del clúster:

   ```bash
   sudo tail -n 50 /var/log/postgresql/postgresql-16-main.log
   ```

4. Compruebe si otro proceso ocupa el puerto 5432:

   ```bash
   sudo ss -ltnp | grep ':5432'
   ```

5. Si no hay errores de configuración y el puerto está disponible, inicie el clúster:

   ```bash
   sudo systemctl start postgresql@16-main
   ```

6. Compruebe nuevamente:

   ```bash
   pg_lsclusters
   sudo -u postgres psql -d postgres -c "SELECT version();"
   ```

7. Si el log informa permisos incorrectos en el directorio de datos, restaure el propietario y permisos seguros:

   ```bash
   sudo chown -R postgres:postgres /var/lib/postgresql/16/main
   sudo chmod 0700 /var/lib/postgresql/16/main
   sudo systemctl restart postgresql@16-main
   ```

---

## Limpieza

Esta práctica crea componentes que serán necesarios en las prácticas posteriores, por lo que **no elimine** PostgreSQL, el rol `lab_admin` ni la base de datos `sales_pg`.

Realice únicamente las siguientes acciones de limpieza operativa:

1. Cierre todas las sesiones interactivas de `psql`:

   ```text
   \q
   ```

2. Compruebe que no existan archivos temporales innecesarios en el directorio personal:

   ```bash
   ls -la ~
   ```

3. Registre la versión instalada y la fecha de instalación en la bitácora del laboratorio:

   ```bash
   {
     echo "Fecha UTC: $(date -u '+%Y-%m-%dT%H:%M:%SZ')"
     echo "Hostname: $(hostname)"
     echo "Cliente: $(psql --version)"
     echo "Servidor: $(sudo -u postgres psql -tAc 'SHOW server_version')"
     echo "Clúster:"
     pg_lsclusters
   } | tee ~/lab-01-00-01-instalacion-verificacion.txt
   ```

4. Proteja el archivo de verificación para que solo su usuario pueda leerlo:

   ```bash
   chmod 600 ~/lab-01-00-01-instalacion-verificacion.txt
   ```

5. Si el procedimiento del curso lo requiere, cree un snapshot de la máquina virtual identificado de forma clara, por ejemplo:

   ```text
   Lab01_PostgreSQL16.2_Instalado
   ```

---

## Resumen

En esta práctica se preparó el nodo `pg-primary` con nombre de host, resolución local, red privada y zona horaria UTC. Se agregó el repositorio PGDG y se instaló PostgreSQL Server 16.2 junto con PostgreSQL Client Utilities 16.2.

También se identificaron los componentes esenciales de la instalación: el clúster `16/main`, el servicio `postgresql@16-main`, el directorio de datos `/var/lib/postgresql/16/main`, los archivos de configuración en `/etc/postgresql/16/main` y los logs en `/var/log/postgresql/`. Finalmente, se verificó el servidor con `psql`, se creó el rol administrativo `lab_admin` y se creó la base de datos `sales_pg`.

Estos recursos forman la base del entorno `lab16-primary` que se utilizará para controles de acceso, migración de datos, respaldos, recuperación y replicación física.

### Recursos opcionales

- [Documentación oficial de PostgreSQL 16](https://www.postgresql.org/docs/16/)
- [Descargas y repositorio PGDG para Linux](https://www.postgresql.org/download/linux/ubuntu/)
- [Documentación de psql](https://www.postgresql.org/docs/16/app-psql.html)
- [Documentación de roles de base de datos](https://www.postgresql.org/docs/16/database-roles.html)
- [Documentación de clústeres PostgreSQL en Ubuntu](https://manpages.debian.org/pg_lsclusters)
