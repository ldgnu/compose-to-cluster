# Diario

Bitácora del proyecto, en orden cronológico.

## 2026-11-23 — Auditoría, limpieza y reset del clúster

### Lo que se hizo

1. **Auditoría de solo lectura** de 34 contenedores: imágenes, volúmenes, redes, mounts y
   nombres de variables de entorno (nunca valores).
2. **Mapeo de dependencias** entre servicios a partir de las redes Docker.
3. **Verificación del backup**: se confirmó que el tarball de 3.6 GB incluye la base de datos,
   el WAL y la clave RSA del gestor de contraseñas.
4. **Restore real**: contenedor paralelo, datos extraídos, login con contraseña maestra. Funcionó.
5. **Limpieza**: dos contenedores basura, 32 imágenes dangling, 13 GB de cache de build, una red
   huérfana y cAdvisor (1.2 GB de RAM). Disco de 35 a 58 GB libres.
6. **Descubrimiento**: un clúster k3s corría desde un mes antes, sin documentación, con workloads
   de un tutorial y componentes del sistema con 236 y 250 reinicios por falta de memoria.
7. **Reset**: desinstalación completa y reinstalación limpia (v1.37.2). Nodo `Ready`, cuatro pods
   de sistema, cero reinicios.

### Errores cometidos (y qué enseñaron)

- **Estimación de espacio inflada**: se sumaron tamaños de `docker image ls`, que incluyen capas
  compartidas. El recuperable real era un tercio de lo estimado.
- **`MemPerc` mal interpretado**: `docker stats` calcula el porcentaje contra el límite de cada
  contenedor, no contra la RAM del host. El ranking por porcentaje es engañoso.
- **Un comando de inspección falló por `sudo` dentro de `$()`**: el shell expande la sustitución
  antes de ejecutar `sudo`, así que el comando interno corría sin privilegios y devolvía vacío.
- **jq es sensible a mayúsculas**: `{type, source}` no lee `.Type` ni `.Source`; devuelve `null`.
- **Tono**: el diagnóstico llegó en ráfagas de problemas sin pausa para consolidar. Aprendizaje:
  una auditoría cansa si no se alterna con avance visible.

### Estado al cerrar

- Clúster k3s vacío y sano.
- Disco resuelto.
- Backup y restore verificados.
- RAM todavía ajustada (6.7 GB usados, 1.8 GB de swap), dominada por el stack de agentes.
- Repositorio de documentación creado.
