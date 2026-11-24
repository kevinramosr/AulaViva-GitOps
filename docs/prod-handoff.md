# Traspaso final de AulaViva PROD

Fecha de cierre: 24 de noviembre de 2026.

**PROD funciona internamente, pero NO está listo para exposición pública.**

Este documento registra las validaciones realizadas durante la preparación y el cierre del entorno. No implica una nueva ejecución de pruebas ni demuestra disponibilidad de producción real. La rama se prepara desde GitOps main en `eafea53ef4857c79cc3cef228d10585fcc7992b6`.

## Estado validado

- PROD desplegado mediante Argo CD: Application `aulaviva-prod`, estado **Synced / Healthy** al cierre.
- DEV: Application `aulvaviva-dev`, estado **Synced / Healthy** al cierre. DEV no fue modificado por las operaciones de PROD ni por la limpieza E2E.
- Imagen PROD fijada por digest:
  `ghcr.io/kulkatpop/aulaviva-api@sha256:3c1aef5217470e0e74368de2bfa36f6be66fed931cc79a65a0c8fbb33b379850`.
- PostgreSQL PROD separado de DEV: contenedor y volumen propios; publicación local PROD en 55433 y DEV en 45655.
- PostgreSQL **16.16** con **pgvector 0.8.7**; migraciones **000–006** ejecutadas y esquema validado.
- Roles y privilegios configurados: `aulaviva_migrator` para propiedad/DDL y `aulaviva_app` para runtime con privilegios limitados. No se concedió UPDATE sobre `aula_alumnos` durante la E2E ni su limpieza.
- Conexión PostgreSQL con **TLS verify-full y TLS 1.3** validada, incluida la comprobación de rechazo con CA incorrecta.
- Backup lógico en formato custom validado. Restauración en una instancia aislada validada, con comparación de esquema, propietarios/ACL y prueba funcional; instancia temporal eliminada.
- Hardening del contenedor desplegado y validado:
  - UID/GID **33**.
  - `runAsNonRoot: true`.
  - Capabilities **ALL** eliminadas.
  - `allowPrivilegeEscalation: false`.
  - `readOnlyRootFilesystem: true`.
  - Seccomp `RuntimeDefault`.
  - `/tmp` mediante `emptyDir` de **64Mi**.
- Sesiones PHP validadas después del hardening: creación, escritura y lectura posterior de la misma sesión con almacenamiento temporal en `/tmp`.
- PostgreSQL y Ollama accesibles desde PROD. Esta conectividad no equivale a la validación completa del comportamiento del Tutor.
- Una réplica y estrategia `Recreate`, con la limitación de sesiones locales.
- Tras la limpieza, `/health/live.php` y `/health/ready.php` respondieron **HTTP 200 — ok**; PostgreSQL PROD continuó accesible.

## E2E

Run ejecutado: **`E2E_20261124_8dc02e71fbf2`**.

### Resultados positivos

- Login web.
- Login JSON.
- Invitación y activación de cuenta.
- Reutilización de invitación rechazada.
- Subida de material.
- Lectura y descarga autorizadas.
- Ingesta.
- Fragmentación y embedding.
- Controles negativos de autorización.
- Tutor sin evidencia: abstención correcta.

### Caso positivo del Tutor pendiente

El caso positivo con evidencia recuperada terminó en **`VALIDATION_FAILED`**, con **HTTP 200** y **1 fragmento recuperado**. No se produjo una respuesta factual sin respaldo.

El diagnóstico del Tutor queda pendiente. No debe considerarse validado el caso positivo ni interpretarse HTTP 200 como éxito funcional.

Evaluación/rendición no se ejecutó en este bloque.

## Limpieza E2E

El run fue eliminado posteriormente, utilizando su inventario y comprobando las identidades exactas de los registros antes de borrar.

| Tabla | Registros eliminados |
| --- | ---: |
| material_fragmentos | 1 |
| material_ingestas | 1 |
| materiales | 1 |
| invitaciones | 1 |
| aula_alumnos | 2 |
| aula_profesores | 2 |
| aulas | 2 |
| usuarios | 5 |
| **Total** | **15** |

La limpieza se ejecutó en una única transacción con `ON_ERROR_STOP=1`, por IDs y claves compuestas explícitas, respetando las dependencias. No se utilizaron DELETE por prefijo, TRUNCATE ni CASCADE.

Las **16 tablas quedaron vacías nuevamente**, igual que antes de la prueba, comprobado antes y después del COMMIT. No quedaron fixtures en PostgreSQL ni se encontraron registros ajenos o dependencias inesperadas. No se modificaron permisos, esquema, DEV, Secrets ni infraestructura.

## Pendientes

1. Diagnosticar el caso positivo del Tutor que terminó en `VALIDATION_FAILED`.
2. Ejecutar evaluación/rendición E2E. Resolver previamente su bloqueo de permisos si resulta necesario y se autoriza.
3. Revisar si realmente se necesita `UPDATE(alumno_id)` sobre `aula_alumnos` para `aulaviva_app` y, si se aprueba, conceder únicamente el privilegio mínimo. No está autorizado por este documento.
4. Validar flujos autenticados completos.
5. Diseñar HTTPS/Ingress antes de exposición pública.
6. Validar NetworkPolicy/enforcement; la existencia de manifiestos no demuestra su aplicación efectiva.
7. Implementar backup externo.
8. Definir recuperación ante pérdida del equipo.
9. Evaluar disponibilidad y sesiones para un entorno de producción real.

## Límites del traspaso

El entorno validado depende de infraestructura local de laboratorio. La restauración aislada demuestra la recuperación probada del backup, pero no demuestra recuperación ante pérdida completa del equipo ni alta disponibilidad.

Este documento no contiene contraseñas, secretos, claves privadas, CA privada, backups, archivos de credenciales ni datos personales. No autoriza nuevas pruebas, cambios de permisos, despliegues ni exposición pública.
