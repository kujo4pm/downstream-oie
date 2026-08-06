# Learning Integration Engines by Doing: HL7v2 → Your SENAITE Instrument Workflow

**Tool:** Open Integration Engine (OIE) — free, open-source, drop-in compatible with Mirth Connect channels.
**Project:** Rebuild the instrument-result path of your SENAITE FHIR IG in HL7v2, so you're translating a workflow you already understand at the FHIR/FSH level into the interface-engine world.
**Time:** ~5 sessions of 2-3 hours. Do them in order; each produces a working channel you keep.

---

## Why OIE and not Mirth

NextGen moved Mirth Connect to a fully commercial license starting at v4.6 (2025), ending new open-source releases. OIE is the community fork that split off at that point — same engine, same channel format, same JavaScript-based transformer model, actively developed, and it's what most NHS trusts and mid-market integrators are migrating to rather than paying for Mirth licenses. Skills transfer 1:1 either direction.

- Docs: https://docs.openintegrationengine.org/
- Project site / downloads: https://openintegrationengine.org/
- GitHub: https://github.com/OpenIntegrationEngine
- Quick Docker start (no installer needed): search "Open Integration Engine docker" on the GitHub repo's README — a container pull gets you running in minutes.

If you'd rather use the reference material NextGen already produced (still broadly accurate for concepts, just re-labelled OIE in your head), the getting-started docs are here: https://docs.nextgen.com/en-US/mirthc2ae-connect-by-nextgen-healthcare-user-guide-3273569/getting-started-with-mirthc2ae-connect-13713

---

## Session 1 — Install, orient, "Hello Channel" (2-3 hrs)

**Do:**
1. Install OIE (Docker is fastest: pull the image, expose port 8443, open `https://localhost:8443` in the Administrator).
2. Tour the four things you'll live in: **Channels** list, the **Channel Editor** (Source → Filter/Transformer → Destination(s)), the **Dashboard** (live status, message counts, start/stop), and the **Message Browser** (every message that's passed through, raw and transformed).
3. Concepts to nail down, since they map directly onto things you already do in FSH/SUSHI:
   - **Channel** ≈ your IG's Implementation Guide scope — one bounded workflow.
   - **Source connector** ≈ where a message enters (LLP listener, HTTP listener, file reader).
   - **Filter** ≈ your FSH invariants — decide whether a message proceeds at all.
   - **Transformer** ≈ your FSH mapping — reshape source data into your target model, step by step, using small JavaScript snippets against segment/field indices instead of FSH rules against FHIRPath.
   - **Destination connector** ≈ where the transformed message goes out (another LLP endpoint, HTTP POST, database write, file).
   - **Channel Map / Global Map** ≈ variables you stash mid-pipeline, like intermediate values in a FSH example instance.

**Build:** A trivial channel — LLP listener on port 6661 → a JavaScript transformer that just logs the raw message → a "Channel Writer" or dummy destination. Use the built-in **Send Message** test tool in the Administrator to fire a test HL7 string at it and watch it appear in the Dashboard and Message Browser. Goal for today is purely: message in → visible in engine → message out, nothing more.

---

## Session 2 — HL7v2 anatomy, hands-on (2-3 hrs)

You've been living in FHIR R5; HL7v2 will feel like a much older, positional, pipe-delimited cousin. The segments you'll need for lab results:

| Segment | Purpose | FHIR-world analogue |
|---|---|---|
| MSH | Message header (sending/receiving app, message type, timestamp) | Bundle.meta / MessageHeader |
| PID | Patient identification | Patient |
| ORC | Common order data | ServiceRequest (shared fields) |
| OBR | Observation request (the "test ordered") | ServiceRequest (specific) |
| OBX | Observation result — one per analyte/result value | Observation |
| NTE | Free-text note, often attached to OBX | Observation.note |

**Do:** Hand-craft a minimal ORU^R01 (unsolicited result) message representing one instrument result — pick something concrete from your SENAITE analysis services, e.g. a chemistry analyte with a numeric result, units, and reference range. Send it into last session's channel using the Send Message tool and step through the Message Browser's parsed tree view so you can see OIE break MSH/PID/OBR/OBX apart automatically.

