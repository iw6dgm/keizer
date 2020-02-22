# Keizer

## Summary

Keizer project contains a SQLite3 database implementation `keizer.db` of the chess Keizer pairing system as described in [Sevilla](https://www.jbfsoftware.com/wordpress/sevilla-keizer/).
You can access this database using `sqlite3` command line executable or a JDBC compatible client with the connection string `jdbc:sqlite:<path>/keizer.db`.

## Data structure

`keizer.db` database already includes the following tables and views with pre-calculated data from the example provided above (see link).

### Tables

- `player`: This table stores players data. No explicit primary key is defined as the built-in `rowid` is used for that purpose.
Category is optional while rating is mandatory and may be used to create the first ranking.

-----------------------------------------
| COLUMN_NAME | TYPE_NAME | IS_NULLABLE |
| ----------- | --------- | ----------- |
| name        | TEXT      | NO          |
| category    | TEXT      | YES         |
| rating      | INT       | NO          |
-----------------------------------------

- `tournament`: This table stores tournaments data. No explicit primary key is defined as the built-in `rowid` is used for that purpose.
Status is currently unused (for future uses).

-----------------------------------------
| COLUMN_NAME | TYPE_NAME | IS_NULLABLE |
| ----------- | --------- | ----------- |
| description | TEXT      | NO          |
| status      | SMALL INT | NO          |
-----------------------------------------

- `x_player_tournament`: Cross-table between `player` and `tournament`, this is where the public player score is stored (1 point for a win, 0.5 for a draw, 0 otherwise).
`player_id` and `tournament_id` are foreign keys to `player` and `tournament` tables respectively. 
Status is currently unused (for future uses).

-------------------------------------------
| COLUMN_NAME   | TYPE_NAME | IS_NULLABLE |
| ------------- | --------- | ----------- |
| player_id     | INT       | NO          |
| tournament_id | INT       | NO          |
| score         | REAL      | NO          |
| status        | SMALL INT | NO          |
-------------------------------------------

Primary key:

---------------------------
| KEY_SEQ | COLUMN_NAME   |
| ------- | ------------- |
| 1       | round_number  |
| 2       | tournament_id |
| 3       | white         |
| 4       | black         |
---------------------------

- `ranking`: This table stores the Keizer scores to calculate the ranking after each round.
`player_id` and `tournament_id` are foreign keys to `player` and `tournament` tables respectively. 

-------------------------------------------
| COLUMN_NAME   | TYPE_NAME | IS_NULLABLE |
| ------------- | --------- | ----------- |
| tournament_id | INT       | NO          |
| player_id     | INT       | NO          |
| round_number  | SMALL INT | NO          |
| score         | SMALL INT | NO          |
-------------------------------------------

Primary key:

---------------------------
| KEY_SEQ | COLUMN_NAME   |
| ------- | ------------- |
| 1       | tournament_id |
| 2       | player_id     |
| 3       | round_number  |
---------------------------


- `round`: This table stores the results of each round.
`white`, `black` are foreign keys to `player` table while `tournament_id` is foreign key to `tournament` table.
Values for `result` column are '1-0', '0-1' and '½-½'. Other values can be used for forfeits or non-standard results but will be ignored (not scored).

-------------------------------------------
| COLUMN_NAME   | TYPE_NAME | IS_NULLABLE |
| ------------- | --------- | ----------- |
| round_number  | SMALL INT | YES         |
| tournament_id | INT       | NO          |
| white         | INT       | NO          |
| black         | INT       | NO          |
| result        | TEXT      | YES         |
| last_updated  | TIMESTAMP | YES         |
-------------------------------------------

### Views

- `results`: This view displays players classical (no Keizer) scores and ranking after each round.
-----------------------------
| COLUMN_NAME   | TYPE_NAME |
| ------------- | --------- |
| tournament_id | INT       |
| player_id     | INTEGER   |
| name          | TEXT      |
| score         | REAL      |
-----------------------------

This is view is created by:

```sql
CREATE VIEW results as
SELECT
tournament_id,player_id,name,sum(score) as score
FROM
(
   SELECT
   r.tournament_id,
   p.rowid as player_id,
   p.name,
   case r.result when '1-0' then 1.0 when '½-½' then 0.5 else 0.0 end as score
   FROM player p,
   round r
   WHERE p.rowid = r.white
   UNION
   SELECT
   r.tournament_id,
   p.rowid as player_id,
   p.name,
   case r.result when '0-1' then 1.0 when '½-½' then 0.5 else 0.0 end as score
   FROM player p,
   round r
   WHERE p.rowid = r.black
)
GROUP BY player_id,tournament_id
;
```

- `scores`: This view displays Kerizer scores calculated for each round. It's used as _helper_ table for the update query and created by:

```sql
CREATE view scores AS
SELECT
r.tournament_id,
rnk1.round_number,
p.rowid as player_id,
p.name,
case r.result when '1-0' then rnk2.score * 1.0 when '½-½' then rnk2.score / 2.0 else 0.0 end as score
FROM player p,ranking rnk1,ranking rnk2,round r
WHERE rnk2.player_id = r.black
AND rnk2.round_number = rnk1.round_number
AND r.round_number <= rnk1.round_number
AND r.white = p.rowid
AND rnk1.player_id = p.rowid
UNION ALL
SELECT
r.tournament_id,
rnk1.round_number,
p.rowid as player_id,
p.name,
case r.result when '0-1' then rnk2.score * 1.0 when '½-½' then rnk2.score / 2.0 else 0.0 end as score
FROM player p,ranking rnk1,ranking rnk2,round r
WHERE rnk2.player_id = r.white
AND rnk2.round_number = rnk1.round_number
AND r.round_number <= rnk1.round_number
AND r.black = p.rowid
AND rnk1.player_id = p.rowid
order by rnk1.round_number
;
```

### Updates

Given a `tournament_id` and the current round this SQL is executed to insert new Keizer scores that will be used for the follow-up round:

```sql
WITH new_ranking
(
   tournament_id,
   round_number,
   player_id,
   name,
   score,
   new_score
)
AS
(
   SELECT
   tournament_id,
   round_number,
   player_id,
   name,
   score,
   51-ROW_NUMBER() OVER (ORDER BY score DESC) as new_score
   FROM
   (
      SELECT
      rnk1.tournament_id,
      rnk1.round_number,
      p.rowid as player_id,
      p.name,
      rnk1.score +
      (
         SELECT
         sum(s.score)
         from scores s
         where s.player_id = p.rowid
         AND s.round_number = rnk1.round_number
         and s.tournament_id = rnk1.tournament_id
      )
      as score
      FROM player p,
      ranking rnk1
      WHERE rnk1.player_id = p.rowid
      AND rnk1.round_number = 2 -- 1st parameter
      AND rnk1.tournament_id = 1 -- 2nd parameter
   )
)
SELECT
tournament_id,player_id,round_number+1,new_score
FROM new_ranking;
```
Note: This query requires SQLite3 to be compatible with the `ROW_NUMBER() OVER` function. First player is assigned `50` points but this value can be changed based on the number of total players (see link for further documentation).

Given a `tournament_id` the following SQL query can be used to update player results:

```sql
UPDATE x_player_tournament SET score = (SELECT score FROM results WHERE x_player_tournament.player_id = results.player_id AND x_player_tournament.tournament_id = results.tournament_id)
WHERE EXISTS (SELECT 1 FROM results WHERE x_player_tournament.player_id = results.player_id AND x_player_tournament.tournament_id = results.tournament_id)
AND x_player_tournament.tournament_id = 1 -- parameter
;
```

## TODO

- [ ] Optimise database / queries
- [ ] User interface
- [ ] Public Web App

If you spot any bug or feel that the SQL (or database structure) can be improved to accomodate better features please let me know.