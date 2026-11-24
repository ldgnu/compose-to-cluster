# Decisiones de arquitectura (ADR)

Registro de las decisiones que cambiaron el rumbo del proyecto, con el contexto que las hizo
necesarias y las alternativas que se descartaron.

---

## ADR-001 — Clúster single-node, no multi-nodo

**Fecha:** 2026-11-23
**Estado:** Aceptada

### Contexto

La infraestructura corre en un único servidor ARM gratuito: 2 vCPU, 12 GB, disco único, sin
posibilidad de ampliar. Existe un segundo servidor más chico (1 GB) y una máquina vieja apagada
en una red doméstica, ambos en dominios de falla distintos y sin conectividad de baja latencia.

### Decisión

El clúster de producción es **un solo nodo** en el servidor principal. No se agregan nodos.

### Alternativas descartadas

- **Segundo nodo con el servidor chico**: 1 GB no sostiene un kubelet con cargas reales, y
  agregaría complejidad de arquitectura mixta (ARM + x86) sin dar alta disponibilidad.
- **La máquina doméstica como nodo**: está en otra red, detrás de NAT, con hardware de 2011.
  Un plano de control atravesando internet público convierte cada microcorte en un fallo de
  clúster.
- **Comprar otro VPS**: no resuelve el problema real (memoria y datos), y dos nodos no dan
  quórum de etcd ni failover real del plano de control.

### Consecuencia

Para practicar lo que un nodo único no puede enseñar (taints, scheduling, drenaje, HA del
control plane) se usa un clúster efímero local con `kind` o `k3d` en la máquina de trabajo.
El clúster de producción es para servicios reales; el local es para aprender.

---

## ADR-002 — El backup va antes que Kubernetes

**Fecha:** 2026-11-23
**Estado:** Aceptada

### Contexto

El único dato irrecuperable es el gestor de contraseñas (~12 MB). No existían pasos de restore
escritos: solo un export desde la aplicación, que no restaura la instancia.

### Decisión

No se migra ningún servicio con datos hasta que exista un backup **verificado con un restore
real** y un runbook escrito.

### Consecuencia

Se insertó una fase 0.5 antes del clúster. Se probó el restore en un contenedor paralelo y se
documentó el procedimiento, incluidos los dos errores de TLS que costaron tiempo. El gestor de
contraseñas se migra último, no primero.
