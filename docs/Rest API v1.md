# Tolaria REST API

Base URL: `https://tolaria.app/api/v1`

---

## Authentication

All endpoints require a Bearer token in the `Authorization` header.

```
Authorization: Bearer <your-api-token>
```

To obtain a token, contact the platform administrator.

---

## Endpoints

### Match Info

Returns player information for a specific match identified by tournament, round, and table.

```
GET /match-info
```

**Query Parameters**

| Parameter     | Type   | Required | Description                                                      |
|---------------|--------|----------|------------------------------------------------------------------|
| tournamentId  | string | Yes      | The tournament document ID                                       |
| roundNumber   | number | No       | The round number. Defaults to the tournament's active round      |
| tableNumber   | number | Yes      | The table number                                                 |

**Response**

```json
{
  "roundNumber": 3,
  "phase": "swiss | bracket",
  "players": [
    {
      "playerDocId": "string",
      "name": "string",
      "record": "2-1-0",
      "points": 6,
      "country": "Sweden | null",
      "clubs": [
        {
          "name": "MTG X-Files",
          "logoUrl": "https://... | null"
        }
      ],
      "deckName": "string",
      "deckId": "string",
      "deckPhotoUrl": "string",
      "deckUrls": {
        "small": "https://...",
        "medium": "https://...",
        "large": "https://...",
        "origin": "https://..."
      }
    }
  ],
  "playerMap": {
    "1": { "playerDocId": "string", "name": "string", "..." },
    "2": { "playerDocId": "string", "name": "string", "..." }
  }
}
```

**Notes**
- `roundNumber` is the resolved round — either the one passed in or the event's active round.
- `phase` is `swiss` or `bracket`, derived from the tournament's current status.
- `players` always contains at most 2 entries. Bye matches will have 1.
- `playerMap` keys are `1` and `2`, mirroring the player order in the `players` array.
- `record` and `points` reflect results from rounds **prior** to the requested round.
- `points` is computed as 3 per match win + 1 per draw.
- `clubs`, `deckName`, `deckId`, `deckPhotoUrl`, and `deckUrls` are empty/null if not applicable.

---

### Pairings

Returns all pairings for a given round in a tournament.

```
GET /pairings
```

**Query Parameters**

| Parameter     | Type   | Required | Description                        |
|---------------|--------|----------|------------------------------------|
| tournamentId  | string | Yes      | The tournament document ID         |
| roundNumber   | number | Yes      | The round number                   |

**Response**

```json
{
  "pairings": [
    {
      "table": 1,
      "players": [
        {
          "name": "string",
          "record": "2-0-0",
          "country": "Finland | null",
          "clubs": [
            {
              "name": "MTG X-Files",
              "logoUrl": "https://... | null"
            }
          ]
        },
        {
          "name": "string",
          "record": "1-1-0",
          "country": "Sweden | null",
          "clubs": null
        }
      ]
    }
  ]
}
```

**Notes**
- Pairings are sorted by table number in ascending order.
- Only Swiss/stage matches are returned. Bracket/playoff matches are excluded.
- `record` reflects results from all rounds **prior** to the requested round.
- Bye matches will have only 1 player in the `players` array.
- Matches without a valid table number are excluded.

---

### Standings

Returns the current standings for a tournament with tiebreakers computed from match results.

```
GET /standings
```

**Query Parameters**

| Parameter     | Type   | Required | Description                        |
|---------------|--------|----------|------------------------------------|
| tournamentId  | string | Yes      | The tournament document ID         |

**Response**

```json
{
  "standings": [
    {
      "rank": 1,
      "seed": 1,
      "playerDocId": "string",
      "name": "string",
      "record": "3-0-0",
      "matchPoints": 9,
      "matchWinPercentage": 1.0,
      "opponentMatchWinPercentage": 0.72,
      "gameWinPercentage": 0.88,
      "opponentGameWinPercentage": 0.65,
      "dropped": false,
      "country": "Sweden | null",
      "clubs": [
        {
          "name": "MTG X-Files",
          "logoUrl": "https://... | null"
        }
      ]
    }
  ]
}
```

**Tiebreaker order**

1. Playoff results (bracket wins/depth, if applicable)
2. Match Points
3. Opponent Match Win Percentage (OMWP)
4. Game Win Percentage (GWP)
5. Opponent Game Win Percentage (OGWP)

**Notes**
- Standings are not available for bracket-only events (returns `400`).
- All percentages use a minimum floor of `0.33` per DCI/MTR rules.
- Records and tiebreakers are computed on the fly from match documents, not stored values.

---

### Player Metadata

Returns the full player profile document for a given player.

```
GET /player-metadata/<playerDocId>
```

**Path Parameters**

| Parameter    | Type   | Required | Description              |
|--------------|--------|----------|--------------------------|
| playerDocId  | string | Yes      | The player document ID   |

**Response**

Returns the full `IPlayerDetails` document as stored in Firestore. Key fields include:

```json
{
  "docId": "string",
  "name": {
    "first": "string",
    "last": "string",
    "nick": "string",
    "display": "string"
  },
  "country": {
    "name": "string",
    "code": "string",
    "region": "string"
  },
  "email": "string",
  "avatar": "https://...",
  "clubDocIds": ["string"]
}
```

---

## Error Responses

All endpoints return JSON error responses in the following format:

```json
{ "error": "Description of the error" }
```

| Status | Meaning                                                   |
|--------|-----------------------------------------------------------|
| 400    | Bad request — missing or invalid query parameters         |
| 401    | Unauthorized — missing or invalid Bearer token            |
| 404    | Not found — tournament, match, or player does not exist   |
| 405    | Method not allowed — only GET requests are accepted       |
