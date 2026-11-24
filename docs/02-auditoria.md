# Auditoría

Qué medí, qué encontré y cómo clasifiqué cada servicio. Fecha: 2026-11-23.

## Cómo lo hice

Todo de solo lectura. Inventario con `docker ps`, `docker inspect`, `docker system df` y
`docker network inspect`, más consumo real con `docker stats` y `ps`.

Una precaución desde el principio: `docker inspect` imprime todas las variables de entorno con
sus valores. Así que filtré la salida para extraer **solo los nombres** de las variables, nunca
los valores. Lo mismo con los archivos de Compose: `docker compose config` interpola y expande
los secretos, así que siempre lo corrí con `--no-interpolate`.

## Números

| Métrica | Antes de limpiar |
|---|---|
| Contenedores | 34 (32 activos) |
| Imágenes | 90 (32 sin tag) |
| Disco usado | 110 GB de 145 |
| Cache de build | 13.1 GB |
| Volúmenes con datos | 3.98 GB |
| Swap en uso | 1.8 GB de 2 |
| Red más poblada | `proxy`, 22 contenedores |

## Lo que encontré, ordenado por gravedad

1. **El host ya estaba saturado.** 7.1 GB de 12 en uso y 1.8 GB de swap activo. Estaba paginando
   antes de que yo empezara a migrar nada.
2. **Un solo disco.** Sistema, imágenes, volúmenes y datos compartiendo `sda1`.
3. **No podía reconstruir 11 contenedores.** Corrían con ID de imagen en vez de tag; la imagen
   original ya no existía en ningún registry.
4. **Red plana.** 22 contenedores en `proxy` sin segmentación, con una base de datos entre ellos.
5. **`watchtower` con el socket de Docker en escritura.** Control total del daemon, combinado con
   tags `:latest` y auto-actualización. Es la superficie de ataque más grande que tenía.
6. **`cadvisor` privilegiado usando 1.2 GB de RAM.** El 40 % del consumo de todos los
   contenedores, para una función que kubelet da de fábrica.
7. **15 servicios sin healthcheck.** En Kubernetes, "pod Running" no significa "pod Ready".
8. **Dos contenedores basura.** Un `sleep infinity` en red host con 837 MB escritos, y una
   instancia vieja de una app que ya corría en otro lado.
9. **Tres fuentes de verdad para los compose.** Un catálogo de ~65 stacks, el árbol de producción
   con 7 repos git propios, y un directorio de stacks parciales. Dos stacks de monitoring
   corriendo en paralelo.
10. **El DNS estaba bien.** AdGuard escuchaba solo en la interfaz de Tailscale; el host resolvía
    por el DNS del proveedor. No había dependencia oculta.

## Cómo clasifiqué cada servicio

| Veredicto | Qué incluí | Por qué |
|---|---|---|
| **REMOVE** | contenedor abandonado, duplicado, red huérfana, imágenes sin tag | No cumplen función |
| **REPLACE** | cAdvisor, node-exporter en Docker, watchtower, monitoring artesanal, NPM | La función sirve, la implementación no |
| **KEEP fuera del clúster** | AdGuard (puerto 53), Home Assistant (red local), NPM en la transición | Acoplados al host o a la red local |
| **MIGRATE temprano** | Excalidraw, sitio estático, mis apps sin estado, FreshRSS | Sin estado o mínimo |
| **MIGRATE tarde** | Vaultwarden, Linkwarden + DB, Evolution API + PG + Redis, monitoring | Datos o criticidad |
| **INVESTIGATE** | tres servicios de origen incierto, el repo de catálogo | Me faltaba información |

## La limpieza

| Acción | Resultado |
|---|---|
| Borré dos contenedores basura | RAM e imágenes |
| `docker image prune` | 1.5 GB |
| `docker builder prune -a` | ~11 GB |
| `docker network prune` | una red huérfana |
| Borré cAdvisor | 1.2 GB de RAM |
| Desinstalé el k3s viejo | estado del clúster e imágenes |

Pasé de 35 GB libres a **58 GB**. El disco dejó de ser el límite.

## El error que me llevé

Estimé que iba a recuperar unos 35 GB y recuperé 13. ¿Por qué? Sumé los tamaños que muestra
`docker image ls`, y ese número **incluye las capas compartidas**. La misma capa se cuenta en
varias imágenes, así que la suma infla el total. Lo que `prune` libera es solo lo que no
referencia nadie.

Moraleja: se mide con `docker system df`, no sumando listados.

## Lo que la limpieza no arregló

La memoria. Mis contenedores sumaban 3 GB, pero el host usaba 7.1. El resto era el stack de
agentes que corro encima, que consume tanto como toda la infraestructura Docker junta. Ese es mi
verdadero competidor por RAM, y define cuánta carga puede sostener el clúster.
