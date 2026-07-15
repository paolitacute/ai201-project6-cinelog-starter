# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I asked Ai how to do a rebase and solve a conflict.

## Comment 1 — Rename

**What I did:**
* Renamed `save_to_watchlist()` to `add_to_watchlist()` inside `services/watchlist_service.py`.
* Updated the known call site in `routes/watchlist/watchlist.py`.
* Performed a project-wide global text search for `save_to_watchlist` in my editor to confirm there were no other hidden or dangling references. 
* Integrated the new deduplication logic directly into the renamed function, mirroring the `CollectionEntry` query pattern to raise a custom `AlreadyInWatchlistError`.

**How I verified:**
* I verified the logic by writing a new set of Pytest tests, using `test_add_to_collection_nonexistent_film_raises` as my direct model for the fixture setup and assertion structure.
* To ensure the test ran correctly without database integrity or setup issues, I extracted the `app` and `sample_user` fixtures into a root `conftest.py` file. 
* I ran the test suite to confirm that attempting to add a duplicate film correctly triggers the application-level deduplication block, and adding a non-existent film still properly raises the `FilmNotFoundError`.

## Comment 2 — Deduplication

**What I did:**
* Inserted an explicit database query into `add_to_watchlist()` in `services/watchlist_service.py` to check for existing entries (`WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()`).
* Followed the exact error-handling pattern established in `add_to_collection()`, raising a custom `AlreadyInWatchlistError` if a match is found to gracefully reject the duplicate at the application level.

**How I verified:**
* Authored a new test case following the same fixture structure (`app`, `sample_user`) used in the collection service tests.
* Ran the test suite to ensure that attempting to insert a duplicate film successfully triggers the `AlreadyInWatchlistError` rather than causing a database integrity exception, while verifying that fresh inserts still commit correctly.

## Comment 3 — Missing test

**What I did:**
* Created a new test file at `tests/test_watchlist.py`.
* Reviewed `tests/test_collection.py` to match the exact structure and assertions of `test_add_to_collection_nonexistent_film_raises`.
* Authored the equivalent `test_add_to_watchlist_nonexistent_film_raises` test, checking that a `FilmNotFoundError` is properly raised when attempting to add a dummy UUID (like `"00000000-0000-0000-0000-000000000000"`).
* Extracted the `app` and `sample_user` fixtures into a shared `conftest.py` file at the root of the `tests/` directory to ensure both test files could access the setup without raising `fixture 'app' not found` errors.

**How I verified:**
* Ran the specific test file via Pytest to confirm the runner successfully discovered the shared global fixtures. 
* Verified the test passes, confirming the function correctly halts and raises the expected domain error rather than attempting a database commit that would throw an integrity exception.

## Comment 4 — Default visibility

**My position:** 
I strongly advocate for keeping `public=True` as the default state for user watchlists.

**Reasoning:** 
We are optimizing for social discovery and network effects. In a film-centric application, a core driver of user engagement and retention is seeing what peers are anticipating or recommending. If we default to private, we introduce significant friction—users rarely dig into settings to flip a visibility toggle. By defaulting to public, we encourage organic sharing, profile exploration, and a more vibrant, connected community ecosystem right out of the gate.

**Tradeoff acknowledged:** 
I recognize the clear tradeoff here: we are prioritizing community engagement over maximum default privacy. The risk is that a user might add a film to their watchlist without realizing their profile is visible to others. By choosing `public=True`, we are accepting that risk in favor of growth, which means we must take on the responsibility of ensuring the frontend UI clearly telegraphs the list's public status (e.g., using a visible globe icon or a clear "Public" badge) so users are informed without needing to explicitly manage permissions.

## Comment 5 — Sort order

**My position:**
I agree with changing the default sort order to "date added" (descending) so that the newest additions appear at the top of the list.

**Reasoning:**
A watchlist functions primarily as an intent queue rather than a static library. When users open their watchlist, they are typically trying to answer "what should I watch tonight?" based on a recent recommendation, a trailer they just saw, or a fleeting impulse. Alphabetical sorting scatters these recent, high-interest additions randomly throughout the list, creating unnecessary friction and forcing the user to hunt for the title they added just yesterday. 

**Engagement with reviewer's point:**
You are completely right that "most users want to see what they added recently." I initially chose alphabetical sorting because I was carrying over the mental model from the `Collection` feature, where a user is browsing a long-term inventory they already own. However, your point highlights that a 'Watchlist' is fundamentally different—it is inherently chronological and tied to recent user interest. I will update the query to sort by `WatchlistEntry.created_at.desc()` and add a note to the PR description documenting this distinction between lists

## Comment 6 — Rebase

**What conflicted:**
The recent refactor on `main` migrating `film_id` from integers to UUIDs clashed with the newly added watchlist logic in `services/watchlist_service.py`, which was originally authored expecting integer IDs.

**How I resolved it:**
I ran an interactive rebase against `origin/main`. When the conflict triggered, I manually updated `services/watchlist_service.py` to expect UUID strings instead of integers. I updated the docstrings to reflect the new `str` type and ensured the arguments passed to `db.session.get()` and `WatchlistEntry.query.filter_by()` conformed to the new UUID standard. 

**How I verified no conflict remains:**
After staging the resolved files and completing the rebase with `git rebase --continue`, I ran `git log --oneline --graph` to verify that the branch history is completely linear. The feature commits now sit cleanly on top of the latest `main` without any merge commits remaining in the history.

## PR Description
The watchlist feature allows users to queue films they plan to watch. I implemented two key design decisions: the default visibility is set to public=True to encourage social discovery, and the default sort order is "date added" (descending) so users see their most recent additions first.

To manually test:

Log in to your account.

Navigate to a film's page and click "Add to Watchlist".

Go to your Watchlist profile to verify the film appears at the top.

Click "Add to Watchlist" on the same film again to ensure deduplication blocks it.