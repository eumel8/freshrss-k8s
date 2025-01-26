# freshrss-k8s

Kubernetes deployment of [FreshRSS](https://github.com/FreshRSS/FreshRSS), a web-based multi-source RSS-Reader.

## Intro

This project contains manifests to deploy the app in a secure way on K8S. For this an extra container image will build for

- readOnlyRootFilesystem into container, separate live data on PVC
- run the app on non-priviledged port and non-priviledged user
- focus on externel authentification with Keycloak (OIDC), all member of a group claim should use the same account
- data store on PVC, held also the Sqlite database file
- (optional), Slack2RSS-reader

## Builds

The repo provides 2 container images on 2 places:

- [fresh-rss mtr](https://mtr.devops.telekom.de/repository/mcsps/fresh-rss)
- [fresh-rss ghr](https://github.com/users/eumel8/packages/container/package/freshrss-k8s/fresh-rss)
- [python-curl mtr](https://mtr.devops.telekom.de/repository/mcsps/python-curl)
- [python-curl ghr](https://github.com/users/eumel8/packages/container/package/freshrss-k8s/python-curl)

## Installation

Create a namespace on the target cluster (tested Kubernetes 1.29.4)

```bash
kubectl create namespace fresh-rss
```

Create a secret based on your needs (skip OIDC keys or set `OIDC_ENABLED=0`).

Assume you have a Keycloak instance with a realm `cloud` and a group `devops` which use the `REMOTE_USER devops`. This user can be the admin user from the FreshRSS installation before you switch to `REMOTE_USER` authentification in FreshRSS:

```bash
kubectl -n fresh-rss create secret generic rss-app --dry-run=client -o yaml \
  --from-literal=OIDC_CLAIM='claim groups:devops' \
  --from-literal=OIDC_DEFAULT_URL='https://rss-app.example.com' \
  --from-literal=OIDC_ENABLED='1' \
  --from-literal=OIDC_PROVIDER_METADATA_URL='https://keycloak.example.com/auth/realms/cloud/.well-known/openid-configuration' \
  --from-literal=OIDC_CLIENT_ID='rss-app' \
  --from-literal=OIDC_CLIENT_SECRET='xxxxxxxxxxxxx' \
  --from-literal=OIDC_REMOTE_USER_CLAIM='devops' \
  --from-literal=OIDC_CLIENT_CRYPTO_KEY='12345' \
  --from-literal=OIDC_X_FORWARDED_HEADERS='X-Forwarded-Host X-Forwarded-Port X-Forwarded-Proto' \
  --from-literal=OIDC_SCOPES='openid'
```


You can create `create-secret.sh` with the content above and apply this with

```bash
sh create-secret.sh | kubectl apply -f -
```

Create PVC (review StorageClass), Deployment, Service:

```bash
kubectl -n fresh-rss apply -f Kubernetes/pvc.yaml
kubectl -n fresh-rss apply -f Kubernetes/deployment.yaml
kubectl -n fresh-rss apply -f Kubernetes/service.yaml
```

(optional) Create Ingress/Certificate. Example files for ingress-nginx and cert-manager with existing ClusterIssuer:

```bash
kubectl -n fresh-rss apply -f Kubernetes/ingress.yaml
kubectl -n fresh-rss apply -f Kubernetes/certificate.yaml
```

## Sync

Call a sync script periodically to refresh all RSS feeds automatically:

```bash
kubectl -n fresh-rss apply -f Kubernetes/sync.yaml
```

## Keycloak

Example snippet of a Keycloak client:

```json
{
  "clientId": "rss-app",
  "name": "",
  "description": "",
  "rootUrl": "https://rss-app.example.com",
  "adminUrl": "https://rss-app.example.com",
  "baseUrl": "https://rss-app.example.com",
  "surrogateAuthRequired": false,
  "enabled": true,
  "alwaysDisplayInConsole": false,
  "clientAuthenticatorType": "client-secret",
  "secret": "xxxxxxxxxxxxxxxxx",
  "redirectUris": [
    "https://rss-app.example.com:443/i/oidc/"
  ],
  "webOrigins": [
    "https://rss-app.example.com"
  ],
  "notBefore": 0,
  "bearerOnly": false,
  "consentRequired": false,
  "standardFlowEnabled": true,
  "implicitFlowEnabled": false,
  "directAccessGrantsEnabled": true,
  "serviceAccountsEnabled": false,
  "publicClient": false,
  "frontchannelLogout": false,
  "protocol": "openid-connect",
  "attributes": {
    "saml.assertion.signature": "false",
    "saml.multivalued.roles": "false",
    "saml.force.post.binding": "false",
    "saml.encrypt": "false",
    "oauth2.device.authorization.grant.enabled": "false",
    "backchannel.logout.revoke.offline.tokens": "false",
    "saml.server.signature": "false",
    "saml.server.signature.keyinfo.ext": "false",
    "exclude.session.state.from.auth.response": "false",
    "oidc.ciba.grant.enabled": "false",
    "backchannel.logout.session.required": "true",
    "saml_force_name_id_format": "false",
    "saml.client.signature": "false",
    "tls.client.certificate.bound.access.tokens": "false",
    "saml.authnstatement": "false",
    "display.on.consent.screen": "false",
    "saml.onetimeuse.condition": "false",
    "login_theme": "",
    "backchannel.logout.url": "",
    "post.logout.redirect.uris": "https://rss-app.example.com/i/"
  },
  "authenticationFlowBindingOverrides": {},
  "fullScopeAllowed": true,
  "nodeReRegistrationTimeout": -1,
  "protocolMappers": [
    {
      "name": "devops",
      "protocol": "openid-connect",
      "protocolMapper": "oidc-hardcoded-claim-mapper",
      "consentRequired": false,
      "config": {
        "introspection.token.claim": "true",
        "claim.value": "devops",
        "userinfo.token.claim": "true",
        "id.token.claim": "true",
        "lightweight.claim": "false",
        "access.token.claim": "true",
        "claim.name": "devops",
        "jsonType.label": "String",
        "access.tokenResponse.claim": "false"
      }
    },
    {
      "name": "groups",
      "protocol": "openid-connect",
      "protocolMapper": "oidc-group-membership-mapper",
      "consentRequired": false,
      "config": {
        "full.path": "false",
        "introspection.token.claim": "true",
        "userinfo.token.claim": "true",
        "id.token.claim": "true",
        "lightweight.claim": "false",
        "access.token.claim": "true",
        "claim.name": "groups"
      }
    }
  ],
  "defaultClientScopes": [
    "web-origins",
    "profile",
    "roles",
    "email"
  ],
  "optionalClientScopes": [
    "address",
    "phone",
    "offline_access",
    "microprofile-jwt"
  ],
  "access": {
    "view": true,
    "configure": true,
    "manage": true
  },
  "authorizationServicesEnabled": false
}
```

## OpenID

- [FreshRSS OpenID Connect](https://freshrss.github.io/FreshRSS/en/admins/16_OpenID-Connect.html)
- [Apach OIDC modul](https://auth.docs.cern.ch/user-documentation/oidc/apache/)

## Slack2RSS-reader

Create a Slack app, invite the app into a channel. This script fetch conversation from the channel

run.sh:

```bash
#!/bin/sh
if [ ! -d '/data/p/slack' ]; then
        mkdir -p /data/p/slack
fi
curl -q -k -X GET 'https://slack.com/api/conversations.history?channel=XXXXX' \
-H 'Authorization: Bearer xxxxxxxxxxxxxxxxxxxxxxx' \
-H 'Content-Type: application/json' -o /data/p/slack/slack.json

GIT_TOKEN=xxxxxx
#GIT_REPO=https://gitlab.com/my/repo
python /slack/slack.py && exit 0
python /slack/gitlog-json.py && exit 0
python /slack/gitlog-xml.py && exit 0
```

Create a secret based on this script:

```bash
kubectl -n fresh-rss create secret generic rss-job --dry-run=client -o yaml --from-file run.sh 
```

Create Configmap/Cronjob:

```bash
kubectl -n fresh-rss apply -f Kubernetes/configmap.yaml
kubectl -n fresh-rss apply -f Kubernetes/cronjob.yaml
```

## Credits

* Frank Kloeker <f.kloeker@telekom.de>

Life is for sharing. If you have an issue with the code or want to improve it, feel free to open an issue or an pull
request.

