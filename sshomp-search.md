---
name: sshomp-search
description: >
  Query the SSH Open Marketplace (SSHOMP) API to find resources for social sciences and humanities research. 
  Use this skill whenever the user wants to find, explore, or look up items in the SSH Marketplace — including 
  tools, training materials, datasets, publications, workflows, or actors. Trigger on phrases like 
  "find me a tool for X", "search the marketplace for Y", "what's available for Z in SSHOMP", 
  "look up resources for digital humanities / SSH", "find training materials on X", or any request 
  to discover or retrieve SSH Open Cloud Marketplace resources. Also trigger when the user asks 
  about a specific persistent ID (e.g. "what is n4Dv8y?") or wants to explore what categories of 
  resources are available.
---
 
# SSH Open Marketplace Search Skill
 
The SSH Open Marketplace (SSHOMP) is a discovery portal for tools, training materials, datasets, 
publications, and workflows relevant to Social Sciences and Humanities research.
 
**Base URL:** `https://marketplace-api.sshopencloud.eu`
 
---
 
## Core Workflow
 
### Step 1: Understand the intent
 
Determine what the user is looking for:
- **Free-text search** (most common): user describes a topic, task, or need
- **Category-filtered search**: user wants a specific type (tool, dataset, training material, etc.)
- **Item detail**: user has a persistent ID or wants deep info on a known item
- **Actor/concept search**: user wants to find people, organisations, or controlled vocabulary terms
 
### Step 2: Choose and execute the right API call
 
Use `curl` or Python `urllib` for all requests. The API is public and requires no authentication for read operations.
 
```bash
# Basic search
curl -s "https://marketplace-api.sshopencloud.eu/api/item-search?q=QUERY&perpage=10" \
  -H "Accept: application/json"
 
# Category-filtered search
# categories: tool-or-service | training-material | publication | dataset | workflow
curl -s "https://marketplace-api.sshopencloud.eu/api/item-search?q=QUERY&categories=tool-or-service&perpage=10" \
  -H "Accept: application/json"
 
# Get full item detail (replace {type} and {persistentId})
# type: tools-services | training-materials | publications | datasets | workflows
curl -s "https://marketplace-api.sshopencloud.eu/api/tools-services/PERSISTENT_ID" \
  -H "Accept: application/json"
 
# Autocomplete (for suggestions)
curl -s "https://marketplace-api.sshopencloud.eu/api/item-search/autocomplete?q=PARTIAL_QUERY" \
  -H "Accept: application/json"
 
# Actor search (people/organisations)
curl -s "https://marketplace-api.sshopencloud.eu/api/actor-search?q=QUERY&perpage=10" \
  -H "Accept: application/json"
```
 
**Python alternative** (if curl not available):
```python
import urllib.request, json
 
url = "https://marketplace-api.sshopencloud.eu/api/item-search?q=text+analysis&perpage=5"
req = urllib.request.Request(url, headers={"Accept": "application/json"})
with urllib.request.urlopen(req) as r:
    data = json.loads(r.read())
```
 
### Step 3: Parse and present results
 
**Search response structure (`PaginatedSearchItems`):**
```
{
  "hits": <total matching>,
  "count": <items in this page>,
  "page": 1,
  "perpage": 10,
  "pages": <total pages>,
  "items": [ SearchItem, ... ],
  "categories": { "tool-or-service": { "label": ..., "count": ... }, ... }
}
```
 
**Each `SearchItem` has:**
- `persistentId` — stable identifier (e.g. `n4Dv8y`)
- `category` — `tool-or-service`, `training-material`, `publication`, `dataset`, `workflow`, `step`
- `label` — name of the resource
- `description` — short description
- `accessibleAt` — list of URLs where the resource is available
- `status` — `approved`, `suggested`, `ingested`, etc. (prefer `approved`)
- `lastInfoUpdate` — ISO timestamp
 
**Full item detail adds:**
- `properties` — array of typed properties (activity, language, terms of use, etc.)
- `relatedItems` — linked resources (see Step 4 below)
- `contributors` — actors who contributed
- `media` — thumbnails, screenshots
 
### Step 4: Retrieve and present related items
 
When you fetch an item's detail, **always process `relatedItems`** — they provide essential context about how a resource fits into the broader ecosystem.
 
Each related item already contains enough information for a rich summary:
```
{
  "persistentId": "s89flK",
  "category": "step",
  "label": "Using an existing NLP pipeline",
  "description": "In practice, the above steps are combined to pipelines...",
  "relation": { "code": "is-mentioned-in", "label": "Is mentioned in" },
  "accessibleAt": ["https://..."]
}
```
 
