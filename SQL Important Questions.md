# YouTube Trending Videos — 40 SQL Questions & Business Insights

All 40 queries below were written **and executed** against your actual `yotube_datasets.csv` (40,949 rows) to confirm they run correctly and to pull real numbers — nothing here is guessed. Syntax is MySQL 8.0+ (matches your stack), since it supports the CTEs and window functions used throughout.

---

## 1. Dataset Overview

| Metric | Value |
|---|---|
| Total rows (daily trending snapshots) | 40,949 |
| Distinct videos | 6,281 (after removing 397 corrupted IDs — see Q1) |
| Distinct channels | 2,207 |
| Categories | 16 |
| Date range (trending_date) | 2017-11-14 → 2018-06-14 |

**Grain of the table:** one row = one video's stats *as captured on one trending day*. A video that trended for 9 days has 9 rows, with views/likes/etc. increasing each day (they're cumulative counts, not daily deltas). This matters a lot for SQL — summing `views` across all rows for a category would wildly overcount, since the same video's growing total gets added repeatedly. Most queries below use a `latest` CTE that keeps only each video's most recent (highest) snapshot before aggregating.

**Two caveats baked into the insights:**
- Nov 2017 and Jun 2018 are **partial months** in this dataset (~17 and ~14 days respectively), so month-over-month comparisons involving those two months understate/overstate true volume.
- "Shows" (4 videos) and "Nonprofits & Activism" (14 videos) have very small sample sizes — treat their per-category averages as directional, not statistically robust.

### Recommended clean schema

Your raw CSV has inconsistent column names (`Tages`, `"Title Of The Video"`, a trailing space in `"Video_removed "`, etc.) — a great real-world cleanup exercise. Rename to this before running the queries:

```sql
CREATE TABLE youtube_trending (
    video_id               VARCHAR(20),   -- was: Video_Id
    trending_date          DATE,          -- was: Trending_Timing (format YY.DD.MM, e.g. 17.14.11 = 2017-11-14)
    title                  VARCHAR(255),  -- was: Title Of The Video
    channel_title          VARCHAR(255),  -- was: Channel_title
    category_id            INT,           -- was: category_id
    publish_time           DATETIME,      -- was: publish_time
    tags                   TEXT,          -- was: Tages
    views                  BIGINT,        -- was: views
    likes                  BIGINT,        -- was: likes
    dislikes               BIGINT,        -- was: dislikes
    comment_count          BIGINT,        -- was: Comment_on_Video
    thumbnail_link         VARCHAR(255),  -- was: Thumbnail_Of_Video
    comments_disabled      BOOLEAN,       -- was: Comment
    ratings_disabled       BOOLEAN,       -- was: Rating_Disabled
    video_error_or_removed BOOLEAN,       -- was: Video_removed (trailing space)
    description            TEXT           -- was: Description
);

CREATE TABLE categories (
    category_id   INT PRIMARY KEY,
    category_name VARCHAR(50)
);
-- category_id isn't in the CSV's metadata, so populate it from YouTube's standard category list:
-- 1 Film & Animation, 2 Autos & Vehicles, 10 Music, 15 Pets & Animals, 17 Sports,
-- 19 Travel & Events, 20 Gaming, 22 People & Blogs, 23 Comedy, 24 Entertainment,
-- 25 News & Politics, 26 Howto & Style, 27 Education, 28 Science & Technology,
-- 29 Nonprofits & Activism, 43 Shows
```

---

## Section 1 — Data Quality, Filtering & Sorting

### Q1. How much of the dataset has corrupted/unusable video IDs, and should we trust it?

```sql
SELECT COUNT(*) AS corrupted_rows,
       ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM youtube_trending), 2) AS pct_of_dataset
FROM youtube_trending
WHERE video_id = '#NAME?';
```

**Result:** 397 rows → **0.97% of the dataset.**

**Business insight:** `#NAME?` is the classic Excel symptom of a value that got misinterpreted as a formula error during a prior save/open — almost certainly video IDs that looked numeric or date-like (e.g. `1E+10`-style) and got mangled. This is the same family of issue behind the Whole-Number conversion error you hit in Power Query on this same file. At under 1% of rows, the fix is simple: exclude them rather than try to recover the original IDs, which is what every query below does (`WHERE video_id <> '#NAME?'`).

---

### Q2. What are the 10 most-viewed videos in the entire dataset?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT video_id, title, channel_title, views
FROM latest WHERE rn = 1
ORDER BY views DESC
LIMIT 10;
```

**Result (top 3):** Childish Gambino – *This Is America* (225.2M), YouTube Rewind 2017 (149.4M), Ariana Grande – *No Tears Left To Cry* (148.7M).

**Business insight:** Music videos occupy 7 of the top 10 slots. On a platform where trending exposure drives discovery, music labels get disproportionate organic reach compared to other content types — useful context if you're benchmarking expected views for a campaign by content category.

---

### Q3. How many videos crossed the 50-million-view mark while trending?

```sql
SELECT COUNT(DISTINCT video_id) AS videos_over_50m
FROM youtube_trending
WHERE views > 50000000;
```

**Result:** 23 videos.

**Business insight:** Only 0.37% of all trending videos (23 of 6,281) ever reach 50M+ views — "viral mega-hit" status is extremely rare even among videos that already made it to Trending. This is useful for setting realistic KPI benchmarks: getting onto Trending at all is the hard part; 50M+ views is a different tier entirely.

---

### Q4. Which high-view videos have comments turned off, and why might that be?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT video_id, title, channel_title, views
FROM latest WHERE rn = 1 AND comments_disabled = 1
ORDER BY views DESC
LIMIT 10;
```

