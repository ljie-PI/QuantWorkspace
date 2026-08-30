# Domain Docs

Before exploring, read root `CONTEXT.md` when present and relevant ADRs under `docs/adr/`. Missing files are not errors; domain-modeling creates them lazily.

## Layout

This is a single-context repository:

```text
/
├── CONTEXT.md
├── docs/adr/
└── src/
```

Use the glossary vocabulary from `CONTEXT.md`. If work contradicts an ADR, surface the conflict explicitly instead of silently overriding it.