**Relation codes you'll encounter:**
- `is-mentioned-in` — this item is cited/used in the related resource
- `documents` — this item is documented by the related resource
- `is-documented-by` — the related resource documents this item
- `is-part-of` / `has-part` — hierarchical membership (e.g. step within a workflow)
- `suggests` / `is-suggested-by` — complementary resources
 
**How to present related items:**
 
Group by `category` and note the relation. Example output format:
```
Related resources (4):
 
  Workflows using this tool:
    • Using an existing NLP pipeline (step) — "In practice, above steps are combined to pipelines…"
 
  Publications citing this tool:
    • Beyond Shot Lengths (publication) — quantitative movie analysis using language data
      → https://doi.org/...
 
  Training materials:
    • Cleaning OCR'd text with Regular Expressions (training-material)
      → https://programminghistorian.org/…
```
 
**When to go deeper on a related item:**
 
If a related item looks highly relevant (e.g. it's a workflow that uses the tool, or a training material that teaches it), **fetch its full detail** using the category → endpoint mapping below. This gives you its own properties, URLs, and a second level of related items — enough to give the user a useful onward path without fetching the whole graph.
 
```bash
# e.g. a related step turned out to be interesting — fetch it
curl -s "https://marketplace-api.sshopencloud.eu/api/workflows/s89flK" \
  -H "Accept: application/json"
```
 
Limit deep fetches to **2–3 of the most relevant** related items to avoid flooding the context. Use the relation code and category to prioritise: workflows and training materials are usually the most actionable for a user trying to get things done.
 
### Step 5: Synthesise for the user
 
- Lead with the most relevant search results (highest score / best match)
- Show: name, category, short description, URL(s) if available
- For each item fetched in detail, follow up with its related items (grouped and annotated)
- Group search results by category when multiple types are returned
- Note total hits so user knows how many more exist
- Offer to fetch detail on a specific item, explore a related item further, or refine the search
 
---
 
## Category → Endpoint Mapping
 
When fetching full detail, use the correct endpoint:
 
| Category value     | Detail endpoint             |
|--------------------|----------------------------|
| `tool-or-service`  | `/api/tools-services/{id}` |
| `training-material`| `/api/training-materials/{id}` |
| `publication`      | `/api/publications/{id}`   |
| `dataset`          | `/api/datasets/{id}`       |
| `workflow`         | `/api/workflows/{id}`      |
 
---
 
## Search Parameter Reference
 
| Parameter     | Values / Notes                                          |
|---------------|---------------------------------------------------------|
| `q`           | Free text query                                         |
| `categories`  | Comma-separated or repeated: `tool-or-service`, `training-material`, `publication`, `dataset`, `workflow` |
| `order`       | `score` (default), `label`, `modified-on`               |
| `page`        | Integer, default 1                                      |
| `perpage`     | Integer, default 10, max ~50                            |
| `advanced`    | Boolean — enables field-specific syntax in `q`          |
 
---
 
## Common Patterns
 
**"Find a tool for sentiment analysis"**
```bash
curl -s "https://marketplace-api.sshopencloud.eu/api/item-search?q=sentiment+analysis&categories=tool-or-service&perpage=5" -H "Accept: application/json"
```
 
**"Show me training materials on digital humanities"**
```bash
curl -s "https://marketplace-api.sshopencloud.eu/api/item-search?q=digital+humanities&categories=training-material&perpage=5" -H "Accept: application/json"
```
 
**"Find resources related to TEI/XML"**
```bash
curl -s "https://marketplace-api.sshopencloud.eu/api/item-search?q=TEI+XML&perpage=10" -H "Accept: application/json"
```
 
**"Get details for item n4Dv8y"** — first determine category from a search, then fetch:
```bash
curl -s "https://marketplace-api.sshopencloud.eu/api/tools-services/n4Dv8y" -H "Accept: application/json"
```
 
**"What tools exist for corpus analysis?"**
```bash
curl -s "https://marketplace-api.sshopencloud.eu/api/item-search?q=corpus+analysis&categories=tool-or-service&perpage=10" -H "Accept: application/json"
```
 
---
 
## Error Handling
 
- **403 on detail endpoint via Python urllib**: Use `curl` instead — some endpoints block Python's default User-Agent
- **Empty results**: Try broader query, remove category filter, or check spelling
- **Status filtering**: Results include all statuses; prefer `approved` ones but show others with a note
- **Pagination**: If `hits` >> `count`, offer to fetch next page with `&page=2`
 
---
 
## Marketplace URL for Users
 
Item page on the frontend: `https://marketplace.sshopencloud.eu/[category]/[persistentId]`
 
Category path mapping:
- `tool-or-service` → `tool-or-service`
- `training-material` → `training-material`  
- `publication` → `publication`
- `dataset` → `dataset`
- `workflow` → `workflow`
 
Example: `https://marketplace.sshopencloud.eu/tool-or-service/n4Dv8y`
 