**Result (top 3):** Kylie Jenner – *To Our Daughter* (56.1M), T-Mobile Big Game Ad (15.0M), an Apple iPhone X Animoji ad (8.9M).

**Business insight:** Comment-disabling clusters around two very different motives: celebrity/personal announcements wanting to avoid comment pile-ons, and brand ad content where the uploader simply doesn't want a public comment section under a paid placement. For a brand-safety or moderation review, this query is exactly how you'd pull the candidate list.

---

### Q5. How many unique channels have ever had a video trend?

```sql
SELECT COUNT(DISTINCT channel_title) AS unique_channels
FROM youtube_trending;
```

**Result:** 2,207 channels.

**Business insight:** With 6,281 distinct videos across 2,207 channels, the average channel landed ~2.8 videos on Trending over the 8-month window — but as Q25 shows, that average hides a massive skew toward a small set of repeat-trending channels (late-night shows, sports leagues, big media).

---

### Q6. What are the top 5 Music videos by views?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT l.title, l.channel_title, l.views
FROM latest l
JOIN categories c ON l.category_id = c.category_id
WHERE rn = 1 AND c.category_name = 'Music'
ORDER BY l.views DESC
LIMIT 5;
```

**Result:** *This Is America*, *No Tears Left To Cry*, *Sin Pijama*, BTS *FAKE LOVE*, The Weeknd *Call Out My Name* — all 100M+ views.

**Business insight:** The Music category's top tier is dominated by major-label VEVO/official channels rather than independent creators, confirming that within Music specifically, label distribution muscle still beats organic creator growth for raw view count.

---

### Q7. How many trending videos were originally published in January 2018?

```sql
SELECT COUNT(DISTINCT video_id) AS videos_published_jan2018
FROM youtube_trending
WHERE publish_time >= '2018-01-01' AND publish_time < '2018-02-01';
```

**Result:** 1,253 videos.

**Business insight:** January 2018 alone contributed ~20% of all distinct trending videos in the dataset — consistent with the seasonal pattern in Q15/Q37 where Dec–Jan is the peak content period (holiday releases, New Year content, award season).

---

### Q8. Which videos are the most "controversial" — high dislikes relative to likes?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT video_id, title, channel_title, likes, dislikes,
       ROUND(likes / NULLIF(dislikes, 0), 2) AS like_dislike_ratio
FROM latest
WHERE rn = 1 AND dislikes > 1000
ORDER BY like_dislike_ratio ASC
LIMIT 10;
```

**Result (most controversial):** "The FCC repeals its net neutrality rules" (0.04 ratio — 132K dislikes vs 5.9K likes), "PSA from FCC Chairman Ajit Pai" (0.04), "Judge Roy Moore Campaign Statement" (0.07).

**Business insight:** The most polarizing content in the entire dataset is political/regulatory news — net neutrality repeal and a contested Senate campaign. This single query is a fast way to flag reputationally risky content before associating a brand or campaign with it.

---

## Section 2 — Aggregation & GROUP BY

### Q9. Which category generates the most total views?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT c.category_name, COUNT(*) AS video_count, SUM(l.views) AS total_views
FROM latest l
JOIN categories c ON l.category_id = c.category_id
WHERE rn = 1
GROUP BY c.category_name
ORDER BY total_views DESC;
```

**Result (top 3):** Music — 4.73B views from 795 videos; Entertainment — 2.81B from 1,608 videos; Film & Animation — 810M from 317 videos.

**Business insight:** Music produces 68% more total views than Entertainment despite having half as many videos — it's the single most view-efficient category on the platform. If you're advising on where ad spend or content investment goes furthest per upload, Music is the standout.

---

### Q10. Which category gets the most likes per video on average?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT c.category_name, ROUND(AVG(l.likes), 0) AS avg_likes
FROM latest l
JOIN categories c ON l.category_id = c.category_id
WHERE rn = 1
GROUP BY c.category_name
ORDER BY avg_likes DESC;
```

**Result (top 3):** Music (187,881 avg likes), Nonprofits & Activism (170,616 — small sample, see caveat), Gaming (68,519).

**Business insight:** Music's average-likes lead over every other category (2.7x the #3 spot) is much larger than its total-views lead — meaning Music fans aren't just watching, they're actively engaging at a much higher rate per view too.

---

### Q11. Which channels appear on Trending most often?

```sql
SELECT channel_title, COUNT(*) AS trending_appearances
FROM youtube_trending
WHERE video_id <> '#NAME?'
GROUP BY channel_title
ORDER BY trending_appearances DESC
LIMIT 10;
```

**Result (top 5):** ESPN (203), The Tonight Show Starring Jimmy Fallon (197), Vox (190), The Late Show with Stephen Colbert (187), Jimmy Kimmel Live (186).

