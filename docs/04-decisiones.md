# Decisiones de arquitectura (ADR)

Las decisiones que cambiaron el rumbo, con el contexto que las hizo necesarias y lo que descarté.

---

## ADR-001 — Clúster single-node, no multi-nodo

**Fecha:** 2026-11-23
**Estado:** Aceptada

### Contexto

Todo corre en un único servidor ARM gratuito: 2 vCPU, 12 GB, disco único, sin posibilidad de
ampliar. Tengo un segundo servidor más chico (1 GB) y una máquina vieja apagada en mi red
doméstica, pero están en dominios de falla distintos y sin conectividad de baja latencia.

### Decisión

El clúster es **un solo nodo**. No agrego más.

### Qué descarté

- **El servidor chico como segundo nodo.** 1 GB no sostiene un kubelet con cargas reales, y me
  obligaría a arquitectura mixta (ARM + x86) sin darme alta disponibilidad.
- **La máquina de casa como nodo.** Está en otra red, detrás de NAT, con hardware de 2011. Un
  plano de control atravesando internet público convierte cada microcorte en un fallo de clúster.
- **Comprar otro VPS.** No resuelve el problema real (memoria y datos), y dos nodos no dan quórum
  de etcd ni failover del plano de control.

### Consecuencia

Para practicar lo que un nodo único no puede enseñarme —taints, scheduling, drenaje, HA del
control plane— uso un clúster efímero local con `kind` o `k3d`. El de producción es para
servicios reales; el local, para aprender.

---

## ADR-002 — El backup va antes que Kubernetes

**Fecha:** 2026-11-23
**Estado:** Aceptada

### Contexto

Mi único dato irrecuperable es el gestor de contraseñas (~12 MB). No tenía pasos de restore
escritos: solo el export desde la aplicación, que no restaura la instancia.

### Decisión

No migro ningún servicio con datos hasta tener un backup **verificado con un restore real** y un
runbook escrito.

### Qué descarté

- **Migrar primero y respaldar después.** Es el orden natural y el equivocado: si algo sale mal
  en el medio, no hay red.
- **Confiar en el export de la aplicación.** Sirve para migrar de gestor, no para recuperar el
  servicio: no trae usuarios, organizaciones, adjuntos ni la clave del servidor.

### Consecuencia

Inserté una fase 0.5 antes del clúster. Probé el restore en un contenedor paralelo y documenté el
procedimiento, incluidos los dos errores de TLS que me costaron tiempo. El gestor de contraseñas
se migra último.
