# FNRIC Resources for the Reboot the Earth Hackathon

Resources for the **"Counting the Cold: Help California Growers Track Chill and Growing Degree Days"** challenge at the UC Davis Reboot the Earth hackathon (Oct 23–24, 2026). They come from the [Fruit & Nut Research & Information Center (FNRIC)](https://fruitsandnuts.ucdavis.edu/).

## What's here

| File | What it is | Use it to |
| --- | --- | --- |
| [`FNRIC-Weather-Data-Cheat-Sheet.md`](FNRIC-Weather-Data-Cheat-Sheet.md) | Chill and GDD basics, the FNRIC API reference, common gotchas, a forecasting method, and starter code in Python and JavaScript | Get oriented and start pulling data. **Read this first.** |
| [`fnric-api.openapi.yaml`](fnric-api.openapi.yaml) | OpenAPI 3 spec of the FNRIC weather models API | Import into Postman, Insomnia, Bruno, or Hoppscotch, or generate a client |
| [`fnric-api.postman_collection.json`](fnric-api.postman_collection.json) | Postman v2.1 collection with ready-to-run requests and variables (`station_id`, `chill_year`, `bloom_date`, `lat`, `lon`), plus Open-Meteo and NWS forecast requests | Try every endpoint in a couple of clicks. Insomnia and Bruno import it too. |

## Quick start

1. Read the cheat sheet's first two sections (about 5 minutes).
2. Import either the YAML or the JSON file into your API client (in Postman: **Import**, then drop the file), and run **Stations** and **Chill models download**.
3. Copy the starter code and check your season-to-date numbers against the [FNRIC chill calculator](https://fruitsandnuts.ucdavis.edu/chill-calculator).

The API is public and needs no key, but it runs on a limited budget. Please cache responses and don't loop on it.

## Questions

Ask the FNRIC mentors at the event.
