# Driver Reliability Score & Match Blocking

## 1. Overview

The driver reliability system calculates a driver's score based on their active penalties. This score influences driver ranking during route matching and determines whether a driver is eligible to receive new route matches.

The base reliability score is **100 points**.

```python
reliability_score = 100 - active_penalty_points - heavy_penalty
```

| Reliability Score | Status | Behavior |
|---|---|---|
| `>= 0` | Eligible | Driver can participate in route matching. |
| `< 0` | Match-blocked | Driver is excluded from all route matches. |

**Important:** A match-blocked driver is not deactivated. They retain access to the app, but they are excluded from every matching process until their score recovers to `0` or above.

---

## 2. Penalty Types and Points

| Penalty Type | Points | Conditions |
|---|---:|---|
| Quick abandon | 5 | Driver drops an accepted route within 15 minutes of accepting it, regardless of when the ride starts. |
| Abandon | 30 | Driver drops the route more than 15 minutes after accepting it, with 12–24 hours remaining before the ride starts. |
| Last-minute abandon | 100 | Driver drops the route more than 15 minutes after accepting it, with less than 12 hours remaining before the ride starts. |
| No-show | 200 | An admin explicitly flags the driver as a no-show while unassigning the route. This penalty is never generated automatically. |
| Heavy penalty | 100 additional points | Applied when the driver has two or more active last-minute abandon penalties within a 15-day window. |

### Penalty classification rules

1. **Quick abandon takes precedence.** If a driver drops a route within 15 minutes of accepting it, they receive only the 5-point quick-abandon penalty, even if the ride is starting soon.
2. **No penalty for early abandonment.** Dropping a route more than 24 hours before its start time does not incur a penalty.
3. **Abandonment timing is measured relative to the ride start time**, except for the 15-minute acceptance grace window.
4. **No-shows require an explicit admin action.** The system does not infer no-shows automatically.

---

## 3. Penalty Expiration

Each penalty has an expiration date, calculated when the penalty is created.

```python
expire_date = created_at + penalty_expiration_duration
```

The configurable setting `driver-penalty-expiration-minutes` controls the duration.

- **Default duration:** 14 days.
- **Last-minute abandon duration:** At least 15 days, ensuring two last-minute abandons can remain active together throughout the heavy-penalty window when created within that window.
- **Configuration changes:** Updating the expiration setting affects only newly created penalties. Existing penalties retain their original expiration dates.
- **No cleanup job:** Expired penalty records remain in the database as historical records.
- **Active penalty calculation:** A penalty counts only while `EXPIRE_DATE >= now`.
- **Automatic recovery:** When a penalty expires, its points stop counting at the next score calculation. The driver's ranking improves, and match blocking is automatically lifted when the score returns to `0` or above.

### Admin-initiated early expiration

Admins can end a penalty before its scheduled expiration. This is implemented by setting its expiration date to one second before the current time.

```python
expire_date = now - 1 second
```

The penalty record is retained for historical purposes but no longer contributes to the reliability score.

---

## 4. Heavy Penalty Logic

The heavy penalty is an additional deduction of **100 points** applied when a driver has at least two active last-minute abandon penalties within a 15-day window.

Conceptually:

```python
heavy_penalty = 100 if active_last_minute_abandons >= 2 else 0
```

The final score becomes:

```python
reliability_score = (
    100
    - sum(active_penalty_points)
    - heavy_penalty
)
```

The heavy penalty is separate from the individual last-minute abandon penalties. Each last-minute abandon still contributes its own 100 points.

With the default expiration settings, last-minute abandons remain active for at least 15 days, supporting the heavy-penalty calculation.

---

## 5. Worked Examples

### Example A: One standard abandon

A driver abandons a route with 18 hours remaining, more than 15 minutes after accepting it.

```text
Base score:          100
Abandon penalty:     -30
------------------------
Final score:          70
```

**Result:** The driver remains eligible for matching, but their ranking is reduced for the duration of the active penalty.

For example, a nearby driver with a clean record may rank higher than this driver, even if the clean driver is slightly farther away.

