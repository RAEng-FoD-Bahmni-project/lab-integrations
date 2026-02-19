# HL7 Lab Middleware

Java middleware that connects lab analyzer machines to Bahmni/OpenELIS. Listens for HL7 messages over TCP, parses test results, and writes them into the OpenELIS PostgreSQL database.

```
Lab Analyzer  ──HL7/TCP──▶  Middleware  ──SQL/JDBC──▶  OpenELIS DB
```

## How It Works

1. Lab analyzer transmits results as HL7 messages over TCP
2. Middleware extracts sample ID and test results using configurable regex
3. Checks OpenELIS for a matching pending sample
4. Inserts results, auto-signs them, marks sample as processed
5. Saves raw HL7 messages to disk as backup

## Source Files

| File | Purpose |
|------|---------|
| `Main.java` | Entry point — loads config, spawns one thread per analyzer |
| `WorkingThread.java` | Core loop — listens on TCP port, parses HL7, writes to DB |
| `Analyzer.java` | Analyzer config + sample-end detection + ACK sending |
| `DataBaseUtility.java` | PostgreSQL connection pool (reads `openelisDB.json`) |
| `PostgresDBConnectionInfo.java` | DB connection config POJO |
| `Result.java` | Test result data class (name, value, flag) |

## Configuration

**Analyzer config (JSON array):** Each analyzer needs a name, TCP port, regex patterns for parsing, and an output path for HL7 backups. The `name` must match the `machineid` in the `machine_mapping` DB table.

**Database config (`openelisDB.json`):** Server IP, port, DB name, credentials, max connections.

**`machine_mapping` table:** Maps analyzer test codes to OpenELIS test IDs. One row per test per analyzer.

## Quick Setup

1. Install Java 8+
2. Place the JAR + `openelisDB.json` + analyzer config in `/opt/hl7-middleware/`
3. Populate `machine_mapping` in the OpenELIS database
4. Open the TCP port(s) on the firewall
5. Run: `java -DLOG_DIR=/var/log/hl7/logs -jar captureHl7Messages.jar`

## Dependencies

Jackson 2.16.1, PostgreSQL JDBC, Log4j 2.23.0 — all bundled in the fat JAR.
