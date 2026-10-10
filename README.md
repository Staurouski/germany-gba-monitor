# Germany G-BA Drug Benefit Rating Lookup

Look up a drug substance's **G-BA** (Gemeinsamer Bundesausschuss, Germany's statutory
health-technology-assessment body) **early-benefit assessment rating** under **Section
35a SGB V** ("fruehe Nutzenbewertung") before a market-access or pricing decision: added-
benefit rating, evidence category, comparator therapy, indication, and decision date --
straight from G-BA's own official machine-readable data export, not scraped off a web
page. Can also run on a schedule to monitor new and changed ratings.

## What you get

One record per new, changed, or withdrawn decision/patient-group combination. Every
record carries an explicit `matchState`:

- `FIRST_SEEN` -- your own baseline run.
- `NEW` -- a decision/patient-group not seen on a previous run.
- `RATING_CHANGED` -- the added-benefit rating itself changed (extent, evidence level,
  comparator, or the headline rating sentence). The record includes both
  `previousBenefitRating` (before) and `benefitRating` (after).
- `AMENDED` -- some other tracked field changed (validity date, trade name, indication
  text, ICD code) without the rating itself changing.
- `REMOVED` -- a decision that was tracked on a previous run has dropped out of G-BA's
  currently-implementing export (repealed or superseded).

Unchanged decisions are never pushed or charged -- only what actually moved.

Each record includes: decision ID and case number, substance and trade name(s),
indication (patient subgroup), ICD-10 code(s), decision date, benefit-rating extent and
evidence category, the headline rating sentence, per-endpoint (mortality/morbidity/
quality of life/adverse events) advantage-or-disadvantage symbols where published, and
a link back to G-BA's own procedure page.

## No personal data

