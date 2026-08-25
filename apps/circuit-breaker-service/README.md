# circuit-breaker-service — circuit breaker con Service Mesh (Istio)

Demuestra **circuit breaker** con el producto Red Hat **OpenShift Service Mesh 3**
(Istio). A diferencia del canary (que es Gateway API / Connectivity Link), el
circuit breaker NO es Kuadrant: es una **DestinationRule** de Istio con
`outlierDetection` (expulsa instancias que fallan) y `connectionPool` (limita
conexiones concurrentes, el "breaker" propiamente dicho).

## Como funciona

- Un Deployment del servicio, CON sidecar de Istio (el circuit breaker vive en el
  sidecar Envoy, no en la app).
- Una `DestinationRule` con:
  - `connectionPool`: limita conexiones TCP/HTTP concurrentes y peticiones en
    cola. Al superarlas, Envoy corta ("abre el circuito") con 503 en vez de saturar
    el backend.
  - `outlierDetection`: si una instancia devuelve N errores 5xx seguidos, la
    expulsa del balanceo durante un tiempo (ejection). Evita mandar trafico a un
    pod enfermo.

## Producto: Service Mesh (no Connectivity Link)
El circuit breaker y el outlier detection son de Istio. El canary por peso es
Gateway API (ver canary-service). Son escenarios de productos distintos a proposito.

## Requisito
El namespace debe tener inyeccion de sidecar activa (istio-injection=enabled o el
IstioRevisionTag) para que Envoy aplique la DestinationRule.
