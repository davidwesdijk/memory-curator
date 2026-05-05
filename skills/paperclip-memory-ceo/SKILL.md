---
name: paperclip-memory-ceo
description: >
  CEO-skill voor Paperclip AI om geheugen (memory) van onderliggende agents te
  optimaliseren, te reorganiseren en te ontdubbelen met minimale tokenkosten en
  maximale bruikbaarheid.
version: 1.0.0
owner: Paperclip AI CEO
language: nl
---

# Paperclip Memory CEO Skill

## Doel

Deze skill helpt de **CEO-agent** van Paperclip AI om alle memory-lagen van het
agent-netwerk structureel te verbeteren:

1. **Optimaliseren**: alleen relevante kennis behouden en compacter formuleren.
2. **Reorganiseren**: geheugen onderbrengen in consistente categorieën en niveaus.
3. **Ontdubbelen**: duplicaten samenvoegen zonder betekenisverlies.
4. **Tokenkosten verlagen**: context slim inkorten, normaliseren en prioriteren.

Resultaat: betere kwaliteit van beslissingen met minder contexttokens per run.

---

## Scope & architectuur

Deze skill gaat uit van drie memory-niveaus:

- **Global/Org Memory (CEO-niveau)**
  - Strategische doelen, bedrijfsregels, kwaliteitscriteria, SLA’s.
- **Team/Domain Memory (afdelingsniveau)**
  - Domeinkennis (sales, support, engineering), playbooks, policies.
- **Agent/Task Memory (operationeel niveau)**
  - Task-specifieke notities, korte-termijn inzichten, uitvoeringstraces.

### Canonieke memory-objectstructuur

Gebruik per memory-item dit schema:

```yaml
id: mem_<uuid>
layer: global|team|agent
domain: string
topic: string
summary: string            # 1-3 zinnen, compact
facts:
  - string                 # atomische feiten
rules:
  - string                 # expliciete regels/besliskaders
examples:
  - input: string
    output: string
source:
  type: user|system|tool|derived
  ref: string
confidence: 0.0-1.0
impact: 1-5                # waarde voor toekomstige taken
volatility: low|medium|high
last_verified_at: ISO-8601
created_at: ISO-8601
updated_at: ISO-8601
expiry_at: ISO-8601|null
tags: [string]
checksum: string           # hash op genormaliseerde inhoud
```

---

## CEO-werkingsmodel

De CEO voert cyclisch een **Memory Governance Loop** uit.

## Fase 1 — Inname & normalisatie

**Doel:** ruwe memory-items uniform maken.

Checklist:

- Zet tekst om naar vaste stijl (kort, actief, zonder ruis).
- Splits samengestelde items in atomische claims.
- Label met `layer`, `domain`, `topic`, `volatility`, `impact`.
- Verwijder triviale conversational fluff.
- Voeg `checksum` toe op genormaliseerde inhoud.

Normalisatieregels:

- Gebruik één taal per item (bij voorkeur dezelfde als primaire workspace-taal).
- Vermijd synoniemen-sprawl; kies één canonieke term per concept.
- Schrijf getallen en datums expliciet (bijv. `2026-05-05`).
- Verplaats meningen naar `confidence` + bron, feiten naar `facts`.

---

## Fase 2 — Detectie van duplicaten

**Doel:** exacte en semantische duplicaten vinden.

Pas 3 detectielagen toe:

1. **Exact hash match**
   -zelfde `checksum` ⇒ directe duplicate.
2. **Near-duplicate lexical**
   - Hoge overlap op keyphrases + entities.
3. **Semantic similarity**
   - Zelfde intentie/betekenis met andere formulering.

Beslisboom bij matches:

- Zelfde claim + nieuwere verificatie → update timestamp, behoud één item.
- Zelfde claim + conflict in details → markeer als conflictcluster.
- Subset/superset-relatie → merge naar compacter item met meerdere `facts`.

---

## Fase 3 — Merge, compressie & herstructurering

**Doel:** maximale informatiedichtheid met minimale tokens.

### Merge-principes

- Bewaar **canonieke versie** met hoogste `confidence` en beste bron.
- Voeg unieke feiten samen, verwijder redundante formuleringen.
- Behoud provenance (`source.ref`) van samengevoegde items.

### Compressie-principes

- Gebruik “statement packing”:
  - In plaats van 5 losse zinnen → 1 compacte samenvatting + lijst `facts`.
- Verwijder stopzinnen (“belangrijk om te onthouden dat...”).
- Vervang lange narratieven door beslisregels.
- Houd voorbeelden alleen als ze gedrag daadwerkelijk sturen.

### Herstructurering

- Promoveer stabiele, high-impact patronen naar `global`.
- Houd veranderlijke, low-impact details op `agent` niveau.
- Split grote items > 250 tokens in thematische subitems.

---

## Fase 4 — Retentie & lifecycle

**Doel:** alleen waardevolle memory actief houden.

Retentiebeleid per item:

- **Keep-Hot**: high impact + recent gebruikt.
- **Warm**: medium impact of seizoensgebonden.
- **Cold Archive**: lage impact, alleen historisch relevant.
- **Expire**: verouderd, niet gevalideerd, lage confidence.

Aanbevolen thresholds:

