# Auditoría

Qué se midió, qué se encontró y cómo se clasificó cada servicio. Fecha: 2026-11-23.

## Método

Solo lectura. Comandos de inventario (`docker ps`, `docker inspect`, `docker system df`,
`docker network inspect`), más medición de consumo con `docker stats` y `ps`. Nunca se
publicaron valores de variables de entorno: la inspección extrajo **solo nombres de variables**,
nunca valores.

## Números

| Métrica | Antes de limpiar |
|---|---|
| Contenedores | 34 (32 activos) |
| Imágenes | 90 (32 dangling) |
| Disco usado | 110 GB de 145 |
| Build cache | 13.1 GB |
| Volúmenes con datos | 3.98 GB |
| Swap en uso | 1.8 GB de 2 |
| Red más poblada | `proxy`, 22 contenedores |

## Hallazgos, por gravedad

1. **Host saturado**: 7.1 GB de 12 en uso, con 1.8 GB de swap activo. El host ya paginaba
   antes de cualquier migración.
2. **Disco único**: SO, imágenes, volúmenes y datos comparten `sda1`.
3. **Reproducibilidad rota**: 11 contenedores corrían con ID de imagen en vez de tag; la imagen
   original ya no era resoluble desde ningún registry.
4. **Red plana**: 22 contenedores en `proxy` sin segmentación; una base de datos entre ellos.
5. **`watchtower` con `/var/run/docker.sock` en escritura**: control total del daemon Docker,
   combinado con tags `:latest` y auto-actualización.
6. **`cadvisor` privilegiado consumiendo 1.2 GB de RAM** — el 40 % del consumo de contenedores,
   para una función que kubelet provee de fábrica.
7. **15 servicios sin healthcheck**: en Kubernetes "pod Running" no implica "pod Ready".
8. **Dos contenedores basura**: un `sleep infinity` en red host con 837 MB escritos y una
   instancia vieja de una app duplicada.
9. **Tres fuentes de verdad para los compose**: un catálogo de ~65 stacks, el árbol de producción
   con 7 repos git propios, y un directorio de stacks parciales. Dos stacks de monitoring
   corriendo en paralelo.
10. **DNS**: AdGuard escuchaba solo en la interfaz de Tailscale; el host resolvía por el DNS del
    proveedor. No había dependencia crítica oculta.

## Clasificación de servicios

| Veredicto | Servicios | Criterio |
|---|---|---|
| **REMOVE** | contenedor interactivo abandonado, duplicado en red default, red huérfana, imágenes dangling | Sin función |
| **REPLACE** | cAdvisor, node-exporter en Docker, watchtower, monitoring artesanal, NPM | La función sobrevive, la implementación cambia |
| **KEEP fuera del clúster** | AdGuard (puerto 53), Home Assistant (red local), NPM durante la transición | Acoplamiento al host o a la red local |
| **MIGRATE temprano** | Excalidraw, sitio estático, apps propias stateless, FreshRSS | Sin estado o mínimo |
| **MIGRATE tarde** | Vaultwarden, Linkwarden + DB, Evolution API + PG + Redis, monitoring | Datos persistentes o criticidad |
| **INVESTIGATE** | tres servicios de origen incierto, el repo de catálogo | Falta información |

## Limpieza ejecutada

| Acción | Recuperado |
|---|---|
| `docker rm` de dos contenedores basura | RAM + imágenes |
| `docker image prune` | 1.5 GB |
| `docker builder prune -a` | ~11 GB |
| `docker network prune` | red huérfana |
| `docker rm cadvisor` | 1.2 GB de RAM |
| Desinstalación del k3s previo | estado del clúster + imágenes |

Resultado: de 35 GB libres a **58 GB libres**. El disco dejó de ser el límite.

## La lección del cálculo

La primera estimación de espacio recuperable fue de ~35 GB y el resultado real fue ~13 GB.
La causa: `docker image ls` reporta el tamaño de cada imagen **incluyendo capas compartidas**.
Sumar esos números cuenta la misma capa varias veces. Lo que `prune` libera es solo lo que no
referencia nadie. Lección: medir con `docker system df`, no sumando listados.

## Lo que no se pudo resolver con limpieza

La memoria. Los contenedores sumaban 3 GB, pero el host usaba 7.1. El resto era el stack de
agentes (Hermes + opencode), que consume tanto como toda la infraestructura Docker. Ese es el
verdadero competidor por RAM, y condiciona cuánta carga puede sostener el clúster.
