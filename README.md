# League of Legends Match History Pipeline

An end-to-end data pipeline that collects, enriches, and stores League of Legends match data using the Riot Games API and a local PostgreSQL database.

## What it does

1. **Collection** — Fetches match data for a target player via one of two lookup modes:
   - **Ladder-based**: pulls the top N players from the NA SoloQ ladder (Challenger, Grandmaster, or Master) and selects a player by rank
   - **Direct lookup**: targets a specific player by Riot ID and tag (e.g. `TaricMidGG#NA1`)
   
   Match history can be fetched in two ways:
   - **Date range**: pulls all matches between a start and end date (e.g. a full season)
   - **N most recent**: pulls the most recent N matches
2. **Enrichment** — Maps raw numeric IDs for champions, items, runes, and rune shards to human-readable names using the [CommunityDragon](https://communitydragon.org/) open data source
3. **Storage** — Upserts the resulting DataFrame into a local PostgreSQL database using `pangres`, with a composite `match_id + puuid` primary key to prevent duplicates

## Schema

Table: `soloq.regional_player_matches`

Each row represents a single player's performance in a single match. Key fields include champion, KDA, gold, damage stats, items, runes, game mode, patch, and win/loss.

## Usage

**Step 1: Identify the target player**
```python
# Option A: Ladder-based lookup (pull from top N ranked players)
ladder = get_ladder(top=10)
puuid = ladder['puuid'].iloc[0]

# Option B: Direct lookup by Riot ID
puuid = get_puuid(gameName="TaricMidGG", tagLine="NA1", region="americas")
```

**Step 2: Fetch match history**
```python
# Option A: Pull all matches in a date range
start_date = int(time.mktime(time.strptime("2025-01-01", "%Y-%m-%d")))
end_date   = int(time.mktime(time.strptime("2026-01-01", "%Y-%m-%d")))
match_ids = get_match_history(region=REGION, puuid=puuid, start_time=start_date, end_time=end_date)

# Option B: Pull N most recent matches
match_ids = get_match_history(region=REGION, puuid=puuid, count=50)
```

Note: development API keys are rate-limited. The pipeline includes automatic retry logic and a 2-second delay between requests. Fetching a full season (~400+ matches) takes roughly 15 minutes.

## Setup

**Requirements**

- Python 3.9+
- A local PostgreSQL instance
- A [Riot Games API key](https://developer.riotgames.com/) (development keys are rate-limited but sufficient)

**Install dependencies**

```bash
pip install -r requirements.txt
```

**Configure environment variables**

Create a `.env` file in the project root with the following:

```
riot_api_key=your_riot_api_key
db_username=your_db_username
db_password=your_db_password
db_host=localhost
db_port=5432
db_name=your_db_name
```

**Run the notebook**

Open `soloq.ipynb` in Jupyter and run cells top to bottom. The notebook will create the `soloq` schema and `regional_player_matches` table automatically on first run.

## Tech stack

- **Python** — `requests`, `pandas`, `sqlalchemy`, `pangres`, `python-dotenv`
- **PostgreSQL** — local instance via `psycopg2`
- **Data source** — [Riot Games API](https://developer.riotgames.com/), [CommunityDragon](https://communitydragon.org/)
