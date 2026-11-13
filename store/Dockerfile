ARG JAVA_VERSION=25
ARG ALPINE_VERSION=3
ARG JENA_VERSION=6.2.0
ARG FUSEKI_SHA1=d3490295a2b95677c1227d25bca8564d40f993a7
ARG GRAPH_BASE=https://ns.example.org/
# Space-separated graph names (data/ subfolder paths, or ontology/ file paths
# without extension) excluded from inference, along with the names under them
ARG INFERENCE_EXCLUDE=""
# true to serve the dataset with full-text search (text.ttl, Lucene index),
# false to serve the dataset directly
ARG FULL_TEXT=true


# Internal, passed between stages.
## Fuseki
ARG FUSEKI_DIR=/fuseki
ARG FUSEKI_JAR=jena-fuseki-server-${JENA_VERSION}.jar
## Java
ARG JAVA_MINIMAL=/opt/java-minimal


## ---- Stage: download Fuseki JAR
FROM docker.io/library/alpine:${ALPINE_VERSION} AS fuseki-download

ARG JENA_VERSION
ARG FUSEKI_DIR
ARG FUSEKI_SHA1
ARG FUSEKI_JAR
ARG REPO=https://repo1.maven.org/maven2
ARG FUSEKI_JAR_URL=${REPO}/org/apache/jena/jena-fuseki-server/${JENA_VERSION}/${FUSEKI_JAR}

WORKDIR ${FUSEKI_DIR}

# BusyBox wget (with ssl_client) ships in the Alpine base image, so there is
# no need to install curl. Integrity is enforced by the checksum either way.
RUN wget -q -O "$FUSEKI_JAR" "$FUSEKI_JAR_URL" && \
    echo "${FUSEKI_SHA1}  ${FUSEKI_JAR}" | sha1sum -c -


## ---- Stage: build minimal Java runtime
FROM docker.io/library/eclipse-temurin:${JAVA_VERSION}-alpine AS jre

ARG JAVA_MINIMAL
ARG FUSEKI_DIR
ARG FUSEKI_JAR

# Alpine: objcopy is needed by jlink --strip-debug
RUN apk add --no-cache binutils

WORKDIR ${FUSEKI_DIR}
COPY --from=fuseki-download ${FUSEKI_DIR}/${FUSEKI_JAR} ./

ARG JDEPS_EXTRA="jdk.incubator.vector"
RUN JDEPS="$(jdeps --multi-release base --print-module-deps --ignore-missing-deps "${FUSEKI_JAR}")" && \
    jlink \
        --compress=zip-6 --strip-debug --no-header-files --no-man-pages --generate-cds-archive \
        --output "${JAVA_MINIMAL}" \
        --add-modules "${JDEPS},${JDEPS_EXTRA}"


## ---- Stage: Default build context value for the config folder
FROM scratch AS config
COPY store/config/ /

## ---- Stage: Default build context value for the data folder
FROM scratch AS data
COPY store/data/ /

## ---- Stage: Default build context value for the ontology folder
FROM scratch AS ontology
COPY store/ontology/ /

## ---- Stage: Default build context value for the shapes folder
FROM scratch AS shapes
COPY store/shapes/ /


## ---- Stage: load data and build text index
FROM jre AS loader

ARG JAVA_MINIMAL
ARG FUSEKI_DIR
ARG FUSEKI_JAR
ARG DATABASE=./db
ARG GRAPH_BASE
ARG INFERENCE_EXCLUDE
ARG FULL_TEXT

ENV JAVA_HOME=${JAVA_MINIMAL}

WORKDIR ${FUSEKI_DIR}

COPY --from=data . ./data/
COPY --from=ontology . ./ontology/
COPY --from=shapes . ./shapes/
COPY --from=config ./config.ttl ./text.ttl ./dataset.ttl ./

