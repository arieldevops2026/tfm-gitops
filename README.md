# tfm-gitops

Repositorio GitOps del TFM *Diseño y evaluación de una plataforma GitOps con entrega progresiva gobernada por SLO y telemetría OpenTelemetry para microservicios en Kubernetes* (UNIR, Máster en DevOps).

Contiene el estado deseado del clúster. Argo CD lo vigila y aplica los cambios; nadie ejecuta `kubectl apply` a mano.

## Estructura

```text
apps/
└── tfm/                  # namespace "tfm": los tres microservicios
    ├── namespace.yaml
    ├── ms-products.yaml  # Deployment + Service
    ├── ms-orders.yaml
    ├── ms-payments.yaml
    └── ingress.yaml      # Traefik: expone solo ms-orders
```

## Servicios

| Servicio | Puerto | Repositorio |
|---|---|---|
| `ms-orders` | 18000 | `tfm-ms-orders` |
| `ms-products` | 18001 | `tfm-ms-products` |
| `ms-payments` | 18002 | `tfm-ms-payments` |

## Notas

- En esta fase las imágenes son `tfm-ms-*:local`, importadas a K3s a mano. Después pasarán a `ghcr.io` fijadas por digest.
- Este repositorio es público: no debe contener secretos.