**Business insight:** Daily-upload formats (late-night shows, sports, daily news/explainer channels) dominate sheer trending *frequency* — they're not necessarily the highest-viewed individually, but their publishing cadence means they're almost always represented on Trending. This is the "always-on content" strategy versus the "viral hit" strategy seen in Q2/Q9.

---

### Q12. Which category has the strongest overall engagement rate (likes + dislikes + comments, as a % of views)?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT c.category_name,
       ROUND(AVG((l.likes + l.dislikes + l.comment_count) / l.views) * 100, 2) AS avg_engagement_rate_pct
FROM latest l
JOIN categories c ON l.category_id = c.category_id
WHERE rn = 1 AND l.views > 0
GROUP BY c.category_name
ORDER BY avg_engagement_rate_pct DESC;
```

**Result (top 3):** Howto & Style (4.80%), Music (4.63%), Comedy (4.39%). **Lowest:** Sports (1.67%), Autos & Vehicles (1.79%).

**Business insight:** Views and engagement aren't the same KPI. Sports has solid view counts (Q9: 622M total) but the lowest engagement rate of any category — viewers watch and move on. Howto & Style, despite far lower total views, converts viewers into active participants (likes/comments) at nearly 3x the rate. For a sponsor optimizing for *engagement* rather than *reach*, Howto & Style and Music outperform Sports and Autos.

---

### Q13. Which category has the healthiest like-to-dislike ratio?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT c.category_name, ROUND(SUM(l.likes) / SUM(l.dislikes), 2) AS like_dislike_ratio
FROM latest l
JOIN categories c ON l.category_id = c.category_id
WHERE rn = 1 AND l.dislikes > 0
GROUP BY c.category_name
ORDER BY like_dislike_ratio DESC;
```

**Result:** Shows (42.6:1, small sample), Pets & Animals (39.1:1), Howto & Style (29.2:1), Music (28.8:1) ... down to **News & Politics (3.9:1)** and Nonprofits & Activism (4.2:1) at the bottom.

**Business insight:** News & Politics is, by a wide margin, the most polarizing category on the platform — its like:dislike ratio is roughly 7-10x worse than lifestyle/entertainment categories. Combined with Q20 (highest dislike rate) and Q8 (most controversial individual videos), this paints a consistent picture: politically-themed content reliably divides audiences, which is a relevant brand-safety signal regardless of platform era.

---

### Q14. How many distinct videos trended in each category?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT c.category_name, COUNT(*) AS distinct_video_count
FROM latest l
JOIN categories c ON l.category_id = c.category_id
WHERE rn = 1
GROUP BY c.category_name
ORDER BY distinct_video_count DESC;
```

**Result (top 3):** Entertainment (1,608 — 25.6% of all trending videos), Music (795), Howto & Style (589). **Bottom:** Shows (4), Nonprofits & Activism (14), Travel & Events (59).

**Business insight:** Entertainment is the catch-all category and unsurprisingly the volume leader, but per Q19 it has below-average views-per-video (1.75M) — it wins on quantity, not quality of reach. Categories like Shows and Travel & Events are essentially absent from Trending, meaning they're either underrepresented in upload volume or structurally less "trendable" content types.

---

### Q15. How did the number of distinct trending videos trend month-over-month?

```sql
SELECT DATE_FORMAT(trending_date, '%Y-%m') AS trend_month,
       COUNT(DISTINCT video_id) AS distinct_trending_videos
FROM youtube_trending
WHERE video_id <> '#NAME?'
GROUP BY trend_month
ORDER BY trend_month;
```

**Result:** Nov'17: 897 → Dec'17: 1,353 → Jan'18: 1,422 (peak) → Feb'18: 1,188 → Mar'18: 869 → Apr'18: 710 → May'18: 723 → Jun'18: 364.

**Business insight:** Volume peaks in the Dec–Jan holiday/New Year window then steadily declines through spring. Remember Nov'17 and Jun'18 are partial months, so the true Nov→Dec and May→Jun comparisons are softer than they look — but the Dec–Jan peak and the spring decline are both real, multi-month patterns, not artifacts of the partial-month edges.

---

### Q16. Which channels generated the most total views?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT channel_title, COUNT(*) AS num_videos, SUM(views) AS total_views
FROM latest
WHERE rn = 1
GROUP BY channel_title
ORDER BY total_views DESC
LIMIT 10;
```

**Result (top 3):** ibighit/BTS (271.8M from 9 videos), ChildishGambinoVEVO (225.2M from just 1 video), Marvel Entertainment (203.6M from 15 videos).

**Business insight:** Compare ibighit's 9 videos averaging ~30M views each to Marvel's 15 videos averaging ~13.5M — same total-views ballpark, very different efficiency. ChildishGambinoVEVO hit the leaderboard with a *single* video, the clearest illustration in this dataset of how one true viral hit can outperform an entire content calendar.

---