# Load data and ontologies:
# - files directly in data/ go into the default graph
# - files in a subfolder of data/ go into the named graph
#   ${GRAPH_BASE}<subfolder path>, e.g. data/people/a.ttl -> ${GRAPH_BASE}people
# - each file in ontology/ goes into its own named graph
#   ${GRAPH_BASE}<file path without extension>,
#   e.g. ontology/skos.ttl -> ${GRAPH_BASE}skos
# - files in shapes/ go into the named graph ${GRAPH_BASE}shapes, which is
#   always excluded from inference
# Then complete dataset.ttl: every named graph not matched by INFERENCE_EXCLUDE
# (a space-separated list of graph names, which also matches the names under
# them, e.g. shapes matches shapes/person) feeds the reasoner, and every named
# graph is exposed with GRAPH.
RUN <<EOF
set -eu
LOADER="$JAVA_HOME/bin/java --add-modules jdk.incubator.vector -cp $FUSEKI_DIR/$FUSEKI_JAR tdb2.tdbloader --loader=parallel --loc $DATABASE"

# ontology/skos.ttl -> skos, ontology/vocab/foo.ttl.gz -> vocab/foo
ontology_graph() {
    echo "$1" | sed -E 's|^ontology/||; s/\.(gz|bz2)$//; s/\.[^./]*$//'
}

find data -maxdepth 1 -type f -exec $LOADER {} +

for file in $(find ontology -type f); do
    graph=$(ontology_graph "$file")
    $LOADER --graph="${GRAPH_BASE}${graph}" "$file"
done

graphs=$(
    find data -mindepth 2 -type f -exec dirname {} \; | sed 's|^data/||'
    for file in $(find ontology -type f); do ontology_graph "$file"; done
)
graphs=$(echo "$graphs" | sort -u)

touch named-graphs.ttl

