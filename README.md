# workshop-demo-app-config — configuración declarativa del workshop

Este repositorio contiene **toda la configuración** del flujo GitOps del
workshop: la plataforma, la aplicación de demostración y las políticas de
exposición. Es el repositorio que observa Argo CD.

El **código** de la aplicación vive aparte, en `workshop-app`. Esa separación es
deliberada y es uno de los principios que enseña el material:

```
workshop-app (código)
  -> Pipeline de Tekton construye la imagen
  -> registro de imágenes
  -> workshop-config (este repositorio)
  -> Argo CD sincroniza con Kustomize
  -> Connectivity Link expone la API con políticas
```

Argo CD **nunca** mira el repositorio de código: solo este. Un despliegue se
produce porque cambia la configuración en Git, no porque alguien ejecute un
comando contra el cluster.

## Estructura

| Ruta | Contenido |
|---|---|
| `apps/demo-service/` | Deployment, Service y ConfigMap de la aplicación, con overlays `dev` y `test` |
| `apps/canary-service/` | Despliegue canary en dos variantes: MANUAL (dos versiones con reparto por pesos en la HTTPRoute) y AUTOMÁTICO (Rollout de Argo Rollouts + AnalysisTemplate que promociona o revierte solo por métricas) |
| `apps/bluegreen-service/` | Despliegue blue-green: dos versiones con conmutación total de tráfico |
| `apps/circuit-breaker-service/` | Frontend y backend con DestinationRule de circuit breaker |
| `policies/` | AuthPolicy y RateLimitPolicy del servicio |

> La **plataforma** NO vive aquí: su fuente de verdad es el repositorio
> `workshop-demo-platform-config`. Eso incluye el Gateway compartido, el CR de
> Kuadrant, los **pipelines de Tekton** (montan el token de escritura sobre este
> repositorio: gobierno de plataforma) y las **Applications de Argo CD**: los
> servicios de `apps/` los descubren los ApplicationSets `workshop-*` de ese
> repositorio, así que aquí no se declara ningún objeto de Argo. Este
> repositorio solo contiene lo que es de los equipos de aplicación: las apps y
> sus políticas.
>
> La configuración del **propio Argo CD** (el CR `ArgoCD`, los AppProjects y el
> RBAC del controller) tampoco vive aquí: la aplica el **bootstrap** del cluster
> (`workshop-demo-platform-config/bootstrap/`), porque lo que Argo necesita para
> existir no lo gestiona Argo.

Cada servicio de `apps/` sigue el mismo patrón: `base/` + `overlays/<entorno>/`.
Las imágenes viven en el registro corporativo (Quay) y los overlays las fijan
por digest; cada namespace descarga con su pull secret (`quay-pull-credentials`).

Todos los directorios siguen el patrón **base + overlays** de Kustomize: la
`base` declara el *qué* y cada overlay el *dónde* y el *cuánto*.

## Despliegue: quién crea las Applications

Ningún objeto de Argo CD vive en este repositorio. Los **ApplicationSets
`workshop-services-dev` y `workshop-gateway-routes-dev`** (en
`workshop-demo-platform-config/gitops/appsets/`) recorren este repositorio con
un generador `git` de directorios:

- `apps/*/overlays/dev` → una Application por servicio (onda 2).
- `apps/*/gateway-route/overlays/dev` → una Application por Route del gateway,
  desplegada en `platform-gateway` (onda 1).

Además, el ApplicationSet `workshop-previews-pr` (mismo repositorio de
plataforma) crea un **preview efímero por cada Pull Request abierto** contra
este repositorio: despliega `apps/demo-service/overlays/dev` tal como lo deja
la rama del PR en un namespace propio `demo-service-pr-<n>`, expuesto por el
gateway en el path `/pr-<n>`. Al cerrar el PR, la Application y su namespace se
borran solos. Así dos features en paralelo dejan de pisarse el `overlays/dev`.

**Añadir un servicio nuevo** consiste en crear su carpeta
`apps/<componente>-service/` siguiendo el patrón existente; el ApplicationSet
la detecta solo. La plantilla (project, destino, syncPolicy) la define
plataforma, y el AppProject `workshop-platform` (allow-list de kinds y
namespaces, falla cerrado) acota lo que este repositorio puede desplegar. El
namespace de destino sale del nombre de la carpeta: `<componente>-demo-dev`
(el servicio original `demo-service` conserva `workshop-demo-dev`).

Las Applications de `policies` (y la de los pipelines) se declaran
estáticamente en `workshop-demo-platform-config/gitops/apps/workshop/`.

| Onda | Componente | Motivo del orden |
|---|---|---|
| 1 | Rutas del gateway | La Route de exposición existe antes que el servicio |
| 2 | Aplicaciones y pipelines | Necesitan el gateway compartido ya disponible (lo despliega `workshop-demo-platform-config`) y el AppProject `workshop-platform`, que crea el bootstrap |
| 3 | Políticas | Se declaran sobre el HTTPRoute de la aplicación, que debe existir antes |