- Archiveer als `impact <= 2` en > 45 dagen niet geraadpleegd.
- Expire als `volatility=high` en `last_verified_at` > 14 dagen oud.
- Verplicht revalidatie voor alle `global` items elke 30 dagen.

---

## Fase 5 — Context assemblage met tokenbudget

**Doel:** per run alleen de beste memory inladen.

### Budgetstrategie

Definieer per run:

- `B_total` = maximaal contextbudget
- `B_system`, `B_task`, `B_memory`, `B_tools`

Vuistregel:

- `B_memory` = 20%–35% van `B_total`.

### Selectieformule (ranking)

Score per item:

`score = (0.35 * relevance) + (0.25 * impact) + (0.20 * confidence) + (0.10 * recency) + (0.10 * layer_weight)`

Waar:

- `layer_weight`: global=1.0, team=0.8, agent=0.6
- Penalty bij hoge `volatility` zonder recente verificatie.

Selecteer items op score/tokens-ratio totdat `B_memory` is gevuld.

---

## Conflictresolutiebeleid

Wanneer twee memory-items botsen:

1. Geef voorrang aan recenter geverifieerde bron.
2. Bij gelijkheid: hogere bronkwaliteit (`system > user > derived`).
3. Bij blijvend conflict: bewaar beide in conflictcluster met expliciete flags.
4. Trigger “human review” als conflict op global policy zit.

Conflictcluster-structuur:

```yaml
conflict_id: c_<uuid>
topic: string
items: [mem_id_1, mem_id_2]
reason: stale|source_mismatch|numeric_disagreement|policy_conflict
recommended_action: keep_a|keep_b|merge|human_review
```

---

## KPI’s voor de CEO

Meet wekelijks:

- **Tokenreductie (%)** op memory-injectie per agent-run.
- **Duplicate ratio** voor/na deduplicatie.
- **Memory precision@k** (hoeveel ingeladen items waren echt nuttig).
- **Conflict rate** en gemiddelde oplostijd.
- **Stale memory ratio** (verouderde items in actieve set).

Streefwaarden (startpunt):

- 25–50% minder memory-tokens binnen 2 weken.
- < 5% exacte duplicaten in hot/warm sets.
- > 80% precision@k op top-10 ingeladen items.

---

## Operationele runbook (CEO)

### Dagelijks (light pass, 10–20 min)

1. Ingest nieuwe memory van alle agents.
2. Run exact + near-duplicate check.
3. Merge low-risk duplicaten automatisch.
4. Markeer conflicts voor review.

### Wekelijks (deep pass, 45–90 min)

1. Herbereken impact en recency-scores.
2. Archiveer/expire op basis van retentiepolicy.
3. Reorganiseer taxonomy (domain/topic consistency).
4. Evalueer KPI’s en pas thresholds aan.

### Maandelijks (governance)

1. Audit global memory op juistheid.
2. Herziening van policies en conflictregels.
3. Benchmark tokengebruik vs. outputkwaliteit.

---

## Prompt-template voor CEO-agent

Gebruik dit als system of meta-instructie:

```text
Je bent de CEO Memory Curator van Paperclip AI.
Je taak is memory optimaliseren, herstructureren en ontdubbelen met minimaal tokengebruik.
Werk in 5 fases: normaliseren, duplicaatdetectie, merge/compressie, lifecycle, contextselectie.
Behoud alleen informatie met hoge toekomstige waarde.
Bij conflicten: expliciet clusteren, bronkwaliteit wegen, en escaleren bij policy-conflicten.
Lever output als:
1) bijgewerkte canonieke memory-items,
2) lijst van merges/deletes,
3) conflictclusters,
4) voorgestelde contextset binnen budget.
```

---

## Output-contract (verplicht)

Elke CEO-run produceert:

```yaml
run_id: r_<uuid>
date: ISO-8601
stats:
  input_items: int
  duplicates_found: int
  merged_items: int
  deleted_items: int
  archived_items: int
  expired_items: int
  token_before: int
  token_after: int
  reduction_pct: float
artifacts:
  canonical_memory_delta: path_or_id
  conflict_clusters: path_or_id
  context_bundle: path_or_id
notes:
  - string
```

---

## Implementatietips voor onderliggende agents

- Laat iedere agent writes doen in gestandaardiseerd schema (geen vrije tekstdump).
- Voeg per write minimaal toe: `topic`, `summary`, `confidence`, `source`, `updated_at`.
- Forceer max lengte per `summary` (bijv. 60–90 tokens).
- Voeg “why this matters” toe als `impact`-input.

---

## Anti-patterns (vermijden)

- Alles in global memory stoppen.
- Historische chatlogs 1-op-1 bewaren als memory.
- Duplicaten alleen op string-match detecteren.
- Geen vervaldata gebruiken voor volatiele feiten.
- Lange context injecteren zonder ranking en budget.

---

## Snelle start (30 minuten)

1. Definieer schema + checksum in je memory store.
2. Bouw dedupe pipeline: exact → near → semantisch.
3. Zet retentie-thresholds en cron cadans.
4. Introduceer ranking op score/tokens-ratio.
5. Monitor KPI’s en tune wekelijks.

Met deze skill kan de CEO-agent van Paperclip AI memorykwaliteit verhogen én
structureel tokens besparen zonder verlies van operationele intelligentie.