for graph in $graphs; do
    iri="${GRAPH_BASE}${graph}"
    if [ -d "data/$graph" ]; then
        find "data/$graph" -maxdepth 1 -type f -exec $LOADER --graph="$iri" {} +
    fi

    printf '<urn:kb:dataset> ja:namedGraph [ ja:graphName <%s> ; ja:graph <urn:kb:graph:%s> ] .\n' "$iri" "$graph" >> named-graphs.ttl
    printf '\n<urn:kb:graph:%s> a tdb2:GraphTDB2 ; tdb2:location "./db" ; tdb2:graphName <%s> .\n' "$graph" "$iri" >> dataset.ttl

    excluded=no
    for pattern in $INFERENCE_EXCLUDE; do
        case "$graph" in "$pattern"|"$pattern"/*) excluded=yes ;; esac
    done
    if [ "$excluded" = no ]; then
        printf '<urn:kb:inference-base> ja:subModel <urn:kb:graph:%s> .\n' "$graph" >> dataset.ttl
        echo "Graph $iri: inference"
    else
        echo "Graph $iri: excluded from inference"
    fi
done

if [ -n "$(find shapes -type f)" ]; then
    iri="${GRAPH_BASE}shapes"
    find shapes -type f -exec $LOADER --graph="$iri" {} +
    printf '<urn:kb:dataset> ja:namedGraph [ ja:graphName <%s> ; ja:graph <urn:kb:graph:shapes> ] .\n' "$iri" >> named-graphs.ttl
    printf '\n<urn:kb:graph:shapes> a tdb2:GraphTDB2 ; tdb2:location "./db" ; tdb2:graphName <%s> .\n' "$iri" >> dataset.ttl
    echo "Graph $iri: SHACL shapes, excluded from inference"
fi
EOF

# Assemble config.ttl: the service, full-text search (if FULL_TEXT), then the
# dataset. Index before the named graphs are exposed, so full-text search only
# covers the inferred view (excluded vocabularies stay out of search results).
# Without full-text search the service serves the dataset directly, and
# fts-index/ stays empty.
RUN <<EOF
set -eu
case "$FULL_TEXT" in
    true)
        cat text.ttl dataset.ttl >> config.ttl
        "$JAVA_HOME/bin/java" --add-modules jdk.incubator.vector \
            -cp "$FUSEKI_DIR/$FUSEKI_JAR" \
            jena.textindexer --desc config.ttl
        ;;
    false)
        sed -i 's|<urn:kb:text-dataset>|<urn:kb:dataset>|' config.ttl
        cat dataset.ttl >> config.ttl
        mkdir -p fts-index
        echo "Full-text search disabled"
        ;;
    *)
        echo "FULL_TEXT must be true or false, got '$FULL_TEXT'"; exit 1
        ;;
esac
cat named-graphs.ttl >> config.ttl
EOF

# Validate the inferred view against the SHACL shapes in shapes/, with Jena's
# command-line tools: materialise the inferred view from dataset.ttl (the same
# dataset Fuseki serves, without the search wrapper), then validate it. No server runs, and the SHACL engine never reaches the
# runtime config.
RUN <<EOF
set -eu
if [ -z "$(find shapes -type f)" ]; then
    echo "No shapes: skipping SHACL validation"
    exit 0
fi

JENA="$JAVA_HOME/bin/java --add-modules jdk.incubator.vector -cp $FUSEKI_DIR/$FUSEKI_JAR"
work=$(mktemp -d)

$JENA riotcmd.riot --output=nt $(find shapes -type f) > "$work/shapes.nt"
$JENA arq.sparql --desc=dataset.ttl --results=ttl \
    'CONSTRUCT { ?s ?p ?o } WHERE { ?s ?p ?o }' > "$work/inferred.ttl"

$JENA shacl.shacl validate --shapes="$work/shapes.nt" --data="$work/inferred.ttl" > "$work/report.ttl"

if grep -Eq 'conforms>?[[:space:]]+true' "$work/report.ttl"; then
    echo "SHACL validation: conforms"
else
    echo "SHACL validation failed:"; cat "$work/report.ttl"; exit 1
fi
EOF


## ---- Stage: runtime
FROM docker.io/library/alpine:${ALPINE_VERSION}

ARG JENA_VERSION
ARG JAVA_MINIMAL
ARG FUSEKI_DIR
ARG FUSEKI_JAR
ARG JENA_USER=fuseki
ARG JENA_GROUP=$JENA_USER
ARG JENA_GID=1000
ARG JENA_UID=1000

WORKDIR ${FUSEKI_DIR}

# -H : no home directory, -D : no password
RUN addgroup -g "${JENA_GID}" "${JENA_GROUP}" && \
    adduser "${JENA_USER}" -G "${JENA_GROUP}" -s /bin/ash -u "${JENA_UID}" -H -D && \
    mkdir -p run && \
    chown "${JENA_USER}:${JENA_GROUP}" run

# JRE, JAR and config stay root-owned (read-only for the service user).
COPY --link --from=jre ${JAVA_MINIMAL} ${JAVA_MINIMAL}
COPY --link --from=fuseki-download ${FUSEKI_DIR}/${FUSEKI_JAR} ./
COPY --link --from=config ./log4j2.properties .
COPY --link --from=loader --chown=${JENA_UID}:${JENA_GID} ${FUSEKI_DIR}/db/ ./db/
COPY --link --from=loader --chown=${JENA_UID}:${JENA_GID} ${FUSEKI_DIR}/fts-index/ ./fts-index/
# config.ttl as completed by the loader (named graphs and inference set)
COPY --link --from=loader ${FUSEKI_DIR}/config.ttl .

USER ${JENA_USER}

# Heap sized from the container memory limit. TDB2 uses memory-mapped files
# outside the heap, so leave headroom rather than giving the heap everything.
ENV \
    JAVA_HOME=${JAVA_MINIMAL} \
    JAVA_OPTIONS="--enable-native-access=ALL-UNNAMED --add-modules=jdk.incubator.vector -XX:MaxRAMPercentage=50 -XX:InitialRAMPercentage=50" \
    JENA_VERSION=${JENA_VERSION} \
    FUSEKI_JAR="${FUSEKI_JAR}" \
    FUSEKI_DIR="${FUSEKI_DIR}"

EXPOSE 3030

ENTRYPOINT ["/bin/sh", "-c", "exec \"$JAVA_HOME/bin/java\" $JAVA_OPTIONS -jar \"$FUSEKI_DIR/$FUSEKI_JAR\" \"$@\"", "--"]

CMD ["--conf", "./config.ttl"]
