# workshop-config — configuración declarativa del workshop

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
| `bootstrap/` | La Application raíz (patrón *app of apps*) y las Applications que componen el workshop |
| `platform/argocd/` | Configuración del propio Argo CD: el AppProject que acota qué puede desplegarse |
| `platform/pipelines/` | Los dos pipelines de Tekton (CI y CD), la identidad que los ejecuta, y en `ejecuciones/` los PipelineRun de ejemplo (se lanzan a mano, Argo no los toca) |
| `platform/connectivity-link/` | Custom Resource de Kuadrant, plano de control de Connectivity Link |
| `platform/gateway/` | Gateway compartido, TLSPolicy y AuthPolicy por defecto |
| `apps/servicio-demo/` | Deployment, Service y ConfigMap de la aplicación, con overlays `dev` y `test` |
| `policies/` | AuthPolicy y RateLimitPolicy del servicio |

Todos los directorios siguen el patrón **base + overlays** de Kustomize: la
`base` declara el *qué* y cada overlay el *dónde* y el *cuánto*.

## Instalación

La Application raíz es el **único** recurso que se aplica a mano. A partir de
ella, Argo CD despliega el resto:

```bash
oc apply -f bootstrap/root-application.yaml
```

Las Applications hijas se ordenan con `argocd.argoproj.io/sync-wave`, porque el
orden importa:

| Onda | Componente | Motivo del orden |
|---|---|---|
| 0 | Configuración de Argo CD | El AppProject debe existir antes que las Applications que lo declaran |
| 1 | Gateway compartido | Una ruta no puede adjuntarse a un gateway inexistente |
| 2 | Aplicación y pipelines | Necesitan el gateway ya disponible |
| 3 | Políticas | Se declaran sobre el HTTPRoute de la aplicación, que debe existir antes |

## Pipelines: CI y CD separados

El flujo está partido en dos pipelines, cada uno con una responsabilidad clara:

- **`ci-construir-imagen`** — clona `workshop-app`, construye la imagen con
  Buildah, la firma con Tekton Chains, la publica, y escribe el digest recién
  construido en este repositorio. Ese `git push` es lo que dispara el CD.
- **`cd-desplegar-aplicacion`** — clona este repositorio, valida que el overlay
  renderiza, y le pide a Argo CD que sincronice la Application. No reconstruye ni
  toca el código.

Cada uno se lanza con su PipelineRun de ejemplo:

```bash
oc create -f platform/pipelines/ejecuciones/ci-construir-imagen-pipelinerun.yaml -n workshop-demo-dev
```

En este entorno la imagen se publica en el **registro interno de OpenShift**
(`image-registry.openshift-image-registry.svc:5000`). En una plataforma real
sería el **registro corporativo (Quay)**, y la credencial de publicación se
inyectaría como un Secret de tipo `docker` referenciado por su **nombre** desde
el workspace de la tarea de construcción.

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

- Los manifiestos se nombran `<descriptor>-<kind>.yaml`, de modo que el nombre
  del archivo delate el tipo de recurso. Los Custom Resources de operador
  (`gateway.yaml`, `kuadrant.yaml`) llevan el nombre del recurso, que ya es
  autoexplicativo.
- **Ningún secreto en Git.** Las credenciales se referencian por el nombre del
  Secret; su valor se crea en el cluster o lo sincroniza un gestor de secretos.
