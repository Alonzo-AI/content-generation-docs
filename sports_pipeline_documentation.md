
## 📘 Stage 1: Data Sources & ETL

### Overview
This stage ingests diverse sports data from multiple sources and transforms them into a standardized format for downstream processing. The data spans multiple time periods and comes in different formats, requiring tailored ETL pipelines for each source.

### Data Sources

| Source       | Format | Time Coverage     | Granularity        | Notes |
|--------------|--------|-------------------|---------------------|-------|
| **Genius**   | JSON   | 2014–present       | Game-level (2014–2020), Quarter-level (2021–present) | Provides football and basketball stats via API |
| **StatCrew** | XML    | ~1998–2013         | Game-level          | Legacy format requiring custom parsers |
| **Record Book** | PDF (manual) | Varies by school | Aggregated (not game-specific) | Maintained by schools, used for historical or individual records |
| **Others**   | Mixed  | TBD                | TBD                 | Placeholder for future or external sources |

### Current ETL Implementation
- **Tooling:** Apache Hop is used for the majority of ETL tasks, enabling workflows for parsing, cleaning, and normalizing data.
- **Transformation Need:** Due to format diversity (JSON, XML, PDF), each source has a customized ETL sub-pipeline before data is funneled into the **Common Data Lake**.

---

Awesome — your answers are crystal clear and allow us to document **Stage 2: Common Data Lake** accurately.

---

## 📘 Stage 2: Common Data Lake

### Overview
The **Common Data Lake (CDL)** serves as the centralized repository for structured sports data after source-specific ETL. It organizes statistics by **entity** (Player or Team) and **statistical period** (Game, Season, Career), forming the foundation for event extraction and analytics.

### Core Design Concepts

- **Entity Types:**
  - **Player:** Individual-level statistics and identifiers.
  - **Team:** Aggregated team-level statistics or team-specific records.

- **Statistical Periods:**
  - **Game:** Stats recorded per match or game.
  - **Season:** Aggregated stats for a single season.
  - **Career:** Accumulated stats over multiple seasons.

### Data Organization

| Table                          | Entity  | Period     | Notes |
|-------------------------------|---------|------------|-------|
| `player_game_statistics`      | Player  | Game       | Includes metadata like game ID, season, and opponent |
| `player_season_statistics`    | Player  | Season     | Includes player’s name, team, year |
| `player_career_statistics`    | Player  | Career     | Aggregation across multiple seasons |
| `team_game_statistics`        | Team    | Game       | Includes team performance stats per game |
| `team_season_statistics`      | Team    | Season     | Season summary of team performance |

- All tables are stored in **PostgreSQL**.
- **Traceability** is ensured through shared fields like `season`, `player_name`, and unique game identifiers. This enables relational linking between game, season, and career records.

### Data Usage
The structured format and relational integrity of the CDL tables allow downstream stages—like event extraction, aggregation, and statistical modeling—to perform consistent and traceable analytics.

Got it — thanks for the detailed direction. Here's the **revised and expanded documentation for Stage 3**, incorporating all your feedback:

---

## 📘 Stage 3: Atomic Event Extraction & Aggregation

### Overview
This stage transforms structured game-level statistics into **fine-grained atomic events** and then computes meaningful aggregations over diverse population types. While the system is designed to eventually support all stat periods (e.g., **career**, **season**, **game**, and even **play-by-play**), the current implementation focuses on **game-level data**.

---

### 🔄 Transformation: From Structured Stats to Atomic Events

Structured tables like `player_game_statistics` (PGS) contain both **metadata** and **statistical fields**, where each row corresponds to a single game.

#### Example PGS Entry:
```json
{
  "playername": "BENSON, TREY",
  "teamname": "Florida St.",
  "opponentteamname": "Virginia Tech",
  "gamedate": "2023-10-07",
  "position": "RB",
  "srushingyardsnet": 200,
  "srushingattempts": 11,
  "sreceptionsyards": 15,
  ...
}
```

From this row, we extract **one atomic event per stat field**, preserving all relevant metadata.

#### Example Atomic Events Derived:
| Player        | Team         | Date       | Opponent        | Stat Type         | Value |
|---------------|--------------|------------|------------------|-------------------|--------|
| BENSON, TREY  | Florida St.  | 2023-10-07 | Virginia Tech    | sRushingYardsNet  | 200    |
| BENSON, TREY  | Florida St.  | 2023-10-07 | Virginia Tech    | sRushingAttempts  | 11     |

Each **atomic event** is thus defined by:
- **Entity**: Player or Team
- **Stat Period**: Game (for now)
- **Stat Type**: e.g., sRushingYardsNet

This structure powers everything downstream — patterns, probabilities, and stories — by reducing complex game logs into **granular, analyzable rows**.

---

