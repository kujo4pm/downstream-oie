

# Initial Set up
Please note these are the steps for Mac Silicon M2 chip which includes a LOT of fiddling about.

Steps here: 
- [ ] Download the OIE Imagine `$ docker pull --platform linux/arm64 openintegrationengine/engine:latest`
- [ ] Create app data area for persistence across reboots:
```
# 1. Create a folder anywhere on your machine to hold persistent data
$ mkdir -p ~/oie-appdata

# 2. Run the container, mounting that folder into the engine's appdata path
$ docker compose up
```
- [ ] Install the Admin launcher: https://www.meditecs.com/download-administrator-launcher/
## Database
- [ ] `docker exec -it downstream-oie-db-1 psql -U enginedb -c "CREATE DATABASE senaite_tracking;"`
- [ ] run
```bash
docker exec -it downstream-oie-db-1 psql -U enginedb -d senaite_tracking -c "
CREATE TABLE diagnostic_report_ids (
    id TEXT PRIMARY KEY,
    fetched_at TIMESTAMP DEFAULT NOW(),
    processed BOOLEAN DEFAULT FALSE
);"
```


- [] Download DBeaver community edition. Connect to the database and create:

```sql
CREATE TABLE instrument_worklist (
    senaite_id TEXT PRIMARY KEY,
    device_fhir_id TEXT,
    fhir_id TEXT NOT NULL,
    analyte_code TEXT NOT NULL,
    analyte_display TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending', -- pending | dispatched | resulted
    fetched_at TIMESTAMP DEFAULT NOW(),
    dispatched_at TIMESTAMP,
    resulted_at TIMESTAMP,
    attempts INT DEFAULT 0,
    last_error TEXT
);

CREATE TABLE instrument_results (
    id SERIAL PRIMARY KEY,
    senaite_id TEXT NOT NULL REFERENCES instrument_worklist(senaite_id),
    value TEXT,
    unit TEXT,
    ref_range TEXT,
    received_at TIMESTAMP DEFAULT NOW(),
    posted_to_fhir BOOLEAN DEFAULT FALSE,
    fhir_observation_id TEXT
);

CREATE TABLE dispatched_orders_log (
    id SERIAL PRIMARY KEY,
    senaite_id TEXT NOT NULL,
    hl7_message TEXT NOT NULL,
    logged_at TIMESTAMP DEFAULT NOW()
);
```

- [ ]

## Notes
- Just use the HTTP listener - there is NO explicity FHIR layer here.

## Idea
- Need to set up an OIE instance on a server that speaks with Senaite.fhir running on demo.


```
[SENAITE FHIR API] --poll-->[Ch1: fetch-instrument-worklist]--INSERT-->[Postgres: instrument_worklist]
                                                                              |
                                                            [Ch2: dispatch-instrument-orders]
                                                            (polls table WHERE status='pending')
                                                                              |
                                                                    HL7 ORM^O01 → instrument
                                                                              v
                                                                [Ch3: simulated-instrument]
                                                                              |
                                                                    HL7 ORU^R01 (auto result)
                                                                              v
                                                          [Ch4: receive-instrument-results]
                                                          (local DB lookup, not live search)
                                                          POST Observation → SENAITE FHIR API
```