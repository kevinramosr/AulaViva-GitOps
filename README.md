
# AulaViva GitOps

Repositorio para gestionar los manifiestos Kubernetes de AulaViva
mediante Kustomize y una estrategia GitOps.

## Estructura

- `base/`: recursos Kubernetes comunes.
- `overlays/dev/`: configuración de desarrollo.
- `overlays/prod/`: configuración de producción.

## Flujo previsto

1. GitHub Actions valida el código de la aplicación.
2. La imagen se construye y publica en GHCR.
3. Un cambio revisable actualiza la imagen en Dev.
4. Argo CD sincroniza los manifiestos con Kubernetes cuando
   la integración esté configurada.

## Estado

Estructura inicial en preparación. La integración con GHCR,
Argo CD y Kubernetes debe validarse antes de considerarse operativa.

