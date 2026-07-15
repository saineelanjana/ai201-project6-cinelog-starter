# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->
Used Claude Code throughout this PR cycle, mainly for:
- Drafting written responses to reviewer comments (Comments 1–6 above) — I gave the decision or fix I'd already made/wanted, and had it turn that into clear position/reasoning/tradeoff writeups.
- Debugging the rebase onto `main`: after `git rebase main`, it caught that the UUID-migration refactor had silently deleted the `WatchlistEntry` model entirely (no textual conflict, so git didn't flag it) and helped re-add it with UUID-typed `film_id`, plus fixed a stale docstring and an outdated integer test fixture.
- Cleaning up commit history: squashed the first five commits into one (`git merge --squash` + `git rebase --onto`, since interactive rebase isn't usable here) to get down to 6 commits, verifying tests still passed after.
- Drafting this PR description from the actual code/tests in the branch.

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
- `.gitignore` and `pr-response.md` had trivial add/add and content conflicts (duplicate `.venv`/`venv` entries, a stray blank line) — resolved by keeping both sides' real content and dropping the noise.
- The bigger issue wasn't a textual conflict at all: main's `refactor: migrate film IDs from integer to UUID` commit deleted the `WatchlistEntry` model entirely (it only existed pre-refactor), and no commit on this branch ever re-added it. Since git saw no overlapping lines to flag, the rebase completed "cleanly" but silently dropped `WatchlistEntry` from `models.py`.
- `services/watchlist_service.py` still had a docstring claiming `film_id` was an `int` ("pre-refactor"), and `tests/test_watchlist.py` used a bare integer (`99999`) as a fake film ID.
**How I resolved it:**
- Re-added `WatchlistEntry` to `models.py` with `film_id` typed as `db.String(36)` (matching the new `Film.id` and `CollectionEntry.film_id`), keeping the `public` default and `to_dict` shape unchanged.
- Updated the `add_to_watchlist` docstring to say `film_id (str): UUID of the film.` `db.session.get(Film, film_id)` in `add_to_watchlist` already worked with either ID type, so no logic change was needed there.
- Updated the nonexistent-film test to use a fake UUID string instead of an integer, so it stays meaningful post-refactor.
**How I verified no conflict remains:**
- Ran `grep` for `WatchlistEntry` across `models.py` before and after the fix to confirm it was actually missing, then present.
- Ran the full test suite (`pytest`) after rebasing: all 6 tests pass, including the deduplication and nonexistent-film tests against the UUID schema.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->

### Overview
Adds a watchlist feature: users can save films they want to watch later, view their list, and are protected from adding duplicates or nonexistent films.

- `WatchlistEntry` model (`models.py`) — tracks `user_id`, `film_id`, `date_added`, and `public`.
- `add_to_watchlist(user_id, film_id)` (`services/watchlist_service.py`) — adds a film, raising `FilmNotFoundError` for an unknown film and `AlreadyInWatchlistError` for a duplicate.
- `get_watchlist(user_id)` (`services/watchlist_service.py`) — returns a user's watchlist as a list of film dicts with watchlist metadata attached.
- `POST /watchlist/<user_id>/add` and `GET /watchlist/<user_id>` (`routes/watchlist/watchlist.py`) — endpoints exposing the above.

### Design decisions
1. **Visibility defaults to public** — `WatchlistEntry.public` defaults to `True`. A watchlist's value comes from being shareable/discoverable (friends can see what you plan to watch), so the common case is opt-out rather than opt-in. See Comment 4 above for the full reasoning and the privacy tradeoff this implies.
2. **Sort order is most-recently-added first** — `get_watchlist` orders by `date_added` descending rather than alphabetically by title. A watchlist is a queue of what to get to next, not a reference list to search through, so recency is the more useful default. See Comment 5 above for the full discussion, including where alphabetical would still make sense (the collection view).

### Manual testing
1. Install dependencies and activate the environment: `pip install -r requirements.txt` (or use the existing `.venv`).
2. Run the full test suite: `pytest`
   - Confirms adding a film to the watchlist succeeds and is retrievable.
   - Confirms adding a duplicate film raises `AlreadyInWatchlistError` and does not create a second row.
   - Confirms adding a nonexistent `film_id` raises `FilmNotFoundError`.
3. All 6 tests should pass, covering the watchlist add/dedupe/not-found flow end to end against the current UUID-based schema.