<div align="center">
  <b>English</b> |
  <a href="./README.ar.md">العربية</a>
</div>

---

# Lab Integrations — Bahmni (Arabic)

Middleware for connecting laboratory analyzer machines to Bahmni/OpenELIS. Results flow automatically from the analyzer into the patient record.

```
Lab Analyzer  ──HL7/TCP──▶  Middleware  ──SQL──▶  OpenELIS  ──▶  OpenMRS
```

## Contents

| File | Description |
|------|-------------|
| `captureHl7Messages_final.rar` | HL7 middleware source code (Java) |
| `MIDDLEWARE.md` | Technical documentation + setup guide |

## Quick Summary

- Java TCP server — one listening thread per analyzer
- Parses HL7/ASTM messages using configurable regex patterns
- Writes results directly into the OpenELIS PostgreSQL database
- Auto-signs results and updates sample status
- Saves raw HL7 messages to disk as audit trail
- Tested with Sysmex XP-300 hematology analyzer

See [MIDDLEWARE.md](MIDDLEWARE.md) for full details and setup instructions.

## License

AGPL-3.0
