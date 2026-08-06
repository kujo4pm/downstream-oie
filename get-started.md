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

## Notes
- Just use the HTTP listener - there is NO explicity FHIR layer here.

## Idea
- Need to set up an OIE instance on a server that speaks with Senaite.fhir running on demo.
- 