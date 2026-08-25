# canary-service — despliegue canary con Connectivity Link

Demuestra **canary deployment** con el producto Red Hat **Connectivity Link**
(Gateway API nativo): el HTTPRoute reparte el trafico entre dos versiones del
servicio con `weight`. No hace falta Service Mesh para esto — el weighted routing
es nativo de Gateway API y lo gobierna Connectivity Link.

## Como funciona

- Dos Deployments del mismo servicio: `canary-service-v1` (estable) y
  `canary-service-v2` (nueva version), cada uno con su Service.
- Un unico HTTPRoute con dos `backendRefs`, cada uno con un `weight`:
  - v1: weight 90  (90% del trafico)
  - v2: weight 10  (10% del trafico -> el canary)
- Se sube el peso de v2 gradualmente (10 -> 50 -> 100) editando el HTTPRoute en
  Git; Argo sincroniza y el reparto cambia sin downtime.

## Producto: Connectivity Link (no Service Mesh)
El canary por peso es Gateway API puro. Circuit breaker es otra cosa (ese SI es
Service Mesh / Istio DestinationRule) — ver el servicio circuit-breaker-service.
