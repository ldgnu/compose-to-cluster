# Compose to Cluster

**Proyecto personal de infraestructura:** migración documentada de un homelab de 34 servicios en
Docker a Kubernetes.

## En treinta segundos

Mantengo 34 servicios corriendo en un servidor propio: gestor de contraseñas, notas, lector de
RSS, mensajería, monitoreo y ocho aplicaciones que escribí. Todo lo administro yo, sobre un
servidor ARM de 2 vCPU.

Este repositorio documenta cómo los estoy migrando a Kubernetes sin romper nada: qué auditar
antes de mover, qué servicios se quedan y cuáles no, cómo garantizar que los datos sobrevivan, y
cómo probar que el backup sirve antes de confiar en él.

No es un tutorial ni una lista de comandos. Es el registro real de las decisiones, con lo que medí
y lo que me equivoqué.

## Resultados hasta ahora

- **34 servicios** inventariados y clasificados para migración, con sus dependencias de red.
- **23 GB de disco** liberados y **1,2 GB de RAM** recuperada, sin tocar un solo dato.
- **Backup y restore verificados** con una prueba real de recuperación del gestor de contraseñas.
- **Clúster k3s administrado desde cero**, con nodo `Ready` y cero reinicios.
- **Riesgos corregidos**: contenedores privilegiados, sockets de Docker expuestos y bases de
  datos en redes compartidas.
- **Decisiones documentadas** en ADRs y procedimientos operativos en runbooks.

## Stack

Docker · Docker Compose · Kubernetes (k3s) · containerd · Linux (Ubuntu ARM64) · redes de
contenedores · SQLite · Prometheus · Cloudflare Tunnel · Tailscale

## Estado por fase

| Fase | Qué hice | Estado |
|---|---|---|
| 0 | Auditar la infraestructura Docker | Completada |
| 0.5 | Verificar backup y restore del gestor de contraseñas | Completada |
| 1 | Limpiar el host | Completada |
| 2 | Instalar un clúster k3s single-node | Completada |
| 3 | Migrar el primer servicio | En curso |
| 4 | Servicios sin estado | Pendiente |
| 5 | Servicios con datos | Pendiente |
| 6 | El gestor de contraseñas, último | Pendiente |

## La regla que me puse

No migro por moda. Migro cuando Kubernetes resuelve un problema real o me enseña algo que vale
la pena. Lo que no cumple eso se queda en Docker, y lo justifico por escrito.

## Mapa del repositorio

- `docs/01-arquitectura-actual.md` — cómo era todo al empezar
- `docs/02-auditoria.md` — hallazgos, mediciones y clasificación de servicios
- `docs/03-backup-y-restore.md` — el sistema de backup y cómo lo verifiqué
- `docs/04-decisiones.md` — ADRs: qué decidí y por qué
- `docs/05-diario.md` — bitácora del trabajo
- `docs/06-como-contarlo.md` — cómo presentar este proyecto en una entrevista
- `docs/runbooks/` — procedimientos que probé

## Sobre los datos personales

El repo es público, así que reemplacé IPs, dominios y rutas personales por placeholders
(`<tailscale-ip>`, `ejemplo.com`, `<usuario>`). Los nombres de servicios quedan porque no son
sensibles y ayudan a entender el contexto.

## Licencia

Documentación bajo CC BY 4.0. El código de ejemplo, MIT.