## Pipelines: CI y CD separados

Las definiciones viven en `workshop-demo-platform-config/workshop-pipelines/`
(el pipeline monta el token de escritura sobre este repositorio, así que su
definición es gobierno de plataforma), pero operan SOBRE este repositorio.
El flujo está partido en dos pipelines, cada uno con una responsabilidad clara:

- **`ci-build-image`** — clona `workshop-app`, construye la imagen con
  Buildah, la firma con Tekton Chains, la publica, y escribe el digest recién
  construido en este repositorio. Ese `git push` es lo que dispara el CD.
- **`cd-deploy-application`** — clona este repositorio, valida que el overlay
  renderiza, y le pide a Argo CD que sincronice la Application. No reconstruye ni
  toca el código.

Cada uno se lanza con su PipelineRun de ejemplo:

```bash
# Desde un clon de workshop-demo-platform-config:
oc create -f workshop-pipelines/runs/pipelinerun-ci-build-image.yaml -n workshop-demo-dev
```

La imagen se publica en el **registro corporativo (Quay)** con su tag inmutable
`git-<sha-corto>`; la credencial de publicación es un Secret de tipo `docker`
referenciado por su **nombre** desde el workspace de la tarea de construcción.

> Tekton corre **solo en el cluster hub**: el CI construye y el CD le pide a Argo
> que despliegue. Ninguna de las dos cosas necesita ejecutarse en los clusters de
> destino — de eso se encarga Argo CD.

## Exposición y políticas

El servicio se expone por el gateway compartido mediante un `HTTPRoute`. Dos
políticas de Connectivity Link gobiernan su consumo:

- **AuthPolicy** — exige un JWT válido emitido por el proveedor de identidad. Sin
  token, el gateway responde `401`.
- **RateLimitPolicy** — limita el consumo por identidad. Superado el límite, el
  gateway responde `429`.

El contador del límite usa el **sujeto del token**, no la dirección de origen:
`source.address` incluye el puerto efímero de cada conexión, de modo que cada
petición contaría como un origen distinto y el límite nunca se dispararía.

### Requisito del namespace

Para que una aplicación quede expuesta por el gateway compartido, su namespace
necesita **dos** etiquetas:

```yaml
shared-gateway-access: "true"   # el listener del Gateway admite sus rutas
istio-discovery: enabled        # istiod descubre el namespace
```

Si falta la segunda, el `HTTPRoute` parece correcto pero el listener queda con
`attachedRoutes: 0` y el tráfico no llega.

## Verificación funcional

```bash
# Sin token: 401
curl -sk -o /dev/null -w '%{http_code}\n' https://<host>/

# Con token válido: 200
curl -sk -H "Authorization: Bearer $TOKEN" https://<host>/

# Superando el límite: 429 a partir de la sexta petición en 60 segundos
for i in $(seq 1 8); do
  curl -sk -o /dev/null -w '%{http_code} ' -H "Authorization: Bearer $TOKEN" https://<host>/
done; echo
```

## Convenciones

- Los manifiestos se nombran `<kind>-<descriptor>.yaml`: el **kind en minúscula
  va primero** (al listar una carpeta los archivos quedan agrupados por tipo) y
  después un **descriptor que explica qué hace** el recurso en el workshop, no
  solo cómo se llama. `kustomization.yaml` nunca se renombra, y un archivo
  multi-documento se nombra por su recurso principal. Los nombres de archivo van
  en inglés kebab-case; el `metadata.name` del recurso NO cambia con el archivo.

  | Kind | Prefijo de archivo |
  |------|--------------------|
  | Deployment | `deployment-` |
  | Service | `service-` |
  | ConfigMap | `configmap-` |
  | Namespace | `namespace-` |
  | HTTPRoute | `httproute-` |
  | Route (OpenShift) | `route-` |
  | AuthPolicy | `authpolicy-` |
  | RateLimitPolicy | `ratelimitpolicy-` |
  | DestinationRule | `destinationrule-` |
  | RoleBinding / ClusterRole | `rolebinding-` / `clusterrole-` |
  | ServiceAccount | `serviceaccount-` |

  Ejemplos:
  - `httproute-canary-weighted-split.yaml` — HTTPRoute que reparte el tráfico
    90/10 entre v1 y v2 por peso.
  - `deployment-bluegreen-green-standby.yaml` — Deployment de la versión green,
    desplegada en reserva a la espera de la conmutación.
- **Ningún secreto en Git.** Las credenciales se referencian por el nombre del
  Secret; su valor se crea en el cluster o lo sincroniza un gestor de secretos.
