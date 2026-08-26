<p align="center">
  <a href="https://supabase.com">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/supabase/supabase/master/packages/common/assets/images/supabase-logo-wordmark--dark.svg">
      <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/supabase/supabase/master/packages/common/assets/images/supabase-logo-wordmark--light.svg">
      <img src="https://raw.githubusercontent.com/supabase/supabase/master/packages/common/assets/images/logo-preview.jpg" alt="Supabase" width="280">
    </picture>
  </a>
</p>

<h1 align="center">Despliegue autoalojado de Supabase</h1>

<p align="center">supabase.erp.spin-suite.com</p>

El despliegue se divide en dos proyectos independientes de Docker Compose:

- `supabase/`: stack oficial autoalojado de Supabase, configurado para `supabase.erp.spin-suite.com`.
- `nginx/`: Nginx Proxy Manager y su base de datos MariaDB, basado en el compose de referencia.

## DNS y configuración inicial

Crear un registro `A` para `supabase.erp.spin-suite.com` con la IPv4 pública de este servidor. Si el servidor tiene IPv6 pública, crear también un `AAAA`. El dominio debe resolver antes de solicitar el certificado de Let's Encrypt.

Iniciar primero Supabase; crea la red Docker compartida que usa el proxy:

```sh
cd supabase
docker compose up -d
```

Después iniciar Nginx Proxy Manager:

```sh
cd ../nginx
docker compose up -d
```

Abrir `http://IP_DEL_SERVIDOR:81`, completar la cuenta inicial de Nginx Proxy Manager y crear un **Proxy Host** con:

- Nombre de dominio: `supabase.erp.spin-suite.com`
- Esquema: `http`
- Host/IP de destino: `supabase-envoy`
- Puerto de destino: `8000`
- Soporte de WebSockets: activado
- SSL: solicitar un certificado Let's Encrypt y activar **Force SSL**

Los puertos 80 y 443 deben ser accesibles desde Internet. El proxy es el único punto de entrada público; no exponer puertos de base de datos de Supabase.

## Respaldos

Realizar un respaldo antes de actualizar y de forma periódica. No copiar `volumes/db/data` mientras Postgres está activo: un respaldo físico en caliente puede quedar inconsistente. En su lugar, crear un respaldo lógico de la base de datos y archivar los archivos persistentes.

Con Supabase iniciado:

```sh
cd /opt/development/github/supabase-deployment/supabase

umask 077
BACKUP_DIR="/var/backups/supabase/$(date +%F_%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Base de datos completa: roles, esquemas y datos
docker exec supabase-db pg_dumpall -h localhost -U supabase_admin \
  > "$BACKUP_DIR/postgres.sql"

# Clave requerida para recuperar secretos de Vault/pgsodium
docker compose run --rm db cat /etc/postgresql-custom/pgsodium_root.key \
  > "$BACKUP_DIR/pgsodium_root.key"

# Archivos persistentes: Storage, Edge Functions y snippets
tar -C volumes -czf "$BACKUP_DIR/storage-functions-snippets.tar.gz" \
  storage functions snippets

# Configuración y verificación de integridad
cp .env "$BACKUP_DIR/supabase.env"
sha256sum "$BACKUP_DIR"/* > "$BACKUP_DIR/SHA256SUMS"
```

Copiar el directorio generado a un almacenamiento externo o a otro servidor. Un respaldo que permanece únicamente en el mismo VPS no protege ante pérdida de disco o del servidor.

El archivo `postgres.sql` es el respaldo principal de la base de datos. Conservar también `pgsodium_root.key`; perderlo puede impedir recuperar secretos cifrados. Para hacer una copia física de `volumes/db/data`, detener Supabase antes de archivarla.

## Actualizar Supabase

La configuración y las imágenes de Supabase se actualizan como una versión conjunta. No se recomienda cambiar una única etiqueta de imagen salvo que Supabase documente explícitamente su compatibilidad.

Antes de la primera actualización, confirmar que la versión base está registrada:

```sh
cd supabase
cat .supabase-version
# ref=self-hosted/v0.8.0
```

### Procedimiento seguro

1. Respaldar la base de datos y los archivos de `supabase/volumes/`.
2. Consultar los cambios sin modificar archivos:

   ```sh
   cd supabase
   sh update.sh --dry-run
   ```

3. Revisar las notas de versión y ejecutar la actualización:

   ```sh
   sh update.sh
   ```

   El script realiza una fusión de tres vías desde la versión base, conserva los valores existentes en `.env` y no modifica los datos de Postgres ni Storage. Si presenta conflictos o cambios incompatibles, resolverlos antes de continuar.

4. Descargar las imágenes y recrear los servicios:

   ```sh
   sh run.sh pull
   sh run.sh recreate
   ```

5. Confirmar el estado y revisar los logs:

   ```sh
   sh run.sh status
   sh run.sh logs
   ```

Para fijar una versión concreta en vez de actualizar a la última disponible:

```sh
sh update.sh --to self-hosted/vX.Y.Z
```

`reset.sh` no es un comando de actualización: elimina contenedores y datos persistentes. No usarlo para actualizar.

`supabase/.env` y `nginx/.env` contienen secretos y están excluidos intencionalmente de Git.
