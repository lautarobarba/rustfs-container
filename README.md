# RustFS

RustFS es un servidor de almacenamiento de objetos escrito en Rust y compatible con la API de Amazon S3.

## Descripción

Permite almacenar y recuperar archivos (objetos) mediante la API S3. Incluye una consola web para administrar buckets y objetos de forma visual.

## Configuración

### Variables de entorno

Crea un archivo `.env` a partir de `.env.example`:

```bash
RUSTFS_WEB_PORT=5002
RUSTFS_SERVER_PORT=9002

RUSTFS_ACCESS_KEY=rustfsadmin
RUSTFS_SECRET_KEY=rustfsadmin
```

> Cambia `RUSTFS_ACCESS_KEY` y `RUSTFS_SECRET_KEY` antes de exponer el servicio en una red.

### Puertos

- **9002**: API S3 (puerto 9000 dentro del contenedor)
- **5002**: Consola web (puerto 9001 dentro del contenedor)

### Permisos

El contenedor corre como el usuario `rustfs` (UID/GID 10001). El servicio `rustfs-perms` se ejecuta antes de `rustfs` y asigna la propiedad de `./data` a ese usuario.

## Uso

### Iniciar los servicios

```bash
docker compose up -d
```

### Acceder a la consola web

Abre tu navegador en: `http://localhost:5002/rustfs/console/`

Inicia sesión con las credenciales definidas en `.env` (`RUSTFS_ACCESS_KEY` y `RUSTFS_SECRET_KEY`).

### Configurar tu aplicación

Configura tu aplicación para usar RustFS como almacenamiento S3 con los siguientes parámetros:

- **Endpoint**: `http://localhost:9002`
- **Access Key**: valor de `RUSTFS_ACCESS_KEY`
- **Secret Key**: valor de `RUSTFS_SECRET_KEY`
- **Region**: (cualquier valor, ej. `us-east-1`)
- **Path style**: habilitado

### API S3

Comprobar el estado del servidor:

```bash
curl http://localhost:9002/health
```

Listar buckets:

```bash
curl http://localhost:9002 \
  --aws-sigv4 "aws:amz:us-east-1:s3" \
  -u "rustfsadmin:rustfsadmin"
```

### Cliente mc (MinIO Client)

RustFS es compatible con `mc`:

```bash
# Agregar el servidor como alias
mc alias set rustfs http://localhost:9002 rustfsadmin rustfsadmin

# Crear un bucket
mc mb rustfs/mi-bucket

# Subir un archivo
mc cp archivo.txt rustfs/mi-bucket/

# Listar objetos
mc ls rustfs/mi-bucket
```

### Detener los servicios

```bash
docker compose down
```

## Backups

Los backups se hacen **por bucket**: se descarga el contenido completo del bucket y se comprime en un `.tar.gz`. Los archivos se guardan en **/backups**.

Para backups automáticos (a otro servidor con rsync o a AWS S3 Glacier Deep Archive), cómo descargarlos y cómo restaurarlos en una instancia nueva, ver **[BACKUPS.md](BACKUPS.md)**.

### Cliente S3 (servicio `backup`)

_compose.yaml_ incluye el servicio `backup` con `aws-cli`, ya configurado con las credenciales del _.env_ y conectado a RustFS. No se inicia con `docker compose up`; se usa con `docker compose run`:

```bash
# Listar buckets
docker compose run --rm backup s3 ls

# Listar objetos de un bucket
docker compose run --rm backup s3 ls s3://mi-bucket --recursive
```

### Backup manual

Reemplazar `mi-bucket` por el bucket a respaldar (con `sudo`, porque los archivos descargados quedan con dueño root):

```bash
docker compose run --rm backup s3 sync s3://mi-bucket /backups/mi-bucket --only-show-errors
sudo tar -czf mi-bucket-$(date +%Y%m%d-%H%M).tar.gz -C backups/mi-bucket .
sudo rm -rf backups/mi-bucket
```

### Restaurar backup

Ver [Restaurar en una instancia nueva](BACKUPS.md#restaurar-en-una-instancia-nueva).
