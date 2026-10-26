# Deploying the Salesforce → SAP order demo to CloudHub 2.0

Two synchronous apps, shared space **us-east-1**:

```
  SFDC ──HTTPS──▶  sfdc-order-intake-api   ──HTTPS/443──▶  sap-order-system-api  ──JCo/RFC──▶  SAP ECC
  (front door)     (App1, public ingress)                 (App3, the only        (BAPI validate
                                                            SAP-touching app)      + create + commit)
```

- **App3 — `sap-order-system-api`**: validate-then-create against SAP (BAPI `..._CREATEFROMDAT2` TESTRUN → real + `BAPI_TRANSACTION_COMMIT`). Returns 201 / 422 / 503.
- **App1 — `sfdc-order-intake-api`**: the endpoint SFDC calls. Forwards to App3 over **public HTTPS** and passes the result straight back (adds a `trackingId`).

App1 calls App3 by its **public CloudHub URL**, so **App3 must be deployed first** to know that host.

---

## 0. Prerequisites

- **Java 17** for the build (`export JAVA_HOME=/Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home`).
- A **Connected App** (client credentials grant) in the target org with **Runtime Manager → deploy** and **Exchange** permissions. You need its **client id** and **secret**.
- Your business group ID (pass `-Danypoint.bg=<id>`) and environment, default `Sandbox` (override with `-Danypoint.env=`).
- Each app's `src/main/resources/secrets.yaml` filled in (copy `secrets.example.yaml`). It is gitignored but gets bundled into the jar at build time.
- Target: `Cloudhub-US-East-1` (shared space, us-east-1) — already set as `ch2.target` in both POMs.
- The `~/.m2/settings.xml` EE repo credentials only reach already-cached artifacts. All required artifacts (SAP connector 5.9.13, JCo 3.1.13 incl. the **Linux `.so`** native lib, http-connector 1.11.1) are already in `.m2`, so an **offline-capable** build works.

### The OS-specific JCo native library

The JCo native binary differs per OS. App3's POM handles this with Maven profiles:

| Profile | Native lib packaged | Use for |
|---|---|---|
| `local` (default) | `libsapjco3-3.1.13-external-library.dylib` | macOS standalone runtime |
| `ch2` | `libsapjco3-3.1.13-external-library.so` | CloudHub 2.0 (Linux x86_64) |

**Always build App3 for CH2.0 with `-Pch2`** so the Linux `.so` is bundled instead of the Mac `.dylib`. Omitting it ships a Mac binary that won't load on CloudHub.

---

## 1. Deploy App3 — `sap-order-system-api` (first)

```bash
export JAVA_HOME=/Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home
cd sap-order-system-api

mvn -Pch2 clean deploy -DmuleDeploy \
  -Danypoint.client.id=<CONNECTED_APP_CLIENT_ID> \
  -Danypoint.client.secret=<CONNECTED_APP_CLIENT_SECRET> \
  -Danypoint.env=Sandbox \
  -Danypoint.bg=<BUSINESS_GROUP_ID> \
  -Dsap.password='<SAP_PASSWORD>'
```

