# Colección Postman · Workshop Demo (OpenShift + Connectivity Link)

Colección para probar los cuatro servicios del workshop expuestos por el gateway
compartido (Gateway API + Kuadrant): servicio base, canary, blue-green y circuit
breaker. Todas las peticiones validadas en vivo contra `cluster-qthlq`.

## Importar

1. En Postman: **Import** → arrastra `workshop-demo.postman_collection.json` y
   `workshop-demo.postman_environment.json`.
2. Selecciona el environment **Workshop Demo · cluster-qthlq** (arriba a la derecha).
3. Rellena la variable `client_secret` (ver abajo) — viene vacía a propósito.
4. Ejecuta primero **00 · Autenticación** para guardar el `token`.

Para apuntar a otro cluster basta con cambiar `base_domain` (y `keycloak_host`)
en el environment.

## AVISO: certificado propio del gateway

El gateway sirve un **certificado autofirmado**. En Postman hay que desactivar la
verificación SSL: **Settings → General → SSL certificate verification → OFF**.
Si no, **todas** las peticiones fallarán con error de certificado.

## Obtener el `client_secret`

El client `user1` del realm `workshop` usa `client_credentials`. Su secret se
obtiene con la API de admin de Keycloak (credenciales admin en el cluster):

```bash
KC=https://sso.apps.cluster-qthlq.dyn.redhatworkshops.io
ADMIN_USER=$(oc get secret keycloak-initial-admin -n keycloak -o jsonpath='{.data.username}' | base64 -d)
ADMIN_PASS=$(oc get secret keycloak-initial-admin -n keycloak -o jsonpath='{.data.password}' | base64 -d)

# Token de admin (realm master)
AT=$(curl -sk "$KC/realms/master/protocol/openid-connect/token" \
  -d grant_type=password -d client_id=admin-cli \
  -d username="$ADMIN_USER" -d password="$ADMIN_PASS" | jq -r .access_token)

# UUID interno del client user1 y su secret
CID=$(curl -sk -H "Authorization: Bearer $AT" \
  "$KC/admin/realms/workshop/clients?clientId=user1" | jq -r '.[0].id')
curl -sk -H "Authorization: Bearer $AT" \
  "$KC/admin/realms/workshop/clients/$CID/client-secret" | jq -r .value
```

Copia el valor en la variable `client_secret` del environment. **Nunca** lo
commitees a Git.

## Qué esperar (comportamiento validado en vivo)

- **demo-dev** — lo sirve la app `servicio-demo`: responde **200 con el mismo
  JSON en todas las rutas** (incluida `/api/error`, que aquí NO da 500). Tiene
  **RateLimitPolicy 5 peticiones/60 s por client_id**: la 6ª petición da **429**.
- **canary-dev** — reparto ~90/10 entre `1.0.0-stable` y `2.0.0-canary`
  (20 peticiones → 17/3 en la validación). `/api/echo` y `/api/slow` dan 404
  en esta imagen; `/api/error` sí da 500.
- **bluegreen-dev** — siempre `1.0.0-blue`; con la cabecera `x-version: green`
  responde `2.0.0-green`.
- **cb-dev** — `/api/call?path=/api/error` devuelve **502** con
  `upstream_status: 500`; tras 3 errores seguidos el circuito **no queda
  abierto** (la siguiente llamada responde 200). La llamada lenta
  (`path=/api/slow%3Fs%3D3`) tarda ~3000 ms.
- **Sin token o token inválido** → **401** en todos los hosts.
- La cabecera `x-identidad` la inyecta la AuthPolicy **hacia el upstream**; el
  cliente no la ve en la respuesta (se observa vacía en el eco de `cb-dev`).

## Rate limit (carpeta 05)

Ejecuta la petición «Rate limit en demo-dev» con el **Collection Runner**
(8 iteraciones): verás 5×200 y luego 429. Espera 60 s antes para partir de
cuota limpia. El límite es por `client_id`, no por IP.
