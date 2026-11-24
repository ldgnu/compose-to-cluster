# Backup y restore

## El principio

Un backup que nunca se restauró no es un backup: es una esperanza con nombre de archivo.
Este documento registra cómo se verificó que el backup existe, está completo y **restaura**.

## Qué se respalda

Un `tar.gz` completo de `/home/ubuntu/self-hosted` más las configuraciones del agente, generado
cada 6 horas por un script con estas características:

- `sudo -n`: falla en el acto si no hay permisos, en vez de colgarse pidiendo contraseña.
- Helper root acotado: `NOPASSWD` sobre un único binario, no sobre `tar` en general.
- Fallback degradado: si no hay permisos root, respalda lo legible y deja constancia de lo
  omitido en un archivo `backup-skipped-*.txt`, en lugar de morir en silencio.
- Exclusión de directorios pesados e irrelevantes del agente.

Tamaño típico: ~3.6 GB. Retención: dos tarballs.

## El problema del WAL

Vaultwarden usa SQLite en modo WAL. Las escrituras recientes viven en `db.sqlite3-wal`, no en
`db.sqlite3`. Un backup que copie solo la base principal pierde todo lo escrito desde el último
checkpoint —en este caso, tres semanas de datos—.

Verificación del contenido real del tarball:

```
self-hosted/vaultwarden/data/db.sqlite3
self-hosted/vaultwarden/data/db.sqlite3-wal
self-hosted/vaultwarden/data/db.sqlite3-shm
self-hosted/vaultwarden/data/rsa_key.pem
```

Los cuatro presentes. El backup es completo.

## Riesgo residual

`tar` no toma un snapshot atómico: lee `db.sqlite3` en un instante y el `-wal` milisegundos
después. Si el servicio escribe justo en el medio, la copia puede quedar incoherente. Con uso
personal la probabilidad es baja, pero no nula. La mitigación definitiva es `sqlite3 .backup` o
pausar el contenedor durante la copia.

## Verificación automática

Un drill corre cada ~15 días y valida en cadena:

1. Existe un tarball y no tiene más de 36 h.
2. Pesa más de 1 GB.
3. Contiene `db.sqlite3`, el WAL y `rsa_key.pem`.
4. La base abre y tiene usuarios y elementos.
5. La cantidad de elementos no cayó más de 10 % respecto del último drill.

No levanta contenedores ni usa TLS: valida por SQLite directamente. Si falla, notifica con el
motivo exacto. Si pasa, solo deja registro.

## El restore, probado

Procedimiento completo en `runbooks/restore-vaultwarden.md`.

Resultado de la prueba real: se extrajo la carpeta de datos del tarball, se levantó un
contenedor paralelo en un puerto alternativo y se ingresó con la contraseña maestra. Todas las
cajas fuertes, carpetas y elementos estaban ahí.

## Dos trampas que costaron tiempo

**1. Secure context.** Vaultwarden cifra en el cliente con WebCrypto, que solo funciona en
contextos seguros: `https://`, `http://localhost`, `http://127.0.0.1` o `file://`. Acceder por
`http://<ip>:puerto` falla con un aviso naranja, aunque el servidor responda.

**2. Validación del servidor.** Además del navegador, Vaultwarden rechaza orígenes no-HTTPS
del lado del servidor: `Insecure URL not allowed. All URLs must use HTTPS.` No alcanza con
`localhost`; hay que terminar el TLS en algún proxy.

Solución usada para la prueba: `tailscale serve --https` (certificado válido del tailnet).
En producción cumple ese rol el reverse proxy; en Kubernetes, el Ingress.

## Lo que falta

- Copia offsite cifrada del directorio de datos (son ~12 MB) a un segundo servidor.
- Automatizar esa copia con retención.
- Documentar el restore en un host nuevo, no solo en paralelo.