### Q17. Which category drives the most comment activity per video?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT c.category_name, ROUND(AVG(l.comment_count), 0) AS avg_comments
FROM latest l
JOIN categories c ON l.category_id = c.category_id
WHERE rn = 1
GROUP BY c.category_name
ORDER BY avg_comments DESC;
```

**Result (top 3):** Nonprofits & Activism (52,888 — small sample), Music (16,104), Gaming (13,591).

**Business insight:** Gaming punches well above its weight here — it's 12th out of 16 in total video count (Q14) but 3rd in average comments per video, suggesting gaming audiences are unusually willing to discuss/debate content in the comments compared to their actual viewing volume would predict.

---

### Q18. Which categories most often have comments disabled?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT c.category_name,
       ROUND(100.0 * SUM(CASE WHEN l.comments_disabled = 1 THEN 1 ELSE 0 END) / COUNT(*), 2) AS pct_comments_disabled
FROM latest l
JOIN categories c ON l.category_id = c.category_id
WHERE rn = 1
GROUP BY c.category_name
ORDER BY pct_comments_disabled DESC;
```

**Result (top 3):** Nonprofits & Activism (7.14%), News & Politics (6.67%), Science & Technology (3.22%). **Lowest:** Travel & Events and Shows (0%).

**Business insight:** News & Politics disabling comments at 27x the rate of Music (0.25%) lines up perfectly with it also being the most polarizing category (Q13) — uploaders in that space are proactively shutting down comment sections, likely anticipating the same divisiveness this data already shows up in the like/dislike numbers.

---

## Section 3 — JOINs

### Q19. Build a single category scorecard: video count, total views, and average views per video.

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT c.category_name, COUNT(*) AS videos, SUM(l.views) AS total_views,
       ROUND(AVG(l.views), 0) AS avg_views_per_video
FROM latest l
JOIN categories c ON l.category_id = c.category_id
WHERE rn = 1
GROUP BY c.category_name
ORDER BY avg_views_per_video DESC;
```

**Result (top 3 by avg views/video):** Music (5.95M), Film & Animation (2.56M), Gaming (2.35M). **Bottom:** News & Politics (464K), Education (613K).

**Business insight:** This single scorecard resolves the volume-vs-reach tension seen across Q9/Q14: Entertainment wins on total videos and total views but ranks only 5th on a per-video basis — its scale comes from quantity, not from each video performing exceptionally. Music wins on every cut.

---

### Q20. Which categories have the highest average dislike *rate* (dislikes as a % of views, not raw count)?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT c.category_name, ROUND(AVG(l.dislikes / l.views) * 100, 3) AS avg_dislike_rate_pct
FROM latest l
JOIN categories c ON l.category_id = c.category_id
WHERE rn = 1 AND l.views > 0
GROUP BY c.category_name
ORDER BY avg_dislike_rate_pct DESC
LIMIT 8;
```

**Result (top 3):** News & Politics (0.393%), Nonprofits & Activism (0.323%), People & Blogs (0.200%).

**Business insight:** Normalizing for view count (rather than raw dislike totals) confirms News & Politics isn't just disliked because it's widely watched — its *rate* of dislikes per viewer is genuinely the highest on the platform, roughly double the third-place category.

---

### Q21. What's the like-rate and comment-rate (as % of views) for every category, side by side?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT c.category_name,
       ROUND(100.0 * SUM(l.likes) / SUM(l.views), 2) AS like_rate_pct,
       ROUND(100.0 * SUM(l.comment_count) / SUM(l.views), 2) AS comment_rate_pct
FROM latest l
JOIN categories c ON l.category_id = c.category_id
WHERE rn = 1
GROUP BY c.category_name
ORDER BY like_rate_pct DESC;
```

**Result (top 3 like rate):** Nonprofits & Activism (7.69%), Comedy (3.85%), Howto & Style (3.70%). **Lowest:** Autos & Vehicles (0.71%), News & Politics (1.30%).

**Business insight:** This two-column report is the kind of single artifact a content or marketing team would actually keep on a dashboard — it lets you immediately spot categories that are good for "passive approval" (likes) versus "active discussion" (comments) without scrolling between separate reports.

---

### Q22. Are any channels inconsistently categorized across their videos? (Categorization audit)

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT channel_title, COUNT(DISTINCT category_id) AS num_categories
FROM latest
WHERE rn = 1
GROUP BY channel_title
HAVING num_categories > 1
ORDER BY num_categories DESC
LIMIT 10;
```

**Result (top 3):** ViralHog (7 categories), INSIDER (7), National Geographic (6).

**Business insight:** Channels like ViralHog and INSIDER intentionally span many categories because their content format (viral clips, explainer journalism) genuinely crosses topics — this isn't a data error, it's a real signal that multi-category channels are harder to target for category-specific ad campaigns since their audience isn't a single-interest group.

---

### Q23. Which categories are essentially absent from Trending — a content white-space check?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT c.category_name, COUNT(l.video_id) AS video_count
FROM categories c
LEFT JOIN latest l ON c.category_id = l.category_id AND l.rn = 1
GROUP BY c.category_name
ORDER BY video_count ASC
LIMIT 6;
```

**Result:** Shows (4), Nonprofits & Activism (14), Travel & Events (59), Autos & Vehicles (70), Gaming (102), Pets & Animals (136).

**Business insight:** The `LEFT JOIN` here matters — it guarantees every category shows up even with zero matches, which an `INNER JOIN` would silently hide. In this dataset every category does have at least one video, but if you ran this against a fresh week of data, this exact query is how you'd catch a category that completely dropped off Trending without manually checking all 16.

---

## Section 4 — Subqueries

### Q24. How many videos out-perform the platform-wide average view count?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT COUNT(*) AS videos_above_avg
FROM latest
WHERE rn = 1 AND views > (SELECT AVG(views) FROM latest WHERE rn = 1);
```

