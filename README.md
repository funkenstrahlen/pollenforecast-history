# pollenforecast-history

A daily archive of the **Pollenflug-Gefahrenindex** (pollen forecast index) published by
[Deutscher Wetterdienst](https://www.dwd.de/) (DWD) for Germany.

DWD publishes only the *current* forecast at a single URL and overwrites it every day —
there is no official history. This repository snapshots that file once a day and keeps it,
so the past is queryable.

**Coverage:** 2019-04-18 to present — 2753 daily snapshots, 3 days missing
(2021-02-10, 2021-02-11, 2021-06-29).

## Accessing the data

Every snapshot is served over HTTPS via GitHub Pages:

```
https://pollenflug.stefantrauth.de/data/YYYYMMDD.json
```

```bash
curl https://pollenflug.stefantrauth.de/data/20261102.json
```

Served as `application/json; charset=utf-8`, so it is usable directly from the browser:

```js
const res  = await fetch('https://pollenflug.stefantrauth.de/data/20261102.json')
const data = await res.json()
```

Two caveats:

- **There is no index and no directory listing.** You have to construct the filename from
  the date. The root URL returns 404 by design — this is a data archive, not a website.
- Responses are CDN-cached for 10 minutes (`cache-control: max-age=600`), so a snapshot
  taken moments ago may take up to that long to appear.
- **The path changed on 2026-11-02.** Snapshots used to sit at the repository root and were
  served as `/YYYYMMDD.json`; they now live in `data/`. The old URLs return 404 — GitHub
  Pages cannot redirect without Jekyll, and Jekyll is disabled here. The move was needed
  because GitHub's web UI truncates a directory listing at 1000 entries, which left
  `README.md` invisible on the repository page as entry 2756 of 2756.

## File format

One file per day in `data/`, named `YYYYMMDD.json` after the day it was fetched. Each file
is a **byte-for-byte copy** of what DWD served at that moment — nothing is normalised,
reformatted or corrected, so the archive stays faithful to the original even where DWD
changed its own structure over the years.

```json
{
  "name": "Pollenflug-Gefahrenindex für Deutschland ausgegeben vom Deutschen Wetterdienst",
  "sender": "Deutscher Wetterdienst - Medizin-Meteorologie",
  "last_update": "2026-11-02 11:00 Uhr",
  "next_update": "2026-11-03 11:00 Uhr",
  "legend": { "id1": "0", "id1_desc": "keine Belastung", "...": "..." },
  "content": [
    {
      "region_id": 10,
      "region_name": "Schleswig-Holstein und Hamburg",
      "partregion_id": 11,
      "partregion_name": "Inseln und Marschen",
      "Pollen": {
        "Birke":    { "today": "0", "tomorrow": "0", "dayafter_to": "0" },
        "Graeser":  { "today": "0", "tomorrow": "0", "dayafter_to": "0" },
        "...":      {}
      }
    }
  ]
}
```

`content` holds 27 entries — the regions and sub-regions DWD divides Germany into. Each
carries a forecast for **8 pollen types**: Ambrosia, Beifuss, Birke, Erle, Esche, Graeser,
Hasel, Roggen. Each type has three horizons: `today`, `tomorrow`, `dayafter_to`.

### Reading the index values

The forecast values are **strings, not numbers**, and some of them are *ranges*:

| Value | Meaning (`legend`) |
|---|---|
| `"0"` | keine Belastung — none |
| `"0-1"` | keine bis geringe Belastung — none to low |
| `"1"` | geringe Belastung — low |
| `"1-2"` | geringe bis mittlere Belastung — low to moderate |
| `"2"` | mittlere Belastung — moderate |
| `"2-3"` | mittlere bis hohe Belastung — moderate to high |
| `"3"` | hohe Belastung — high |

So `parseInt()` / `float()` will silently turn `"2-3"` into `2` and `"0-1"` into `0`. Map
the strings explicitly, or treat each value as the interval it is. The `legend` object in
every file is authoritative for that file.

Note also that a region without sub-regions carries `"partregion_id": -1` and an empty
`"partregion_name"`.

## How the archive is produced

A cron job on a small shared host fetches
[`s31fg.json`](https://opendata.dwd.de/climate_environment/health/alerts/s31fg.json)
from DWD Open Data every day at 12:00 (Europe/Berlin) — DWD updates it at 11:00 — then
commits it and pushes here. The script validates the download before committing, so a DWD
outage leaves a gap rather than an empty or truncated file, and it alerts by mail when any
step fails.

Between 2023-03-24 and 2026-11-02 the push half of that job was broken: GitHub rotated the
SSH host key it had leaked, the host still trusted the old one, and nothing logged or
alerted. Collection never stopped, so no data was lost, but this repository stood still for
three years until the backlog was pushed on 2026-11-02. The logging and alerting exist
because of it.

## Data source and licence

The data is produced and owned by **Deutscher Wetterdienst**, retrieved from
[DWD Open Data](https://opendata.dwd.de/climate_environment/health/alerts/). DWD offers it
free of charge and without registration, under the terms of use its own
[README](https://opendata.dwd.de/README.txt) points to:
[dwd.de legal notice](https://www.dwd.de/EN/service/legal_notice/legal_notice_node.html).
Read those before reusing the data — they are the authoritative source, and they ask for
DWD to be credited.

This repository only archives and republishes those files unmodified. It is not affiliated
with or endorsed by DWD.
