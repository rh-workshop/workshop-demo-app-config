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

## Variante AUTOMATICA: canary con Argo Rollouts (/canary-auto)

El canary anterior avanza A MANO (alguien edita los pesos en Git). La variante
automatica — `rollout-canary-auto.yaml` + `analysistemplate-canary-success-rate.yaml`
+ su Service/HTTPRoute/AuthPolicy propios en el path `/canary-auto` — la decide
una maquina: el `Rollout` despliega la version candidata por pasos (25% -> 50%
-> 100%) y en cada paso un `AnalysisTemplate` consulta Prometheus (tasa de
respuestas no-5xx y latencia p95); si una metrica viola su umbral, el Rollout
ABORTA y revierte solo a la version estable. Las dos variantes CONVIVEN a
proposito: la manual ensena el mecanismo, la automatica ensena la practica.

Detalles y limites, documentados en los propios manifiestos:

- El controlador lo despliega la plataforma (`RolloutManager` en
  `workshop-demo-platform-config/rollouts/gitops/`); aqui solo viven los CRs.
- SIN `trafficRouting`: el peso se aproxima por proporcion de replicas (4
  replicas -> 25% = 1 pod), que es nativo de Argo Rollouts. El control fino del
  peso con Gateway API exige el plugin community
  `rollouts-plugin-trafficrouter-gatewayapi`, fuera de la norma "solo Red Hat"
  del proyecto, asi que no se usa.
- La consulta al Thanos Querier del cluster esta PENDIENTE de validar en vivo
  (autenticacion por token); mientras tanto la AnalysisTemplate va con
  `insecure: true` y esta anotado en el backlog de plataforma.

```bash
# Seguir un despliegue canary automatico (plugin kubectl-argo-rollouts)
oc argo rollouts get rollout canary-service-auto -n canary-demo-dev --watch
```