**Reference:** any HL7v2 message structure lookup (Caristix's HL7 database or the raw HL7.org tables) for exact segment/field numbering — you don't need to memorize it, just know where OBX-5 (value), OBX-6 (units), OBX-7 (reference range) and OBX-8 (abnormal flag) live.

---

## Session 3 — The real project: instrument result → FHIR (2-3 hrs)

This is where it becomes *your* workflow instead of a generic tutorial.

**Recall what you already modeled:** your SENAITE IG's instrument/results side carries detection limits, reference ranges, performer/verifier identity, and method coding on Observation. That's the target shape. Now build the HL7v2 source side that would feed it in a real lab.

**Build a channel:**
- **Source:** LLP listener simulating an instrument sending ORU^R01.
- **Transformer (the meat):** JavaScript steps that pull each OBX field into channel map variables — value, units, reference range low/high, abnormal flag, performer (OBX-16), method (stash in NTE or a custom Z-segment if your instrument uses one — many chemistry analyzers do).
- **Destination:** an HTTP Sender connector that POSTs a FHIR `Observation` (or a `DiagnosticReport` bundling several OBX-derived Observations) to a target URL. Point this at a mock endpoint first (e.g. https://webhook.site for a throwaway URL you can watch fill up with your JSON), then — once it's clean — at a real SENAITE FHIR endpoint you control, so you're closing the loop from instrument-shaped HL7v2 to your own IG's profile shape.

**Stretch goal:** carry over the specific fields you fought with in FSH recently — detection limits and the SMILES/cheminformatics field. Map an OBX or Z-segment value into the matching Observation.component or extension slot, so you're proving out the same conformance rules end-to-end rather than in isolation.

---

## Session 4 — Orders direction: ServiceRequest and your `$revoke` operation (2-3 hrs)

Since you've already designed a custom `$revoke` operation for ServiceRequest cancellation, this session closes the other direction of the workflow.

- Inbound message type: **OML^O21** (lab order) or the simpler **ORM^O01**.
- Key field: **ORC-1** (order control code) — `NW` (new order), `CA` (cancel), `XO` (change).
- **Build:** a second channel, LLP listener for OML/ORM → transformer maps ORC-1/OBR into a FHIR `ServiceRequest` → destination HTTP POST. Branch the transformer logic (or use a second destination with a filter) so that `ORC-1 = CA` triggers a call to your `$revoke` operation instead of a plain create/update — this is the piece that actually rehearses the workflow you designed rather than a generic order message.

---

## Session 5 — Round-trip, validation, and failure handling (2-3 hrs)

- Wire Session 3's result channel and Session 4's order channel so a simulated order → simulated instrument result → FHIR output flows end to end using consistent identifiers (same specimen/accession number threaded through both).
- Add a validation step in the transformer (simple JS assertions, or OIE's built-in message validator) that rejects malformed OBX segments and generates an HL7 ACK/NACK back to the source — this is the piece every real instrument interface needs and every tutorial skips.
- Deliberately break something (drop a required OBX-5, mistype a date) and watch how the Dashboard surfaces the error, then fix your transformer defensively. This is the part of interface-engine work that's genuinely different from FHIR IG authoring — you're now responsible for a live, always-on pipe, not a static conformance artifact.

---

## Where this leaves you

By the end you'll have two working channels (results-in and orders-in) that mirror the exact instrument workflow you've already specified in FSH, just built from the other side of the interface. That's the most transferable thing to show a client or employer: not "I read the OIE docs" but "I took a real IG I authored and built the engine-side interface for it."

### Reference links
- OIE docs: https://docs.openintegrationengine.org/
- OIE project / downloads: https://openintegrationengine.org/
- OIE GitHub (source, issues, community): https://github.com/OpenIntegrationEngine
- NextGen's original getting-started guide (concepts still apply): https://docs.nextgen.com/en-US/mirthc2ae-connect-by-nextgen-healthcare-user-guide-3273569/getting-started-with-mirthc2ae-connect-13713
- Community: OIE Discord (linked from the project site) — good for "why won't my transformer fire" questions specific to the engine.
