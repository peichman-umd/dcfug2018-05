title: Fixity
author:
  name: Peter Eichman
  email: peichman@umd.edu
theme: theme
output: index.html

--

# Flexible Fixity Checking in Fedora

## sha256:5c60ad5c70208fcfbf88ce4cdcd25c73ad246cb3e9399cb70beee0f5b9a9c2fc

--

### When?

- On ingest
  - Did the resource get transferred correctly?
- "Every so often"
  - Has the resource maintained integrity over time?

--

### Fedora's Existing Infrastructure

- [fcrepo-fixity](https://github.com/fcrepo4-exts/fcrepo-camel-toolbox/tree/master/fcrepo-fixity) route
- Part of <https://github.com/fcrepo4-exts/fcrepo-camel-toolbox>

--

### Our Extra Routes

- SPARQL query transformation route: [umd-fcrepo-sparql-query](https://github.com/umd-lib/umd-fcrepo-camel-extensions/tree/develop/umd-fcrepo-sparql-query)
- Notification route: [umd-fcrepo-notification](https://github.com/umd-lib/umd-fcrepo-camel-extensions/tree/develop/umd-fcrepo-notification)
- Extension repo: <https://github.com/umd-lib/umd-fcrepo-camel-extensions>

--

### Camel Side Note

- Multiply-instantiatable Camel routes
- Factory and instance abstract base classes
- One config file == one instance
- Part of [umd-osgi-extensions](https://github.com/umd-lib/umd-fcrepo-camel-extensions/tree/develop/umd-osgi-extensions)

--

### Common Fixity Pipeline

- Send message to fcrepo-fixity
- On success:
  - Log to filesystem
  - Record fixity result in audit triplestore
- On failure:
  - Log to filesystem
  - Record fixity result in audit triplestore
  - Send an email notification

--

### Fixity Flow

![fixity-flow.png](fixity-flow.png)

--

### Checking On Ingest

- Add `fedorafixity` to the list of broadcast queues
- Whenever a resource is modified, a message will appear on the `fedorafixty` queue
- Follows the established fixity flow pipeline

--

### Checking "Every so Often"

- Run this nightly via `cron`
- Query index and audit triplestores for a list of fixity check candidates
- Take the first 7000 candidates
- Checking takes ~1 hour

--

### Determining Fixity Check Candidates

1. Resources that have never had a fixity check
    * sort by last modified date, oldest to newest
2. Resources that have had a fixity check
    * sort by most recent fixity check date, oldest to newest
3. Omit resources that have had a fixity check within the last six months

--

### All Binaries

```
PREFIX fedora: <http://fedora.info/definitions/v4/repository#>

SELECT ?sub WHERE {
  ?sub a fedora:Binary ; 
    fedora:lastModified ?timestamp .
} ORDER BY ?timestamp
```

--

### All Resources with Fixity

```
PREFIX premis: <http://www.loc.gov/premis/rdf/v1#> 

SELECT ?resource WHERE {
  ?event premis:hasEventType <http://id.loc.gov/vocabulary/preservation/eventType/fix> ;
    premis:hasEventRelatedObject ?resource ;
    premis:hasEventDateTime ?timestamp .
} ORDER BY ?timestamp
```

--

### All Resources with Fixity, Timeboxed

```
PREFIX premis: <http://www.loc.gov/premis/rdf/v1#>
PREFIX xs: <http://www.w3.org/2001/XMLSchema#>

SELECT ?resource
WHERE {
  {
    SELECT ?resource (MAX(?date) AS ?most_recent_date)
    WHERE {
      SELECT ?resource ?date
      WHERE {
        ?event premis:hasEventRelatedObject ?resource ;
          premis:hasEventType <http://id.loc.gov/vocabulary/preservation/eventType/fix> ;
          premis:hasEventDateTime ?date .
      }
    }
    GROUP BY ?resource
  }
  FILTER (?most_recent_date < "$TIMEBOX"^^xs:dateTime)
}
ORDER BY ASC(?most_recent_date)
```

--

### Regular Fixity Checking

```bash
log "Checking a maximum of $MAX_CANDIDATES resources with fixity checks \
    no more recent than $TIMEBOX"

head -n "$MAX_CANDIDATES" <(./all_fcrepo_uris_without_fixity.sh) > candidates.txt

# use redirection so the filename doesn't get output
count=$(wc -l <candidates.txt)

if [ "$count" -lt "$MAX_CANDIDATES" ]; then
    extra=$((MAX_CANDIDATES - count))
    # ensure that we are only getting resources that actually still exist
    ./all_fcrepo_uris_with_fixity_timeboxed.sh "$TIMEBOX" \
        | grep -F -f <(./all_fcrepo_uris_in_fuseki.sh) \
        | head -n "$extra" \
        >> candidates.txt
fi

log "Found $(wc -l <candidates.txt) resources for fixity checking"

if [ -z "$DRY_RUN" ]; then
    tr -d "\r" <candidates.txt | ./fixitycheck.sh
fi
```