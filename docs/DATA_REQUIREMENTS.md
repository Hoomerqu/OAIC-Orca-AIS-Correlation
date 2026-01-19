```markdown
# Datenanforderungen & Start‑Schema (MVP)

Zweck
- Dokumentiert die minimal benötigten Datensätze, Felder, Formate und Verarbeitungs‑Empfehlungen für OAIC (Nenner = AIS, Zähler = Vorfälle).

1) AIS‑Rohdaten (Nenner) — Minimales Schema
- Format: Parquet (empfohlen), alternativ CSV/JSONL; Roh‑NMEA für Archiv.
- Zeitbereich: 2020–aktuell (start MVP: 1 Monat).
- Raum: Kernregion Huelva → Algarve + Puffer (z. B. 50 NM).
- Felder (per Zeile / Nachricht):
  - timestamp (string, ISO8601, UTC)
  - latitude (float, WGS84)
  - longitude (float, WGS84)
  - mmsi (string/int)
  - sog (float)  // speed over ground (knots)
  - cog (float)  // course over ground (degrees)
  - heading (int, optional)
  - navigational_status (int/string, optional)
  - message_type (int, optional)
  - source (string: terrestrial|satellite)
  - static_fields (obj, optional): {imo, callsign, name, vessel_type, length, beam}
- Hinweise:
  - Partitioniere Parquet nach year/month/region für schnelle Queries.
  - Kennzeichne Quelle (terrestrial vs satellite) für Coverage‑Bias.

2) Incident / Vorfalls‑Daten (Zähler) — Minimales Schema
- Format: CSV/JSON.
- Felder:
  - incident_id (string)
  - incident_datetime (ISO8601, UTC)
  - latitude, longitude (float) OR textual_location (string)
  - mmsi (optional)
  - vessel_description (string: length/type/name)
  - interaction_type (enum: "no_damage","rudder_damage","propeller_damage","boarding_attempt",...)
  - damage_severity (low|medium|high)
  - source (string: official/media/witness)
  - source_url (string)
  - confidence (0..1 optional)
  - attachments (array of urls to photos/video)
- Hinweise:
  - Wenn MMSI fehlt: dokumentiere Evidenzbasis und ggf. candidate MMSIs aus matching.

3) Vessel‑Master / Registry
- Felder: mmsi, imo, name, type, length, beam, build_year, owner (anonymisiert), hull_type (wenn verfügbar).

4) Coverage & Provenance
- Receiver coverage map (time ranges per receiver).
- Data provenance table: {dataset_name, source_url, license, fetch_date, contact}.

5) Quality Checks (erste Pipeline‑Steps)
- Vereinheitliche timezone → UTC; de‑dupliziere identische messages.
- Plausibilitätsregeln: SOG < 100 kn, Lat in [-90,90], Lon in [-180,180].
- Track fragmentation: berechne max gap per MMSI; markiere kurze Tracks (< X points) als low confidence.
- Filter on‑land positions mit Küstenpuffer.

6) Matching‑Heuristik (erste Version)
- Wenn incident.mmsi vorhanden → direct match.
- Sonst: spatial‑temporal join: AIS points within radius R (e.g. 500 m — 1 NM) and time window ±T (e.g. 30 min).
- Setze Konfidenzscore basierend auf distance/time/unique_mmsi_count.

7) Speicherung & Infrastruktur
- Objekt‑Storage (S3/GCS) für Roh‑Parquet, partitioniert.
- Analytik: PostGIS for spatial joins OR Parquet + DuckDB/Polars for ad‑hoc.
- Große Dateien: git‑ignore / Git LFS ab > 100 MB.

8) Datenschutz & rechtliche Hinweise
- Keine personenbezogenen Daten oder private Kontakte in öffentlichen Commits.
- Keine Original‑Fotos/Videos mit persönlicher Identifikation ohne Einwilligung.
- Prüfe Provider‑Terms (AISHub/MarineTraffic) vor Redistribution.
- Bei öffentlichen Repo: nur Beispiel‑/synthetische Daten oder anonymisierte Auszüge commiten.

9) Praktische Hinweise für Repo
- Ablage:
  - docs/DATA_REQUIREMENTS.md (dieses Dokument)
  - data_samples/ (synthetische Beispiel‑CSV/Parquet) — gitignored große Dateien, stattdessen small sample
  - scripts/etl_demo.py (optional demo matching script)
- Commit‑Hinweis (Beispiel): "docs: add data requirements and minimal schemas for AIS and incident datasets"
- PR‑Beschreibung (Beispiel): "Adds initial data requirements and schema for AIS/incident datasets; includes guidance on storage, quality checks and privacy. This is a groundwork doc for building ingestion and analysis pipelines."

10) Nächste Schritte (MVP)
- 1) Commit dieses Dokuments in docs/.
- 2) Leg ein kleines synthetisches AIS‑Parquet (<=1MB) in data_samples/ für Integrationstests.
- 3) Implementiere scripts/etl_demo.py: Einlesen sample, apply QC, spatial‑temporal join gegen sample incidents.
```
