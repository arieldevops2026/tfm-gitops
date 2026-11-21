# tfm-gitops

Repositorio GitOps del TFM *Diseño y evaluación de una plataforma GitOps con entrega progresiva gobernada por SLO y telemetría OpenTelemetry para microservicios en Kubernetes* (UNIR, Máster en DevOps).

Contiene el estado deseado del clúster. Argo CD lo vigila y aplica los cambios: nadie ejecuta `kubectl apply` a mano.

## Estructura

```text
argocd/
├── tfm-app.yaml              # Application "tfm": apunta al escenario activo (apps/tfm/overlays/<b0|b1|b2>)
└── observability-app.yaml    # Application "observability": apps/observability
apps/
├── observability/            # Prometheus, Tempo, Loki, OpenTelemetry Collector y Grafana
└── tfm/
    ├── base/                 # común a los tres escenarios
    │   ├── namespace.yaml
    │   ├── ms-products.yaml  # Deployment + Service
    │   ├── ms-payments.yaml  # Deployment + Service
    │   └── ms-orders.yaml    # Deployment + Service: única fuente del pod template de ms-orders
    └── overlays/
        ├── b0/               # B0: rolling update nativo (Deployment), sin canary ni SLO
        ├── canary-common/    # común a B1 y B2: Service canary, TraefikService, IngressRoute con pesos
        ├── b2/               # B2: Rollout canary con pasos de tiempo fijos, sin SLO
        └── b1/               # B1: Rollout canary con análisis de SLO (AnalysisTemplate slo-ms-orders)
```

Se usa Kustomize: cada escenario es la `base` más sus diferencias. Argo CD renderiza el overlay al que apunta la Application.

## Escenarios de despliegue de ms-orders

| Escenario | Qué ejecuta los pods | Tráfico | Decide el avance |
|---|---|---|---|
| B0 | `Deployment` | IngressRoute → Service `ms-orders` | Kubernetes (probes) |
| B2 | `Rollout` con `workloadRef` al Deployment | IngressRoute → TraefikService (pesos) | El tiempo: 10 % → 1 min → 50 % → 1 min → 100 % |
| B1 | `Rollout` con `workloadRef` al Deployment | IngressRoute → TraefikService (pesos) | Los SLO: p95 < 500 ms, 5xx < 1 %, mínimo 500 peticiones |

En B1 y B2 el Deployment `ms-orders` queda con 0 réplicas: solo aporta el pod template, y es el Rollout quien crea los pods. Así la imagen y la configuración están en un único archivo (`base/ms-orders.yaml`) en los tres escenarios.

### Cambiar de escenario

El escenario activo es el `path` de la Application `tfm`. Para cambiarlo de forma permanente se edita `argocd/tfm-app.yaml`. Para los experimentos hay un script en la VM del laboratorio:

```bash
./cambiar-escenario.sh b0   # o b1, b2
```

Cambiar de escenario **no es transparente**: al pasar de Deployment a Rollout o al revés, `ms-orders` deja de responder unos segundos (medido en la VM local: ~5-6 s de B1 a B0 y de B0 a B2; sin fallos de B2 a B1). Se cambia **entre** experimentos, nunca durante uno.

## Flujo de una versión nueva

1. Push en el repositorio del microservicio.
2. GitHub Actions compila la imagen nativa, la publica en `ghcr.io` y abre un Pull Request aquí que cambia la imagen (por digest) y `APP_VERSION` en `apps/tfm/base/<servicio>.yaml`.
3. Al fusionar el PR, Argo CD lo aplica y el escenario activo decide cómo se despliega.

## Servicios

| Servicio | Puerto | Repositorio |
|---|---|---|
| `ms-orders` | 18000 | `tfm-ms-orders` |
| `ms-products` | 18001 | `tfm-ms-products` |
| `ms-payments` | 18002 | `tfm-ms-payments` |

## Notas

- Las imágenes se fijan por digest (`ghcr.io/arieldevops2026/tfm-ms-*@sha256:...`), nunca por etiqueta.
- Este repositorio es público: no debe contener secretos. La contraseña de Grafana vive en el Secret `grafana-admin`, creado a mano en el clúster.
