# Keycloak Open Policy Integration Demo

Example code for integrating Open Policy Agent with Keycloak, originally presented at Keycloak Dev Day 2024.

This repository was adapted and validated for Red Hat Build of Keycloak 26.6 in a local runtime environment. The original work and upstream project remain credited to the original author and project maintainers.

Original project: https://github.com/thomasdarimont/keycloak-opa-authz-demo

[Slides](keycloak-devday-2024-flexible-authz-for-keycloak-with-openpolicyagent.pdf)

## Attribution and validation

This fork is based on work originally created by Thomas Darimont and the Keycloak OPA demo project.

The migration, validation, and documentation updates were supported with AI assistance and reviewed/verified by a human operator before being retained in this repository.

## Status

This repo has been validated against Red Hat Build of Keycloak (RHBK) 26.6 and the runtime stack was confirmed working in this environment.

The verified runtime stack is:

- Keycloak image: `registry.redhat.io/rhbk/keycloak-rhel9:26.6`
- OPA: `openpolicyagent/opa:0.62.1`
- keycloak-config-cli: compatible image for Keycloak 26.x
- Build target: Keycloak 26.6.0

---

## Build

```bash
mvn clean package -DskipTests
```

This produced the expected jar artifact:

```text
target/keycloak-opa.jar
```

---

## Run with HTTP

This project was validated with Podman in the local environment, but the equivalent Docker command is also supported when Docker is available.

### Podman

```bash
podman compose -f dev/docker-compose.yml up -d --force-recreate
```

### Docker

```bash
docker compose -f dev/docker-compose.yml up
```

Once the stack is up, the admin console is available at:

```text
http://localhost:8080/auth
```

Default admin credentials used in the demo:

```text
admin / admin
```

---

## Run with HTTPS

This example uses `https://id.kubecon.test:8443/auth` as the Keycloak auth server URL.

To use HTTPS, add a mapping for `id.kubecon.test` to your `/etc/hosts` file and regenerate certificates with [mkcert](https://github.com/FiloSottile/mkcert).

```bash
(cd dev/config/certs && mkcert -install && mkcert -cert-file kubecon.pem -key-file kubecon-key.pem "*.kubecon.test")
```

Then start the HTTPS compose file:

```bash
docker compose -f dev/docker-compose-https.yml up
```

---

## Demo realm

The demo contains a realm named `opademo`, provisioned by `dev/config/realms/opademo.yaml` through [keycloak-config-cli](https://github.com/adorsys/keycloak-config-cli).

### Users

The demo includes the following users:

- Username: `tester` / Password: `test`
- Username: `admin` / Password: `test`
- Username: `guest` / Password: `test`

### Clients

The `opademo` realm includes several sample client applications to demonstrate access decisions expressed in REGO via OPA.

---

## OPA access policy

The client access policies are defined in:

```text
dev/opa/policies/keycloak/realms/opademo/access/policy.rego
```

OPA watches the file for updates and is configured to evaluate access decisions through the Keycloak SPI.

To enable the policy check in the realm UI, navigate to:

```text
opademo Realm -> Authentication -> Required Actions -> Enable OPA Policy Check
```

---

## Realm configuration

The realm, clients, roles, groups and users are defined in:

```text
dev/config/realms/opademo.yaml
```

To adjust access behavior for the `tester` user, uncomment the relevant role or group assignment in `opademo.yaml` and re-import the realm with the provisioner.

Typical re-import/restart flow:

```bash
podman compose -f dev/docker-compose.yml up -d --force-recreate
```

---

## Verified runtime checks

The following checks were executed successfully in this environment:

```bash
mvn clean package -DskipTests
podman compose -f dev/docker-compose.yml up -d --force-recreate
curl -sS http://localhost:8181/health
curl -sS http://localhost:8181/v1/data/keycloak/realms/opademo/access
curl -sS -X POST 'http://localhost:8080/auth/realms/opademo/protocol/openid-connect/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'client_id=admin-cli' \
  --data-urlencode 'grant_type=password' \
  --data-urlencode 'username=tester' \
  --data-urlencode 'password=test'
```

Observed results in the verification run:

- `BUILD SUCCESS`
- Keycloak reported version `26.6.7.redhat-00003`
- OPA responded with data under `keycloak/realms/opademo`
- token request returned a valid bearer token

---

## Notes for migration to RHBK 26.x

The main migration issues were not the Java build itself, but runtime compatibility items such as:

- wrong Red Hat image reference
- deprecated runtime flags incompatible with Keycloak 26.x
- incompatibility between the old provisioner and the Keycloak 26.x target
- fragile local JAR dependency patterns

The repo was adjusted to work with the supported enterprise image and a compatible provisioning flow.