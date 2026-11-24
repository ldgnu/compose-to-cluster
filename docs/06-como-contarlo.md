# Cómo contar este proyecto

Notas para mí, para cuando tenga que presentarlo. Sirve para un CV, para LinkedIn y para una
entrevista técnica.

## La etiqueta correcta

Es un **proyecto personal de infraestructura / homelab**, no experiencia laboral. Presentarlo
como trabajo profesional es el error más caro: se cae en la primera pregunta de seguimiento.

Lo que sí es cierto y verificable:

- Infraestructura real, con servicios que uso todos los días y datos que no puedo perder.
- Decisiones tomadas con criterio documentado (ADRs), no copiadas de un tutorial.
- Mediciones concretas antes y después.
- Un restore probado de punta a punta.

Eso, en un candidato junior o de perfil generalista, vale más que la lista de tecnologías.

## Versión CV (bullets)

Para la sección de proyectos personales:

> **Compose to Cluster** — Migración de un homelab de 34 servicios en Docker Compose a Kubernetes.
> - Audité la infraestructura completa: inventario de 34 contenedores, mapeo de dependencias de
>   red y clasificación de cada servicio para migración (migrar / mantener / reemplazar / eliminar).
> - Identifiqué y corregí riesgos de seguridad y operación: contenedores privilegiados, sockets
>   de Docker con escritura, bases de datos en redes compartidas y servicios fuera del ingress.
> - Reduje el uso de disco en 23 GB y liberé 1.2 GB de RAM eliminando imágenes huérfanas, cache
>   de build y un agente de monitoreo redundante.
> - Desplegué un clúster k3s single-node desde cero, tras detectar y resetear un clúster previo
>   inestable con componentes reiniciándose por falta de memoria.
> - Implementé verificación automática de backup y restore (drill con baseline y alertas),
>   validada con un restore real en un contenedor aislado.
> - Documenté decisiones de arquitectura en ADRs y procedimientos operativos en runbooks.

**Tecnologías:** Docker, Docker Compose, Kubernetes (k3s), containerd, Linux (Ubuntu ARM64),
redes de contenedores, SQLite, Prometheus, Cloudflare Tunnel, Tailscale.

## Versión de una línea (LinkedIn)

> Migré mi homelab de 34 contenedores Docker a Kubernetes documentando cada decisión: auditoría,
> clasificación de servicios, backup verificado con restore real y un clúster k3s administrado
> desde cero.

## Qué NO decir

- ❌ "Migré toda mi infraestructura a Kubernetes." → Todavía no. Está en curso.
- ❌ "Tengo experiencia en Kubernetes en producción." → No. Es un homelab.
- ❌ "Administro un clúster en alta disponibilidad." → Es un solo nodo.
- ❌ "Implementé GitOps / CI-CD / observabilidad con Prometheus Operador." → No todavía.
- ❌ "Manejo Kubernetes a nivel experto." → Sé lo que hice y por qué; el resto lo estoy aprendiendo.

Decir "esto todavía no lo hice" genera más confianza que inflar. En una entrevista técnica se
nota en treinta segundos quién copió comandos y quién entiende lo que corrió.

## Preguntas probables y respuestas honestas

**¿Por qué un solo nodo?**
Porque el hardware es una instancia gratuita de 2 vCPU y 12 GB, y porque dos nodos no dan quórum
de etcd ni failover real. Para practicar multi-nodo uso un clúster efímero local con `kind`.
Está en el ADR-001.

**¿Qué pasa si se cae el nodo?**
Se cae todo. Es la limitación principal y la conozco. Por eso el backup y el restore están
verificados, y por eso el gestor de contraseñas se migra último: no quiero que sea el conejillo
de indias.

**¿Cómo manejás los secretos?**
Hoy viven en archivos `.env` y montajes de solo lectura fuera del repositorio. En el clúster van
a ser Secrets de Kubernetes, y más adelante evaluaría External Secrets o SOPS. No lo implementé
todavía.

**¿Y el ingress?**
Hoy entra todo por un túnel de Cloudflare hacia un reverse proxy en Docker. k3s trae Traefik por
defecto, así que tengo dos opciones y todavía no decidí cuál queda. Es una decisión pendiente, no
un olvido.

**¿Por qué no usaste GitOps?**
Porque primero quiero entender los objetos de Kubernetes a mano. Argo CD o Flux sobre una
infraestructura que no entiendo es automatizar el caos.

**¿Qué fue lo más difícil?**
No fue Kubernetes. Fue darme cuenta de que mi backup no estaba verificado: la base de datos
guardaba los cambios recientes en un archivo WAL que el script no copiaba. Lo descubrí probando
un restore, no leyendo el script.

## El activo real

Lo que este proyecto demuestra no es que sepa instalar k3s. Es que:

- Mido antes de decidir.
- Documento el porqué, no solo el cómo.
- Pruebo que las cosas funcionan, en vez de asumirlo.
- Sé cuándo una herramienta no es la respuesta.