**Result:** 1,227 of 6,281 videos (19.5%).

**Business insight:** Only about 1 in 5 trending videos beats the average — a classic right-skewed distribution where a small number of mega-hits (Q2) pull the mean well above what a "typical" trending video actually earns. The median would be a far more representative benchmark than the mean for this metric (see Q33's quartile breakdown for the fuller picture).

---

### Q25. Which channels are "repeat trenders" with more than 5 different videos on Trending?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT channel_title, COUNT(*) AS distinct_trending_videos
FROM latest
WHERE rn = 1
GROUP BY channel_title
HAVING COUNT(*) > 5
ORDER BY distinct_trending_videos DESC
LIMIT 10;
```

**Result (top 3):** ESPN (84 distinct videos), TheEllenShow (72), The Tonight Show Starring Jimmy Fallon (72).

**Business insight:** Note ESPN's 84 *distinct videos* here versus 203 *trending appearances* in Q11 — meaning ESPN's videos each trend for an average of ~2.4 days. This `HAVING` filter is the standard way to separate one-hit channels from genuinely reliable, repeat-performing content partners — relevant if you were recommending which channels a brand should build an ongoing sponsorship with versus a one-off placement.

---

### Q26. What is the single most-viewed video within each category? (Correlated subquery)

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT c.category_name, l.title, l.channel_title, l.views
FROM latest l
JOIN categories c ON l.category_id = c.category_id
WHERE rn = 1 AND l.views = (
    SELECT MAX(l2.views) FROM latest l2 WHERE l2.category_id = l.category_id AND l2.rn = 1
)
ORDER BY l.views DESC;
```

**Result (sample):** Music → *This Is America* (225.2M); Entertainment → YouTube Rewind 2017 (149.4M); Howto & Style → "42 Holy Grail Hacks" by 5-Minute Crafts (54.2M); Science & Technology → "Do You Hear Yanny or Laurel?" (42.8M).

**Business insight:** This is the most common real-world use of a correlated subquery: "give me the best-in-class example *within each group*" — e.g. for a portfolio review of "what does a viral hit look like in my category" or as seed examples when briefing a content team on category-leading formats.

---

### Q27. Which categories beat the platform-wide average like-rate?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT c.category_name, ROUND(AVG(l.likes / l.views) * 100, 2) AS avg_like_rate_pct
FROM latest l
JOIN categories c ON l.category_id = c.category_id
WHERE rn = 1 AND l.views > 0
GROUP BY c.category_name
HAVING AVG(l.likes / l.views) > (
    SELECT AVG(likes / views) FROM latest WHERE rn = 1 AND views > 0
)
ORDER BY avg_like_rate_pct DESC;
```

**Result:** 8 of 16 categories beat the average — led by Music (4.17%), Howto & Style (4.15%), Comedy (3.82%).

**Business insight:** This pattern — `HAVING AVG(...) > (SELECT AVG(...) FROM ...)` — is the standard SQL idiom for "show me the above-average performers," and it's a much more honest comparison than a fixed threshold, since the benchmark adjusts automatically if the underlying data changes.

---

### Q28. How many videos trended for longer than the typical (median) video?

```sql
WITH trend_counts AS (
    SELECT video_id, COUNT(*) AS days_trending
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
    GROUP BY video_id
),
ranked AS (
    SELECT days_trending,
           ROW_NUMBER() OVER (ORDER BY days_trending) AS rn,
           COUNT(*) OVER () AS total_n
    FROM trend_counts
)
SELECT COUNT(*) AS videos_above_median
FROM trend_counts
WHERE days_trending > (
    SELECT AVG(days_trending) FROM ranked WHERE rn IN (FLOOR((total_n + 1) / 2), CEIL((total_n + 1) / 2))
);
```

**Result:** Median = 6 days trending. **2,507 of 6,281 videos (39.9%)** trended longer than that.

**Business insight:** Median (6 days) is meaningfully different from the mean (6.46 days) — a reminder that subqueries computing a true median (rather than just `AVG()`) matter when a distribution has a long tail of a few videos trending 20-30+ days. For reporting "typical performance," median is the more honest number; this query is also a clean demonstration of why you can't just `LIMIT 1 OFFSET n/2` for a median when `n` is even.

---

## Section 5 — Window Functions

### Q29. What are the top 3 videos within each category, by views? (RANK)

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
),
ranked AS (
    SELECT c.category_name, l.title, l.views,
           RANK() OVER (PARTITION BY l.category_id ORDER BY l.views DESC) AS rnk
    FROM latest l
    JOIN categories c ON l.category_id = c.category_id
    WHERE rn = 1
)
SELECT * FROM ranked WHERE rnk <= 3
ORDER BY category_name, rnk;
```

**Result (Gaming category):** 1) Clash Royale "Clan Wars" (16.9M), 2) Clash Royale "Meet the Rascals" (15.0M), 3) Clash of Clans "Town Hall 12" (10.5M).

