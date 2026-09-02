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

## Promocionar el canary

El reparto vive en `httproute-canary-weighted-split.yaml`. Promocionar es editar
los pesos en Git y dejar que Argo sincronice:

```bash
# Ver el reparto vigente
oc get httproute canary-service -n canary-demo-dev \
  -o jsonpath='{range .spec.rules[0].backendRefs[*]}{.name}={.weight}{"\n"}{end}'

# Comprobar el reparto real: la respuesta trae la version que atendio
for i in $(seq 1 20); do
  curl -sk -H "Authorization: Bearer $TOKEN" https://$GATEWAY_HOST/canary | jq -r .version
done | sort | uniq -c
```

Secuencia habitual: `90/10` -> `50/50` -> `0/100`. Si la version candidata falla,
se revierte devolviendo el peso a v1 en el mismo fichero. La decision de
promocionar o revertir la toma quien opera, mirando las metricas del servicio en
la consola de OpenShift.
