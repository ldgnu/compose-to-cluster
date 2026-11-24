# Backup y restore

## La regla que adopté

Un backup que nunca restauré no es un backup: es una esperanza con nombre de archivo.
Antes de migrar cualquier cosa con datos, quise probar que podía volver atrás.

## Qué respaldo

Un `tar.gz` completo de mis servicios más las configuraciones del agente, generado cada 6 horas.
El script tiene tres decisiones que me gustan:

- **`sudo -n`**: falla en el acto si no hay permisos, en vez de quedarse colgado pidiendo
  contraseña.
- **Helper root acotado**: `NOPASSWD` sobre un único binario, no sobre `tar` en general.
- **Fallback degradado**: si no hay permisos root, respalda lo que puede leer y deja constancia
  de lo omitido en un archivo aparte, en vez de morir en silencio.

Tamaño típico: ~3.6 GB. Retención: dos tarballs.

## El problema que casi me costó caro

Vaultwarden usa SQLite en modo WAL. Las escrituras recientes viven en `db.sqlite3-wal`, no en
`db.sqlite3`. Si el backup copia solo la base principal, pierde todo lo escrito desde el último
checkpoint.

En mi caso, `db.sqlite3` no se modificaba desde hacía **tres semanas**. Los datos reales estaban
en el WAL. Si el script lo hubiera ignorado, mis backups habrían tenido tres semanas de atraso
sin que yo lo supiera.

Revisé el contenido real del tarball y estaban los cuatro archivos:

```
self-hosted/vaultwarden/data/db.sqlite3
self-hosted/vaultwarden/data/db.sqlite3-wal
self-hosted/vaultwarden/data/db.sqlite3-shm
self-hosted/vaultwarden/data/rsa_key.pem
```

Completo. Pero lo confirmé mirando, no suponiendo.

## El riesgo que queda

`tar` no toma un snapshot atómico: lee la base en un instante y el WAL milisegundos después. Si
el servicio escribe justo en el medio, la copia puede quedar incoherente. Con uso personal la
probabilidad es baja, pero existe. La solución definitiva es `sqlite3 .backup` o pausar el
contenedor durante la copia.

## Verificación automática

Armé un drill que corre cada ~15 días y valida en cadena:

1. Existe un tarball y no tiene más de 36 horas.
2. Pesa más de 1 GB.
3. Contiene la base, el WAL y la clave RSA.
4. La base abre y tiene usuarios y elementos.
5. La cantidad de elementos no cayó más de 10 % respecto del último drill.

No levanta contenedores ni usa TLS: valida por SQLite directamente. Si falla, me manda un mensaje
con el motivo exacto. Si pasa, solo deja registro. No quiero ruido cuando todo está bien.

## El restore, probado

Procedimiento completo en `runbooks/restore-vaultwarden.md`.

Extraje la carpeta de datos del tarball, levanté un contenedor paralelo en un puerto alternativo
y entré con mi contraseña maestra. Estaban todas las cajas fuertes, las carpetas y los elementos.

Recién ahí consideré que el backup servía.

## Dos trampas que me costaron tiempo

**1. Contexto seguro.** Vaultwarden cifra en el cliente con WebCrypto, que solo funciona en
contextos seguros: `https://`, `http://localhost`, `http://127.0.0.1` o `file://`. Entrar por
`http://<ip>:puerto` muestra un aviso naranja aunque el servidor responda. `localhost` cuenta
como seguro; una IP cruda no.

**2. Validación del servidor.** Además del navegador, Vaultwarden rechaza orígenes no-HTTPS del
lado del servidor: `Insecure URL not allowed. All URLs must use HTTPS.` No alcanza con localhost;
hay que terminar el TLS en algún proxy.

Para la prueba usé `tailscale serve --https`, que me dio un certificado válido del tailnet. En
producción ese rol lo cumple el reverse proxy; en Kubernetes, el Ingress.

## Lo que me falta

- Copia offsite cifrada del directorio de datos (son ~12 MB) a un segundo servidor.
- Automatizar esa copia con retención.
- Documentar el restore en un host nuevo, no solo en paralelo.