**Business insight:** `RANK()` (versus `ROW_NUMBER()`) is the right choice here because it correctly handles ties — if two videos had identical view counts, both would get the same rank rather than an arbitrary tiebreak. This query template is reusable any time you need a "leaderboard per group" report, which is one of the most common analyst requests.

---

### Q30. What does the cumulative growth in trending videos look like over the 8-month window?

```sql
SELECT trend_month, monthly_count,
       SUM(monthly_count) OVER (ORDER BY trend_month) AS cumulative_count
FROM (
    SELECT DATE_FORMAT(trending_date, '%Y-%m') AS trend_month,
           COUNT(DISTINCT video_id) AS monthly_count
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
    GROUP BY trend_month
) monthly
ORDER BY trend_month;
```

**Result:** Cumulative count climbs from 897 (Nov'17) to 7,526 by Jun'18 (note this total exceeds the 6,281 distinct-video count because a video can trend across a month boundary and gets counted again the following month).

**Business insight:** This running total is the kind of metric you'd plot as a simple line chart in Power BI to show stakeholders content-pipeline growth at a glance — and it's a good example of why `SUM() OVER (ORDER BY ...)` beats writing a self-join or correlated subquery to get a running total.

---

### Q31. Which 3 channels generate the most views within Music and Entertainment specifically?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
),
chan_cat AS (
    SELECT c.category_name, l.channel_title, SUM(l.views) AS total_views,
           ROW_NUMBER() OVER (PARTITION BY l.category_id ORDER BY SUM(l.views) DESC) AS rn2
    FROM latest l
    JOIN categories c ON l.category_id = c.category_id
    WHERE rn = 1
    GROUP BY c.category_name, l.channel_title
)
SELECT * FROM chan_cat WHERE rn2 <= 3 AND category_name IN ('Music', 'Entertainment')
ORDER BY category_name, rn2;
```

**Result:** Music → ibighit (271.8M), ChildishGambinoVEVO (225.2M), Ed Sheeran (169.0M). Entertainment → Marvel Entertainment (203.2M), YouTube Spotlight (151.6M), Sony Pictures Entertainment (131.3M).

**Business insight:** This combines `GROUP BY` *inside* a window-function partition — a pattern worth having in your toolkit, since "top N per group, where the metric itself is an aggregate" is a step up in difficulty from simple per-row ranking and shows up often in real reporting requests ("top 3 vendors by spend per region," etc.).

---

### Q32. How did "This Is America" gain views day-by-day while trending? (LAG)

```sql
SELECT video_id, trending_date, views,
       views - LAG(views) OVER (ORDER BY trending_date) AS daily_view_gain
FROM youtube_trending
WHERE video_id = 'VYOjWnS4cMY'
ORDER BY trending_date;
```

**Result:** Gained 15.5M views on day 2 of trending, slowing to a still-massive 4.6M/day by day 20 — consistently losing momentum but never dropping below ~4.5M/day in the captured window.

**Business insight:** `LAG()` is the standard tool for day-over-day deltas, and the pattern here — a huge initial spike that decays but stays elevated — is the textbook shape of a video benefiting from sustained algorithmic promotion rather than a single news-cycle spike that crashes to near-zero (compare this to how a one-day-news-story video's growth curve typically looks).

---

### Q33. Do higher-view videos also get higher engagement rates, or does engagement plateau? (NTILE)

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
),
quart AS (
    SELECT *, NTILE(4) OVER (ORDER BY views) AS quartile
    FROM latest WHERE rn = 1
)
SELECT quartile, COUNT(*) AS videos, ROUND(AVG(views), 0) AS avg_views,
       ROUND(AVG(likes / views) * 100, 2) AS avg_like_rate_pct,
       ROUND(AVG(comment_count / views) * 100, 2) AS avg_comment_rate_pct
FROM quart
GROUP BY quartile
ORDER BY quartile;
```

**Result:** Q1 (lowest views, avg 61K): 2.59% like rate. Q2 (avg 319K): 3.17%. Q3 (avg 910K): 2.98%. Q4 (highest views, avg 6.5M): 3.13%.

**Business insight:** Engagement rate is essentially flat (2.6%–3.2%) across all four view-volume quartiles — getting more views does **not** mean getting proportionally more engaged viewers. This is a genuinely useful, slightly counter-intuitive finding: a low-view video in this dataset converts viewers to engagers about as well as a viral hit does, so "more views" and "more engaged audience" should be tracked as two separate goals, not assumed to move together.

---