### 👥 Population Types

Populations define **logical groupings** of atomic events to compute contextual statistics (e.g., “career best”, “best vs opponent”, etc.).

Each **population type** is a **metadata grouping template**, and each population is an **instance** of that type.

#### ✅ List of Current Population Types (Grouped by Meaning):

##### 🧍 Player-based
- `["playerID", "stat"]`: Career total for player
- `["playerID", "season", "stat"]`: Season total
- `["playerID", "opponentTeamName", "stat"]`: Career vs team
- `["playerID", "opponentTeamName", "season", "stat"]`: Season vs team
- `["playerID", "opponentConferenceName", "stat"]`: Career vs conf
- `["playerID", "opponentConferenceName", "season", "stat"]`: Season vs conf

##### 🧑‍🤝‍🧑 Team-based
- `["teamName", "stat"]`
- `["teamName", "season", "stat"]`
- `["teamName", "opponentTeamName", "stat"]`
- `["teamName", "opponentTeamName", "season", "stat"]`
- `["teamName", "opponentConferenceName", "stat"]`
- `["teamName", "opponentConferenceName", "season", "stat"]`

##### 🏟 Opponent and Conference
- `["opponentTeamName", "stat"]`
- `["opponentTeamName", "season", "stat"]`
- `["opponentConferenceName", "stat"]`
- `["opponentConferenceName", "season", "stat"]`

##### 🎓 Player Class
- `["playerClass", "stat"]`
- `["playerClass", "teamName", "stat"]`
- `["playerClass", "season", "stat"]`
- `["playerClass", "opponentTeamName", "stat"]`
- `["playerClass", "opponentTeamName", "season", "stat"]`
- `["playerClass", "opponentConferenceName", "stat"]`
- `["playerClass", "opponentConferenceName", "season", "stat"]`

##### 🧑 Position
- `["position", "stat"]`
- `["position", "teamName", "stat"]`
- `["position", "season", "stat"]`
- `["position", "opponentTeamName", "stat"]`
- `["position", "opponentTeamName", "season", "stat"]`
- `["position", "opponentConferenceName", "stat"]`
- `["position", "opponentConferenceName", "season", "stat"]`

##### 🕒 Global
- `["season", "stat"]`
- `["stat"]`

Each of these populations serves as the basis for computing statistical significance, detecting patterns, and triggering storytelling logic.

---

### 📊 Aggregations

For each **(population, stat)** pair, we compute:

- `sum`, `count`, `max`, `min`, `mean`, `std` (standard deviation)
- `rank`: Relative position of a stat within a parent population

##### 🏁 Rank Computation
Given population type: `["playerID", "opponentTeamName", "season", "stat"]`
→ Parent: `["opponentTeamName", "season", "stat"]`
→ Rank: Position of player stat within that opponent-season group.

---

### 📐 Quantiles

- For **each population**, we compute **quantiles** (e.g., 10th, 25th, 50th, 75th, 90th, 99th percentiles).
- Quantiles are essential to define **thresholds for pattern detection** (e.g., “top 10% rushing yard performances”).
- Used directly in streaks, NTIMES, and LTW logic.

---

### ⏳ Time Bounds

- For each population, we identify **time-based milestones** (e.g., earliest occurrence, last occurrence).
- By **combining time bounds and quantiles**, we create temporal narratives:

  > _"In how many games since {T} years ago did X achieve more than {Q} rushing yards?"_

This allows the system to blend **performance** and **rarity over time** — a key input into probability and storytelling logic.

Perfect — now we have a strong understanding of how patterns are defined and computed. Here’s the detailed documentation for:

---

## 📘 Stage 4: Pattern Detection

### Overview
This stage identifies **recurring or significant performance patterns** based on atomic events. These patterns help surface stories such as consistency, standout performances, or long-awaited returns — which are then fed into probability modules and storytelling.

---

### 🔍 Types of Patterns

| Pattern Type | Definition | Example |
|--------------|------------|---------|
| **Streak**   | A sequence of **consecutive games** where a stat exceeds a threshold | “5 straight games with 100+ rushing yards” |
| **NTIMES**   | A count of **non-consecutive games** where a stat exceeds a threshold | “8 games this season with 2+ touchdowns” |
| **LTW (Last Time When)** | Identifies the **most recent occurrence** of a stat exceeding a threshold | “Last time this player had 150+ rushing yards was in 2021 vs Clemson” |

- **Thresholds** for these patterns are typically **quantile-based**, dynamically determined from the distribution of stat values in the population.
- Each pattern instance is **parameterized** by `X` (number of games), `Y` (threshold), and the time context (season, opponent, etc.).

---

### ⚙️ Pattern Computation

