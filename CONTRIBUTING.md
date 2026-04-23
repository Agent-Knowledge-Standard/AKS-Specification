# Contributing to AKS

AKS is a community-governed spec. The reference implementation is maintained by ThoughtFoundry, but the spec itself belongs to the community. Proposals, corrections, and implementations from anyone are welcome.

---

## What Can Be Contributed

**Spec proposals** — new fields, new endpoints, changes to existing semantics, or corrections to the spec prose. Proposals go through the RFC process below.

**Conformance tests** — additional test cases for the conformance checklist. Pull requests against `CONFORMANCE.md` or a new `tests/` directory.

**Glossary additions** — terms used in practice that are not yet defined in `GLOSSARY.md`. Pull requests with precise, single-paragraph definitions are welcome without an RFC.

**Alternative implementations** — if you have built an AKS-compliant server in a different language or on a different stack, open an issue to have it listed in the ecosystem section of the README.

**Example bundles** — real or synthetic AKS bundles that demonstrate the spec in specific domains. Add them to `examples/` with a README explaining the domain and what it demonstrates.

**Corrections** — typos, ambiguous wording, broken links. Small pull requests are welcome without an RFC.

---

## The RFC Process

Significant changes to the spec require an RFC (Request for Comments). RFC stands for Request for Comments — a process borrowed from the internet engineering community that produced the specs for HTTP, TCP/IP, and JSON. The name is intentionally humble: you are not announcing a decision, you are asking people to comment on a proposal before it becomes official.

An RFC is required for:
- Adding a new required field to any schema object
- Removing an existing field
- Changing an existing field's type or enum values
- Adding a new required endpoint
- Removing an existing endpoint
- Changing the meaning of an existing term in the glossary

An RFC is not required for:
- Adding an optional field (still RECOMMENDED to open an issue first)
- Fixing a typo or clarifying prose
- Adding an example bundle
- Updating the conformance checklist tests

### RFC Steps

1. Open an issue with the title `RFC: [brief description]`
   Example: `RFC: add document_count to AKSBundle`

2. In the issue body, describe:
   - The problem you are solving, not just the solution you want
   - The specific change — exact field names, types, and descriptions as they would appear in README.md and SCHEMA.json
   - How existing conformant implementations would be affected
   - Whether this is a breaking change

3. Label the issue `rfc`

4. Allow two weeks for community comment. If significant concerns are raised, the comment period extends until consensus is reached or the RFC is withdrawn.

5. If there is rough consensus and no unresolved blocking objections, a maintainer opens a pull request implementing the RFC. The pull request references the RFC issue.

6. Breaking changes that introduce a new `aks_version` value require a separate migration guide.

---

## What Is Out Of Scope

The spec governs the export format and query interface of compiled knowledge. Pull requests about the following will be closed:

- How knowledge is compiled internally (chunking strategy, embedding model, extraction prompt)
- How agents use compiled knowledge downstream
- Pricing, licensing, or commercial use of any specific implementation
- UI or product decisions in ThoughtFoundry or any other implementation
- Tool definitions or MCP server configuration (these belong in the MCP ecosystem)

---

## Code of Conduct

Be direct. Be precise. Disagree on ideas, not on people. Proposals that are vague, that change the spec without evidence of a real problem, or that serve a specific commercial interest without community benefit will be closed without comment.

---

## Local Development

Clone the repo and validate the schema and example bundles:

    git clone https://github.com/your-org/aks-spec
    cd aks-spec
    pip install check-jsonschema jsonschema

Validate the schema is itself valid JSON Schema:

    check-jsonschema --check-metaschema SCHEMA.json

Validate example bundles against the schema:

    check-jsonschema --schemafile SCHEMA.json examples/ecosystems-stack.json

Run all validations:

    for f in examples/*.json; do
      echo -n "Validating $f... "
      python3 -c "
    import json, jsonschema
    schema = json.load(open('SCHEMA.json'))
    bundle = json.load(open('$f'))
    jsonschema.validate(bundle, schema)
    print('ok')
      "
    done
