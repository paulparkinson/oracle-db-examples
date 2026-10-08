# Oracle JDBC token authentication demo

Runnable Java examples for Autonomous AI Database with OCI IAM and Microsoft Entra ID: OCI/Entra token files, OCI SDK proof-of-possession tokens, Entra SDK bearer tokens, JDBC resource providers, and a concurrent Universal Connection Pool (UCP). No database password is passed by the demo. Each successful connection executes a read-only identity query; no application tables are required.

Companion article source: [Passwordless Java connections with OCI IAM and Microsoft Entra ID](https://github.com/paulparkinson/oracledb-java-security/blob/main/java-jdbc-token-authentication-oci-iam-entra.html) (the article repository currently requires access).

**Complete workload examples:** [OCI Compute instance principal and Data Science resource principal](docs/oci-workload-principals.md), including IAM policy, global schema, full Java source, JDBC/UCP commands and explicit verification boundaries.

## Prerequisites

- JDK 17 or 21 and Maven 3.9+; a shell for `run.sh` (or run Java directly).
- An Autonomous Database configured for your selected identity provider, reachable over TCPS.
- The appropriate IAM or Entra identity, policy/app-role permission and database mapping. Follow [the complete setup guide](docs/setup.md) first. OCI IAM and Entra use separate lab database configurations.
- An OCI CLI installation for file-token examples; cloud runtime credentials for workload identities.

Pinned and build-tested: Oracle JDBC `ojdbc17` and `ucp17` **23.26.3.0.0**, OCI and Azure JDBC providers **1.1.0**, OCI SDK **3.86.2**, Azure Identity **1.18.0**. Database branding (26ai) and Maven versions are different. Provider dependencies bring their cloud SDKs. The providers declare `ojdbc8` as `provided`, so no driver exclusion is needed; this application explicitly uses `ojdbc17`. Original JARs remain separate so ServiceLoader descriptors are preserved.

## Build and configure

```bash
git clone https://github.com/paulparkinson/oracle-db-examples.git
cd oracle-db-examples/java/jdbc-token-auth
mvn -B verify
./run.sh --help

# Replace with the host, TCPS port, and service from Database Connection.
export ADB_JDBC_URL='jdbc:oracle:thin:@tcps://<adb-host>:1522/<service>'
export OCI_COMPARTMENT_ID='<compartment-ocid>'
export OCI_DATABASE_ID='<database-ocid>'
export OCI_CONFIG_FILE="$HOME/.oci/config"
export OCI_PROFILE=DEFAULT
export OCI_DB_SCOPE='urn:oracle:db::id::<compartment-ocid>::<database-ocid>'
```

Use EZConnect+ with the actual TCPS host, port, and service for your database. Do not use a naming alias or descriptor. The pinned driver enables server identity matching by default; no redundant DN-match URL flag is needed. For mTLS add wallet properties to an ignored local copy of a profile as explained in [setup](docs/setup.md#tls-and-wallets). The demo loads `${VARIABLE}` placeholders itself and rejects missing variables without logging their values. It does not load `.env` files automatically.

```bash
./run.sh config/oci-config-file.properties check
./run.sh config/oci-config-file.properties jdbc
./run.sh config/oci-config-file.properties ucp
```

`check` only validates configuration and makes no cloud/database connection. `jdbc` opens a physical connection per round. `ucp` performs four concurrent tasks through a pool of at most two connections, propagates worker failures, and destroys the pool on exit. Windows users can invoke `java -cp "target/oracle-jdbc-token-auth-demo-1.0.0.jar;target/lib/*" demo.TokenAuthDemo ...` after Maven builds it.

## 1. OCI CLI token files

```bash
umask 077
oci iam db-token get --profile "$OCI_PROFILE"
export OCI_DB_TOKEN_DIR="$HOME/.oci/db-token"
./run.sh config/oci-file.properties jdbc
./run.sh config/oci-file.properties ucp

# Alternatives: run only in a configured identity environment.
oci --auth instance_principal iam db-token get
oci --auth resource_principal iam db-token get
# A session-token profile must be created/renewed with oci session authenticate.
oci --auth security_token --profile SESSION iam db-token get
```

The directory must contain `token` and its matching `oci_db_key.pem`. JDBC reads them with `oracle.jdbc.tokenAuthentication=OCI_TOKEN` and `oracle.jdbc.tokenLocation`. These are a database token and PoP key, not the API signing key or TLS wallet. File mode does **not** invoke the CLI or acquire replacements. For this demo, stop the application, regenerate the pair, and restart. A long-lived producer needs coordinated reads/writes of both files: replacing two files individually is not an atomic pair update. Prefer SDK/provider mode for a continuously running service.

## 2. Application-owned OCI SDK acquisition

```bash
./run.sh config/oci-sdk.properties builder
./run.sh config/oci-sdk.properties jdbc 3
./run.sh config/oci-sdk.properties ucp
```

[OciTokens.java](src/main/java/demo/OciTokens.java) generates a 2048-bit RSA pair, sends the public key and database scope to the OCI identity data plane, and constructs `AccessToken.createJsonWebToken(jwt, privateKey)`. It keeps the private key in memory. `builder` supplies one newly acquired token through `createConnectionBuilder().accessToken(...)`; `jdbc` and `ucp` attach a cached supplier created by `AccessToken.createJsonWebTokenCache(...)` to the data source. No custom refresh scheduler is required.

For other SDK identities copy the profile to `config/oci-sdk.local.properties`, change `app.oci.authentication` to `session-token`, `instance-principal`, or `resource-principal`, and remove unused `app.oci.configFile` and `app.oci.profile` lines for workload identities. Session credentials must remain valid for future token requests. Do not combine the supplier with username/password or an access-token provider on the same data source.

## 3. OCI JDBC resource provider

```bash
./run.sh config/oci-config-file.properties jdbc
./run.sh config/oci-instance-principal.properties ucp
./run.sh config/oci-resource-principal.properties ucp
./run.sh config/oci-cloud-shell.properties jdbc
./run.sh config/oci-interactive.properties jdbc
```

Run the applicable command in its intended environment. Each file uses `oracle.jdbc.tokenAuthentication` to select `OCI_API_KEY`, `OCI_INSTANCE_PRINCIPAL`, `OCI_RESOURCE_PRINCIPAL`, `OCI_DELEGATION_TOKEN` (Cloud Shell), or `OCI_INTERACTIVE`. The driver selects the matching provider automatically; retain the OCI provider JAR, but omit the explicit access-token-provider properties. Set `oracle.jdbc.ociCompartment` and `oracle.jdbc.ociDatabase` to restrict scope. The API-key profile also sets `oracle.jdbc.ociConfigFile` and `oracle.jdbc.ociProfile`. The provider acquires and caches tokens. Cloud Shell uses its managed session; interactive authentication opens a browser and requires a usable desktop callback. The config-file provider path is for API-key configuration; use SDK `session-token` mode for an OCI CLI session-token profile.

## 4. Microsoft Entra ID provider

Use the URL of the **Entra-configured** database. Set these non-secret identifiers:

```bash
export AZURE_TENANT_ID='<tenant-id>'
export AZURE_CLIENT_ID='<java-client-application-id>'
export ENTRA_DB_APP_ID_URI='https://<tenant-domain>/<database-application-id>'
# SDK/file acquisition needs a full scope; provider profiles need the bare resource URI.
export ENTRA_DB_SCOPE="${ENTRA_DB_APP_ID_URI}/.default"
```

For a client secret, inject `AZURE_CLIENT_SECRET` through the runtime's secret facility. For a local Bash demonstration, read it without terminal echo or a literal in shell history:

```bash
read -r -s -p 'Client secret: ' AZURE_CLIENT_SECRET; echo
export AZURE_CLIENT_SECRET
./run.sh config/entra-service-principal.properties jdbc
./run.sh config/entra-service-principal.properties ucp
unset AZURE_CLIENT_SECRET
```

For certificate authentication, set `AZURE_CLIENT_CERTIFICATE_PATH` (and `AZURE_CLIENT_CERTIFICATE_PASSWORD` for an encrypted PFX), unset the client secret, and use the same service-principal profile. Register the corresponding certificate on the client application. The identity credential is not a database password.

Other examples:

```bash
# On Azure with a configured managed identity and database app-role assignment:
# For a system-assigned identity, unset AZURE_CLIENT_ID first.
./run.sh config/entra-managed-identity.properties ucp

# Public client with delegated permission, user role assignment and redirect URI:
# ENTRA_DB_APP_ID_URI remains the bare resource URI for both profiles.
./run.sh config/entra-interactive.properties jdbc
./run.sh config/entra-device-code.properties jdbc
```

These profiles use `oracle.jdbc.tokenAuthentication=AZURE_SERVICE_PRINCIPAL`, `AZURE_MANAGED_IDENTITY`, `AZURE_INTERACTIVE`, or `AZURE_DEVICE_CODE`, plus `oracle.jdbc.azureDatabaseApplicationIdUri`. Do not put `/.default` or `/session:scope:connect` into that resource-URI property. Public-client profiles additionally set `oracle.jdbc.tenantId`, `oracle.jdbc.clientId`, and, for browser login, `oracle.jdbc.redirectUri`. The Azure provider JAR is still required; explicit `oracle.jdbc.provider.accessToken` properties are not.

Browser/device-code paths authenticate a human; service-principal/managed-identity paths authenticate a workload. Never use the database resource ID as the Java client's ID. See [setup](docs/setup.md#microsoft-entra-id) for registration, roles, consent and token-version details.

## 5. Entra token files and application-owned SDK acquisition

The Entra SDK flow uses the same service-principal environment as section 4. Re-inject the client credential if you unset it:

```bash
./run.sh config/entra-sdk.properties builder 2
./run.sh config/entra-sdk.properties jdbc 2
./run.sh config/entra-sdk.properties ucp
```

[EntraTokens.java](src/main/java/demo/EntraTokens.java) uses Azure Identity to acquire the database-scoped JWT. It constructs `AccessToken.createJsonWebToken(jwt.toCharArray())` **without an OCI proof-of-possession key**. Builder mode supplies a token directly; JDBC and UCP share an expiry-aware cached supplier.

For other SDK credentials, copy the profile to `config/entra-sdk.local.properties` and set `app.entra.authentication` to `managed-identity`, `interactive`, `device-code`, or `azure-cli`. Human flows need `app.entra.scope=<database-URI>/session:scope:connect`; interactive/device-code also need `app.entra.tenantId` and `app.entra.clientId`. Register the public client, permission, redirect and user role first. Azure CLI uses its signed-in identity and its own client consent; a working Graph token does not prove database permission. Each credential is selected explicitly, not through an automatic credential fallback chain.

An external approved token producer can instead save **only the Entra database JWT** to a private UTF-8 file (no JSON wrapper, no OCI key):

```bash
export ENTRA_DB_TOKEN_FILE='/private/path/to/entra-db-token'
./run.sh config/entra-file.properties jdbc
./run.sh config/entra-file.properties ucp
```

Protect the directory and set file permissions to `0600`. The profile uses `oracle.jdbc.tokenAuthentication=OAUTH` and `oracle.jdbc.tokenLocation`. The producer, not JDBC file mode, obtains and replaces the token before expiry. For a CLI producer with database consent, `az account get-access-token --resource "$ENTRA_DB_APP_ID_URI" --query accessToken -o tsv` returns the JWT: redirect it to a private temporary file and atomically rename it into place; never print it or commit it. Prefer provider/SDK acquisition for a continuously running service.

## Token refresh and pool verification

```bash
# Four tasks per round; purge physical sessions between completed rounds.
# About 74 minutes between first and final rounds, spanning a typical OCI token lifetime.
./run.sh config/oci-config-file.properties ucp-fresh 75 60
# Or use config/oci-sdk.properties / config/entra-service-principal.properties.
```

Token expiry governs creation of new authenticated connections. A still-open pooled session can keep working after its login token expires, so repeated borrows alone do not prove refresh. `ucp-fresh` purges the pool after all four tasks in a round return their connections, verifies that available/borrowed counts are zero, and retains the same data source/token cache. The next round must create new authenticated physical sessions. It does not invalidate connections during concurrent borrowing. It allows a bounded 60-second replacement wait. Output includes session IDs for diagnosis, although the server can recycle those IDs. Run longer than the actual issuer token lifetime; no unit test proves cloud refresh.

Illustrative successful output (values depend on your identity/server):

```text
round=1 user=TOKEN_DEMO identity=<mapped-principal> authentication=<server-reported-method> sid=<session-id>
```

## Code map and checks

| File | Responsibility |
|---|---|
| [TokenAuthDemo.java](src/main/java/demo/TokenAuthDemo.java) | Executable JDBC/builder/UCP flows, concurrent tasks, read-only query and shutdown |
| [OciTokens.java](src/main/java/demo/OciTokens.java) | SDK identity selection, scoped token request, PoP key, cached supplier |
| [EntraTokens.java](src/main/java/demo/EntraTokens.java) | Explicit Azure Identity credential, database scope, bearer JWT, cached supplier |
| [Settings.java](src/main/java/demo/Settings.java) | Environment substitution, TCPS and token-mode validation |
| [config](config) | 13 complete profiles for file, SDK, OCI and Entra methods |
| [sql](sql) | Administrator setup templates for each identity provider |
| [troubleshooting](docs/troubleshooting.md) | Failure diagnosis and live acceptance procedure |

`mvn verify` checks configuration security boundaries, OCI/Entra concurrent token caches, all shipped profiles, UCP settings and discovery of the actual released OCI/Azure providers. CI runs JDK 17 and 21. Cloud tests require configured external identities; see [validation](docs/validation.md) for exactly what was run.

## Relationship to the original articles

This project supplies runnable implementations of the flows discussed in [Accessing Autonomous Database with IAM token using Java](https://blogs.oracle.com/developers/accessing-autonomous-database-with-iam-token-using-java) (Jean de Lavarene, Michael McMahon and Nirmala Sundarappa, 2022) and [23c JDBC seamless authentication with OCI IAM and Azure AD](https://blogs.oracle.com/developers/23c-jdbc-extensions-for-oci-iam-azure-ad) (Jean de Lavarene, 2023). It uses the documented simplified `oracle.jdbc.tokenAuthentication` values for provider acquisition, with the matching provider JARs. File and application-owned SDK modes remain separate alternatives; do not combine acquisition mechanisms on one data source.

Authoritative references: [Oracle JDBC security](https://docs.oracle.com/en/database/oracle/oracle-database/26/jjdbc/client-side-security.html), [OCI provider](https://github.com/oracle/ojdbc-extensions/tree/main/ojdbc-provider-oci), [Entra provider](https://github.com/oracle/ojdbc-extensions/tree/main/ojdbc-provider-azure), [AccessToken API](https://docs.oracle.com/en/database/oracle/oracle-database/26/jajdb/oracle/jdbc/AccessToken.html).
