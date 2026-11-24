# AulaViva PROD de laboratorio

Este overlay prepara el despliegue; no crea la Application de Argo CD ni expone
HTTP al exterior. El Service heredado es ClusterIP, puerto 80 hacia Apache/PHP.
El Namespace existente queda representado declarativamente en `namespace.yaml`.

## Imagen y dependencias

La imagen queda fijada al digest aprobado, disponible en GHCR y utilizado en DEV.
La transformacion coincide con `nginx` en la base, sin modificar base ni DEV.

Los Secrets se aprovisionan fuera de Git:

- `aulaviva-prod-db`: `DB_DSN`, `DB_USER`, `DB_PASSWORD`.
- `aulaviva-prod-db-ca`: exclusivamente `ca.crt`, certificado publico de la CA.

El DSN existente apunta a `host.docker.internal:55433`, base `aulaviva_prod`,
con `sslmode=verify-full`. `PGSSLROOTCERT` indica a libpq la CA montada en
`/etc/aulaviva/postgresql-ca/ca.crt`. No se almacena ninguna credencial aqui.

Ollama sigue externo al Pod en `host.docker.internal:11434`, con
`nomic-embed-text` y `qwen3:1.7b`. La opcion explicita
`OLLAMA_ALLOW_DOCKER_HOST=true` habilita ese host en el proveedor; el codigo
no comprueba APP_ENV. Esta configuracion depende de Docker Desktop.

## Sesiones, recursos y disponibilidad

PHP utiliza sesiones en archivos locales, sin almacenamiento compartido validado.
Se configura una replica con estrategia Recreate para evitar solapamiento de
Pods durante un rollout. Los rollouts tienen interrupcion y los reinicios o
reemplazos pueden perder sesiones; no hay alta disponibilidad ni HPA.
No se agrega PVC para sesiones. Materiales PDF y embeddings residen en PostgreSQL.

Requests iniciales: 100m CPU y 128Mi RAM. Limites: 500m CPU y 384Mi RAM;
el margen sobre el memory_limit PHP de 256M contempla Apache y otros consumos.
Son valores iniciales para el laboratorio, no resultados de una prueba de carga.

Liveness usa `/health/live.php`, independiente de PostgreSQL y Ollama.
Readiness usa `/health/ready.php`, con SELECT 1 a PostgreSQL; no consulta Ollama.
Su timeout de 5 segundos contempla los timeouts de conexion y consulta del PHP.

## Revision previa al despliegue

Desde la raiz del repositorio:

```sh
kubectl kustomize overlays/prod
kubectl apply --dry-run=server -k overlays/prod
git diff --check
```

El dry-run no despliega ni demuestra scheduling, conectividad o salud real.
La Application, sincronizacion Argo CD y exposicion HTTPS se preparan despues.