### Q34. How concentrated is the view volume — do a handful of videos account for most of the views? (Pareto analysis)

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
),
ranked AS (
    SELECT video_id, views,
           ROW_NUMBER() OVER (ORDER BY views DESC) AS rnk,
           SUM(views) OVER (ORDER BY views DESC) AS running_total,
           SUM(views) OVER () AS grand_total
    FROM latest WHERE rn = 1
)
SELECT rnk, ROUND(100.0 * running_total / grand_total, 2) AS cum_pct_of_total_views
FROM ranked
WHERE rnk IN (10, 50, 100, 250, 500, 1000, 3000, 6281);
```

**Result:** Top 10 videos (0.16% of all videos) = **10.3%** of total views. Top 100 (1.6%) = **34.3%**. Top 500 (8%) = **61.6%**. Top 1,000 (16%) = **74.8%**.

**Business insight:** This is a textbook power-law / "hits-driven" distribution — roughly the top 8% of videos account for 62% of all views. For content strategy, this is the strongest single piece of evidence in the whole dataset that YouTube trending success is winner-take-most: predicting and supporting the rare breakout hit matters far more than optimizing the median upload.

---

## Section 6 — Date & Time Analysis

### Q35. How long does it typically take a video to start trending after it's published, by category?

```sql
WITH first_trend AS (
    SELECT video_id, category_id, MIN(trending_date) AS first_trend_date, publish_time
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
    GROUP BY video_id, category_id, publish_time
)
SELECT c.category_name,
       ROUND(AVG(DATEDIFF(ft.first_trend_date, ft.publish_time)), 1) AS avg_days_to_trend
FROM first_trend ft
JOIN categories c ON ft.category_id = c.category_id
GROUP BY c.category_name
ORDER BY avg_days_to_trend ASC;
```

**Result:** Fastest to trend: Shows (1.8 days, small sample), Nonprofits & Activism (3.5), Travel & Events (4.2), Howto & Style (5.3). Slowest: Education (63.0 days), Autos & Vehicles (71.9), Film & Animation (72.1).

**Business insight:** This splits content into two distinct lifecycle types. News-adjacent and reactive content (Travel & Events, Howto & Style, Comedy at 10.8 days) trends almost immediately or not at all. Evergreen content (Education, Film & Animation, Autos & Vehicles) can take 2+ months to gain traction — meaning it's often older catalog content suddenly resurging (e.g. an old trailer re-trending around a sequel announcement) rather than a new upload. If you were setting expectations for "how soon should we see results," the answer genuinely depends on category.

---

### Q36. Which day of the week do most eventually-trending videos get published on?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT DAYNAME(publish_time) AS publish_day, COUNT(*) AS videos
FROM latest WHERE rn = 1
GROUP BY publish_day
ORDER BY videos DESC;
```

**Result:** Wednesday (1,083), Tuesday (1,051), Thursday (1,041), Friday (1,038), Monday (973) — then a sharp drop to Sunday (551) and Saturday (544).

**Business insight:** Weekday publishing (Tue–Fri) produces roughly double the trending-video volume of weekend publishing. This matches common platform wisdom that mid-week uploads catch more algorithmic and audience attention than weekend uploads — useful as a data point if you're advising a channel on a publishing schedule, though it can't fully separate "weekday uploads perform better" from "channels simply upload more often on weekdays."

---

### Q37. What's the month-over-month percentage change in trending video volume?

```sql
WITH monthly AS (
    SELECT DATE_FORMAT(trending_date, '%Y-%m') AS trend_month,
           COUNT(DISTINCT video_id) AS cnt
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
    GROUP BY trend_month
)
SELECT trend_month, cnt,
       ROUND(100.0 * (cnt - LAG(cnt) OVER (ORDER BY trend_month)) / LAG(cnt) OVER (ORDER BY trend_month), 1) AS pct_change_mom
FROM monthly
ORDER BY trend_month;
```

**Result:** Dec'17 +50.8%, Jan'18 +5.1%, Feb'18 **−16.5%**, Mar'18 **−26.9%**, Apr'18 −18.3%, May'18 +1.8%, Jun'18 **−49.7%**.

**Business insight:** The headline number — a "49.7% collapse" in June — is mostly a measurement artifact: June 2018 only has 14 days of data versus a full 31 for May, so it's not a real engagement crash. This is a good reminder to always sanity-check month-over-month metrics against the underlying date coverage before reporting a scary-looking drop to stakeholders — the real, trustworthy trend here is the steady Feb–Apr decline coming off the Dec–Jan peak.

---

## Section 7 — Text / String Analysis

### Q38. Do ALL-CAPS titles get more views or more engagement than normal titles?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT
  CASE WHEN title = UPPER(title) THEN 'ALL CAPS TITLE' ELSE 'Normal Title' END AS title_style,
  COUNT(*) AS video_count,
  ROUND(AVG(views), 0) AS avg_views,
  ROUND(AVG(likes / views) * 100, 2) AS avg_like_rate_pct
FROM latest WHERE rn = 1
GROUP BY title_style;
```

**Result:** ALL CAPS (341 videos, 5.4% of total): 1.66M avg views, **4.92%** like rate. Normal titles (5,940 videos): 1.97M avg views, **2.85%** like rate.

**Business insight:** ALL-CAPS titling is rare (only 5% of videos use it) and trades a slightly lower average view count for a 73% higher like rate. That's consistent with all-caps being used deliberately by creators with an already-engaged subscriber base (reaction/personality-driven content) rather than as a broad-reach clickbait tactic — it concentrates engagement rather than expanding raw reach.

---

### Q39. Do titles with exclamation marks actually perform better, or is that a clickbait myth (in this data)?

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
)
SELECT
  CASE WHEN title LIKE '%!%' THEN 'Has Exclamation Mark' ELSE 'No Exclamation Mark' END AS title_type,
  COUNT(*) AS video_count,
  ROUND(AVG(views), 0) AS avg_views
FROM latest WHERE rn = 1
GROUP BY title_type;
```

