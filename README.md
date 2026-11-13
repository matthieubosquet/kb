# kb

## Reasoning

The triple store (Apache Jena Fuseki) applies the **OWL Micro** rule reasoner at query time. OWL Micro covers RDFS (`subClassOf`, `subPropertyOf`, `domain`, `range`) and a useful subset of OWL (`inverseOf`, transitive and symmetric properties, `equivalentClass`, `equivalentProperty`, and some restrictions). It does not cover cardinality or property chains.

### Graphs

Files are loaded according to where they sit in `data/` and `ontology/`:

- A file directly in `data/` goes into the **default graph**.
- A file in a subfolder of `data/` goes into a **named graph** called `GRAPH_BASE` followed by the subfolder path. For example, `data/people/a.ttl` goes into `https://ns.example.org/people`.
- Each file in `ontology/` goes into its **own named graph**, called `GRAPH_BASE` followed by the file path without its extension. For example, `ontology/skos.ttl` goes into `https://ns.example.org/skos`.
- Every file in `shapes/` goes into the `https://ns.example.org/shapes` named graph, which is always excluded from inference (see [Validation](#validation)).

`GRAPH_BASE` is a build argument set in `compose.yaml`. Each vocabulary is its own file, and so its own named graph, so that each one can be included or excluded independently.

### What a query sees

A query without a `GRAPH` clause sees the **inferred view**:

- the default graph,
- every named graph that is not excluded,
- and everything the reasoner infers from those.

Named graphs listed in the `INFERENCE_EXCLUDE` build argument in `compose.yaml` are left out of both inference and the inferred view, but can still be queried explicitly with `GRAPH <…> { … }`. `INFERENCE_EXCLUDE` is a space-separated list of graph names (the part after `GRAPH_BASE`); an entry also excludes the graphs under it, so `people` excludes `people/staff`. When it is empty, every named graph is included. Because the reasoner runs over everything it can see, inclusion in the inferred view and inclusion in inference are the same setting.

Full-text search (`text:query`) covers the inferred view only, so excluded vocabularies do not appear in search results. It is on by default; set the `FULL_TEXT` build argument in `compose.yaml` to `"false"` to serve the dataset without a Lucene index, in which case `text:query` finds nothing.

### What to include

Include a vocabulary when the data uses it and its axioms give useful entailments. Examples:

- **SKOS:** `skos:prefLabel` is a subproperty of `rdfs:label`, and `broader` and `narrower` are inverses.
- **PROV-O**, **DCAT**, **DCMI terms**, **ORG**, **OWL-Time:** include these only if the domain uses them.

The domain ontology is always included, in its own file so that it can be told apart from the data, versioned, or switched off while debugging.

### What to exclude

Keep a vocabulary out of the inferred view when it:

- **is already implemented** by the software (see [Ignored ontologies](#ignored-ontologies)), so loading it adds noise and nothing for the reasoner to use;
- **causes unwanted entailments**, typically through broad `rdfs:domain`/`rdfs:range`, `owl:equivalentClass` or `owl:sameAs` alignments;
- **is not about the domain**, such as SHACL shapes, test fixtures, drafts, or older versions of an ontology;
- **is large or untrusted**, because the reasoner works in memory over everything it can see.

### Validation

SHACL shapes live in the `shapes/` folder, are loaded into their own excluded named graph, and are validated against the **inferred view**. That way, facts the reasoner can derive, such as a type implied by `rdfs:range`, count towards conformance. Jena's SHACL engine supports SHACL Core and SPARQL-based constraints, but not SHACL-AF rules.

Validation runs at build time. The build validates the inferred view against every shape in `shapes/`, and fails with the validation report if the data does not conform. When `shapes/` is empty, validation is skipped. Since the data only changes with a rebuild, this is the check before every update, and the running store has no SHACL endpoint.

## Ignored ontologies

These vocabularies are loaded into their own named graphs but excluded from inference and from the default graph, because the software already implements their semantics. They remain queryable with `GRAPH <…> { … }`, for example to look up labels and comments.

| Ontology | Namespace | Implemented by | Reason for excluding |
|---|---|---|---|
| RDF | `http://www.w3.org/1999/02/22-rdf-syntax-ns#` | Jena OWL Micro reasoner | Rules built into the reasoner |
| RDFS | `http://www.w3.org/2000/01/rdf-schema#` | Jena OWL Micro reasoner | Rules built into the reasoner |
| OWL | `http://www.w3.org/2002/07/owl#` | Jena OWL Micro reasoner | Rules built into the reasoner |
| XSD | `http://www.w3.org/2001/XMLSchema#` | Jena datatype system | Literal parsing and comparison are native; the reasoner ignores facets such as `xsd:pattern` |
| SHACL | `http://www.w3.org/ns/shacl#` | Jena SHACL engine (`jena-shacl`) | Shapes are constraints for validation, not domain facts |
