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

| Parameter     | Type              | Required | Description                                                      |
|---------------|-------------------|----------|------------------------------------------------------------------|
| tournamentId  | string            | Yes      | The tournament document ID                                       |
| phase         | `swiss\|playoff`  | Yes      | The phase to look up. Use `playoff` for bracket matches          |
| roundNumber   | number            | No       | The round number. Defaults to the tournament's active round      |
| tableNumber   | number            | Yes      | The table number                                                 |

**Response**

```json
{
  "roundNumber": 3,
  "phase": "swiss",
  "players": [
    {
      "playerDocId": "string",
      "name": {
        "first": "string",
        "last": "string",
        "nick": "string",
        "display": "string"
      },
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
      },
      "seed": 1
    }
  ],
  "playerMap": {
    "1": { "playerDocId": "string", "name": { "first": "string", "last": "string", "nick": "string", "display": "string" }, "seed": 1, "..." },
    "2": { "playerDocId": "string", "name": { "..." }, "seed": 4, "..." }
  }
}
```

**Notes**
- `phase` must be provided by the caller. Use `swiss` for regular rounds and `playoff` for bracket matches.
- `roundNumber` is the resolved round — either the one passed in or the event's active round.
- `players` always contains at most 2 entries. Bye matches will have 1.
- `playerMap` keys are `1` and `2`, mirroring the player order in the `players` array.
- For `swiss` phase: `record` and `points` reflect rounds **prior** to the requested round; `seed` is `null`.
- For `playoff` phase: `record` and `points` reflect the **full swiss phase** (all non-bracket matches); `seed` is the player's swiss seed (1 = top seed).
- `points` is computed as 3 per match win + 1 per draw.
- `name.display` is `"first last"` from the player document.
- `clubs`, `deckName`, `deckId`, `deckPhotoUrl`, and `deckUrls` are empty/null if not applicable.

---

### Pairings

Returns all pairings for a given round in a tournament.

```
GET /pairings
```

**Query Parameters**

| Parameter     | Type              | Required | Description                                                      |
|---------------|-------------------|----------|------------------------------------------------------------------|
| tournamentId  | string            | Yes      | The tournament document ID                                       |
| roundNumber   | number            | Yes      | The round number                                                 |
| phase         | `swiss\|playoff`  | Yes      | The phase to look up. Use `playoff` for bracket matches          |

**Response**

```json
{
  "phase": "swiss",
  "pairings": [
    {
      "table": 1,
      "players": [
        {
          "name": {
            "first": "string",
            "last": "string",
            "nick": "string",
            "display": "string"
          },
          "seed": 1,
          "record": "2-0-0",
          "points": 6,
          "deckName": "string",
          "country": "Finland | null",
          "clubs": [
            {
              "name": "MTG X-Files",
              "logoUrl": "https://... | null"
            }
          ]
        },
        {
          "name": {
            "first": "string",
            "last": "string",
            "nick": "string",
            "display": "string"
          },
          "seed": 4,
          "record": "1-1-0",
          "points": 3,
          "deckName": "string",
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
- For `swiss` phase: only swiss/stage matches are returned; `record` and `points` reflect rounds **prior** to the requested round; `seed` is always `null`.
- For `playoff` phase: bracket matches are returned; `record` and `points` reflect the **full swiss phase** (all non-bracket matches); `seed` is the player's swiss seed (1 = top seed).
- Table numbers for playoff matches are pre-assigned at bracket creation time, sorted by lowest seed (seed 1 = table 1).
- `points` is computed as 3 per match win + 1 per draw.
- `name.display` is `"first last"` from the player document.
- Bye matches will have only 1 player in the `players` array.
- Matches without a valid table number are excluded.
- `deckName` is empty string if no deck has been submitted.

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
      "name": {
        "first": "string",
        "last": "string",
        "nick": "string",
        "display": "string"
      },
      "record": "3-0-0",
      "matchPoints": 9,
      "matchWinPercentage": 1.0,
      "opponentMatchWinPercentage": 0.72,
      "gameWinPercentage": 0.88,
      "opponentGameWinPercentage": 0.65,
      "dropped": false,
      "deckName": "string",
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
- `name.display` is `"first last"` from the player document.
- `deckName` is empty string if no deck has been submitted.

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
