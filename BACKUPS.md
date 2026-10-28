# Backups

Hay dos scripts de backup automático. Se pueden usar por separado o los dos a la vez (por ejemplo rsync todos los días y Glacier una vez por semana).

| Script | Destino | Ideal para |
| --- | --- | --- |
| `backup_rsync.sh.example` | Otro servidor (local / LAN) por SSH + rsync | Restauraciones rápidas, copias diarias. |
| `backup_glacier.sh.example` | AWS S3 Glacier Deep Archive | Copia fuera del sitio, barata y muy durable. Restaurar demora horas. |

## Cómo funcionan

- Hacen un backup **por cada bucket** de la variable `BUCKETS` del script.
- Cada bucket se descarga completo con el servicio `backup` de _compose.yaml_ (cliente `aws-cli` que se conecta a RustFS por la red interna de docker) y se comprime en un archivo `NOMBRE_BUCKET-AAAAMMDD-HHMM.tar.gz`. Adentro están los objetos como archivos comunes, respetando las "carpetas" (prefijos) del bucket.
- Usan las credenciales `RUSTFS_ACCESS_KEY` / `RUSTFS_SECRET_KEY` del _.env_, así que **los scripts no guardan contraseñas de RustFS**.
- **No incluyen**:
  - Los usuarios, access keys ni políticas de acceso.
  - La configuración de los buckets (políticas públicas, versionado, reglas de vencimiento, etc.).
  - Las versiones anteriores de los objetos (sólo la versión actual).
  - Los metadatos propios de cada objeto (`x-amz-meta-*`). Al restaurar, el `Content-Type` se vuelve a calcular según la extensión del archivo.

  Al restaurar se vuelven a configurar a mano (ver [Restaurar en una instancia nueva](#restaurar-en-una-instancia-nueva)).
- Los archivos se generan en **/backups**, se envían al destino y luego se borra la copia local. Si el envío falla, la copia local queda en **/backups**.
- Todo queda registrado en **backup.log** (en la carpeta del proyecto).

```bash
$ tail -n 50 backup.log
```

> **Espacio en disco:** durante el backup cada bucket se descarga completo en **/backups** y después se comprime, así que hace falta libre **alrededor del doble del tamaño del bucket más grande**.

> Los scripts se ejecutan como **root** (desde el cron de root), igual que los ejemplos de esta guía.

---

## Opción 1: Backup a otro servidor con rsync

### 1. Preparar el servidor de destino

```bash
$ # ¡¡¡En el servidor de destino!!!
$ # Instalar rsync (tiene que estar en los dos servidores)
$ sudo apt-get install rsync

$ # Crear un usuario y la carpeta para los backups
$ sudo useradd -m backup_user -s /bin/bash
$ sudo passwd backup_user
$ sudo -u backup_user mkdir -p /home/backup_user/backups/rustfs

$ # Habilitar autenticación con clave pública en SSH
$ sudo nano /etc/ssh/sshd_config
$ # Descomentar la linea: PubkeyAuthentication yes
$ sudo systemctl restart ssh.service
```

### 2. Preparar el servidor de origen (donde corre RustFS)

```bash
$ # ¡¡¡En el servidor de origen, como ROOT!!!
root$ apt-get install rsync
root$ # Generar las claves SSH de root (Enter a todo, sin passphrase)
root$ ssh-keygen -t ed25519
root$ # Copiar la clave pública al servidor de destino
root$ ssh-copy-id -p 22 backup_user@SERVIDOR_DESTINO
root$ # Probar que entra SIN pedir contraseña
root$ ssh -p 22 -o BatchMode=yes backup_user@SERVIDOR_DESTINO "ls /home/backup_user/backups/rustfs"
```

### 3. Configurar el script

```bash
root$ cd /ruta/al/proyecto
root$ cp backup_rsync.sh.example backup_rsync.sh
root$ chmod 700 backup_rsync.sh
root$ nano backup_rsync.sh
```

Variables a configurar (sección `CONFIG`):

| Variable | Descripción |
| --- | --- |
| `BUCKETS` | Buckets a respaldar, separados por espacio. |
| `SSH_DEST_HOST` | IP o nombre del servidor de destino. |
| `SSH_DEST_PORT` | Puerto SSH del servidor de destino. |
| `SSH_DEST_USER` | Usuario del servidor de destino (`backup_user`). |
| `SSH_DEST_PATH` | Carpeta de destino (tiene que existir). |

Para ver los buckets que existen:

```bash
root$ docker compose run --rm backup s3 ls
```

Probarlo a mano:

```bash
root$ ./backup_rsync.sh && echo OK
root$ tail -n 20 backup.log
```

### 4. (Opcional) Borrar backups viejos en el destino

El script no borra nada en el destino. Para no llenar el disco, programar una limpieza **en el servidor de destino**:

```bash
$ # ¡¡¡En el servidor de destino!!!
$ sudo crontab -u backup_user -e
$ # Borrar backups de más de 30 días, todos los días a las 5 AM:
0 5 * * * find /home/backup_user/backups/rustfs -name '*.tar.gz' -mtime +30 -delete
```

Luego seguir con [Programar el backup con cron](#programar-el-backup-con-cron).

---

## Opción 2: Backup a AWS S3 Glacier Deep Archive

**S3 Glacier Deep Archive** es la clase de almacenamiento más barata de AWS y una de las más durables:

- Costo aproximado: **~1 USD por TB por mes** (US$ 0,00099 por GB/mes en `us-east-1`; varía según la región).
- Durabilidad: 99,999999999% (11 nueves). Los datos se replican en al menos 3 zonas de disponibilidad.
- **Se cobra un mínimo de 180 días** por cada archivo (si se borra antes, se cobran igual los 180 días).
- **Los archivos no se descargan inmediatamente**: hay que pedir que se restauren y esperar hasta 12 hs (o 48 hs en el modo más barato). Ver [Descargar un backup desde AWS](#desde-aws-s3-glacier-deep-archive).

Los archivos quedan en:

```
s3://BUCKET/rustfs/HOSTNAME/NOMBRE_BUCKET/NOMBRE_BUCKET-AAAAMMDD-HHMM.tar.gz
```

### 1. Crear el bucket

Desde la [consola de S3](https://console.aws.amazon.com/s3/) → **Create bucket**:

- **Bucket name**: un nombre único, por ejemplo `mi-empresa-backups-rustfs`.
- **Region**: la que prefieras (`us-east-1` es de las más baratas).
- Dejar **Block all public access** activado (viene por defecto).
- Opcional pero recomendado: **Bucket Versioning → Enable**, así un archivo pisado o borrado se puede recuperar.

### 2. Regla de vencimiento (lifecycle)

Para que los backups viejos se borren solos. En el bucket → **Management → Create lifecycle rule**:

- **Rule name**: `expirar-backups-rustfs`
- **Prefix**: `rustfs/`
- **Expire current versions of objects**: por ejemplo `365` días (no tiene sentido menos de 180, se cobran igual).
- Si se activó el versionado: **Permanently delete noncurrent versions** → `1` día.
- **Delete expired object delete markers or incomplete multipart uploads** → incomplete multipart uploads: `7` días.

### 3. Crear un usuario IAM sólo para subir backups

El servidor sólo debe poder **subir** archivos, no borrarlos ni descargarlos. Así, si alguien compromete el servidor, no puede borrar los backups.

En la [consola de IAM](https://console.aws.amazon.com/iam/):

1. **Policies → Create policy → JSON**, pegar (reemplazar `mi-empresa-backups-rustfs`) y guardarla como `rustfs-backup-upload`:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "SubirBackups",
            "Effect": "Allow",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::mi-empresa-backups-rustfs/rustfs/*"
        },
        {
            "Sid": "ListarBackups",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::mi-empresa-backups-rustfs",
            "Condition": { "StringLike": { "s3:prefix": "rustfs/*" } }
        }
    ]
}
```

2. **Users → Create user** → nombre `rustfs-backup` → **Attach policies directly** → elegir `rustfs-backup-upload`.
3. En el usuario creado → **Security credentials → Create access key** → _Command Line Interface (CLI)_. Guardar el **Access key ID** y el **Secret access key** (el secreto se muestra una sola vez).

### 4. Instalar y configurar AWS CLI en el servidor de origen

Fuente: [Instalación AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

_El servicio `backup` de compose.yaml ya trae `aws-cli`, pero sólo se usa para leer de RustFS. Para subir a AWS hace falta instalarlo también en el servidor._

```bash
$ # ¡¡¡En el servidor de origen, como ROOT!!!
root$ apt-get install unzip
root$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
root$ unzip awscliv2.zip
root$ ./aws/install
root$ rm -rf aws awscliv2.zip
root$ aws --version

root$ # Configurar las credenciales del usuario IAM en un perfil
root$ aws configure --profile rustfs-backup
AWS Access Key ID: # Access key ID del paso anterior
AWS Secret Access Key: # Secret access key del paso anterior
Default region name: # La región del bucket, ej: us-east-1
Default output format: # Enter

root$ # Probar el acceso
root$ aws s3 ls s3://mi-empresa-backups-rustfs/rustfs/ --profile rustfs-backup
```

_Las credenciales quedan en **/root/.aws/credentials**, sólo accesibles por root._

### 5. Configurar el script

```bash
root$ cd /ruta/al/proyecto
root$ cp backup_glacier.sh.example backup_glacier.sh
root$ chmod 700 backup_glacier.sh
root$ nano backup_glacier.sh
```

| Variable | Descripción |
| --- | --- |
| `BUCKETS` | Buckets de RustFS a respaldar, separados por espacio. |
| `S3_BUCKET` | Nombre del bucket de AWS. |
| `S3_PREFIX` | Carpeta dentro del bucket. Por defecto `rustfs/HOSTNAME`. Tiene que empezar con `rustfs/` para coincidir con la política IAM y la regla de vencimiento. |
| `S3_STORAGE_CLASS` | `DEEP_ARCHIVE` (no cambiar salvo que se quiera otra clase). |
| `AWS_PROFILE` | Perfil configurado con `aws configure --profile`. |

Probarlo a mano:

```bash
root$ ./backup_glacier.sh && echo OK
root$ tail -n 20 backup.log
root$ aws s3 ls s3://mi-empresa-backups-rustfs/rustfs/ --recursive --profile rustfs-backup
```

---

## Programar el backup con cron

Igual para los dos scripts. Se programa en el cron de **root** del servidor de origen.

```bash
root$ crontab -e
```

```bash
# Para probar: un backup cada 5 minutos
*/5 * * * * /ruta/al/proyecto/backup_rsync.sh

# rsync: todos los días a las 3 AM
0 3 * * * /ruta/al/proyecto/backup_rsync.sh

# Glacier: todos los domingos a las 4 AM
0 4 * * 0 /ruta/al/proyecto/backup_glacier.sh
```

Verificar que se ejecutó revisando **backup.log** al día siguiente:

```bash
root$ grep -E "Comenzando|FINALIZADO|INCOMPLETO" /ruta/al/proyecto/backup.log | tail
```

> Con Glacier, cada archivo se cobra mínimo 180 días: con un backup diario se acumulan al menos 180 copias de cada bucket. Para la mayoría de los casos alcanza con un backup semanal en Glacier y uno diario con rsync.

> Cada ejecución guarda **una copia completa** de cada bucket (no es incremental). Con buckets grandes tenerlo en cuenta para la frecuencia y el espacio en el destino.

---

## Descargar un backup

### Desde el servidor de destino (rsync)

```bash
$ # Listar los backups disponibles
$ ssh -p 22 backup_user@SERVIDOR_DESTINO "ls -lh /home/backup_user/backups/rustfs"

$ # Descargar uno a la carpeta restore/ del equipo actual
$ mkdir -p restore
$ rsync -av -e "ssh -p 22" backup_user@SERVIDOR_DESTINO:/home/backup_user/backups/rustfs/mi-bucket-20261021-0300.tar.gz restore/
```

### Desde AWS S3 Glacier Deep Archive

El usuario `rustfs-backup` no puede descargar (a propósito). Usar credenciales de un usuario con permisos `s3:GetObject` y `s3:RestoreObject` sobre el bucket (por ejemplo tu usuario administrador), configuradas en otro perfil:

```bash
$ aws configure --profile admin
```

**1. Buscar el backup**

```bash
$ aws s3 ls s3://mi-empresa-backups-rustfs/rustfs/ --recursive --profile admin
```

**2. Pedir la restauración** (sacarlo de Glacier temporalmente)

```bash
$ aws s3api restore-object \
    --bucket mi-empresa-backups-rustfs \
    --key rustfs/HOSTNAME/mi-bucket/mi-bucket-20261021-0400.tar.gz \
    --restore-request '{"Days":7,"GlacierJobParameters":{"Tier":"Standard"}}' \
    --profile admin
```

- `Tier`: `Standard` (hasta 12 hs) o `Bulk` (hasta 48 hs, más barato).
- `Days`: cuántos días queda disponible la copia restaurada para descargar.

**3. Esperar a que esté lista**

```bash
$ aws s3api head-object \
    --bucket mi-empresa-backups-rustfs \
    --key rustfs/HOSTNAME/mi-bucket/mi-bucket-20261021-0400.tar.gz \
    --profile admin
```

- `"Restore": "ongoing-request=\"true\""` → todavía se está restaurando.
- `"Restore": "ongoing-request=\"false\", expiry-date=..."` → listo para descargar.

**4. Descargar**

```bash
$ mkdir -p restore
$ aws s3 cp \
    s3://mi-empresa-backups-rustfs/rustfs/HOSTNAME/mi-bucket/mi-bucket-20261021-0400.tar.gz \
    restore/ --profile admin
```

_Si se intenta descargar antes de que termine la restauración, AWS responde con el error `InvalidObjectState`._

---

## Restaurar en una instancia nueva

Sirve tanto para un servidor nuevo como para el mismo servidor. Los comandos se ejecutan desde la carpeta del proyecto (con permisos de root o `sudo`, porque los archivos que genera el servicio `backup` quedan con dueño root).

**1. Levantar el servicio de RustFS**

Seguir [Configuración](README.md#configuración) e [Iniciar los servicios](README.md#iniciar-los-servicios) del README (clonar el proyecto, crear el _.env_, `docker compose up -d`).

**2. Descomprimir el backup**

```bash
$ mkdir -p restore/mi-bucket
$ tar -xzf restore/mi-bucket-20261021-0300.tar.gz -C restore/mi-bucket
```

**3. Crear el bucket**

Con el mismo nombre que espera la aplicación:

```bash
$ docker compose run --rm backup s3 mb s3://mi-bucket
```

_Si el bucket ya existe da error `BucketAlreadyOwnedByYou`; en ese caso seguir con el paso siguiente._

**4. Subir los objetos**

```bash
$ docker compose run --rm backup s3 sync /restore/mi-bucket s3://mi-bucket --only-show-errors
```

_La carpeta **restore/** del proyecto está montada en **/restore** dentro del servicio `backup`._

Si el bucket ya existía con datos, se pisan los objetos que están en el backup y los demás no se tocan. Para que quede exactamente igual al backup (borrando los objetos que no estén en él) agregar `--delete` al comando.

**5. Volver a configurar lo que no está en el backup**

Usuarios, access keys, políticas de acceso y la configuración del bucket (política pública, versionado, etc.), desde la consola web.

**6. Verificar**

```bash
$ docker compose run --rm backup s3 ls s3://mi-bucket --recursive --summarize
```

Comparar `Total Objects` y `Total Size` con el contenido del backup (`find restore/mi-bucket -type f | wc -l`).

**7. Borrar los backups descargados**

```bash
$ rm -rf restore/mi-bucket restore/mi-bucket-20261021-0300.tar.gz
```

> **Probar la restauración de vez en cuando.** Un backup que nunca se restauró no está probado. Cada algunos meses, restaurar un backup en una instancia de prueba siguiendo estos pasos.
