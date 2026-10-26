# Salesforce Order → SAP ECC (MuleSoft demo)

Real-time, event-driven order sync from Salesforce into SAP ECC, built with two Mule 4 apps.

When an order is **Activated** in Salesforce, a Change Data Capture event triggers the flow. The order is enriched, **validated by SAP in dry-run mode** (BAPI `TESTRUN`), created and committed in SAP only if validation passes, and the outcome is written back onto the Salesforce order.

```
Salesforce ──CDC──▶ sfdc-order-intake-api ──HTTPS──▶ sap-order-system-api ──JCo/RFC──▶ SAP ECC
 (Order activated)   subscribe · enrich · gate · map    validate (TESTRUN) → create + commit
        ▲                                                         │
        └──── write-back: SAP_Sync_Status__c · SAP_Sales_Document__c · SAP_Sync_Message__c
```

| Outcome | HTTP | Salesforce `SAP_Sync_Status__c` | SAP |
|---|---|---|---|
| Order passes SAP validation | `201` | `Synced` + SAP document number | Order created and committed |
| SAP rejects the order | `422` | `Rejected` + SAP's error message | Nothing created |
| SAP unreachable | `503` | (retryable) | Nothing created |

## Repository layout

| Folder | What it is |
|---|---|
| [`sfdc-order-intake-api/`](sfdc-order-intake-api) | Owns Salesforce. Subscribes to `/data/OrderChangeEvent`, queries the order and line items with SOQL, processes only orders that are `Activated` and not yet synced, maps lines, calls the System API, and writes the result back. Also exposes `GET /health` and `POST /orders` for testing without Salesforce. |
| [`sap-order-system-api/`](sap-order-system-api) | Owns SAP, and is the only app that talks to it. `GET /ping`, `GET /api/salesorders/{salesDoc}`, `POST /api/salesorders`. See [`API.md`](sap-order-system-api/API.md) for the full contract. |
| [`DEPLOY-CH2.md`](DEPLOY-CH2.md) | Step-by-step CloudHub 2.0 deployment and end-to-end checks. |

## The SAP system

The demo runs against **MuleSoft's internal demo SAP ECC system, `sapidp.demos.mulesoft.com`** (system number `00`, client `800`). It's an IDES-style training system. Keep in mind:

- **Access is internal to MuleSoft.** This repo doesn't include the credentials. MuleSoft employees can get them from the internal demo-systems team. Everyone else needs their own SAP ECC system (see [Using a different SAP system](#using-a-different-sap-system)).
- **It goes down now and then.** If `GET /ping` returns `503`, SAP isn't answering, so try again later. That doesn't mean your setup is broken.
- **It's frozen in fiscal year 1998**, and only one material, **`B-7000`**, can be sold. The defaults in `config.yaml` (described below) are set for this. Every Salesforce product is ordered as `B-7000`, and the real product code and name go into SAP's Customer Material Number (`CUST_MAT35`), where you can see them in transaction VA03.
- **Every successful test creates a real sales order** in a system other people also use.

## Prerequisites

