# Mi arquitectura antes de empezar

Foto del estado de mi infraestructura antes de tocar nada. Fecha: 2026-11-23.

## Hardware

Un único servidor en un proveedor cloud, instancia ARM del tier gratuito:

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

Lo que me importa de esta tabla para Kubernetes:

- **Un solo disco.** El sistema, las imágenes, los volúmenes y mis datos comparten `sda1`.
- **Swap activo.** kubelet lo rechaza por defecto.
- **No puedo ampliar.** Mi tier gratuito no permite más CPU ni RAM.

## Servicios

34 contenedores Docker, agrupados por función:

| Capa | Servicios |
|---|---|
| Borde / red | Nginx Proxy Manager, Cloudflare Tunnel, AdGuard Home (host), Tailscale |
| Identidad | Vaultwarden (mi dato crítico), un proxy de LLMs |
| Datos / productividad | Trilium, Linkwarden + Postgres, FreshRSS, Excalidraw, Homepage |
| Mensajería | Evolution API + Postgres + Redis |
| Casa / IoT | Home Assistant |
| Apps propias | 8 aplicaciones mías (Go, Node, Python) |
| Observabilidad | Prometheus, Grafana, Alertmanager, Blackbox, node-exporter, cAdvisor |
| Operación | Watchtower, un contenedor interactivo que dejé olvidado |

## Cómo entra el tráfico

```
Internet
   │
   ▼
Cloudflare Tunnel (solo salida)
   │
   ▼
nginx-proxy :443 ──► red "proxy" (22 contenedores)
                       ├── apps sin estado
                       ├── linkwarden-db  ← base de datos en red compartida
                       └── grafana / alertmanager
```

Lo que veo cuando lo dibujo:

- **No tengo puertos web abiertos en el host.** Todo entra por el túnel. Eso lo hice bien.
- **La red `proxy` es plana.** 22 contenedores se ven entre sí sin restricciones.
- **Dos excepciones publican en `0.0.0.0`** y se saltan el proxy: la API de mensajería y un juego.
- **Dos redes se llaman "internal" y no lo son.** El nombre miente.
- **El Postgres de Linkwarden vive en `proxy`**, alcanzable por 22 vecinos.

## Dónde viven los datos

Dos patrones conviviendo, sin un criterio claro:

1. **Bind mounts** a `/home/ubuntu/self-hosted/<servicio>/data` — 20 servicios.
2. **Volúmenes nombrados de Docker** — 4 servicios.

Es cómodo para respaldar con `tar`, pero mezcla "dato de aplicación" con "mi home". En Kubernetes
cada uno va a ser un PVC o un hostPath, y voy a tener que decidir cuál es cuál.

## Monitoreo

Prometheus corre en el servidor principal y recolecta métricas de un segundo servidor más chico,
a través de Tailscale. El que vigila vive en el mismo lugar que lo vigilado: si el principal cae,
el monitoreo cae con él.

## Consumo real

| Componente | RAM |
|---|---|
| Mis 32 contenedores | ~3 GB |
| Stack de agentes (Hermes + opencode) | ~3 GB |
| Clúster k3s que no sabía que tenía | 432 MB |

## El hallazgo que me cambió el plan

Mientras auditaba descubrí que **tenía un clúster k3s corriendo desde hacía un mes**. Lo instalé
sin documentarlo, dejé workloads de un tutorial y los componentes del sistema se venían
reiniciando solos por falta de memoria. Ese descubrimiento reordenó todo: no tenía que instalar
Kubernetes, tenía que entender qué había y empezar de nuevo con criterio.

Lo cuento en `02-auditoria.md`.
