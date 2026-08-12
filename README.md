# Self Healing API and Agent OS

> Specifications for building self-healing APIs and agent-based systems through evidence, governance, and deterministic maintenance assessment.

## Vision

Modern software can detect failures long before users notice them, yet most systems only expose whether they are alive or ready. This project defines open specifications that enable software to assess whether it can continue operating as intended, identify required maintenance, and communicate that assessment using deterministic, evidence-based protocols.

The project is specification-first.

Reference implementations validate the specifications—not the other way around. This repository currently contains specifications and governance material; it does not contain a reference implementation.

## Principles

- Specifications before implementations.
- Evidence over assertion.
- Deterministic behavior over opinion.
- Governance through RFCs and ADRs.
- Real-world validation before standardization.
- Dogfood every specification.

## Repository Structure

```text
docs/
  adr/
    ADR-0001-project-license.md
  spec/
    operational-intelligence-specification-v0.1.md

CONTRIBUTING.md
LICENSE
NOTICE
```

Schema, OpenAPI, reference implementation, and conformance artifacts are not currently present in this repository.

## Roadmap

1. Formalize the Operational Intelligence v0.1 draft using evidence from the proof of concept.
2. Publish machine-readable schema and OpenAPI artifacts after the response and transport contracts stabilize.
3. Build or link reference implementations that validate the specification.
4. Create a conformance suite from the normative acceptance criteria.
5. Validate through real-world adoption and refine the specification from evidence.

## Status

The Operational Intelligence specification is at version `0.1.0-draft` and is in active development.

Current work is formalizing decisions identified during proof-of-concept review. The response contract now distinguishes grouped findings from occurrence counts and defines bounded evidence output:

- `uniqueFindingCount` counts grouped findings in a response.
- Each finding's `count` reports the occurrences represented by that finding.
- Evidence arrays are bounded by `output.maximumEvidencePerFinding`.
- `output.includeEvidence` controls whether supporting evidence records are returned.

The specification remains experimental until it has been validated by multiple reference implementations and real-world usage.

## Contributing

Please read [CONTRIBUTING.md](CONTRIBUTING.md) before opening issues or pull requests.

## License

This project is licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for details.