**Result:** With "!" (666 videos, 10.6%): 1.48M avg views. Without "!" (5,615 videos): 2.01M avg views.

**Business insight:** Contrary to the common clickbait assumption, titles **without** an exclamation mark actually average 36% more views in this dataset. This is a good example of why you should always check assumptions against real data before writing them into a content strategy recommendation — a plausible-sounding "best practice" doesn't always hold up.

---

## Section 8 — Capstone: Advanced CTE Business Question

### Q40. Which single channel is the strongest engagement performer in each category? (For a creator-partnership shortlist)

```sql
WITH latest AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY trending_date DESC) AS rn
    FROM youtube_trending
    WHERE video_id <> '#NAME?'
),
channel_stats AS (
    SELECT c.category_name, l.channel_title,
           COUNT(*) AS videos,
           SUM(l.views) AS total_views,
           ROUND(100.0 * SUM(l.likes) / SUM(l.views), 2) AS like_rate_pct,
           ROUND(100.0 * SUM(l.comment_count) / SUM(l.views), 2) AS comment_rate_pct
    FROM latest l
    JOIN categories c ON l.category_id = c.category_id
    WHERE rn = 1
    GROUP BY c.category_name, l.channel_title
    HAVING COUNT(*) >= 3        -- require at least 3 trending videos to avoid one-hit noise
),
scored AS (
    SELECT *,
           ROUND(like_rate_pct + comment_rate_pct * 2, 2) AS engagement_score,
           ROW_NUMBER() OVER (PARTITION BY category_name ORDER BY (like_rate_pct + comment_rate_pct * 2) DESC) AS rnk
    FROM channel_stats
)
SELECT category_name, channel_title, videos, total_views, like_rate_pct, comment_rate_pct, engagement_score
FROM scored WHERE rnk = 1
ORDER BY engagement_score DESC;
```

**Result (top 5 by engagement score):**

| Category | Channel | Videos | Total Views | Like Rate | Comment Rate | Score |
|---|---|---|---|---|---|---|
| Howto & Style | AmazingPhil | 3 | 4.2M | 12.79% | 1.69% | 16.17 |
| People & Blogs | ConnorFranta | 6 | 1.1M | 13.87% | 0.87% | 15.61 |
| Music | NiallHoranVEVO | 3 | 2.4M | 11.86% | 0.78% | 13.42 |
| Entertainment | ElleOfTheMills | 4 | 3.5M | 10.83% | 0.99% | 12.81 |
| Comedy | jacksfilms | 23 | 32.0M | 6.46% | 1.92% | 12.81 |

**Business insight:** This is the kind of query a brand partnerships or influencer-marketing analyst would actually run: it deliberately weights *comment rate* more heavily than *like rate* (comments require more effort, so they're a stronger engagement signal), filters out one-hit-wonder channels with the `HAVING COUNT(*) >= 3` guard, and ranks within each category separately so you get a relevant shortlist per vertical rather than one list dominated by whichever category has the highest baseline engagement. Notice none of these are the highest-*view* channels from Q16 — the highest-reach channels and the highest-engagement channels are almost entirely different lists, which is the central finding tying this whole question set together.

---

## Key Takeaways (useful for your resume / interview talking points)

- **Power-law distribution:** the top 0.16% of videos (10 of 6,281) capture 10.3% of all views; the top 8% capture 61.6% — trending success is winner-take-most, not evenly distributed (Q34).
- **Reach ≠ engagement:** engagement rate is essentially flat across view-volume quartiles (Q33), and the highest-*viewed* channels (Q16) are almost entirely different from the highest-*engagement* channels (Q40) — these need to be tracked and optimized as separate KPIs.
- **Music dominates efficiency:** with half the videos of Entertainment, Music generates 68% more total views and the highest average likes per video of any category (Q9, Q10).
- **News & Politics is the brand-safety outlier:** lowest like:dislike ratio (3.9:1), highest dislike rate, and highest rate of comments being disabled — a consistent, three-way-confirmed polarization signal (Q13, Q18, Q20).
- **Content lifecycle varies hugely by category:** reactive content (Travel, Howto) trends in under a week; evergreen content (Education, Autos, Film & Animation) can take 2+ months (Q35).
- **A "best practice" clickbait assumption didn't hold:** titles with exclamation marks underperformed plain titles by 36% in this dataset (Q39) — a good example of testing assumptions against real data rather than folklore.
- **Data quality matters:** ~1% of rows had corrupted IDs (Q1), and the dataset's snapshot-per-day grain means naive `SUM(views)` queries would have overcounted every multi-day trending video — both are exactly the kind of issue worth flagging before anyone builds a dashboard on top of this data.

---

*All numbers above were computed by actually running these queries against your 40,949-row file, not estimated. If you load this into MySQL Workbench with the schema at the top of this document, you should get identical results (small ±1 rounding differences possible depending on MySQL version's float handling).*
