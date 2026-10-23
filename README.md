# Snowstorm developer workshop

Material for the hands-on workshop **Run a SNOMED CT terminology server on your laptop**: Snowstorm 11 and Elasticsearch 8 in Docker, then the FHIR terminology API from Postman and from your own code.

| Folder | What is in it |
|---|---|
| [`slides/`](slides/) | The workshop deck as a PDF (29 slides) |
| [`postman/`](postman/) | The Postman collection (52 requests in 10 folders) and the environment for the shared test server |
| [`local/`](local/) | `compose.yaml` to run Snowstorm and Elasticsearch on your own machine |

The workshop has two halves, and you can do them in either order:

- **[Part A · Hands-on with Postman](#part-a--hands-on-with-postman)** calls our shared test server, so nobody waits for an import. This is what we do together in the session.
- **[Part B · Run Snowstorm on your laptop](#part-b--run-snowstorm-on-your-laptop)** takes about an hour and a half, most of it the import. Start it before the session, or do it at home afterwards.

---

## Part A · Hands-on with Postman

### What you need

- [Postman](https://www.postman.com/downloads/), desktop app or web, signed in to any free account
- The workshop **password**, which the facilitator gives you in the room. It is not in this repository.

### 1. Import the collection and the environment

**From a link (no download):** in Postman choose **Import**, paste each link below in turn, and click **Import**.

```
https://raw.githubusercontent.com/drmayu7/snowstorm-dev-workshop/main/postman/Snowstorm-hands-on.postman_collection.json
https://raw.githubusercontent.com/drmayu7/snowstorm-dev-workshop/main/postman/api-test.mysnomed.com.postman_environment.json
```

**Or from files:** download the repository (**Code → Download ZIP**), unzip it, then in Postman choose **Import** and drop in both `.json` files from `postman/`. Postman cannot import the ZIP itself.

### 2. Select the environment

Top right of Postman, pick **api-test.mysnomed.com** from the environment list.

### 3. Type in the password

Open the environment (the eye icon, then **Edit**) and fill in the **Current value** column:

| Variable | Current value |
|---|---|
| `username` | `workshop` (already filled in) |
| `password` | the password from the facilitator |

Leave **Initial value** empty. Initial values are shared when you export, sync or share a workspace; current values stay on your machine.

### 4. Check you are connected

Open folder **0 · Check you are connected** and send **0.1 Server version**. Expect `200` and `"version": "10.12.2"`. A `401` means step 2 or step 3 is not done.

### The folders

| Folder | What it teaches |
|---|---|
| 0 · Check you are connected | Server version; the code systems it serves, native and FHIR |
| 1 · Look up a code | `$lookup`: display, British English, parent concepts |
| 2 · Search in a value set | `$expand` with ECL: type-ahead search, totals, paging |
| 3 · Validate a code | `$validate-code`: is this code allowed in this field? |
| 4 · Inactive codes and history | Spotting an inactive code and finding its replacement with `$translate` |
| 5 · KKM health facilities | A national (non-SNOMED) code system hosted on the same server |
| 6 · The native SNOMED API | Term search, a whole concept, ECL: the API behind the SNOMED CT browser |
| 7 · Pitfalls, deliberately wrong | `_count`, `count=0` and `Accept-Language: *`: mistakes you will recognise later |
| 8 · Pharmacy examples | Clinical drugs, dose forms, drug classes, allergies, and the salt trap |
| 9 · Clinical examples | Diagnoses, findings, procedures, body sites, ICD-10 maps, LOINC lab orders |

Every request has a **Tests** tab: a green tick in the response pane means you got the expected answer. Request **8.13** also draws a table: open the **Visualize** tab in the response pane.

### House rules for the test server

- **Send requests one at a time. Do not use the Collection Runner.** The whole room shares one internet address, and the server limits each address (you get `503`).
- **Never put the password in a screenshot, a shared workspace or an exported file.**
- The server is **read-only**: nothing you send can change it, so experiment freely.
- For British English names (paracetamol, not acetaminophen) add `displayLanguage=en-GB`. Do not rely on the `Accept-Language` header: responses are cached for 24 hours and the cache ignores that header.
- The server stops at **21:00 Malaysia time** every night, and the `workshop` login is **removed after the workshop**. To keep practising, run your own server (Part B).

### When a request fails

| You see | Most likely | Do this |
|---|---|---|
| `401 Unauthorized` | No environment selected, or the password is only in Initial value | Select api-test.mysnomed.com; fill in Current value |
| `500` on `/version`, `/codesystems`, `/MAIN/...` | A FHIR `Accept` header sent to the native API | Send `application/fhir+json` to `/fhir` only (the collection does this for you) |
| `500 HAPI-0389 ... Accept-Language '*'` | Your client sends `Accept-Language: *` (Node's `fetch` does) | Send `en` or `en-GB` |
| `500 HAPI-0389 ... / by zero` | `count=0` on `$expand` | Use `count=1` and read `expansion.total` |
| `400 Bad Request`, an HTML page | A raw `<` in a URL you built by hand or typed into curl | Encode it: `<<` is `%3C%3C` |
| `200`, but `total` is 0 | An inactive or mistyped SCTID in the ECL, or a filter under 3 letters | `$lookup` the code; type 3 letters or more |
| `200`, but in the other language | A cached response | Use `displayLanguage=en-GB` |
| `503` from nginx | More than 20 requests a second from the room | Wait a few seconds; no Collection Runner |
| `503` from the load balancer | The server is stopped (every night at 21:00) | Tell the facilitator |

### From Postman to your code

Slide 24 shows the same `$expand` call in JavaScript and Python. The short version, in Python:

```python
import os, requests

r = requests.get(
    "https://api-test.mysnomed.com/fhir/ValueSet/$expand",
    params={"url": "http://snomed.info/sct?fhir_vs=ecl/<<763158003",
            "filter": "parac", "count": 10, "displayLanguage": "en-GB"},
    auth=(os.environ["TS_USER"], os.environ["TS_PASS"]),
    headers={"Accept": "application/fhir+json"},
    timeout=10)
print(r.json()["expansion"]["total"])
```

Keep the credentials in environment variables, never in the code. Postman, `URLSearchParams` and `requests` encode the ECL for you; only a URL you build by hand needs `<<` written as `%3C%3C`.

---

## Part B · Run Snowstorm on your laptop

Two containers: **Elasticsearch** holds every SNOMED component on a Docker volume; **Snowstorm** serves the FHIR API on `localhost:8080`. Both ports bind to `127.0.0.1`, so nothing else on the network can reach them.

### Before you start

- **Docker** with Compose v2 (`docker compose version`), with at least **8 GB of memory** for Docker (Docker Desktop: Settings → Resources)
- **20 GB free** disk inside Docker. The loaded index is about 2.6 GB, but Elasticsearch stops placing data once the disk is 90% full.
- Ports **8080** and **9200** free on localhost
- `curl` and `jq`, or Postman
- **Linux or Windows with WSL2 only:** raise `vm.max_map_count` for Elasticsearch

  ```bash
  # Linux
  sudo sysctl -w vm.max_map_count=262144
  # Windows + WSL2 (PowerShell)
  wsl -d docker-desktop sysctl -w vm.max_map_count=262144
  ```

### Step 1 · Get the release file from MLDS

1. Sign in to [MLDS](https://mlds.ihtsdotools.org) as an affiliate in a member country. Malaysia is a member country.
2. Download the latest **International Edition** RF2 release, for example `SnomedCT_InternationalRF2_PRODUCTION_20261001T120000Z.zip` (about 588 MB). Keep the file name: it carries the release date.
3. **Keep it zipped and out of git.** Snowstorm reads the zip as it is. The content is licensed: never commit it or share it.

### Step 2 · Get `compose.yaml`

Clone this repository (or download [`local/compose.yaml`](local/compose.yaml)) and work in `local/`:

```bash
git clone https://github.com/drmayu7/snowstorm-dev-workshop.git
cd snowstorm-dev-workshop/local
```

Four choices in that file, each a lesson from running Snowstorm in production:

1. **Published images, pinned by version** (`snowstorm:11.0.0`, `elasticsearch:8.19.8`). Upstream's compose file builds from source.
2. **Localhost-only ports**: `127.0.0.1:8080` keeps the server off the workshop Wi-Fi.
3. **`JAVA_TOOL_OPTIONS`, not `JAVA_OPTS`**: the image starts `java` directly and ignores `JAVA_OPTS`, so a heap set there silently does nothing.
4. **One read-only switch** (`READONLY`): writable to load the release, then read-only to serve it.

### Step 3 · Start it, and check both containers

```bash
docker compose up -d                                  # 8 GB or more for Docker
ES_HEAP=2g SNOWSTORM_HEAP=2g docker compose up -d     # smaller laptops

curl localhost:8080/version                           # { "version" : "11.0.0", ... }
curl -s localhost:9200/_cluster/health | jq .status   # "green"
curl -s localhost:8080/codesystems | jq '.items[].shortName'   # "SNOMEDCT", empty until step 4
```

On an empty volume Snowstorm answered `/version` 36 seconds after `up`. Then open `http://localhost:8080` for the Swagger UI (every native endpoint) and `http://localhost:8080/fhir/metadata` for the FHIR CapabilityStatement.

### Step 4 · Import the RF2 snapshot

```bash
# 1. Create an import job; the Location header carries its id
curl -i -X POST localhost:8080/imports \
  -H 'Content-Type: application/json' \
  -d '{"branchPath":"MAIN","type":"SNAPSHOT","createCodeSystemVersion":true}'

# 2. Upload the zip to that job
curl -X POST -F file=@SnomedCT_InternationalRF2_PRODUCTION_20261001T120000Z.zip \
  localhost:8080/imports/<id>/archive

# 3. Poll until COMPLETED
curl -s localhost:8080/imports/<id> | jq .status
```

The 20261001 release took **65 minutes** on a MacBook with 6 GB for Docker (upstream says 30 to 60). `docker compose logs -f snowstorm` shows batches of 5,000 components going in. Leave it alone while it runs: a restart halfway means a clean volume and a fresh start.

### Step 5 · Switch to read-only, and keep the data

```bash
READONLY=true docker compose up -d     # recreates Snowstorm only; Elasticsearch and the data stay
curl -s -o /dev/null -w '%{http_code}\n' -X POST localhost:8080/imports    # 403
```

| Command | What happens to the data |
|---|---|
| `docker compose stop` | Stopped, data kept |
| `docker compose down` | Containers removed, data kept |
| `docker compose down -v` | **Index deleted**: you import again |

Why bother with read-only on a laptop? Because your application will meet a read-only server everywhere else. If it only works against a writable one, you find out in production.

Point the Postman environment's `baseUrl` at `http://localhost:8080` (no login needed) to run the same collection against your own server. Expect some red ticks: counts differ with the release, folder 5 and request 9.13 need the KKM and LOINC content that only the test server has, and Snowstorm 11 answers some of the folder 7 mistakes differently from the test server's 10.12.2.

### Pitfalls we hit, so you don't

| You see | What is going on | Do this |
|---|---|---|
| First start takes many minutes; "Creating index" every 30 s | Docker's disk is over 90% full, so Elasticsearch refuses to place data | Free space (`docker builder prune`); keep 20 GB free |
| The heap is not what you set | The image ignores `JAVA_OPTS` | Use `JAVA_TOOL_OPTIONS`; check "Picked up" in the log |
| First search after a restart takes ~30 s | Snowstorm's ECL cache is in memory and starts empty | Warm your application's value sets after every restart |
| `200`, but `total` is 0 | The ECL names an inactive concept | Check every value set's size after each release |
| `400 Unknown SNOMED Concept Id` (11.x) or an empty `200` (10.x) | A mistyped SCTID in the ECL | A minimum-size test per value set |
| Results look right, the `\|term\|` is wrong | Snowstorm does not check `\|terms\|` against codes | Never trust a term you did not look up |
| LOINC or national codes are missing | Unversioned ECL searches the International Edition only | Name the edition: `http://snomed.info/sct/<module>?fhir_vs=...` |
| Acetaminophen, not Paracetamol | US English is the default display language | Send `displayLanguage=en-GB` |

### Your laptop is not a server

| On your laptop | On a server |
|---|---|
| Ports on 127.0.0.1, no login | A gateway: TLS, one login per client system, rate limits |
| Elasticsearch security off | Elasticsearch on a private network only, never a published port |
| Writable to load, then read-only | Read-only always; releases built and tested elsewhere |
| Version tag `11.0.0` | Image pinned by digest, upgraded on purpose |
| One node, restart when you like | Two nodes or a cluster; warm the cache after restarts |

---

## Further reading

- Snowstorm: [github.com/IHTSDO/snowstorm](https://github.com/IHTSDO/snowstorm) ([using Docker](https://github.com/IHTSDO/snowstorm/blob/master/docs/using-docker.md), [loading SNOMED](https://github.com/IHTSDO/snowstorm/blob/master/docs/loading-snomed.md), [the FHIR API](https://github.com/IHTSDO/snowstorm/blob/master/docs/using-the-fhir-api.md), [releases](https://github.com/IHTSDO/snowstorm/releases))
- Expression Constraint Language (ECL): [docs.snomed.org](https://docs.snomed.org/snomed-ct-specifications/snomed-ct-expression-constraint-language)

## SNOMED CT licence

SNOMED CT® is a registered trademark of SNOMED International. The concept identifiers and terms in these slides and requests are quoted for teaching. Using SNOMED CT in your own systems needs a licence: in Malaysia, a member country, register as an affiliate through [MLDS](https://mlds.ihtsdotools.org). This repository contains no SNOMED CT release files.
