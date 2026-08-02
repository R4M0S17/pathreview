## Week 7 — Issue selection

***Issue link:*** https://github.com/ascherj/pathreview/issues/101
***Issue title:*** Add a "Copy link" button to share a public review summary
***Tier:*** [ ] Tier 1  [X] Tier 2  [ ] Tier 3

***Problem summary:***
The objective is to implement a shareable link feature that allows users to generate a read-only view of their review summary. Currently, there is no way to share these summaries publicly without requiring a login, and the requested solution requires generating a link that automatically expires after 30 days. This fix will involve modifying the frontend on ReviewPage.tsx, building out the shareService.ts, and adding the corresponding API routes in reviews.py to handle unauthenticated, time-limited token validation.

***Branch name:*** feat/101-copy-link-public-review-summary
***Setup confirmation:*** [X] App runs locally at localhost:5173
***Cohort ledger:*** [X] Issue added to cohort ledger

### Selection Notes
I selected this issue because it is a Tier 2 challenge that perfectly aligns with my goals to work across the full stack. It requires cross-module understanding between the React frontend and the Python backend routing, ensuring I get hands-on experience handling expiration logic and public/private route visibility.

---

## Check-in 1: Week 8 — Reproduction & solution planning

**Reproduction commit link:** [To be added after Week 9 implementation]

**Reproduction summary:**
Confirmed the issue by clicking "Share" on ReviewPage.tsx, which copies the current authenticated URL (`/reviews/{reviewId}`). When pasting this link in incognito mode, the app redirects to the login window instead of displaying the review—validating that no public-facing endpoint exists and all review routes require authentication.

**PLAN.md link:** [PLAN.md](PLAN.md)

**Blockers or open questions:**
None. All technical dependencies identified and mapped in PLAN.md; ready to begin Week 9 implementation starting with the data layer (`ReviewShare` model + migration).

---

## Check-in 2: Week 9 — PR submission

**PR link:** https://github.com/ascherj/pathreview/pull/470

**Branch:** feat/101-copy-link-public-review-summary

**What was built:** Implemented a token-based public share-link system — a new `ReviewShare` model, a `POST /reviews/{review_id}/share` endpoint that mints a cryptographically random, 30-day-expiring token, and a public `GET /reviews/shared/{share_token}` endpoint plus `/shared-review/:token` frontend route that render a sanitized, read-only review summary with no login required.

**Tests:** Added tests in `tests/unit/test_review_service.py` covering share-token creation, token reuse on repeat `POST` calls (returns the same unexpired token instead of minting a new one), ownership enforcement (404 for non-owners attempting to share someone else's review), and expired/invalid token lookups (both return a generic 404).

**Self-review:**
- [x] `make check` passes
- [x] `make test-unit` passes