### Example B: Multiple standard abandons

A driver abandons four routes on days 0, 3, 6, and 9. Each abandonment incurs a 30-point penalty.

| Event | Active Abandons | Total Penalties | Score | Status |
|---|---:|---:|---:|---|
| Initial state | 0 | 0 | 100 | Eligible |
| Day 0: First abandon | 1 | 30 | 70 | Eligible |
| Day 3: Second abandon | 2 | 60 | 40 | Eligible |
| Day 6: Third abandon | 3 | 90 | 10 | Eligible |
| Day 9: Fourth abandon | 4 | 120 | -20 | Match-blocked |
| Day 14: First penalty expires | 3 | 90 | 10 | Eligible |

**Result:** The fourth abandonment causes the driver to become match-blocked. On day 14, the first penalty expires, the score recovers to `10`, and matching eligibility is automatically restored.

### Example C: One last-minute abandon

A driver abandons a route less than 12 hours before its start time, more than 15 minutes after accepting it.

```text
Base score:                  100
Last-minute abandon:        -100
---------------------------------
Final score:                    0
```

**Result:** The driver is not match-blocked because blocking occurs only when the score is strictly less than zero.

However, the 100-point deduction significantly reduces their ranking for at least 15 days under the default configuration. This makes it much harder for them to rank competitively against drivers with clean records or rider-familiarity bonuses.

### Example D: Two last-minute abandons within 15 days

A driver receives two last-minute abandon penalties within the heavy-penalty window.

```text
Base score:                  100
First last-minute abandon:  -100
Second last-minute abandon: -100
Heavy penalty:              -100
---------------------------------
Final score:                 -200
```

**Result:** The driver is match-blocked.

The heavy penalty applies in addition to both individual penalties.

### Example E: One no-show

An admin explicitly flags a driver as a no-show while unassigning a route.

```text
Base score:          100
No-show penalty:    -200
------------------------
Final score:        -100
```

**Result:** The driver is immediately match-blocked. Once the no-show penalty expires and the score is recalculated, the driver's score returns to `100`, assuming no other active penalties remain.

### Example F: Repeated quick abandons

Each quick abandon incurs only 5 points.

| Quick Abandons | Total Penalties | Score | Status |
|---:|---:|---:|---|
| 1 | 5 | 95 | Eligible |
| 5 | 25 | 75 | Eligible |
| 10 | 50 | 50 | Eligible |
| 20 | 100 | 0 | Eligible |
| 21 | 105 | -5 | Match-blocked |

**Result:** It takes 21 simultaneously active quick-abandon penalties to make the score negative.

Quick abandons therefore have a relatively small individual impact, providing a grace period for accidental route acceptance.

---

## 6. How Reliability Affects Driver Ranking

The reliability score and the route-matching ranking score serve different purposes:

- **Reliability score:** Determines whether the driver is eligible to participate in matching.
- **Ranking score:** Determines how favorably an eligible driver ranks against other eligible drivers.

A driver can have a reliability score of `0` or greater and still rank poorly due to active penalties.

Conversely, a driver with a high ranking score is still excluded from matching if their reliability score is negative.

### Example G: Penalties versus rider familiarity

Consider two drivers with the same distance score.

| Factor | Driver X | Driver Y |
|---|---:|---:|
| Distance score | +100 | +100 |
| Previously served a rider | +200 | 0 |
| Abandon penalty | 0 | -30 |
| **Final ranking score** | **300** | **70** |

**Result:** Driver X is offered the route first because their ranking score is higher.

This demonstrates how rider familiarity can improve ranking, while active abandonment penalties reduce it.

A driver's eligibility is evaluated separately using their reliability score.

---

## 7. Key Business Rules

