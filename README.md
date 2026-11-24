# Compose to Cluster

Migración progresiva de una infraestructura Docker/Compose a Kubernetes, documentada mientras se aprende.

Este repositorio no es un tutorial: es el registro real de un homelab que pasó de 34 contenedores
en un servidor ARM de 2 vCPU a un clúster k3s, con las decisiones, los errores y las mediciones
que justificaron cada paso.

## Estado

| Fase | Descripción | Estado |
|---|---|---|
| 0 | Auditoría de la infraestructura Docker | Completada |
| 0.5 | Backup y restore de Vaultwarden verificados | Completada |
| 1 | Limpieza del host (35 GB, contenedores basura) | Completada |
| 2 | Clúster k3s single-node | Completada |
| 3 | Primer servicio migrado | Pendiente |
| 4 | Servicios stateless | Pendiente |
| 5 | Servicios con datos | Pendiente |
| 6 | Vaultwarden (el crítico, al final) | Pendiente |

## El principio

No se migra por moda. Se migra cuando Kubernetes resuelve un problema o representa una
oportunidad de aprendizaje justificada. Todo lo que no cumple eso se queda donde está.

## Mapa del repositorio

- `docs/01-arquitectura-actual.md` — cómo era la infraestructura al empezar
- `docs/02-auditoria.md` — hallazgos, mediciones y clasificación de servicios
- `docs/03-backup-y-restore.md` — el sistema de backup y cómo se verificó
- `docs/04-decisiones.md` — ADRs: qué se decidió y por qué
- `docs/05-diario.md` — bitácora cronológica del trabajo
- `docs/runbooks/` — procedimientos operativos

## Convenciones de publicación

Este repo es público. Por eso:

- Las IPs, dominios y rutas personales están reemplazadas por placeholders (`<tailscale-ip>`,
  `ejemplo.com`, `<usuario>`).
- No se publican tokens, claves, `.env` ni capturas con datos reales.
- Los nombres de servicios se mantienen porque no son sensibles y ayudan a entender el contexto.

## Licencia

Documentación bajo CC BY 4.0. El código de ejemplo, MIT.