What this does:
- Packages the app **with the Linux `.so`** JCo native lib (`-Pch2`).
- Deploys to CH2.0 shared us-east-1 as `sap-order-system-api`, **Java 17**, 1 replica @ 0.1 vCore.
- Container listens on **8081** (overridden from the local `8091` via the POM's `cloudhub2Deployment/<properties>`).
- `generateDefaultPublicUrl=true` gives it a public HTTPS endpoint.
- `sap.password` is registered as a **secure property** — pass its real value with `-Dsap.password=` (masked in Runtime Manager). Other SAP settings come from `config.yaml`.

### Capture App3's public host

After deploy, from Runtime Manager (or the CLI output) copy the app's public URL, e.g.:

```
https://sap-order-system-api-<random>.<zone>.usa-e1.cloudhub.io
```

You need the **host only** (no scheme, no path) for the next step.

### Verify App3 is up

```bash
curl -s https://sap-order-system-api-<random>.<zone>.usa-e1.cloudhub.io/ping
# → { "app":"sap-order-system-api", "status":"OK", "message":"SAP ECC connection verified via RFC_PING", ... }
```

> ⚠️ **SAP reachability from CloudHub.** `/ping` proves the CH2.0 worker can reach the SAP gateway (RFC on 33xx). A shared demo SAP system can be **intermittently down** at the RFC/JCo layer (independent of SAP GUI login). If `/ping` returns 503, the SAP side is down — retry later; it is **not** a deploy problem. Confirm `/ping` is green before running the intake test.

---

## 2. Deploy App1 — `sfdc-order-intake-api` (second)

Pass App3's public host so intake calls it over HTTPS/443:

```bash
export JAVA_HOME=/Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home
cd sfdc-order-intake-api

mvn -Pch2 clean deploy -DmuleDeploy \
  -Danypoint.client.id=<CONNECTED_APP_CLIENT_ID> \
  -Danypoint.client.secret=<CONNECTED_APP_CLIENT_SECRET> \
  -Danypoint.env=Sandbox \
  -Danypoint.bg=<BUSINESS_GROUP_ID> \
  -Dsystemapi.host=sap-order-system-api-<random>.<zone>.usa-e1.cloudhub.io
```

What this does:
- Deploys `sfdc-order-intake-api`, **Java 17**, 1 replica @ 0.1 vCore, container on **8081**, public URL enabled.
- Sets the downstream target via `cloudhub2Deployment/<properties>`: `systemApi.protocol=HTTPS`, `systemApi.port=443`, `systemApi.path=/api/salesorders`, and `systemApi.host=<the value of -Dsystemapi.host>`.
- The HTTPS request-config uses the **default JVM trust store** (public CAs, which trust `*.cloudhub.io`) — no custom TLS context needed.

> App1 has no `-Pch2` native-lib concern (it never touches SAP/JCo), but pass `-Pch2` for consistency; only the App3 profile changes what's packaged.

---

## 3. End-to-end verification

```bash
INTAKE=https://sfdc-order-intake-api-<random>.<zone>.usa-e1.cloudhub.io

# Health
curl -s $INTAKE/health
# → { "app":"sfdc-order-intake-api", "status":"UP" }

# Happy path — valid demo master data → 201 + SAP sales-document number
curl -s -X POST $INTAKE/orders -H 'Content-Type: application/json' \
  -d '{"soldTo":"0000099999","items":[{"material":"B-7000","quantity":"1","unit":"ST","plant":"1000"}]}'
# → 201  { "validated":true, "submitted":true, "salesDocument":"00001259xx", "trackingId":"..." }

# Rejection path — bad material → 422 (validated:false, not created)
curl -s -X POST $INTAKE/orders -H 'Content-Type: application/json' \
  -d '{"soldTo":"0000099999","items":[{"material":"NOPE","quantity":"1","unit":"ST","plant":"1000"}]}'
# → 422  { "validated":false, "submitted":false, ... }
```

Response codes end-to-end (App3's status passes through App1 unchanged):

| Code | Meaning |
|---|---|
| 201 | Order validated **and** created + committed in SAP |
| 422 | Rejected by SAP validation (TESTRUN) — nothing created |
| 503 | SAP or System API unavailable (`retryable: true`) |

An empty body `{}` is accepted too — App3 applies demo-safe defaults (sold-to `0000099999`, material `B-7000`, dates pinned to FY1998, which is the era this frozen demo system supports).

---

## 4. Notes / production hardening

- **Secrets.** Credentials live in each app's gitignored `secrets.yaml` (plaintext, demo only). In production use **secure properties** (`sap.password` is already declared secure in the System API's POM — extend to `sap.user`/host as needed) and never commit real credentials.
- **Maven deploy needs the jar in Exchange.** CloudHub 2.0 pulls the app from Exchange, so `-DmuleDeploy` only works if the POM's `groupId` is your business group ID and the artifact is published to Exchange first. Otherwise you get `404 There is no asset matching given parameters`. The simplest alternative is to build with `mvn -Pch2 clean package` and upload `target/*-mule-application.jar` in Runtime Manager.
- **Deploy order matters.** Re-deploying App3 keeps its public host (stable per app name), so you normally don't need to re-deploy App1. If you rename App3 or move regions, re-run step 2 with the new `-Dsystemapi.host`.
- **Scaling.** `ch2.replicas` / `ch2.vcores` are POM properties — override at deploy (`-Dch2.replicas=2 -Dch2.vcores=0.2`) for HA.
- **Java version** is pinned to 17 via the top-level `<javaVersion>17</javaVersion>` in each `cloudhub2Deployment` (it is **not** valid inside `<deploymentSettings>` for mule-maven-plugin 4.8.0).