1. Every driver starts with a base reliability score of `100`.
2. Active penalties are deducted from the base score.
3. The additional heavy penalty is applied when the heavy-penalty conditions are met.
4. A score below `0` blocks the driver from all route matching.
5. A score of exactly `0` does not block the driver.
6. Quick abandons incur only 5 points when the driver drops a route within 15 minutes of acceptance.
7. Abandoning a route more than 24 hours before its start time incurs no penalty.
8. No-show penalties require explicit admin action.
9. Penalties expire individually and are excluded from subsequent score calculations after expiration.
10. Expired records remain in the database for historical purposes.
11. Drivers are automatically unblocked when their recalculated reliability score reaches `0` or above.
12. Match blocking does not deactivate the driver or remove their app access.
13. Reliability determines eligibility; the ranking score determines the order in which eligible drivers are considered.

---

## 8. Summary

The system is designed to discourage unreliable behavior without immediately removing drivers from the platform.

- **Quick abandons:** Minor deductions intended to accommodate accidental accepts.
- **Standard abandons:** Moderate deductions that accumulate with repeated cancellations.
- **Last-minute abandons:** Severe deductions that can significantly damage ranking.
- **Repeated last-minute abandons:** Trigger an additional heavy penalty and can lead to match blocking.
- **No-shows:** Severe, admin-assigned penalties that immediately block a driver under the base scoring model.
- **Expiration:** Allows drivers to recover their score and regain matching eligibility automatically over time.

The central principle is that **penalties affect both ranking and eligibility, but only a reliability score below zero blocks a driver from matching.**

## 9. Ranking Score (decides who gets matched)

| Category | How it's computed | Default / seeded value |
|---|---|---|
| Rides completed (last 365 days) | Count of the driver's `COMPLETED` rides on ended, acknowledged route groups. Points come from a bracket table; the highest bracket reached applies (not cumulative). | Bracket comes from `rides-completed-points-bracket`. No code default, so 0 points if unset. |
| Routes completed recently | Number of route groups completed in the last 7 days, multiplied by a per-route value. | `rides-recently-completed-points`, code default: 0. |
| Location familiarity | Per pickup/dropoff location on the route where the driver has completed rides in the last year. | 50 points per location. |
| Rider familiarity | Per rider on the route for whom the driver has completed a ride in the last year. | 200 points per rider in SQL (code default: 100). |
| Gender preference | Flat bonus if the driver's gender matches a rider preference on the route. Skipped when the preference is enforced, because non-matching drivers are filtered out instead. | 125 points (enforcement defaults to on). |
| Distance | The shortest distance among three possible reference points is used to determine the distance score. The tightest matching bracket determines the points. <br><br> **1. Service location:** The driver's saved home or service-area point, available whenever the driver has set one. <br> *Example:* The driver's service location is 6.5 miles from the pickup. <br><br> **2. Live location:** The last location reported by the driver's phone. Used only when the ride is **2 or more hours away**. The location must have been reported within the last 120 minutes and must not have expired. When the driver comes online, the app's fresh ping location is used; otherwise, the latest-location store is queried. <br> *Example:* The driver is at the gym, 2.8 miles from the pickup. <br><br> **3. Nearby ride-end:** Measures proximity to other routes the driver has claimed that end shortly **before** this route starts or start shortly **after** it ends. Time windows are configured by admins (60-minute code fallback), with a safety gap between routes. Cancelled routes are ignored. <br> - **Before case:** Distance from the previous route's end to this route's first stop. <br> - **After case:** Distance from this route's last stop to the next route's start. | SQL distance brackets: ≤3 mi = 100, ≤5 mi = 80, ≤8 mi = 50, ≤12 mi = 20. |
| Reliability penalty | Subtracts the driver's active penalty points (see Section 2). | Negative points based on active penalties. |

## 10. Ideas around performance score

- Drivers with a clean 60 day record get a "Reliable" badge and a ranking boost. Las minute abandons hide the badge and pause their boost.
- Strike 1 is a documented second chance: When a driver moves `< 0` for the first time, system rewards them a +50 immediately (goodwill credit) that boosts them to middle of the matching tier and notifies them - app also shows this record. Driver can drive, but will be matched to limited rides. They'll feel the consequences.
- After the second chance (goodwill) is used, the next qualifying abandon (< 0) puts the driver in a persistent paused state.
- Only an admin can reinstate them, with a reason recorded.
