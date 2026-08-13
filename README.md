# SENAITE FHIR ↔ Instrument Middleware (Open Integration Engine)

A companion to the [SENAITE FHIR Implementation Guide](https://fhir.senaite.org/), specifically the [Instrument Integration workflow](https://fhir.senaite.org/instrument-integration.html).

This describes how to configure [Open Integration Engine](https://openintegrationengine.org/) (OIE) as a middleware layer so instruments speaking **HL7v2 over MLLP** can participate in SENAITE's FHIR R5 laboratory workflow. SENAITE remains the system of record; OIE owns only protocol translation and the delivery state machine.

The intent is that **clients own their own instrument configurations**. The channels here are the reusable scaffold; the instrument-specific parts (message dialect, field positions) are isolated to two channels, so a site can swap in another HL7v2 analyzer without touching the FHIR side.

**ASTM instruments are not covered by this pattern.** OIE has no support for the ASTM E1381 lower-layer protocol, so an ASTM analyzer cannot be served by swapping a connector on the two instrument-facing channels — it needs an additional component in front of OIE. See [§7 ASTM Support](#7-astm-support) for the two candidate routes, both work in progress.

> OIE is a community fork of Mirth Connect, created after NextGen moved Mirth to a commercial licence at v4.6. Channels are format-compatible in both directions, so everything here transfers to a licensed Mirth deployment.

---

## 1. Getting Started

### 1.1 Prerequisites

| Component | Version used | Notes |
|---|---|---|
| OIE | `4.5.2-alpine` | Pin the version; don't use `latest` |
| Java (local, for Administrator) | Temurin 17 | Newer JDKs tighten TLS/crypto policy and can break the Swing client |
| Postgres | 18.x | Middleware state, separate from OIE's own store |
| Postgres JDBC driver | `42.7.x` (JDBC 4.2 / Java 8 build) | Must be added manually |
| Administrator launcher | MCAL | Java Web Start is dead; a launcher is mandatory |

### 1.2 Container setup

On **Apple Silicon**: the published OIE images are single-arch (amd64) despite documentation suggesting otherwise. Omit `--platform` and let Docker Desktop run under Rosetta — slower to boot, functionally fine. Passing `platform: linux/arm64` in Compose fails rather than degrades.

Create the persistent directories under one parent:

```bash
mkdir -p ~/dev/oie-appdata ~/dev/oie-custom-lib ~/dev/oie-pgdata
```

`compose.yaml`:

```yaml
services:
  engine:
    image: openintegrationengine/engine:4.5.2-alpine
    environment:
      - SENAITE_PASSWORD=${SENAITE_PASSWORD}
    volumes:
      - ~/dev/oie-appdata:/opt/engine/appdata
      - ~/dev/oie-custom-lib:/opt/engine/custom-lib
    ports:
      - "8443:8443"
      - "6661:6661"   # orders out to instrument
      - "6663:6663"   # results in from instrument
    depends_on:
      - db
  db:
    image: postgres
    environment:
      - POSTGRES_USER=enginedb
      - POSTGRES_PASSWORD=enginedb
      - POSTGRES_DB=enginedb
    volumes:
      - ~/dev/oie-pgdata:/var/lib/postgresql
    ports:
      - "5432:5432"
```

Two things that will otherwise bite:

- **Postgres 18+ requires the mount at `/var/lib/postgresql`**, not `/var/lib/postgresql/data`. The older path is a hard startup failure.
- **`appdata` holds OIE's Derby store (channel definitions), not your data.** It is small by design — keystore, `server.id`, `mirthdb/`. If channels vanish after a restart, check the mount path before assuming data loss.

```bash
docker compose up -d
docker compose logs -f engine     # wait for "server successfully started"
```

### 1.3 JDBC driver

Download the Postgres JDBC driver (JDBC 4.2 / Java 8 build — runs fine on the container's Java 17) into `~/dev/oie-custom-lib/`, restart the engine, and confirm it appears under **Settings → Resources → Loaded Libraries**.

Leave **Load Parent-First** unchecked. See §2.4 — it does not fix the driver issue and changing it globally destabilises channels that were previously deploying.

### 1.4 Administrator access

The Administrator is a Java Swing desktop app, downloaded from the server and run locally. `https://<host>:8443/webstart.jnlp` will not work in any modern browser — Java Web Start was removed from the JDK in Java 9.

1. `brew install --cask temurin@17`
2. Install [MCAL](https://www.meditecs.com/download-administrator-launcher/) (macOS ARM build).
3. The server's self-signed cert has `CN=oie-engine`. Java enforces hostname matching, so `localhost` fails verification. Add to `/etc/hosts`:
   ```
   127.0.0.1 oie-engine
   ```
4. Connect to `https://oie-engine:8443`, Java Home set to the Temurin 17 path, credentials `admin` / `admin`. Change the password on first login.

### 1.5 Configuration Map

Under **Settings → Configuration Map**, add the environment-specific values. Channels reference these as `${key}`, so the same channel XML promotes across environments unchanged:

| Key | Example |
|---|---|
| `senaiteFHIRApi` | `https://lab.example.org/@@FHIR/r5` |
| `senaiteJSONApi` | `https://lab.example.org/@@API/senaite/v1` |

`${authHeader}` is **not** a Configuration Map entry — it is written to the Global Map at runtime by `fetch-token` (§3.1).

Do not put the SENAITE password in a channel field or in exported XML. Inject it as a container environment variable and read it into the Configuration Map from a **Global Deploy Script**:

```javascript
configurationMap.put('senaitePassword', java.lang.System.getenv('SENAITE_PASSWORD'), false);
```

For production, use orchestrator secrets rather than a plain env var.

### 1.6 Database schema

```bash
docker exec -it downstream-oie-db-1 psql -U enginedb -c "CREATE DATABASE senaite_tracking;"
```

Then against `senaite_tracking` (DBeaver Community is a convenient client — connect to `localhost:5432`, since the container publishes the port to the host):

```sql
CREATE TABLE instrument_worklist (
    senaite_id      TEXT PRIMARY KEY,          -- SENAITE analysis ID, e.g. AA00092
    fhir_id         TEXT NOT NULL,             -- server-owned ServiceRequest logical ID
    device_fhir_id  TEXT,                      -- Device logical ID from performer[0]
    analyte_code    TEXT NOT NULL,             -- LOINC
    analyte_display TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',  -- pending | dispatched | resulted
    fetched_at      TIMESTAMP DEFAULT NOW(),
    dispatched_at   TIMESTAMP,
    resulted_at     TIMESTAMP,
    attempts        INT DEFAULT 0,
    last_error      TEXT
);

CREATE TABLE instrument_results (
    id                  SERIAL PRIMARY KEY,
    senaite_id          TEXT NOT NULL REFERENCES instrument_worklist(senaite_id),
    value               TEXT,
    unit                TEXT,
    ref_range           TEXT,                  -- as received, e.g. "98-107"
    received_at         TIMESTAMP DEFAULT NOW(),
    posted_to_fhir      BOOLEAN DEFAULT FALSE,
    fhir_observation_id TEXT,
    rejected_at         TIMESTAMP,             -- terminal failure (see §2.3)
    rejected_reason     TEXT
);

CREATE TABLE dispatched_orders_log (
    id           SERIAL PRIMARY KEY,
    senaite_id   TEXT NOT NULL,
    hl7_message  TEXT NOT NULL,
    logged_at    TIMESTAMP DEFAULT NOW()
);
```

`instrument_results` is a separate table rather than columns on `instrument_worklist`, mirroring the FHIR relationship it implements: `ServiceRequest` and `Observation` are distinct resources joined by `basedOn`, not one record with result fields appended. It also allows result history — corrections, re-runs, preliminary-then-final — which a last-write-wins column set cannot represent.

---

## 2. Architecture

### 2.1 Layers

| Layer | Component | Responsibility |
|---|---|---|
| Record | SENAITE + FHIR R5 API | Clinical source of truth. Owns FHIR logical IDs. |
| Integration | OIE channels | Protocol translation, field mapping, retry, dispatch state |
| State | Postgres `senaite_tracking` | Delivery watermarks and audit; **never** clinical truth |
| Transport | MLLP over TCP (HL7v2) | Instrument-facing wire protocol |

The state layer holds no clinical data that isn't already in SENAITE. If it is lost, the correct recovery is to re-poll SENAITE, not to reconstruct records from the middleware.

The transport layer is MLLP only. Anything else — ASTM E1381, HLLP, raw serial — requires a component upstream of OIE (§7).

### 2.2 Channels

```
[SENAITE FHIR API]
      │ GET ServiceRequest?intent=filler-order&status=active
      ▼
[fetch-instrument-worklist] ──upsert──► instrument_worklist (status=pending)
                                              │
                                              ▼
                                  [dispatch-instrument-orders]
                                  (WHERE status='pending' AND attempts < 5)
                                  ├─► send-order-tcp   HL7 ORM^O01 (MLLP :6661)
                                  ├─► log-to-db        dispatched_orders_log
                                  └─► update-result-status  status=dispatched
                                              │
                                        ┌──────────┐
                                        │INSTRUMENT│
                                        └──────────┘
                                              │ HL7 ORU^R01 (MLLP :6663)
                                              ▼
                                  [receive-instrument-results]
                                  ──insert──► instrument_results
                                              │
                                              ▼
                                  [post-observations-to-fhir]
                                  (WHERE posted_to_fhir=false
                                     AND rejected_at IS NULL,
                                   JOIN worklist for fhir_id/device_fhir_id)
                                              │ POST Observation
                                              ▼
                                     [SENAITE FHIR API]

[fetch-token] ──► Global Map: senaite_token, authHeader   (independent, 15 min)
```

| Channel | Source | Destination(s) | Instrument-specific? |
|---|---|---|---|
| `fetch-token` | JavaScript Reader, 15 min, poll on start | HTTP Sender → `${senaiteJSONApi}/login` | No |
| `fetch-instrument-worklist` | JavaScript Reader, 5 s | HTTP Sender → FHIR search; response transformer upserts | No |
| `dispatch-instrument-orders` | JavaScript Reader (SQL → list), 5 s | `log-to-db`, `send-order-tcp`, `update-result-status` | **Yes** |
| `receive-instrument-results` | TCP Listener :6663 (HL7, auto-ACK) | JavaScript Writer → insert | **Yes** |
| `post-observations-to-fhir` | JavaScript Reader (SQL JOIN → list), 5 s | HTTP Sender → POST Observation | No |

Only the two marked channels touch the wire format. **To support a different HL7v2 analyzer, replace those two and leave the rest untouched** — the database contract on either side is the interface. In practice that means adjusting segment and field positions (§3.4) and the outbound Message Builder steps (§3.3) to the analyzer's dialect.

This isolation covers HL7v2 dialect variation. It does not extend to ASTM: that is a change of lower-layer protocol, not of field positions, and OIE cannot terminate it at all. See §7.

The 5-second poll intervals are for development. Tune to the site's latency tolerance before deployment.

### 2.3 Key design decisions

**FHIR logical IDs stay server-owned.** SENAITE's internal identifier (`AA00092`) lives only in `ServiceRequest.identifier` with `use=usual`; the FHIR logical ID is assigned by the server. The middleware stores both and correlates on the SENAITE ID, because that is what round-trips through HL7 (`ORC.2`, `MSH.10`).

**`basedOn` must be a literal reference.** `SenaiteObservation` suppresses `basedOn.identifier` (`..0`), so an identifier-based logical reference is not an option — the POST needs `ServiceRequest/{logical-id}`. Capturing `fhir_id` at fetch time means the posting channel does a local JOIN rather than a live FHIR search, avoiding a round-trip and any search-index lag. `device_fhir_id` is captured the same way, from `performer[0].reference`.

**Fetch and dispatch are separate channels.** SENAITE's `status=active` does not change when an order is dispatched — it changes when a result is verified. A fetch-and-immediately-send design therefore re-dispatches the same order every poll. The `status` column is what makes dispatch exactly-once.

**Status only moves forward.** The upsert in `fetch-instrument-worklist` deliberately does not touch `status`; only the dispatch and posting channels advance it, and only on confirmed success.

**Terminal vs. transient failure.** SENAITE returns `409 Conflict` with an `OperationOutcome` when an analysis is locked — e.g. already in `to_be_verified` because it holds a value:

```
Observation or DiagnosticReport is locked because it is state: to_be_verified
and the value cannot be updated
```

Retrying this can never succeed. Without a distinct terminal state, such rows accumulate as permanent retry noise, so `rejected_at`/`rejected_reason` mark them and the source query filters them out. Any other `OperationOutcome` code is treated as potentially transient and left eligible for retry. This locked-state behaviour is server-specific and should be documented in the CapabilityStatement for middleware implementers.

**R5 type distinctions.** `ServiceRequest.code` is a `CodeableReference` — the coding sits under `code.concept.coding[]`. `Observation.code` and `DiagnosticReport.code` are plain `CodeableConcept` with no `concept` wrapper. Easy to conflate; produces confusing null dereferences.

### 2.4 Platform constraints

Real, documented limitations rather than configuration errors. Record them for whoever maintains this next.

**Native Database Reader/Writer connectors do not work with custom JDBC drivers on Java 9+.** They fail at deploy with `NoClassDefFoundError: java/sql/Driver`, because Mirth/OIE's `CustomDriver` isolated classloader cannot see the JDK's `java.sql` module. This affects Oracle and MS SQL drivers equally — not Postgres-specific — and has been open upstream for years. The behaviour is not stable across restarts.

*Consequence:* all database access uses raw JDBC (`java.sql.DriverManager`) inside JavaScript Reader/Writer connectors, which runs in the JS engine's classloader and is unaffected. Treat Database Reader/Writer as unavailable on this stack.

**No ASTM E1381 support.** There is no ASTM connector, transmission mode or data type in OIE. The TCP Listener speaks MLLP or raw TCP; it has no ENQ/ACK/EOT state machine, no `STX`/`ETX` frame handling and no checksum verification, and there is no ASTM E1394 record parser. Pointing an MLLP listener at an ASTM analyzer produces a connection with no parsed messages. This is a platform gap, not a configuration problem — see §7.

**Fanning out one poll into many messages.** A JavaScript Reader returning a `java.util.ArrayList` of strings causes OIE to process one message per entry. This is the documented mechanism that lets a SQL query drive per-row processing while still using native per-message transformers and connectors downstream.

**Outbound HL7 is built with Message Builder steps, not string concatenation.** Paste a skeleton into the destination's **Outbound Message Template** with data type `HL7 v2.x`, then add one Message Builder step per field. The two fields behave differently:

- **Message Segment** is a tree path into the outbound message: `tmp['ORC']['ORC.2']`
- **Mapping** is a JavaScript expression: `msg['senaite_id'].toString()`

Mirth's serializer handles delimiter escaping and MLLP framing; hand-built strings silently corrupt on any analyte name containing `|`, `^`, `~`, `\` or `&`.

**Base64-encoded responses.** If an HTTP Sender's **Binary MIME Types** is set to a literal like `application/json`, Mirth treats the response as binary and base64-encodes it, making logs unreadable. Keep the default regex `application/.*(?<!json|xml)$|image/.*|video/.*|audio/.*` with **Regular Expression** ticked — the negative lookbehind explicitly excludes JSON and XML from binary handling.

**Scripting is Rhino 1.7.13 — write ES5.** Use `var`, string concatenation; no arrow functions, template literals or destructuring. Java classes are directly accessible (`java.sql.DriverManager`, `java.net.Socket`, `java.util.UUID`) without imports, which is why raw JDBC works at all.

**Variable scope by context.** The most common source of confusion:

| Context | Available |
|---|---|
| Source/destination Transformer steps | `msg`, `tmp`, `channelMap` |
| Response Transformer | `msg` (the response), `channelMap`, `responseMap` |
| JavaScript Writer connector script | `connectorMessage`, `channelMap`, `globalMap`, `responseMap` — **no `msg`, no `tmp`** |

Inside a JavaScript Writer, `connectorMessage.getRawData()` returns the original source message and `connectorMessage.getEncodedData()` returns that destination's own serialised output. `tmp` is scoped to the transformer that built it and is not visible to sibling destinations.

**Save and Deploy are separate.** Saving persists the definition; the running channel keeps executing the last-deployed version until redeployed. Test Connection resolves nothing — `${}` substitution happens at runtime, so a templated URL always appears to fail there.

### 2.5 Moving off Docker

Docker is appropriate for development. For a site deployment:

- **Run OIE as a service on a dedicated host** (`systemd`), not in a container, so JVM heap and MLLP port bindings are managed directly by the OS. Instrument connectivity in particular is easier without container network translation.
- **Postgres on its own host or a managed instance**, with real backups. OIE's internal store should also move off embedded Derby to Postgres at this point.
- **One OIE instance per environment.** There is no environment switch inside OIE; dev/test/prod are separate instances differing only in Configuration Map values.
- **Version-control channels with [MirthSync](https://github.com/SagaHealthcareIT/mirthsync)** (OIE-compatible). Author in the Administrator, export to git, review the XML diffs, promote by pushing to the target instance via CI. Secrets never enter the repo.
- **Network placement:** instrument-facing MLLP ports on a segmented lab network; only FHIR egress needs to reach SENAITE. OIE is the only component that sees both.
- **There is no established automated test framework** for Mirth/OIE channels. Practically: scripted integration tests — fire known HL7 payloads at a staging instance, then assert on the resulting FHIR resources and `instrument_worklist` rows. Worth building as a harness; nothing off-the-shelf exists.
- **If an ASTM route is in scope**, note that whichever option in §7 is chosen adds a second process (senaite.astm) or a licensed extension (Meditecs) to the deployment. Serial devices additionally need a serial-to-TCP converter on the lab network.

---

## 3. Configuring the Channels

### 3.1 `fetch-token`

JavaScript Reader, 15-minute interval, **poll on start** enabled. Source data type `Raw`:

```javascript
logger.info('Refreshing SENAITE token');
return true;
```

Destination: HTTP Sender `GET` to `${senaiteJSONApi}/login`, with `__ac_name` and `__ac_password` as query parameters. Set the destination's **outbound** data type to `JSON` and the **response** inbound data type to `JSON`.

Response Transformer:

```javascript
globalMap.put('senaite_token', msg.token);
globalMap.put('authHeader', 'Bearer ' + globalMap.get('senaite_token'));
logger.info('SENAITE token refreshed');
```

Every other channel then uses `${authHeader}` directly in its Authorization header. The login response includes an `expires` field — confirm the real TTL and set the interval comfortably below it.

### 3.2 `fetch-instrument-worklist`

Source: JavaScript Reader (trigger only), data type `Raw`:

```javascript
logger.info('Polling for active InstrumentServiceRequests');
return true;
```

Destination: HTTP Sender `GET` to `${senaiteFHIRApi}/ServiceRequest`, query parameters `intent=filler-order` and `status=active`, header `Authorization: ${authHeader}`.

Response Transformer — `load-service-requests-into-database`:

```javascript
var bundle = JSON.parse(msg.toString());
if (bundle.entry) {
    var DriverManager = java.sql.DriverManager;
    var conn = DriverManager.getConnection('jdbc:postgresql://db:5432/senaite_tracking', 'enginedb', 'enginedb');
    var upsertSql =
        'INSERT INTO instrument_worklist (senaite_id, fhir_id, device_fhir_id, analyte_code, analyte_display) ' +
        'VALUES (?, ?, ?, ?, ?) ' +
        'ON CONFLICT (senaite_id) DO UPDATE SET ' +
        '  fhir_id = EXCLUDED.fhir_id, ' +
        '  device_fhir_id = EXCLUDED.device_fhir_id, ' +
        '  analyte_code = EXCLUDED.analyte_code, ' +
        '  analyte_display = EXCLUDED.analyte_display';
    var stmt = conn.prepareStatement(upsertSql);
    var count = 0;
    for (var i = 0; i < bundle.entry.length; i++) {
        var sr = bundle.entry[i].resource;
        if (!sr || sr.resourceType !== 'ServiceRequest') continue;

        var senaiteId = null;
        for (var j = 0; j < sr.identifier.length; j++) {
            if (sr.identifier[j].use === 'usual') { senaiteId = sr.identifier[j].value; break; }
        }
        if (!senaiteId) {
            logger.warn('Skipping ServiceRequest/' + sr.id + ' - no senaiteId identifier found');
            continue;
        }

        // R5: ServiceRequest.code is CodeableReference, so coding sits under .concept
        if (!sr.code || !sr.code.concept || !sr.code.concept.coding || sr.code.concept.coding.length === 0) {
            logger.warn('Skipping ServiceRequest/' + sr.id + ' - no code.concept.coding found');
            continue;
        }

        var deviceFhirId = null;
        if (sr.performer && sr.performer.length > 0 && sr.performer[0].reference) {
            deviceFhirId = sr.performer[0].reference.split('/')[1];
        }
        if (!deviceFhirId) {
            logger.warn('ServiceRequest/' + sr.id + ' (' + senaiteId + ') has no performer Device reference');
        }

        stmt.setString(1, senaiteId);
        stmt.setString(2, sr.id);
        stmt.setString(3, deviceFhirId);
        stmt.setString(4, sr.code.concept.coding[0].code);
        stmt.setString(5, sr.code.concept.coding[0].display);
        stmt.executeUpdate();
        count++;
    }
    stmt.close();
    conn.close();
    logger.info('Upserted ' + count + ' instrument worklist rows');
} else {
    logger.info('No entries in ServiceRequest bundle this poll');
}
```

Per-entry `continue` guards mean one malformed resource does not abort the batch. `SenaiteInstrumentServiceRequest` declares `performer` as `1..1`, so a missing Device reference means the resource is not conformant to the profile — worth surfacing as a warning rather than silently storing null.

### 3.3 `dispatch-instrument-orders`  *(instrument-specific)*

**Source:** JavaScript Reader, returning one JSON string per pending row so OIE fans the poll out into individual messages:

```javascript
function doScript() {
    var DriverManager = java.sql.DriverManager;
    var conn = DriverManager.getConnection('jdbc:postgresql://db:5432/senaite_tracking', 'enginedb', 'enginedb');

    var selectStmt = conn.prepareStatement(
        "SELECT senaite_id, fhir_id, analyte_code, analyte_display " +
        "FROM instrument_worklist WHERE status = 'pending' AND attempts < 5"
    );

    var rs = selectStmt.executeQuery();
    var messages = new java.util.ArrayList();
    while (rs.next()) {
        messages.add(JSON.stringify({
            senaite_id:      rs.getString('senaite_id'),
            fhir_id:         rs.getString('fhir_id'),
            analyte_code:    rs.getString('analyte_code'),
            analyte_display: rs.getString('analyte_display')
        }));
    }
    rs.close(); selectStmt.close(); conn.close();

    logger.info('Found ' + messages.size() + ' pending orders to dispatch');
    return messages;
}
return doScript();
```

Three destinations, each with **Wait for previous destination** enabled.

**`send-order-tcp`** — TCP Sender, MLLP, `127.0.0.1:6661` (replace with the instrument's address). Inbound data type `JSON`, outbound `HL7 v2.x`, **Validate Response: Yes** so a NACK is a failure rather than a success. Outbound Message Template:

```
MSH|^~\&|OIE|SENAITE|INSTRUMENT|LAB|20260101000000||ORM^O01|MSGID001|P|2.3
ORC|NW|PLACEHOLDER
OBR|1|PLACEHOLDER||PLACEHOLDER^PLACEHOLDER^L
```

Four Message Builder steps:

| Message Segment | Mapping |
|---|---|
| `tmp['MSH']['MSH.10']` | `msg['senaite_id'].toString()` |
| `tmp['ORC']['ORC.2']` | `msg['senaite_id'].toString()` |
| `tmp['OBR']['OBR.4']['OBR.4.1']` | `msg['analyte_code'].toString()` |
| `tmp['OBR']['OBR.4']['OBR.4.2']` | `msg['analyte_display'].toString()` |

**`log-to-db`** — JavaScript Writer recording exactly what went on the wire. Useful permanently for audit; essential when testing without hardware. It carries **its own copy** of the outbound template and the same four Message Builder steps, because `getEncodedData()` returns *this* destination's serialisation — without them it would log JSON rather than HL7:

```javascript
var rawData = connectorMessage.getRawData();
var row = JSON.parse(rawData);
var senaiteId = row.senaite_id;
var encodedMessage = connectorMessage.getEncodedData();

logger.info('Logging dispatched order for senaiteId=' + senaiteId);

var DriverManager = java.sql.DriverManager;
var conn = DriverManager.getConnection('jdbc:postgresql://db:5432/senaite_tracking', 'enginedb', 'enginedb');
var insertStmt = conn.prepareStatement(
    "INSERT INTO dispatched_orders_log (senaite_id, hl7_message) VALUES (?, ?)"
);
insertStmt.setString(1, senaiteId);
insertStmt.setString(2, encodedMessage);
insertStmt.executeUpdate();
insertStmt.close();
conn.close();
```

**`update-result-status`** — JavaScript Writer, gated by a Filter so state only advances on confirmed delivery:

```javascript
// Filter: check-success
var logResult = responseMap.get('log-to-db');

if (logResult != null && logResult.getStatus().toString() == 'SENT') {
    return true;
}

logger.warn('Skipping status update - log-to-db did not succeed');
return false;
```

```javascript
// Script
var rawData = connectorMessage.getRawData();
var row = JSON.parse(rawData);
var senaiteId = row.senaite_id;

var DriverManager = java.sql.DriverManager;
var conn = DriverManager.getConnection('jdbc:postgresql://db:5432/senaite_tracking', 'enginedb', 'enginedb');

var updateStmt = conn.prepareStatement(
    "UPDATE instrument_worklist SET status = 'dispatched', dispatched_at = NOW() WHERE senaite_id = ?"
);
updateStmt.setString(1, senaiteId);
updateStmt.executeUpdate();
updateStmt.close();
conn.close();

logger.info('Marked dispatched for senaiteId=' + senaiteId);
```

> **Test mode vs. production.** As configured, `send-order-tcp` is **disabled** and the filter keys on `log-to-db`, so dispatch is confirmed by the database write rather than by an instrument ACK. This is what allows the pipeline to run without hardware. **Before deployment, enable `send-order-tcp` and repoint the filter to `responseMap.get('send-order-tcp')`** — otherwise orders are marked `dispatched` without ever having been transmitted.

### 3.4 `receive-instrument-results`  *(instrument-specific)*

**Source:** TCP Listener on `6663`, MLLP, data type `HL7 v2.x`, Response `Auto-generate (After source transformer)` so the instrument receives a real ACK.

Mapper steps extract the fields. **These positions are the part a client changes per analyzer:**

| Variable | Mapping |
|---|---|
| `senaiteId` | `msg['OBR']['OBR.2']['OBR.2.1'].toString()` |
| `analyteCode` | `msg['OBX']['OBX.3']['OBX.3.1'].toString()` |
| `analyteDisplay` | `msg['OBX']['OBX.3']['OBX.3.2'].toString()` |
| `value` | `msg['OBX']['OBX.5']['OBX.5.1'].toString()` |
| `unit` | `msg['OBX']['OBX.6']['OBX.6.1'].toString()` |
| `refRange` | `msg['OBX']['OBX.7']['OBX.7.1'].toString()` |

**Destination:** JavaScript Writer. The reference range is stored verbatim; parsing is deferred until it is actually needed for `Observation.referenceRange` (§6, gap 1):

```javascript
var senaiteId = channelMap.get('senaiteId');
var value = channelMap.get('value');
var unit = channelMap.get('unit');
var refRange = channelMap.get('refRange');

var DriverManager = java.sql.DriverManager;
var conn = DriverManager.getConnection('jdbc:postgresql://db:5432/senaite_tracking', 'enginedb', 'enginedb');

var insertStmt = conn.prepareStatement(
    "INSERT INTO instrument_results (senaite_id, value, unit, ref_range) VALUES (?, ?, ?, ?)"
);
insertStmt.setString(1, senaiteId);
insertStmt.setString(2, value);
insertStmt.setString(3, unit);
insertStmt.setString(4, refRange);
insertStmt.executeUpdate();
insertStmt.close();
conn.close();

logger.info('Recorded result in DB for senaiteId=' + senaiteId);
```

The foreign key on `senaite_id` means a result for an unknown analysis fails loudly rather than orphaning silently.

### 3.5 `post-observations-to-fhir`

The Observation is assembled in the **source**, where the row is already in hand. This avoids per-field `${}` template placeholders entirely — those require Mapper steps to populate channelMap, and unresolved placeholders are passed through literally rather than erroring, which is difficult to diagnose.

**Source:** JavaScript Reader. The JOIN resolves `fhir_id` and `device_fhir_id` without a live FHIR search:

```javascript
function doScript() {
    var DriverManager = java.sql.DriverManager;
    var conn = DriverManager.getConnection('jdbc:postgresql://db:5432/senaite_tracking', 'enginedb', 'enginedb');

    var selectStmt = conn.prepareStatement(
        "SELECT r.id AS result_id, r.senaite_id, w.fhir_id, w.device_fhir_id, " +
        "       r.value, r.unit, w.analyte_code, w.analyte_display " +
        "FROM instrument_results r " +
        "JOIN instrument_worklist w ON r.senaite_id = w.senaite_id " +
        "WHERE r.posted_to_fhir = FALSE AND r.rejected_at IS NULL"
    );

    var rs = selectStmt.executeQuery();
    var messages = new java.util.ArrayList();

    while (rs.next()) {
        var observation = {
            resourceType: 'Observation',
            id: java.util.UUID.randomUUID().toString(),
            meta: {
                profile: ['https://fhir.senaite.org/StructureDefinition/SenaiteObservation']
            },
            status: 'final',
            code: {
                text: rs.getString('analyte_display'),
                coding: [{
                    system: 'http://loinc.org',
                    code: rs.getString('analyte_code'),
                    display: rs.getString('analyte_display')
                }]
            },
            valueQuantity: {
                value: parseFloat(rs.getString('value')),
                unit: rs.getString('unit'),
                system: 'http://unitsofmeasure.org',
                code: rs.getString('unit')
            },
            basedOn: [{
                reference: 'ServiceRequest/' + rs.getString('fhir_id')
            }]
        };

        var deviceFhirId = rs.getString('device_fhir_id');
        if (deviceFhirId != null) {
            observation.device = { reference: 'Device/' + deviceFhirId };
        }

        // Correlation values the response transformer needs, alongside the payload
        var envelope = {
            result_id: rs.getInt('result_id'),
            senaite_id: rs.getString('senaite_id'),
            observation: observation
        };

        messages.add(JSON.stringify(envelope));
        logger.info('Queued Observation for ' + rs.getString('senaite_id'));
    }

    rs.close(); selectStmt.close(); conn.close();
    logger.info('Found ' + messages.size() + ' unposted results');
    return messages;
}
return doScript();
```

The envelope wrapper exists because the Response Transformer needs `result_id` and `senaite_id` to close out the rows. If the message were the bare Observation, those would be lost. The `device` guard omits the element entirely rather than sending `Device/null`.

**Destination:** HTTP Sender `POST` to `${senaiteFHIRApi}/Observation`, `Content-Type: application/json`, header `Authorization: ${authHeader}`, Content field set to `${observationBody}`. Keep **Binary MIME Types** at the default regex (§2.4) or error responses arrive base64-encoded.

Pre-send Transformer — one JavaScript step to unwrap:

```javascript
var envelope = JSON.parse(connectorMessage.getRawData());
channelMap.put('result_id', envelope.result_id);
channelMap.put('senaite_id', envelope.senaite_id);
channelMap.put('observationBody', JSON.stringify(envelope.observation));
```

**Response Transformer** — `handle-post-response`. It branches on `resourceType` rather than HTTP status, which is both more robust and avoids a specific trap: the error `OperationOutcome` carries its own `id`, so a naive `body.id` read would store the error's ID as though the post had succeeded.

```javascript
var resultId = parseInt(channelMap.get('result_id'), 10);
var senaiteId = channelMap.get('senaite_id');

var body;
try {
    body = JSON.parse(msg.toString());
} catch (e) {
    logger.error('Unparseable response for ' + senaiteId + ': ' + msg.toString());
    return;
}

var DriverManager = java.sql.DriverManager;
var conn = DriverManager.getConnection('jdbc:postgresql://db:5432/senaite_tracking', 'enginedb', 'enginedb');

if (body.resourceType === 'Observation') {
    var observationId = body.id;

    var a = conn.prepareStatement(
        'UPDATE instrument_results SET posted_to_fhir = TRUE, fhir_observation_id = ? WHERE id = ?');
    a.setString(1, observationId);
    a.setInt(2, resultId);
    a.executeUpdate();
    a.close();

    var b = conn.prepareStatement(
        "UPDATE instrument_worklist SET status = 'resulted', resulted_at = NOW() WHERE senaite_id = ?");
    b.setString(1, senaiteId);
    b.executeUpdate();
    b.close();

    logger.info('Posted Observation ' + observationId + ' for ' + senaiteId);

} else if (body.resourceType === 'OperationOutcome') {
    var reason = 'unspecified';
    var code = 'unknown';
    if (body.issue && body.issue.length > 0) {
        code = body.issue[0].code;
        if (body.issue[0].details && body.issue[0].details.text) {
            reason = body.issue[0].details.text;
        }
    }

    if (code === 'conflict') {
        // Terminal: the analysis is locked (already has a value). Retrying cannot succeed.
        var c = conn.prepareStatement(
            'UPDATE instrument_results SET rejected_at = NOW(), rejected_reason = ? WHERE id = ?');
        c.setString(1, reason);
        c.setInt(2, resultId);
        c.executeUpdate();
        c.close();
        logger.warn('Rejected (conflict) for ' + senaiteId + ': ' + reason);
    } else {
        // Anything else may be transient - leave the row eligible for retry.
        logger.error('Post failed for ' + senaiteId + ' [' + code + ']: ' + reason);
    }

} else {
    logger.error('Unexpected resourceType for ' + senaiteId + ': ' + body.resourceType);
}

conn.close();
```

`fhir_observation_id` stores what the **server returns**, not the client-generated UUID — see §6, gap 2.

---

## 4. Example Run

### 4.1 Testing without an instrument

Two options; the second is preferable for demonstrating the full round trip.

**Option A — dispatch logged, results injected by hand.** This is how the channels ship. `send-order-tcp` is disabled and `log-to-db` is enabled, so every dispatch is recorded in `dispatched_orders_log` with the exact HL7 that would have gone on the wire. Drive the result side manually with the Administrator's **Send Message** tool against `receive-instrument-results` (paste plain text — MLLP framing is added for you).

**Option B — simulated instrument.** Add a `simulated-instrument` channel: TCP Listener on the port `send-order-tcp` targets (6661), data type `HL7 v2.x`, which reads `ORC.2` and `OBR.4` from the inbound order and returns a fabricated ORU^R01 to port 6663. This exercises the loop unattended, including real ACKs in both directions. Name it clearly as a simulator and never deploy it to a site.

### 4.2 End-to-end walkthrough

1. **Create a Sample** in SENAITE — e.g. an Electrolytes panel.
2. **Assign an analyte to an instrument.** For Chloride, set the analysis's instrument to a blood analyzer. This is what makes SENAITE emit a `SenaiteInstrumentServiceRequest` with `intent = filler-order` and `performer` referencing the Device.
3. **Receive the Sample.** The ServiceRequest becomes `status = active` and enters the query surface.
4. **Confirm the fetch:**
   ```sql
   SELECT senaite_id, fhir_id, device_fhir_id, analyte_code, status
   FROM instrument_worklist ORDER BY fetched_at DESC;
   ```
   Expect a `pending` row with both logical IDs populated. A null `device_fhir_id` means `performer` was absent — check the warning in the server log.
5. **Confirm the dispatch.** The channel builds ORM^O01 and advances the row to `dispatched`. Inspect the outbound HL7 in the Message Browser's Encoded tab, or:
   ```sql
   SELECT senaite_id, hl7_message, logged_at FROM dispatched_orders_log ORDER BY logged_at DESC;
   ```
6. **Return a result.** Either the simulator, or Send Message into port 6663:
   ```
   MSH|^~\&|INSTRUMENT|LAB|OIE|SENAITE|20260812150000||ORU^R01|AA00092|P|2.3
   ORC|RE|AA00092
   OBR|1|AA00092||2075-0^Chloride [Moles/volume] in Serum or Plasma^L
   OBX|1|NM|2075-0^Chloride [Moles/volume] in Serum or Plasma^L||101|mmol/L|98-107|N|||F
   ```
7. **Confirm receipt:**
   ```sql
   SELECT senaite_id, value, unit, ref_range, posted_to_fhir, rejected_reason
   FROM instrument_results ORDER BY received_at DESC;
   ```
8. **Confirm the FHIR post:**
   ```sql
   SELECT w.senaite_id, w.status, r.fhir_observation_id, r.rejected_reason
   FROM instrument_worklist w
   JOIN instrument_results r ON r.senaite_id = w.senaite_id
   WHERE w.senaite_id = 'AA00092';
   ```
   Expect `status = resulted` and a populated `fhir_observation_id`.
9. **Verify in SENAITE.** The analysis should show the result, and `GET {fhir}/Observation/{id}` should return it with `basedOn` pointing at the original ServiceRequest.

### 4.3 Negative paths worth exercising

- **Locked analysis (409).** Re-send a result for an analysis already in `to_be_verified`. Expect `rejected_at`/`rejected_reason` to be set and the row to drop out of the retry query. This is the most likely real-world failure, since any manually-entered result locks the analysis.
- **Out-of-range result.** Send a value outside the reference range and confirm SENAITE's flagging behaves as the IG describes.
- **Instrument unreachable.** Stop the listener and confirm the row stays `pending` rather than advancing. Note gap 3 below: nothing currently increments `attempts`, so this will retry indefinitely.
- **Unknown `senaite_id` in a result.** Confirm the foreign key rejects it rather than orphaning the row.
- **Expired token.** Clear the Global Map and confirm `fetch-token` recovers on its next poll.

---

## 5. ASTM Support

**Status: work in progress. Neither route below is implemented or proven against this channel set.**

### 5.1 Why OIE cannot do this alone

ASTM instrument communication is two standards stacked:

- **ASTM E1381** (also CLSI LIS01) — the lower-layer protocol. A stateful session: `ENQ` to request the line, `ACK`/`NAK` in reply, then records wrapped as `STX` + frame number + data + `ETB`/`ETX` + checksum, each individually acknowledged, closed with `EOT`.
- **ASTM E1394** (also CLSI LIS02) — the message content. `H`/`P`/`O`/`R`/`C`/`Q`/`L` record types with their own field, repeat and component delimiters.

OIE implements neither. Its TCP Listener speaks MLLP or raw TCP, so it has no handshake state machine, no frame checksum verification and no E1394 parser. An ASTM analyzer pointed at an MLLP listener opens a connection and delivers nothing parseable.

This is not a field-mapping problem and cannot be solved by editing the two instrument-specific channels (§2.2). It requires either a component in front of OIE that terminates E1381 and forwards structured data (§5.2), or a connector plugin that adds the protocol to OIE itself (§5.3).

### 5.2 Route A — senaite.astm upstream of OIE

[`senaite/senaite.astm`](https://github.com/senaite/senaite.astm) is a Python asyncio server maintained in the SENAITE organisation. It already implements the parts OIE lacks.

**What it provides:**

| Capability | Location |
|---|---|
| E1381 framing and session state machine | `transports/astm/framing.py`, `transports/astm/protocol.py` |
| E1394 record parsing | `records.py`, `codec.py` |
| MLLP + HL7v2 transport (separate listener) | `transports/hl7/`, `cli/hl7_server.py` |
| Synthesiser for devices that do not emit valid ASTM | `transports/astm/synthesizer.py` |
| Pluggable output pipeline | `core/pipeline.py` |
| Push to SENAITE via `senaite.jsonapi` | `core/lims.py`, `sender.py` |

**Instrument drivers present** (`instruments/`): Roche cobas c111 and c311, Sysmex XN and XP, Horiba Pentra XLR and Yumizen H5xx, Abbott Afinion2, Hitachi 7600, Cepheid GeneXpert, Siemens DCA Vantage, bioMérieux mini VIDAS, Arkray Spotchem EL. Captured message samples for most of these live in `tests/data/`.

`instruments/genexpert.py` is written against GeneXpert LIS Interface Protocol Specification 302-2261 Rev. E (2022), covering Dx v4.7b+, Omni Mobile 1.2+, Xpress 5.1+, Infinity Xpertise v6.4b+ and Cepheid OS 1.0+.

**Proposed wiring.** `core/pipeline.py` runs an ordered list of async handlers with per-handler exception isolation and a dead-letter hook. `cli/astm_server.py` currently assembles `DiskCaptureHandler` and `LimsPushHandler`. Adding an OIE forwarder means a sibling handler on that list — the existing SENAITE push path can stay enabled alongside it.

```
instrument --E1381/TCP--> senaite.astm --HTTP JSON--> OIE HTTP Listener
                          (protocol only)              │
                                                       ▼
                                           [receive-astm-results]
                                           ──insert──► instrument_results
                                                       │
                                                       ▼
                                           [post-observations-to-fhir]   (unchanged)
```

**Forward the parsed envelope, not HL7v2.** Converting to HL7v2 first would let the existing `receive-instrument-results` channel be reused verbatim, but everything then has to fit through `OBX`. `SenaiteObservation` carries detection limits, reference ranges, method coding, performer and verifier identity and cheminformatics extensions; the overlap with `OBX` is incomplete, so the remainder lands in `NTE` or a Z-segment and the OIE transformer scrapes free text to rebuild structure that was already parsed cleanly one hop earlier. Forwarding the envelope keeps OIE as a router and transformer over structured data. The cost is a new source channel rather than reuse of the HL7 one.

`core/lims.py` `Session` preflights for `senaite.jsonapi` at the target and builds paths as `{url}/{API_BASE_URL}/push`, so it is SENAITE-shaped and cannot simply be repointed at OIE with `--url`. Hence a new handler rather than a configuration change. It is plain `requests`; nothing about it is difficult.

**What is missing:**

- **No FHIR.** There is no occurrence of `fhir` anywhere in the codebase. Output is LIS2-A/ASTM payloads to the JSON API.
- **No orders outbound.** The `Q` (query) record type is defined in `records.py`, but nothing in the protocol layer handles host-query mode. IG step 3 — worklist transmission to the instrument — has no implementation. This gap is independent of the routing decision and OIE cannot fill it.
- **No published releases.** Pin a commit and own that decision.
- **Forks outnumber stars**, which usually indicates sites patching in their own drivers rather than contributing upstream. Confirm with the Naralabs team what is considered supported before depending on it.

### 5.3 Route B — Meditecs ASTM Extension for OIE

A commercial connector plugin: [ASTM Extension for Open Integration Engine](https://www.meditecs.com/astm-extension-for-open-integration-engine/). This is the option that keeps everything inside OIE.

**Take the OIE build, not the Mirth one.** They are separate products; the Mirth extension supports open-source Mirth up to 4.5.2 only and is explicitly incompatible with 4.6.0 and later.

**What it adds:** "ASTM Listener" and "ASTM Sender" connectors with a graphical configuration panel; outbound message templates for channel destinations; all three E1381 revisions (-91, -95, -02); TCP client or server operation; multiple connections on one port; CP-1252 encoding; and listener and sender sharing a single ASTM connection to one device. Serial instruments need a serial-to-TCP converter (MOXA is named). Roche Elecsys and Cobas frame structures are called out as specifically supported.

To prove, in order:

1. ASTM Listener receives and frames decode
2. Decoded payload reaches a transformer and maps to `SenaiteObservation`
3. Observation lands in SENAITE against the correct ServiceRequest
4. ASTM Sender delivers an order the instrument accepts

Capture a byte-level trace of a successful exchange while the trial is live. If Route A is chosen later, that trace is the test fixture.

**Open question:** an OIE maintainer, replying to [OIE discussion #170](https://github.com/OpenIntegrationEngine/engine/discussions/170), noted the extension should work with OIE but suggested confirming with the vendor. A user in the same thread reports running it in two labs without issues but does not specify OIE or Mirth. Confirm compatibility with the pinned OIE version when requesting the trial.


### 5.4 Identifying what an instrument actually speaks

Framing and content vary independently, and datasheets conflate them. Capture the opening bytes before configuring anything:

| First byte | Framing |
|---|---|
| `0x05` (`ENQ`) | ASTM E1381 |
| `0x0B` (`VT`) | MLLP |

Then check the first record: `MSH|` is HL7v2 content, `H|\^&` is ASTM E1394 content. Four combinations, and three of them occur in the field:

| Framing | Content | Seen on |
|---|---|---|
| E1381 | E1394 | Most chemistry, haematology and coagulation analyzers |
| E1381 | HL7v2 | GeneXpert in HL7 mode — Cepheid documents HL7 running over E1381-02 |
| MLLP | HL7v2 | Mindray BC series, BD FACS Workflow Manager, HemoScreen |
| E1381 | proprietary | Clinitek Status+, McKesson UA |

Two consequences worth recording:

- **Configure GeneXpert for ASTM mode, not HL7.** Its HL7 mode is HL7 content inside E1381 framing, so the MLLP listener in §3.4 will never see a frame, and `instruments/genexpert.py` in senaite.astm is written for ASTM mode. Choosing HL7 mode creates work for no benefit.
- **Check whether the instrument dials out or listens.** Some analyzers are the TCP client and connect to the host on startup. Half of all "no data arriving" symptoms are this rather than framing.

## 6. Known Gaps

Carried forward deliberately rather than hidden.

| # | Gap | Impact |
|---|---|---|
| 1 | `Observation.referenceRange` and `note` are not sent. `ref_range` is stored verbatim and never parsed. | Reference ranges captured but not transmitted. Parsing must handle negative bounds and open-ended forms (`<5`), which a naive `split('-')` does not. |
| 2 | The client generates a random UUIDv4 `id` on POST. SENAITE's own IDs appear to be UUIDv5 (deterministic). | A conformant server ignores `id` on create. If this one honours it, a timed-out-but-successful POST retried with a fresh v4 creates a duplicate; a deterministic ID would collide and be safely rejected. Also sits against the server-owned-logical-ID principle — worth being deliberate about. |
| 3 | Nothing increments `attempts` or writes `last_error`. | The dispatch query filters `attempts < 5`, but there is no failure path to advance the counter, so a permanently-failing order retries forever. |
| 4 | `send-order-tcp` is disabled and the `update-result-status` filter keys on `log-to-db`. | Test-mode arrangement. Must be switched before deployment (§3.3) or orders are marked dispatched without transmission. |
| 5 | No `simulated-instrument` channel exists; ports 6661/6663 are not yet bridged. | Full loop currently requires manual result injection. |
| 6 | Source inbound data types on both JavaScript Reader channels are `XML` rather than `JSON`/`Raw`. | Harmless as written, because destination scripts read `getRawData()` directly, but misleading. Would break if any Mapper step were added. |
| 7 | ASTM is unimplemented and cannot be added by editing channels (§5). | Only HL7v2 over MLLP is proven. Serving an ASTM instrument requires senaite.astm upstream (§5.2) or the Meditecs extension (§5.3) — a deployment decision, not a configuration one. |
| 8 | No route delivers orders to an ASTM instrument. | senaite.astm has no host-query implementation; the Meditecs ASTM Sender is untested here. IG step 3 is unproven for ASTM either way. |
| 9 | A result received but never posted sits at `dispatched` indefinitely. | Needs a stale-row alert. |
| 10 | Exploratory channels (`fetch-users`, `fetch-diagnostic-reports-summary`, `dispatch-instrument-orders-old`) still present. | Delete before any site deployment. |
| 11 | No automated test harness. | All verification is manual; see §2.5. |

---

## 7. Reference

- [SENAITE FHIR IG](https://fhir.senaite.org/) — [Instrument Integration](https://fhir.senaite.org/instrument-integration.html)
- [OIE documentation](https://docs.openintegrationengine.org/) · [GitHub](https://github.com/OpenIntegrationEngine)
- [MirthSync](https://github.com/SagaHealthcareIT/mirthsync) — channel version control
- [Administrator Launcher (MCAL)](https://www.meditecs.com/download-administrator-launcher/)
- [Postgres JDBC driver](https://jdbc.postgresql.org/download/)

ASTM:

- [senaite.astm](https://github.com/senaite/senaite.astm) — ASTM/HL7 instrument server for SENAITE
- [Meditecs ASTM Extension for OIE](https://www.meditecs.com/astm-extension-for-open-integration-engine/)
- [OIE discussion #170](https://github.com/OpenIntegrationEngine/engine/discussions/170) — community ASTM approaches and maintainer response
- [Cepheid GeneXpert LIS Interface Protocol Specification](https://infomine.cepheid.com/sites/default/files/2025-02/302-2261,%20Rev.%20F%20LIS%20Protocol%20Specification.pdf) (302-2261 Rev. F)
- [BD MGIT 960 LIS Vendor Interface Document](https://www.bikalims.org/downloads/instrument-interface-specifications/bd-mgit-960)