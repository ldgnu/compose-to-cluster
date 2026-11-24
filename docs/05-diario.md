# Diario

Bitácora del proyecto, en orden.

## 2026-11-23 — Auditoría, limpieza y reset del clúster

### Qué hice

1. **Auditoría de solo lectura** de mis 34 contenedores: imágenes, volúmenes, redes, mounts y
   nombres de variables de entorno, nunca valores.
2. **Mapeé dependencias** entre servicios usando las redes Docker.
3. **Verifiqué el backup**: confirmé que el tarball de 3.6 GB incluye la base de datos, el WAL y
   la clave RSA del gestor de contraseñas.
4. **Probé un restore real**: contenedor paralelo, datos extraídos, login con contraseña maestra.
   Funcionó.
5. **Limpié**: dos contenedores basura, 32 imágenes sin tag, 13 GB de cache, una red huérfana y
   cAdvisor (1.2 GB de RAM). Pasé de 35 a 58 GB libres.
6. **Descubrí** que tenía un clúster k3s corriendo desde hacía un mes, sin documentación, con
   workloads de un tutorial y componentes que se reiniciaban solos por falta de memoria.
7. **Lo reseteé**: desinstalación completa y reinstalación limpia (v1.37.2). Nodo `Ready`, cuatro
   pods de sistema, cero reinicios.

### Errores que cometí

- **Estimé mal el espacio recuperable.** Sumé los tamaños de `docker image ls`, que incluyen
  capas compartidas. Recuperé un tercio de lo que calculé.
- **Interpreté mal `MemPerc`.** `docker stats` calcula el porcentaje contra el límite de cada
  contenedor, no contra la RAM del host. El ranking por porcentaje es engañoso; hay que mirar MiB.
- **Un comando me devolvió vacío por el orden de expansión del shell.** Puse `sudo docker inspect
  $(docker ps -aq)`: el shell expande la sustitución **antes** de ejecutar `sudo`, así que el
  comando interno corría sin privilegios y devolvía nada.
- **jq distingue mayúsculas.** Escribí `{type, source}` esperando leer `.Type` y `.Source`. Me
  devolvió `null` en todos los campos.
- **Me costó encontrar el ritmo.** Encadené diagnósticos sin pausas para consolidar. Una auditoría
  cansa si no alterna con algo que se pueda ver funcionando.

### Dónde quedé

- Clúster k3s vacío y sano.
- Disco resuelto.
- Backup y restore verificados.
- RAM todavía ajustada (6.7 GB usados, 1.8 GB de swap), dominada por el stack de agentes.
- Repositorio de documentación creado.
