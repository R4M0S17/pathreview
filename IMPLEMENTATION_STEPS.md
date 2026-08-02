# IMPLEMENTATION_STEPS.md — Issue #101 (local only, not for GitHub)

Step-by-step, modular build order. Each step is small enough to compile/test in isolation before moving to the next. Follow the order — later steps depend on earlier ones existing.

## Git workflow for this feature

One commit per step, created only after that step's checkpoint passes — never commit code you haven't verified. This gives a bisectable, reviewable history instead of one opaque blob.

**Commit message convention** (matches this repo's existing style — see `git log`: `fix:`, `feat:`, `docs:`, `deps:`):
```
<type>: <imperative summary>

<optional body: why, not what — only if the "why" isn't obvious from the summary>
```
Types used across the steps below: `feat` (models, migration, schemas, service, routes, frontend code), `test` (test-only commits), `chore` (housekeeping, e.g. checkpoint fixes), `docs` (PR description).

**Rules for every commit in this feature:**
- Stage only the files that step named — never `git add -A`. Check `git status` before committing to confirm nothing unrelated is staged.
- Run the step's checkpoint first. A red checkpoint means no commit yet — fix, re-check, then commit.
- Write the commit body around *why* a non-obvious decision was made (e.g. "reuse unexpired token to avoid link-churn," "404 not 403 to avoid confirming existence") — these are the same judgment calls PLAN.md flagged as risks, and they're exactly what a reviewer will ask about later if they're not in the commit body.
- Never `--amend` a commit once you've moved to the next step. If a later step reveals a bug in an earlier one, fix it as its own small commit (or fold the fix into the current step if the earlier commit never left your machine) — don't rewrite history that might already be visible to a reviewer.
- Push only when explicitly told to. Work locally through all 13 steps first.

---

## Step 0 — Branch check ✅ DONE
You're already on `feat/101-copy-link-public-review-summary`. Good, stay there for the whole feature.

No commit for this step — nothing changed yet.

---

## Step 1 — `ReviewShare` model (data layer) ✅ DONE

**File:** `core/models/review_share.py` (new)

Model on `Review` (`core/models/review.py`) as the template — same `id: UUID(as_uuid=False)` pattern, same `datetime.utcnow` convention, same `Index`/`ForeignKey` style.

Fields:
- `id: str` — UUID PK, `default=lambda: str(uuid4())`
- `review_id: str` — UUID, `ForeignKey("reviews.id", ondelete="CASCADE")`, `nullable=False`, `index=True`
- `share_token: str` — `String(64)`, `unique=True`, `nullable=False`, `index=True`, `default=lambda: secrets.token_urlsafe(32)`
- `created_at: datetime` — `DateTime(timezone=True)`, `default=datetime.utcnow`
- `expires_at: datetime` — `DateTime(timezone=True)`, `nullable=False` (set explicitly at creation time in the service, not via column default — you need `created_at + timedelta(days=30)`, which a column default can't compute)

Relationship: add `review: Mapped["Review"] = relationship("Review")` (no `back_populates` needed unless you want `review.shares` — see Step 1b).

`__table_args__`: `Index("ix_review_shares_share_token", "share_token")`, `Index("ix_review_shares_review_id", "review_id")`.

Import `secrets` at the top — this is the token generator per PLAN.md risk #1 (cryptographically random, not sequential).

**Step 1b (optional but recommended):** in `core/models/review.py`, add
```python
shares: Mapped[list["ReviewShare"]] = relationship("ReviewShare", back_populates="review", cascade="all, delete-orphan")
```
and add the matching `back_populates="shares"` on the `ReviewShare.review` relationship. This makes the CASCADE delete behavior (PLAN.md risk: "Review deleted after share created") explicit and ORM-visible, not just DB-level.

**File:** `core/models/__init__.py`
Add `from core.models.review_share import ReviewShare` and add `"ReviewShare"` to `__all__`.

**Checkpoint:** `python -c "from core.models import ReviewShare"` should import cleanly with no circular-import errors.

**Commit:**
```
git add core/models/review_share.py core/models/__init__.py core/models/review.py
git commit -m "feat: add ReviewShare model for time-limited public review links

Stores a cryptographically random share_token per review with an
explicit expires_at, per issue #101."
```

---

## Step 2 — Alembic migration ✅ DONE

**File:** `alembic/versions/003_add_review_shares.py` (new)

Copy the header/structure of `alembic/versions/002_add_error_message_to_reviews.py` exactly:
- `revision = "003"`, `down_revision = "002"`, `branch_labels = None`, `depends_on = None`

`upgrade()`:
```python
op.create_table(
    "review_shares",
    sa.Column("id", postgresql.UUID(as_uuid=False), primary_key=True),
    sa.Column("review_id", postgresql.UUID(as_uuid=False), sa.ForeignKey("reviews.id", ondelete="CASCADE"), nullable=False),
    sa.Column("share_token", sa.String(64), nullable=False, unique=True),
    sa.Column("created_at", sa.DateTime(timezone=True), nullable=False),
    sa.Column("expires_at", sa.DateTime(timezone=True), nullable=False),
)
op.create_index("ix_review_shares_share_token", "review_shares", ["share_token"])
op.create_index("ix_review_shares_review_id", "review_shares", ["review_id"])
```
Note: import `from sqlalchemy.dialects import postgresql` at top (check how `001_initial_schema.py` imports the UUID type — mirror it exactly for consistency).

`downgrade()`:
```python
op.drop_index("ix_review_shares_review_id", table_name="review_shares")
op.drop_index("ix_review_shares_share_token", table_name="review_shares")
op.drop_table("review_shares")
```

**Checkpoint:** run `make migrate` (or the equivalent `alembic upgrade head` target in the Makefile) against your dev DB. Confirm the table exists: `\d review_shares` in psql, or equivalent. Then run `alembic downgrade -1` followed by `alembic upgrade head` once more to prove the downgrade path doesn't error — do this now, before any app code depends on the table, while rollback is cheap.

**Commit:**
```
git add alembic/versions/003_add_review_shares.py
git commit -m "feat: add review_shares migration

Verified upgrade/downgrade/upgrade cycle applies cleanly."
```

---

## Step 3 — Schemas ✅ DONE

**File:** `api/schemas/review.py`

Add near the bottom, following the existing `BaseModel` + `model_config = {"from_attributes": True}` pattern used by `ReviewResponse`:

```python
class ShareCreateResponse(BaseModel):
    share_token: str
    share_url: str
    expires_at: datetime


class SharedReviewResponse(BaseModel):
    sections: list[FeedbackSection] | None
    overall_score: float | None
    created_at: datetime

    model_config = {"from_attributes": True}
```

Deliberately exclude `id`, `profile_id`, `status`, `error_message` from `SharedReviewResponse` — this is the PLAN.md "no PII / no internal IDs" boundary (risk #2). `share_url` in `ShareCreateResponse` is a relative path (`/shared-review/{token}`); the frontend prefixes `window.location.origin`, matching PLAN.md §4.

**Checkpoint:** these are pure data classes — confirm they import cleanly (`python -c "from api.schemas.review import ShareCreateResponse, SharedReviewResponse"`) and that `mypy`/`make typecheck` doesn't flag them.

**Commit:**
```
git add api/schemas/review.py
git commit -m "feat: add ShareCreateResponse and SharedReviewResponse schemas

SharedReviewResponse intentionally omits id/profile_id/status/error_message
to avoid leaking internal identifiers through the public endpoint."
```

---

## Step 4 — Service functions ✅ DONE

**File:** `core/services/review_service.py`

Add two functions, following the existing `get_review`/`create_review` style (plain functions taking `db` first, using `select`/`and_`, `db.add`/`db.commit`/`db.refresh`):

```python
async def create_share_token(db, review_id: UUID, user_id: UUID) -> ReviewShare | None:
    """
    Create or reuse a share token for a review.
    Returns None if the review doesn't exist, isn't owned by user_id, or isn't complete.
    Reuses an existing unexpired token if one exists (PLAN.md: avoid link-churn).
    """
    review = await get_review(db=db, review_id=review_id, user_id=user_id)
    if not review or review.status != "complete":
        return None

    now = datetime.utcnow()
    stmt = select(ReviewShare).where(
        and_(ReviewShare.review_id == str(review_id), ReviewShare.expires_at > now)
    )
    result = await db.execute(stmt)
    existing = result.scalars().first()
    if existing:
        return existing

    share = ReviewShare(
        review_id=str(review_id),
        expires_at=now + timedelta(days=30),
    )
    db.add(share)
    await db.commit()
    await db.refresh(share)
    return share


async def get_review_by_share_token(db, share_token: str) -> Review | None:
    """
    Get a review via its share token. Returns None if token is unknown or expired.
    Caller (route) is responsible for turning None into 404.
    """
    now = datetime.utcnow()
    stmt = (
        select(Review)
        .join(ReviewShare, ReviewShare.review_id == Review.id)
        .where(and_(ReviewShare.share_token == share_token, ReviewShare.expires_at > now))
    )
    result = await db.execute(stmt)
    return result.scalars().first()
```

Add imports at the top: `from datetime import timedelta` (datetime already imported), `from core.models.review_share import ReviewShare`.

**Design decisions baked into this code (match PLAN.md exactly):**
- 409-worthy condition (`status != "complete"`) is checked in the service and signaled via `None` — the route turns that into the actual HTTP error. Keep this split: service returns data/`None`, routes decide status codes (matches existing `get_review` → route 404 pattern).
- Ownership check reuses `get_review`, which already joins `Profile.user_id == user_id` — so a non-owner gets `None` → the route should still respond 404 (not 403), matching PLAN.md's "don't confirm existence" edge case.
- Expiry (`expires_at > now`) is checked at fetch time on every call, not just once — matches PLAN.md risk #3.
- Single active token per review — the reuse branch is the whole "regeneration" policy from PLAN.md §5.

**Checkpoint:** write/extend `tests/unit/test_review_service.py` with a `TestReviewShareService` class mirroring the existing `mock_db_session`/`mock_review` fixture style. Cover: token created when none exists; existing unexpired token returned; `None` on non-complete review; `None` on non-owned review; `None`/valid on `get_review_by_share_token` for expired/valid tokens. Run `make test-unit` before moving on — do not proceed to routes until this passes.

**Commit (split in two, service then tests — keeps the diff reviewable and lets a reviewer see the contract before the proof):**
```
git add core/services/review_service.py
git commit -m "feat: add create_share_token and get_review_by_share_token services

Single active token per review (reuse-unexpired to avoid link-churn);
ownership + completeness checks return None so routes control status codes."

git add tests/unit/test_review_service.py
git commit -m "test: cover ReviewShare creation, reuse, ownership, and expiry paths"
```

---

## Step 5 — Routes ✅ DONE

**File:** `api/routes/reviews.py`

Two new endpoints, following the existing try/except/log/HTTPException shape used by every other handler in this file.

**5a — Authenticated generation, placed after `get_review_endpoint`:**
```python
@router.post("/{review_id}/share", response_model=ShareCreateResponse)
async def create_share_endpoint(
    review_id: UUID,
    current_user: User = Depends(get_current_user),
    db=Depends(get_db),
):
    try:
        share = await create_share_token(db=db, review_id=review_id, user_id=current_user.id)

        if not share:
            # Distinguish "not found/not owned" from "not complete" for a useful error,
            # but keep the not-found case generic (no existence leak).
            review = await get_review(db=db, review_id=review_id, user_id=current_user.id)
            if not review:
                raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Review not found")
            raise HTTPException(status_code=status.HTTP_409_CONFLICT, detail="Review is not yet complete")

        return ShareCreateResponse(
            share_token=share.share_token,
            share_url=f"/shared-review/{share.share_token}",
            expires_at=share.expires_at,
        )
    except HTTPException:
        raise
    except Exception as exc:
        log.error("create_share_error", error=str(exc))
        raise HTTPException(status_code=status.HTTP_500_INTERNAL_SERVER_ERROR, detail="Failed to create share link")
```

**5b — Public fetch, placed as its own block, NOT nested under `/reviews/{review_id}` prefix conflicts.** Careful: your router prefix is already `/reviews`, and FastAPI matches routes in declaration order. `GET /reviews/shared/{share_token}` must be declared *before* `GET /{review_id}` would otherwise be fine since `/shared/...` doesn't collide with a UUID path param — but double check by declaring it right after `list_reviews_endpoint` and before `get_review_status`, and confirm with the OpenAPI docs (`/docs`) that `shared` isn't being swallowed as a `review_id` path value.

```python
@router.get("/shared/{share_token}", response_model=SharedReviewResponse)
async def get_shared_review_endpoint(
    share_token: str,
    db=Depends(get_db),
):
    try:
        review = await get_review_by_share_token(db=db, share_token=share_token)

        if not review:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Share link not found")

        return SharedReviewResponse.model_validate(review)
    except HTTPException:
        raise
    except Exception as exc:
        log.error("get_shared_review_error", error=str(exc))
        raise HTTPException(status_code=status.HTTP_500_INTERNAL_SERVER_ERROR, detail="Failed to load shared review")
```

No `current_user` dependency here — that's what makes it public. Do not import or reference `get_current_user` in this handler.

Update the imports at the top of `reviews.py`:
```python
from api.schemas.review import (
    ReviewCreate, ReviewResponse, ReviewListResponse,
    ShareCreateResponse, SharedReviewResponse,
)
from core.services.review_service import (
    create_review, get_review, list_reviews, process_review,
    create_share_token, get_review_by_share_token,
)
```

PLAN.md picked 410 Gone for expired tokens as an option but the recommendation in §6 is to keep it generic — this guide follows the §6 recommendation (404 for both unknown and expired) to avoid leaking "this token used to exist." If you'd rather return 410 for expired specifically, that requires `get_review_by_share_token` to distinguish "not found" from "found but expired" (return a sentinel/raise instead of `None`) — a deliberate scope decision, make it consciously, don't default into it.

**Checkpoint:** start the API (`make run` or equivalent), hit `POST /reviews/{id}/share` with a valid bearer token via curl/Postman against a `complete` review you own, confirm you get back `share_token`/`share_url`/`expires_at`. Then hit `GET /reviews/shared/{token}` with *no* auth header and confirm it returns the trimmed payload. Then hit it with a garbage token and confirm 404.

**Commit:**
```
git add api/routes/reviews.py
git commit -m "feat: add share generation and public shared-review endpoints

POST /reviews/{id}/share (auth, 404 on not-owned/not-found, 409 if not
complete). GET /reviews/shared/{token} (public, 404 for unknown or
expired token — generic to avoid confirming a token ever existed)."
```

---

## Step 6 — Frontend types ✅ DONE

**File:** `frontend/src/types/index.ts`

Add:
```typescript
export interface ShareResponse {
  share_token: string
  share_url: string
  expires_at: string
}

export interface SharedReview {
  sections?: FeedbackSection[]
  overall_score?: number
  created_at: string
}
```

**Checkpoint:** `tsc --noEmit` (or your usual frontend typecheck command) passes with no errors from this file.

**Commit:**
```
git add frontend/src/types/index.ts
git commit -m "feat: add ShareResponse and SharedReview types"
```

---

## Step 7 — Frontend API client methods ✅ DONE

**File:** `frontend/src/services/api.ts`

Add two methods to `ApiClient`, using the existing `this.request<T>()` helper (same as `getReview`/`createReview`):

```typescript
async createShareLink(reviewId: string): Promise<ShareResponse> {
  return this.request(`/reviews/${reviewId}/share`, { method: 'POST' })
}

async getSharedReview(token: string): Promise<SharedReview> {
  return this.request(`/reviews/shared/${token}`)
}
```

Update the top import: `import { AuthResponse, Profile, Review, ReviewListResponse, ShareResponse, SharedReview } from '../types'`.

Note: `this.request()` always attaches the auth header if a token exists in `localStorage` (`getAuthHeader()`), but it degrades to `{}` if there's no token — so calling `getSharedReview` from an incognito/logged-out session is safe and won't be rejected by the backend (the route has no `get_current_user` dependency). Verify there's no global fetch interceptor elsewhere that redirects to `/login` on a 401 — grep for `401` across `frontend/src` before trusting this; if a global interceptor exists, it needs a bypass for this one call path (PLAN.md §6 edge case).

**Checkpoint:** confirmed no 401-redirect interceptor blocks the public call path; `tsc --noEmit` passes.

**Commit:**
```
git add frontend/src/services/api.ts
git commit -m "feat: add createShareLink and getSharedReview to ApiClient"
```

---

## Step 8 — `shareService.ts` (thin wrapper, per PLAN.md §2/§4) ✅ DONE

**File:** `frontend/src/services/shareService.ts` (new)

PLAN.md specifies this as a separate module from `api.ts`, presumably to keep `ReviewPage.tsx`'s import surface small and to isolate the "generate + build absolute URL" concern from raw API plumbing:

```typescript
import { apiClient } from './api'

export async function generateShareLink(reviewId: string): Promise<{ shareUrl: string; expiresAt: string }> {
  const { share_url, expires_at } = await apiClient.createShareLink(reviewId)
  return {
    shareUrl: `${window.location.origin}${share_url}`,
    expiresAt: expires_at,
  }
}

export async function getSharedReview(token: string) {
  return apiClient.getSharedReview(token)
}
```

`generateShareLink` is where the relative `/shared-review/{token}` path becomes an absolute, copyable URL — matches PLAN.md §4's `ReviewPage.tsx handleShare()` output spec.

**Checkpoint:** `tsc --noEmit` passes.

**Commit:**
```
git add frontend/src/services/shareService.ts
git commit -m "feat: add shareService wrapping share-link generation and fetch"
```

---

## Step 9 — Update `ReviewPage.tsx` ✅ DONE

**File:** `frontend/src/pages/ReviewPage.tsx`

Replace `handleShare` (lines 32–37) with an async version that calls the new service, and add basic error handling consistent with the existing `fetchError` state pattern already in this component:

```typescript
const handleShare = async () => {
  if (!reviewId) return
  try {
    const { shareUrl } = await generateShareLink(reviewId)
    await navigator.clipboard.writeText(shareUrl)
    alert('Share link copied to clipboard!')
  } catch (err) {
    alert(err instanceof Error ? err.message : 'Failed to generate share link')
  }
}
```

Add the import: `import { generateShareLink } from '../services/shareService'`.

Keep the `alert()` calls consistent with the existing codebase style (the current `handleShare` already uses `alert`) — don't introduce a toast library for this change; that's scope creep not requested by PLAN.md.

Do not touch `handleExport` or anything else in this file.

**Checkpoint:** with backend running, load a completed review in the browser, click "Share," confirm the alert fires and the clipboard contains an absolute URL like `http://localhost:5173/shared-review/<token>` (or whatever the frontend origin is).

**Commit:**
```
git add frontend/src/pages/ReviewPage.tsx
git commit -m "feat: wire ReviewPage Share button to generate a public link

Replaces copying the login-gated window.location.href with a
30-day, token-based public URL from shareService."
```

---

## Step 10 — `SharedReviewPage.tsx` (new, read-only public view) ✅ DONE

**File:** `frontend/src/pages/SharedReviewPage.tsx` (new)

Model on `ReviewPage.tsx`'s completed-state render block (lines 109–153), but strip anything auth-gated or interactive:
- No Share button, no Export button, no "Back to Dashboard" nav (that route is inside `ProtectedRoute`).
- No `useReviewStatus` polling hook — a shared review is always `complete` by construction (only complete reviews get tokens per Step 5a's 409 check), so no polling/pending/failed states are needed here.
- Fetch directly via `getSharedReview(token)` from `shareService.ts` on mount; render loading/error/success states locally with `useState`.

```typescript
import React, { useEffect, useState } from 'react'
import { useParams } from 'react-router-dom'
import { getSharedReview } from '../services/shareService'
import { ReviewSection } from '../components/ReviewSection'
import { SharedReview } from '../types'

export const SharedReviewPage: React.FC = () => {
  const { shareToken } = useParams<{ shareToken: string }>()
  const [review, setReview] = useState<SharedReview | null>(null)
  const [error, setError] = useState('')
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    if (!shareToken) return
    getSharedReview(shareToken)
      .then(setReview)
      .catch((err) => setError(err instanceof Error ? err.message : 'This link is invalid or has expired'))
      .finally(() => setLoading(false))
  }, [shareToken])

  // loading / error / success JSX — reuse ReviewSection for section rendering,
  // reuse the same Tailwind shell (`min-h-screen bg-gray-50`, `max-w-4xl mx-auto ...`)
  // as ReviewPage.tsx for visual consistency, but with a static "Portfolio Review"
  // header (no action buttons) instead of the interactive one.
}
```

Fill in the JSX body once the earlier steps are verified working — the PLAN.md-mandated behavior is: friendly "This link has expired" / "Share link not found" message on error (no raw error dump), otherwise render `overall_score` + `sections` exactly like `ReviewPage.tsx` does today.

`ReviewSection` is already a shared, presentational component (no auth/edit logic inside it per its usage in `ReviewPage.tsx:148`) — confirm that by reading `frontend/src/components/ReviewSection.tsx` before reusing it; if it secretly depends on anything auth-related, that's a dependency to strip, not a variable to route around.

**Checkpoint:** the JSX body is fully written (not left as the placeholder comment above) and renders loading/error/success states correctly when manually pointed at a real token via browser dev tools.

**Commit:**
```
git add frontend/src/pages/SharedReviewPage.tsx
git commit -m "feat: add SharedReviewPage read-only public view

No auth, no polling (shared reviews are always complete), no
edit/export/re-share actions — matches the public-view constraints
in issue #101."
```

---

## Step 11 — Route registration ✅ DONE

**File:** `frontend/src/App.tsx`

Add the import: `import { SharedReviewPage } from './pages/SharedReviewPage'`.

Add the route **outside** any `<ProtectedRoute>` wrapper — sibling to `/login` and `/register`, not sibling to `/reviews/:reviewId`:

```tsx
<Route path="/shared-review/:shareToken" element={<SharedReviewPage />} />
```

Double check `NavBar` (rendered unconditionally above `<Routes>`) doesn't itself force an auth redirect or hide itself in a way that breaks unauthenticated rendering of this page — skim `frontend/src/components/NavBar.tsx` for any redirect-on-no-user logic before considering this step done.

**Checkpoint:** open an incognito window, paste a previously-copied `/shared-review/<token>` URL directly, confirm it loads without any redirect to `/login`.

**Commit:**
```
git add frontend/src/App.tsx
git commit -m "feat: register /shared-review/:shareToken as a public route

Placed outside ProtectedRoute; verified NavBar has no redirect-on-
no-user logic that would block this."
```

---

## Step 12 — End-to-end verification (PLAN.md §3 step 5)

Run through, in order, noting failures rather than fixing as you go (fix after the full pass so you know the total scope of what's broken):

1. Log in, create/select a profile, run a review to `complete`.
2. Click Share → confirm clipboard has an absolute `/shared-review/{token}` URL.
3. Open that URL in an incognito window → confirm it renders the review with no login prompt, no Share/Export buttons.
4. Manually expire a token (`UPDATE review_shares SET expires_at = now() - interval '1 day' WHERE share_token = '...'`) → reload the incognito tab → confirm a friendly expired message, not a raw 404 JSON or a login redirect.
5. Try a made-up token string in the URL → confirm same friendly "not found" behavior.
6. As a second user, attempt `POST /reviews/{review_id}/share` on the first user's review_id → confirm 404 (not 403, not 200).
7. Attempt to generate a share link for a review still in `pending`/`processing` → confirm 409.
8. Click Share twice on the same review without letting the first token expire → confirm the second click returns the *same* `share_token` (reuse behavior), not a new one.

No commit for this step unless it surfaces a bug — if it does, fix it as its own small commit (`fix: ...`) scoped to just the broken file(s), then re-run the failed scenario before moving on.

---

## Step 13 — Tests, typecheck, and integration coverage

- Extend `tests/unit/test_review_service.py` (already covered in Step 4) plus add route-level tests if this repo has an existing `tests/integration/test_reviews_routes.py`-style file — check `tests/integration/` for the existing pattern before creating a new file.
- Run `make check` (lint + format + typecheck) across both backend and frontend before considering the branch done. Fix everything it flags — don't `--no-verify` past it.

**Commit (only if this step adds/changes anything beyond what Step 4 already committed):**
```
git add tests/integration/test_reviews_routes.py   # if created
git commit -m "test: add integration coverage for share endpoints"
```

---

## Step 14 — Push and open the PR

1. Push the branch: `git push -u origin feat/101-copy-link-public-review-summary` (ask before pushing if you haven't already agreed to this with the user in the current session).
2. Update the existing untracked `PR_DESCRIPTION.md` in the repo root to reflect what actually shipped (not what was planned) — summary, the two new endpoints, the frontend route, the 30-day/no-login/read-only behavior, and a "Test plan" checklist mirroring Step 12's 8 scenarios.
3. Open the PR with `gh pr create`, using `PR_DESCRIPTION.md`'s content as the body and referencing the issue (`Closes #101`) so it auto-links and auto-closes on merge.
4. Double check the PR diff on GitHub (or via `gh pr diff`) shows exactly the ~13 commits from Steps 1–13, each with a clear scope — this is the payoff of committing per-step instead of squashing as you go.

**Commit:** none — `PR_DESCRIPTION.md` changes can be included in the PR body directly, or committed as a final `docs: finalize PR description for issue #101` commit if you want it tracked in history too.