- **Java 17** and **Maven 3.8+**.
- **Mule EE runtime 4.11+**. Running locally is easiest with [Anypoint Code Builder](https://docs.mulesoft.com/anypoint-code-builder/) or Anypoint Studio, which bundle a runtime. A standalone runtime needs a valid license, and evaluation licenses expire, so renew it if the runtime logs `Couldn't validate license key!`. A **FIPS license** (`license-fips.lic`) installs and runs only once the runtime is in FIPS mode. That means the Bouncy Castle FIPS jars in `lib/boot/`, `-Dmule.security.model=fips140-2` in `wrapper.conf`, and a `java.security` that registers only the Bouncy Castle providers. Do all of that *before* `mule -installLicense`. See [FIPS 140-2 Compliance Support](https://docs.mulesoft.com/mule-runtime/latest/fips-140-2-compliance-support). Both apps were tested in FIPS mode.
- **MuleSoft Enterprise repository credentials** in `~/.m2/settings.xml` (server id `MuleRepository`). The SAP connector is an Enterprise connector.
- **SAP JCo 3.1.x**, downloaded from SAP with an SAP support account (it isn't in any public Maven repo). Install it into your local Maven repo; see [SAP JCo](#sap-jco) below.
- **macOS or Linux.** The POM bundles the JCo native library for macOS (`.dylib`, the default `local` profile) and Linux (`.so`, the `ch2` profile). **Windows isn't supported**, because there's no `.dll` profile.
- A **Salesforce org** (a Developer Edition org works) with the setup below.

### Salesforce setup

1. **Turn on Orders:** Setup → **Order Settings** → *Enable Orders*.
2. **Turn on CDC for Order:** Setup → **Change Data Capture** → move **Order** to *Selected Entities*.
3. **Add three custom fields on Order** (Setup → Object Manager → Order → Fields & Relationships):

   | Label | API name | Type |
   |---|---|---|
   | SAP Sync Status | `SAP_Sync_Status__c` | Text(20) |
   | SAP Sales Document | `SAP_Sales_Document__c` | Text(20) |
   | SAP Sync Message | `SAP_Sync_Message__c` | Long Text Area(2000) |

   Give the integration user read/edit access to them.
4. **Create products to order.** Each product needs a **Product Code** and an active entry in the **Standard Price Book**. For the rejection demo, add one product whose code starts with `ZZ-` (for example `ZZ-INVALID`, named "Discontinued Item").
5. **Integration user.** The app logs in with username and password through SOAP login, using **API version 64.0 or lower** (Basic Auth isn't available from v65 onward). If the machine running Mule isn't in the org's Trusted IP Ranges, put the user's **security token** in `sfdc.securityToken` (see step 1 below).

### SAP JCo

Download JCo 3.1.x for your OS from the SAP Support Portal, then install the jar and the native library under the coordinates the POM expects. Install the `.dylib` on macOS, and always install the `.so`, since CloudHub 2.0 runs on Linux:

```bash
V=3.1.13
mvn install:install-file -Dfile=sapjco3.jar -DgroupId=com.sap.conn.jco -DartifactId=com.sap.conn.jco.sapjco3 -Dversion=$V -Dpackaging=jar
mvn install:install-file -Dfile=libsapjco3.dylib -DgroupId=com.sap.conn.jco -DartifactId=libsapjco3 -Dversion=$V -Dclassifier=external-library -Dpackaging=dylib
mvn install:install-file -Dfile=libsapjco3.so    -DgroupId=com.sap.conn.jco -DartifactId=libsapjco3 -Dversion=$V -Dclassifier=external-library -Dpackaging=so
# The SAP connector also needs the IDoc library:
mvn install:install-file -Dfile=sapidoc3.jar -DgroupId=com.sap.conn.idoc -DartifactId=com.sap.conn.idoc.sapidoc3 -Dversion=3.1.4 -Dpackaging=jar
```

If your versions differ, update `sapjco.version` and `sapidoc.version` in `sap-order-system-api/pom.xml`.

## Getting started (local)

### 1. Add your credentials

Each app reads its settings from `src/main/resources/config.yaml`, and credentials from a **gitignored** `secrets.yaml` next to it:

```bash
cp sap-order-system-api/src/main/resources/secrets.example.yaml  sap-order-system-api/src/main/resources/secrets.yaml
cp sfdc-order-intake-api/src/main/resources/secrets.example.yaml sfdc-order-intake-api/src/main/resources/secrets.yaml
```

Then fill in:
- **`sap-order-system-api/.../secrets.yaml`**: `sap.applicationServerHost` (`sapidp.demos.mulesoft.com` for the internal demo SAP), `sap.user`, `sap.password`.
- **`sfdc-order-intake-api/.../secrets.yaml`**: `sfdc.username`, `sfdc.password`, `sfdc.securityToken` (leave `""` if your IP is trusted), and `systemApi.host` (`localhost` for a local run).

For a **local** run, also point the intake app at the local System API. In `sfdc-order-intake-api/src/main/resources/config.yaml`, set `systemApi.protocol: "HTTP"` and `systemApi.port: "8091"`.

> `secrets.yaml` is bundled into the jar at build time. Don't share built jars, and use secure properties for anything beyond a demo.

### 2. Build

```bash
export JAVA_HOME=/path/to/jdk-17
(cd sap-order-system-api  && mvn clean package -DskipTests)    # on Linux add -Pch2
(cd sfdc-order-intake-api && mvn clean package -DskipTests)
```

### 3. Run

Run both apps from Code Builder or Studio, or copy both `target/*-mule-application.jar` files into a standalone runtime's `apps/` folder. Locally, the System API listens on **8091** and the intake API on **8071**.

> **Only run one copy of the intake app per Salesforce org.** Every running copy receives the same CDC events, so if one is also deployed on CloudHub, each activated order is sent to SAP twice. Stop the other copy first.

### 4. Try it

```bash
# SAP connectivity
curl -s http://localhost:8091/ping

# Create an order directly in SAP (every field is optional)
curl -s -X POST http://localhost:8091/api/salesorders -H 'Content-Type: application/json' \
  -d '{"poNumber":"DEMO-PO-001","items":[{"material":"B-7000","quantity":"2","unit":"ST","plant":"1000"}]}'

# Read it back (use the salesDocument number from the response above)
curl -s http://localhost:8091/api/salesorders/<salesDocument>

# Rejection path: an unknown material fails SAP validation → 422, nothing created
curl -s -X POST http://localhost:8091/api/salesorders -H 'Content-Type: application/json' \
  -d '{"items":[{"material":"ZZ-INVALID","quantity":"1"}]}'
```

**End to end from Salesforce:** create an Order with at least one product and set its Status to **Activated**. Within a few seconds the order shows `SAP_Sync_Status__c = Synced` and an SAP document number. An order that includes the `ZZ-` product comes back `Rejected` with SAP's error message instead.

## Configuration

Both apps keep their SAP-specific values in `config.yaml`, so you can change them without editing code.

**`sap-order-system-api` → `orderDefaults.*`.** Used when a `POST /api/salesorders` request leaves a field out. Any field in the request overrides it.

| Setting | Demo value | Meaning |
|---|---|---|
| `docType` | `TA` | Sales document type (standard order) |
| `salesOrg` / `distrChannel` / `division` | `1000` / `10` / `00` | Sales area |
| `soldTo` | `0000099999` | Sold-to customer number |
| `poNumber` | `DEMO-PO-001` | Customer PO number |
| `docDate` / `priceDate` / `reqDeliveryDate` | `19980605` / `19980605` / `19980609` | Dates (YYYYMMDD), pinned to FY1998 for the demo system |
| `material` / `plant` / `quantity` / `unit` | `B-7000` / `1000` / `1` / `ST` | Line item defaults |

**`sfdc-order-intake-api` → `sapMapping.*`.** Controls how Salesforce order lines become SAP lines.

| Setting | Demo value | Meaning |
|---|---|---|
| `material` | `B-7000` | SAP material ordered for every Salesforce product. Set to `""` to send each product's own Product Code as the SAP material. |
| `passthroughPrefix` | `ZZ-` | Product codes with this prefix go to SAP unchanged, so SAP rejects them. Set to `""` to turn this off. |
| `unit` / `plant` | `ST` / `1000` | Unit of measure and plant for every line |

### Using a different SAP system

1. Set `sap.applicationServerHost`, `sap.user` and `sap.password` in `secrets.yaml`, and `sap.systemNumber` and `sap.client` in `config.yaml`.
2. Change `orderDefaults.*` to values that exist in your system (sales area, a real sold-to customer, a plant, current dates).
3. Change `sapMapping.*`: either one material your system can sell, or `material: ""` if your Salesforce Product Codes match SAP material numbers.

## Scaling beyond the demo

The flow is synchronous and handles one order at a time. For high volume, put a queue (e.g. Anypoint MQ) between the two apps and use idempotent consumers. Also cap concurrent SAP calls to what SAP can handle, and add retries with a dead-letter queue.

## Deploying to CloudHub 2.0

See [`DEPLOY-CH2.md`](DEPLOY-CH2.md). In short: deploy the System API first (built with **`-Pch2`** so the Linux JCo library is included), then the intake API pointing at its public host.

## Tests

`sap-order-system-api/src/test/munit` has MUnit tests that mock SAP, covering the happy path, rejection, ping and order status.

> **Known issue:** `mvn test` currently fails before any test runs, with `NoClassDefFoundError: org/apache/commons/io/output/ByteArrayOutputStream`. The SAP connector's commons-io classes don't load in MUnit's embedded runtime. The apps themselves run fine. Until this is fixed, check behaviour with the `curl` commands above.

## Troubleshooting

### Build

| Symptom | Cause and fix |
|---|---|
| `Could not transfer artifact … from/to MuleRepository … 401, reason phrase: Unauthorized` | Your MuleSoft Enterprise repository credentials are missing or wrong. Add them to `~/.m2/settings.xml` under `<server><id>MuleRepository</id>…</server>`. |
| `Could not find artifact com.sap.conn.jco:com.sap.conn.jco.sapjco3:jar:3.1.13` (or `libsapjco3`, `sapidoc3`) | JCo isn't installed in your local Maven repo. Follow [SAP JCo](#sap-jco). The `-Dversion` values must match `sapjco.version` / `sapidoc.version` in the POM. |
| Build picks up the wrong Java | Set `JAVA_HOME` to a JDK 17 before running Maven. Check with `mvn -v`. |

### App won't start

| Symptom | Cause and fix |
|---|---|
| `Couldn't find configuration property value for key ${sap.password}` (or `${sfdc.username}`, `${systemApi.host}`, …) | `secrets.yaml` is missing, or a key isn't filled in. Copy `secrets.example.yaml` to `secrets.yaml`, fill it in, and **rebuild**, since the file is bundled into the jar. |
| `UnsatisfiedLinkError … sapjco3` or `JCo initialization failed` | The JCo native library doesn't match the machine. Build with the default profile on macOS and **`-Pch2` on Linux or CloudHub 2.0**. On Apple Silicon, use the **arm64** (`darwinarm64`) JCo build, not x86_64. |
| `Couldn't validate license key!` | The runtime's license has expired or isn't installed. Evaluation licenses last 30 days. A FIPS license needs FIPS mode first (see [Prerequisites](#prerequisites)). |
| `Address already in use` on 8091 or 8071 | Another app is using the port. Stop it, or change `http.port` in that app's `config.yaml`. |

### SAP

| Symptom | Cause and fix |
|---|---|
| `GET /ping` returns `503` | SAP can't be reached or isn't answering. Check `sap.applicationServerHost`, `sap.systemNumber` and `sap.client`, and that your machine can reach the SAP host on ports **32NN and 33NN** (NN = system number) through your network or VPN. The internal demo SAP is sometimes down; if so, try again later. |
| `503` with a logon error in the logs | Wrong `sap.user` or `sap.password`, or the user is locked. |
| **Every** order comes back `422` with "Material … is not defined for sales org …" | The demo material `B-7000` doesn't exist in your SAP system. Set `sapMapping.material` (intake app) and `orderDefaults.material` (System API) to a material your sales area can sell. If your Salesforce Product Codes match SAP material numbers, set `sapMapping.material: ""` instead. |
| `422` mentioning the sold-to party, sales area or plant | `orderDefaults.soldTo`, `salesOrg` / `distrChannel` / `division` or `plant` don't exist, or don't belong together, in your system. Use values from your SAP master data. |
| `422` about dates or a closed period | The default dates are pinned to 1998 for the demo system. On a normal system, set `orderDefaults.docDate`, `priceDate` and `reqDeliveryDate` to current dates. |
| `422` saying you're not authorized | The SAP user lacks authorization for sales order creation (VA01) or the BAPIs listed under [Prerequisites](#prerequisites). Ask your SAP basis team. |

### Salesforce

| Symptom | Cause and fix |
|---|---|
| `INVALID_LOGIN: Invalid username, password, security token; or user locked out` | Check `sfdc.username` and `sfdc.password`. If the Mule machine's IP isn't trusted by the org, put the user's security token in `sfdc.securityToken` (reset it from the user's personal settings). |
| Login fails with a message that **SOAP API `login()` is disabled** | Salesforce is retiring SOAP `login()`, and newer orgs may have it turned off. Search Setup for "SOAP API login" and re-enable it for the integration user, if your org allows that. Otherwise, switch the Salesforce connection to OAuth (JWT bearer or client credentials). |
| `SOAP Login operation is not available in the API version specified` | `sfdc.loginUrl` uses API v65 or higher. Keep it at `…/Soap/u/64.0` or lower. |
| `No such column 'SAP_Sync_Status__c' on entity 'Order'` (or `INVALID_FIELD`) | A custom field is missing, or the integration user can't see it. Check the three fields and their field-level security. |
| Activating an order does nothing (no `[CDC] Order ACTIVATED` line in the intake log) | Make sure Order is selected under **Change Data Capture** and the intake app is running. Also, an order is processed only once: if `SAP_Sync_Status__c` already has a value, the event is skipped. Clear the field to resend it. |
| Salesforce won't let you activate the order | The order needs at least one product, and each product needs an active Standard Price Book entry. |
| Each order is created in SAP **twice** | Two copies of the intake app are subscribed to the same org (for example, one local and one on CloudHub). Stop one. |
| `[CDC] System API unreachable` in the intake log | The intake app can't reach the System API. Locally, use `systemApi.protocol: "HTTP"`, `port: "8091"`, `host: "localhost"`. On CloudHub 2.0, use HTTPS, port 443, and the System API's public host. |
