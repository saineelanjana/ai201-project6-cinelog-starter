# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:**
- Renamed function `save_to_watchlist` to `add_to_watchlist` to match the naming convention of other service functions (verb_to_noun).
**How I verified:**
- Checked all calls to the function and updated them accordingly.
- Verified that the function behaves as expected with the new name.

## Comment 2 — Deduplication
**What I did:**
- Implemented a check in the `add_to_watchlist` function to prevent adding duplicate films to a user's watchlist and raise an error.
**How I verified:**
- Wrote unit tests to ensure the function raises `AlreadyInWatchlistError` when attempting to add a duplicate film.
- Manually tested the function with both duplicate and unique film IDs.

## Comment 3 — Missing test
**What I did:**
- Added a unit test to add a nonexistent film to a user's watchlist and verify that it raises a `FilmNotFoundError`.
**How I verified:**
- Ran the test suite and confirmed that the new test passes and correctly raises the expected error.

## Comment 4 — Default visibility
**My position:**
- Keeping `public` defaulted to `True` on `WatchlistEntry`.
**Reasoning:**
- A watchlist is inherently a social/sharing feature — the value of "what I want to watch" comes from friends being able to see it and get recommendations. Defaulting to public matches that intent without requiring extra setup from every user.
- Opt-out (public by default, toggle to private) keeps the common case frictionless, since most users adding a film to their watchlist have no reason to hide it.
**Tradeoff acknowledged:**
- This is an opt-out privacy model, which is more permissive than opt-in. A user could add a film before realizing their watchlist is visible by default, which is a legitimate privacy concern for anyone with sensitive/personal viewing interests. If we wanted to be more conservative, defaulting to `False` would protect users first and require explicit action to share, at the cost of reduced social engagement out of the box.

## Comment 5 — Sort order
**My position:**
- Changed `get_watchlist` to order by `date_added` descending (most recently added first) instead of `Film.title` ascending (alphabetical).
**Reasoning:**
- A watchlist is a queue of things to get to, not a reference list you look something up in — the dominant use case is "what did I just add / what's new," not "find title X." Recency ordering matches that mental model better than alphabetical.
**Engagement with reviewer's point:**
- Alphabetical does have a real advantage: it's stable and easy to scan when you're hunting for a specific title in a long list. That's a better fit for something like the collection (already-watched films), where users are more likely to browse or look something up.
- For the watchlist specifically, I think recency wins for most users, so I made the change rather than leaving it alphabetical. Open to revisiting if usage shows otherwise, but wanted to make a call rather than leave it unresolved.

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->