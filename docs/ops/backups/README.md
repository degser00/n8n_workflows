# Backups

This section documents backup strategies, configurations, and operational decisions for services in the **n89n** platform.

Focus is on:
- Data durability
- Restore reliability
- Clear separation between source systems and backup targets
- Operational clarity over tooling specifics

---

## Scope

This directory covers:
- What data is backed up
- Where it is backed up
- How often backups run
- Restore assumptions and limitations
- Operational notes and known risks

It does **not** document:
- Application internals
- One-off recovery incidents (those go to ops incident notes)
- Backup tooling installation steps

---

## Currently Covered Systems

### OpenArchiver
- Email ingestion and archival data
- Attachments and metadata handling
- Export and restore assumptions

Documentation lives in:
- `openaarchiver.md` (or subfolder if it grows)

### Immich
- Media originals
- Derived assets (thumbnails, encodings)
- Database and metadata consistency assumptions

Documentation lives in:
- `immich.md` (or subfolder if it grows)

---

## Structure Guidelines

When adding a new backup target, document:
- **Data scope** (what is included / excluded)
- **Backup method** (snapshot, push, pull, export)
- **Frequency**
- **Retention**
- **Restore procedure (high-level)**
- **Known caveats**

Prefer clarity over completeness.

---

## Related Docs

- `docs/ops/` for operational context
- `docs/adr/` for architectural decisions impacting backups
- `OPS-0001-development-prioritization` for sequencing rationale

---

## Status

This is a living document.
Backup strategies may evolve as storage, compute separation, and compliance requirements mature.