- Patterns are not ML-detected but **computed using custom SQL functions** that operate on atomic event tables.
- **Streaks** involve checking for consecutive rows meeting a condition.
- **NTIMES** are cumulative counts.
- **LTW** performs temporal reverse lookup for the last matching row.

---

### ⏱️ Time Modeling

- **Temporal context** is introduced using **time buckets** in the aggregation stage.
  - These buckets enable sorting events chronologically and identifying temporal gaps or durations (e.g., "over the last 6 games").

---

### 📌 Notes

- Patterns are not computed for all stat values, but only for **pre-determined thresholds**.
  - Thresholds may be hardcoded, rule-based, or dynamically selected from **quantile outputs**.

---
Perfect — your explanation makes the probabilistic modeling approach much clearer, especially the distinction between **context-independent** vs **context-rich** significance. Here’s the detailed write-up for:

---

## 📘 Stage 5: Probability Calculations

### Overview
This stage assigns **statistical significance** to events, patterns, and aggregates to assess their "interestingness." It provides quantitative scores that help determine how remarkable a stat or pattern is, given the surrounding data distribution and context.

---

### 🎲 Types of Probabilities

| Probability Type | What It Measures | Example Use Case |
|------------------|------------------|------------------|
| **Event Prob**   | Likelihood of observing a stat value based on normal distribution (μ, σ) of a population | Is 200 rushing yards exceptional for this player? |
| **Min-Max Prob** | Probability relative to known min/max bounds in population | Is this stat approaching all-time bests or worsts? |
| **Streak Prob**  | Probability of a streak of length *X* with threshold *Y*, given historical patterns | Is a 5-game streak of 2+ touchdowns rare in NCAA history? |
| **LTW Prob**     | Probability based on **how long ago** an event last occurred | How rare is it that it took 3 seasons for this event to recur? |
| **Rank Prob**    | Rank percentile of an event/stat among a broader population | Is this the 3rd highest rushing performance vs ACC opponents this season? |

---

### 📊 Computation Details

- **Event Probabilities** are computed using a **normal distribution** derived from aggregations:
  - \( \text{logprob}(x) = \log P(X \geq x) \), where \( X \sim \mathcal{N}(\mu, \sigma^2) \)
- **All probabilities** are expressed as **log-probabilities**, offering better interpretability and range control for significance modeling.

---

### 🧠 Contextual Awareness

- Pattern-based probabilities like **Streak Prob** and **LTW Prob** take into account:
  - **Time dimension** (e.g., how recent, how long ago)
  - **Thresholds** crossed
  - **Statistical rarity** of sequence

- Population context is vital. For example:
  - **Streak of 3 games** with 100 yards may be **common** for RBs, but **rare** for QBs.
  - LTW of 2 years ago is more meaningful if the player has been active continuously.

---

### 🧬 Usage

- Probabilities may be used:
  - **Individually**, to surface standout stats or performances
  - **Together**, in a **learned importance model** that ranks events based on overall rarity and impact

- A downstream model is trained on a combination of all these probabilities to learn **how interesting** a stat or pattern is, based on historical context, audience appeal, and recurrence.

Fantastic — we’re at the final stretch now. Here's the detailed documentation for:

---

## 📘 Stage 6: Interestingness & Story Generation

### Overview
The final stage evaluates the **story potential** of each detected event or pattern and uses **generative AI** to transform the most significant ones into **natural language narratives** suitable for publishing (e.g., tweets, summaries).

---

### 🌟 Interestingness Scoring

**Interestingness** represents the *storyworthiness* of a stat or pattern and is computed as a **function of:**

1. **Statistical Rarity**
   → Derived from the various probabilities computed in Stage 5
2. **Social Media Popularity Signals**
   → Metrics like recent social engagement, player mentions, fan interest, trending events

This model quantifies how likely an event or pattern will **capture attention**, balancing **objective statistical impact** and **subjective relevance**.

- Modeled as a **scoring function** rather than classification or ranking
- Customizable to emphasize different audiences (e.g., analysts vs fans)

---

### 🧹 Filtered Events & Patterns

Once interestingness is scored:

- Events and patterns with a **score above a certain threshold** are selected
- This threshold ensures only high-signal content moves forward
- Acts as a final gatekeeper between data and content generation

---

### 📝 Tweet Generation

Selected high-interest events are passed to a **Generative AI system** for narrative creation.

- **Input:** Filtered atomic events, patterns, and supporting metadata
- **Output:** Natural language tweets or micro-stories

#### 💡 Generation Features
- Contextual storytelling: mentions opponent, game context, time span, etc.
- Supports **both individual and comparative insights**:
  - "Trey Benson rushed for 200 yards — the most by an FSU RB since 2019."
  - "FSU has had a 100-yard rusher in 4 straight games."

- May use prompt engineering, templates, or structured inputs for higher coherence
