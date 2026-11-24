# Runbook — Restore de Vaultwarden

Procedimiento verificado en 2026-11-23. Es el restore completo, para cuando el servicio está
caído o se migra de host. Para la verificación periódica automática existe un drill que valida
sin levantar contenedores.

## Prerrequisitos

- Acceso SSH al servidor
- Contraseña maestra del vault
- `sqlite3` instalado
- Un tarball de backup accesible

## 1. Elegir el backup

Listar los tarballs disponibles y elegir el más reciente **anterior al incidente**. Si el
problema fue corrupción de datos, el último no sirve: hay que ir uno o dos atrás.

## 2. Extraer solo lo necesario

Extraer únicamente la carpeta del servicio desde el tarball completo, usando `--no-same-owner`
para evitar errores de ownership al extraer como root.

Se extrae solo `self-hosted/vaultwarden/data`: el tarball pesa ~3.6 GB y el dato son 12 MB.

## 3. Verificar el contenido antes de levantar nada

Deben estar: `db.sqlite3`, `db.sqlite3-wal`, `db.sqlite3-shm`, `rsa_key.pem`.

Sin el `-wal`, la base tiene datos hasta el último checkpoint, que puede ser semanas atrás.

Validación rápida sin contenedor:

```bash
sqlite3 <ruta>/db.sqlite3 "select count(*) from ciphers; select email from users;"
```

## 4. Levantar el contenedor en un puerto alternativo

```bash
docker run -d --name vw-restore-test \
  -p 127.0.0.1:8222:80 \
  -v /tmp/restore-test/self-hosted/vaultwarden/data:/data \
  vaultwarden/server:latest
```

Sin `--restart`, sin red compartida, sin token de administración, publicado solo en loopback.

## 5. Exponer con TLS

Vaultwarden rechaza orígenes no-HTTPS del lado del servidor, y el navegador exige contexto
seguro para WebCrypto. Dos opciones:

**A. Túnel SSH** (localhost cuenta como contexto seguro):

```bash
ssh -N -L 8222:127.0.0.1:8222 <usuario>@<host>
# abrir http://localhost:8222
```

**B. TLS real con Tailscale:**

```bash
tailscale serve --bg --https=8443 http://127.0.0.1:8222
# abrir https://<host>.<tailnet>.ts.net:8443
```

## 6. Verificar

- [ ] Login con contraseña maestra
- [ ] Cantidad de elementos coincide
- [ ] Carpetas y organización presentes
- [ ] `rsa_key.pem` en su lugar

## 7. Reemplazar el servicio real

Detener el contenedor de prueba, volver a levantar el servicio con su stack habitual apuntando
a los datos restaurados, y verificar el acceso por el dominio normal.

## 8. Limpiar

```bash
docker rm -f vw-restore-test
sudo rm -rf /tmp/restore-test
tailscale serve --https=8443 off
```

## Trampas conocidas

| Síntoma | Causa | Solución |
|---|---|---|
| Aviso naranja de *secure context* | WebCrypto sin HTTPS ni localhost | túnel SSH o Tailscale serve |
| `Insecure URL not allowed` | validación server-side del origen | HTTPS real |
| Login OK pero faltan datos recientes | se copió la base sin el WAL | restaurar con el WAL |
| `database disk image is malformed` | snapshot inconsistente | probar el tarball anterior |
| Reinicio en loop con `permission denied` | ownership incompatible | ajustar permisos al uid del contenedor |