This Actor reads G-BA's own structured XML export (see "How it works" below), which
carries no contact/person-shaped field at all -- every one of its ~99 distinct data
fields is clinical or administrative (substance, ICD code, trade name, dates, rating
category). A full sweep of the live export for title+name patterns found none. As a
second-layer guard, every free-text field this Actor does output (indication text,
rating text, per-endpoint descriptions) is still checked for a German title
(Herr/Frau/Dr./Prof./PD) followed by a capitalised name, and withheld entirely (never
partially redacted) if it ever matches -- only a count of withheld fields is reported,
never the text itself. The one field in the source closest to a hearing-minutes risk
(G-BA's own long-form "Tragende Gruende" reasoning summary) is never read by this
Actor at all, by design, regardless of its content.

## How it works

G-BA publishes a machine-readable XML export ("AIS -- maschinenlesbare Fassung der
Beschluesse nach Section 35a SGB V"), updated twice a month, covering every currently-
implementing early-benefit decision with the added-benefit rating as structured data
(not locked inside a PDF). Getting it requires accepting a one-checkbox terms-of-use
form rather than a plain static download link -- this Actor does that automatically on
every run; no account or manual step is needed on your side.

## Why you might need this

Pharma market-access and HEOR teams, pricing/reimbursement consultancies, and medical-
communications agencies currently track these decisions by paying for a commercial
HTA-intelligence subscription or by manually re-checking G-BA's site. This Actor
surfaces the same decisions and rating changes directly from G-BA's own official feed.
The rating itself is not just informational: under AMNOG (Arzneimittelmarktneuordnungs-
gesetz), G-BA's added-benefit category is the direct input to the price negotiation
between the manufacturer and the GKV-Spitzenverband, so a `RATING_CHANGED` record (a
downgrade or upgrade in `extent`) is a signal with a real negotiating-position
consequence, not a line item to notice weeks later on a routine manual check.

**Scope**: Germany (G-BA) only. France (HAS) and the UK (NICE) are separate national
HTA bodies with their own sites and schemas -- not covered by this Actor.

## Sample output record

A `RATING_CHANGED` record, where the headline added-benefit rating moved between runs:

```json
{
  "recordKey": "1334:7",
  "decisionId": "1334",
  "caseNumber": "2026-04-01-D-1287",
  "patientGroupId": "7",
  "substance": "Retifanlimab",
  "tradeNames": ["Zynyz"],
  "indication": "Erwachsene mit metastasiertem Merkelzellkarzinom",
  "decisionDate": "2026-10-01",
  "benefitRating": {
    "extent": "betraechtlich",
    "evidence": "Hinweis",
    "comparator": "Watch and wait / bestmoegliche unterstuetzende Behandlung",
    "text": "Ein Hinweis auf einen betraechtlichen Zusatznutzen liegt vor."
  },
  "previousBenefitRating": {
    "extent": "gering",
    "evidence": "Anhaltspunkt",
    "comparator": "Watch and wait / bestmoegliche unterstuetzende Behandlung",
    "text": "Ein Anhaltspunkt fuer einen geringen Zusatznutzen liegt vor."
  },
  "matchState": "RATING_CHANGED",
  "changedFields": ["benefitRating.extent", "benefitRating.evidence"],
  "matchesWatchlist": true,
  "sourceUrl": "https://www.g-ba.de/beschluesse/1334/",
  "checkedAt": "2026-10-10T06:00:00.000Z"
}
```

## Input

```json
{
  "watchlist": ["Sugemalimab", "C61"],
  "maxResults": 300
}
```

- `watchlist` (optional, default empty): substance names, trade names, or ICD-10 codes.
  Every record gets a `matchesWatchlist` flag (case-insensitive substring match) so you
  can filter to drugs/indications you actually track. Leave empty to track everything.
- `maxResults` (optional, default 300): caps how many changed records are pushed and
  charged in a single run. **Important for your first run**: G-BA's export currently
  covers roughly 1,700 decision/patient-group records; a first-ever run with no prior
  state treats every one of them as a new baseline record, so with the default cap your
  first run reports the newest ~300 and the rest follow automatically on your next
  scheduled run (nothing is lost or double-charged either way). If you want the full
  history backfilled in one run, raise `maxResults` and raise your own run cost cap in
  Console accordingly (see Pricing below).

## Change tracking across runs

Run this Actor on a schedule -- weekly is a reasonable match for G-BA's own twice-
monthly update cadence. Each decision/patient-group's rating and key fields are stored
per Actor run, so every run after the first reports only what actually changed.

## Pricing

Pay per run started, plus per new/changed/removed decision record reported (unchanged
records are never charged). See the Pricing tab for exact rates. A first run with a
high `maxResults` will cost proportionally more (see the Input section above) -- raise
your own maximum-cost-per-run setting if you deliberately want a one-run full backfill.

**Worked example.** A weekly scheduled run with no watchlist (full feed, after the
one-time backfill) typically sees 5-20 real new/changed decisions a week given G-BA's
twice-monthly update cadence, so roughly USD 1.40-2.60/month. A watchlist-scoped run
tracking a handful of named substances pays close to zero most weeks (just the run fee)
and only incurs a per-record charge on an actual rating change for one of your own
tracked drugs -- the scenario this Actor is built for.

---

## Deutschland G-BA Fruehe-Nutzenbewertung-Monitor

Verfolgt den **Gemeinsamen Bundesausschuss (G-BA)** auf neue und geaenderte
**Beschluesse zur fruehen Nutzenbewertung nach Section 35a SGB V**: Ausmass des
Zusatznutzens, Wahrscheinlichkeit/Evidenzkategorie, zweckmaessige Vergleichstherapie,
Anwendungsgebiet und Beschlussdatum -- direkt aus G-BAs eigenem maschinenlesbarem
Datenexport, nicht von einer Webseite gescrapt.

**Keine personenbezogenen Daten**: Der genutzte XML-Export enthaelt keinerlei
Kontakt-/Personenfeld; alle enthaltenen Freitextfelder werden zusaetzlich auf ein
Titel-plus-Name-Muster (Herr/Frau/Dr./Prof./PD + Name) geprueft und bei einem Treffer
vollstaendig verworfen, nie teilweise geschwaerzt. Das Feld mit dem hoechsten
theoretischen Risiko (die "Tragenden Gruende" als Volltext) wird von diesem Actor
grundsaetzlich nicht gelesen.

**Umfang**: ausschliesslich Deutschland (G-BA). Frankreich (HAS) und Grossbritannien
(NICE) sind eigene nationale HTA-Behoerden mit eigenen Datenquellen und werden von
diesem Actor nicht abgedeckt.

**Eingabe**: `watchlist` (Wirkstoff-/Handelsnamen oder ICD-10-Codes, optional) und
`maxResults` (Obergrenze geaenderter Datensaetze pro Lauf, Standard 300 -- beim ersten
Lauf ohne vorherigen Zustand werden alle ca. 1.700 Datensaetze als neu gewertet; mit der
Standardgrenze werden die juengsten 300 gemeldet, der Rest folgt automatisch bei den
naechsten planmaessigen Laeufen).
