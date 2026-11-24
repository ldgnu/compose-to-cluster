# Arquitectura actual (punto de partida)

Registro del estado de la infraestructura antes de cualquier migración. Fecha: 2026-11-23.

## Hardware

Un servidor único en un proveedor cloud, instancia ARM gratuita:

| Recurso | Valor |
|---|---|
| Arquitectura | aarch64 (ARM64) |
| vCPU | 2 |
| RAM | ~12 GB |
| Swap | 2 GB en archivo |
| Disco | 145 GB, ext4, volumen único |
| SO | Ubuntu 26.04 LTS |
| Kernel | 7.x |
| cgroups | v2 |
| Docker | 29.x, cgroup driver systemd, overlayfs |

Características relevantes para Kubernetes:

- **Un solo disco**: el SO, las imágenes, los volúmenes y los datos de usuario comparten `sda1`.
- **Swap activo**: bloqueante por defecto para kubelet.
- **Sin capacidad de ampliación**: el tier gratuito del proveedor no permite más CPU/RAM.

## Servicios

34 contenedores Docker en ejecución, agrupados por función:

| Capa | Servicios |
|---|---|
| Borde / red | Nginx Proxy Manager, Cloudflare Tunnel, AdGuard Home (host), Tailscale |
| Identidad | Vaultwarden (crítico), un proxy de LLMs |
| Datos / productividad | Trilium, Linkwarden + Postgres, FreshRSS, Excalidraw, Homepage |
| Mensajería | Evolution API + Postgres + Redis |
| Casa / IoT | Home Assistant |
| Apps propias | 8 aplicaciones propias (Go, Node, Python) |
| Observabilidad | Prometheus, Grafana, Alertmanager, Blackbox, node-exporter, cAdvisor |
| Operación | Watchtower, un contenedor interactivo abandonado |

## Topología de red

```
Internet
   │
   ▼
Cloudflare Tunnel (outbound-only)
   │
   ▼
nginx-proxy :443 ──► red "proxy" (22 contenedores)
                       ├── apps stateless
                       ├── linkwarden-db  ← base de datos en red compartida
                       └── grafana / alertmanager
```

Hallazgos estructurales:

- **No hay puertos web publicados en el host.** Todo entra por el túnel de Cloudflare. Bien.
- **La red `proxy` es plana**: 22 contenedores se ven entre sí sin restricciones.
- **Dos excepciones publican en `0.0.0.0`** (saltando el ingress): la API de mensajería y un juego web.
- **Dos redes se llaman "internal" pero no lo son** (`Internal: false`): el nombre miente.
- **Postgres de Linkwarden vive en la red `proxy`**, alcanzable por 22 vecinos.

## Persistencia

Dos patrones conviviendo:

1. **Bind mounts** a `/home/ubuntu/self-hosted/<servicio>/data` — 20 servicios.
2. **Volúmenes nombrados de Docker** — 4 servicios.

Consecuencia: los datos son visibles y respaldables con `tar`, pero no hay separación entre
"dato de aplicación" y "home del usuario". En Kubernetes cada uno se convierte en un PVC o un
hostPath, y hay que decidir cuál es cuál.

## Observabilidad

Prometheus en el host principal recolecta métricas de un segundo servidor (más chico) a través
de Tailscale. El colector vive en el mismo host que vigila: si el host cae, el monitoreo cae con él.

## Consumo de recursos medido

| Componente | RAM |
|---|---|
| Contenedores (32) | ~3 GB |
| Stack de agentes (Hermes + opencode) | ~3 GB |
| Clúster k3s (previo, no documentado) | 432 MB |

## El hallazgo que cambió el plan

Durante la auditoría se descubrió un **clúster k3s corriendo desde un mes antes**, instalado
sin documentación, con workloads de un tutorial abandonado y componentes del sistema con
cientos de reinicios por falta de memoria. Ese hallazgo reordenó el proyecto: no había que
instalar Kubernetes, había que entender qué había y empezar de nuevo con criterio.

Ver `02-auditoria.md`.
