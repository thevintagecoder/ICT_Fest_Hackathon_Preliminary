# bug_report.md — ICT Fest Hackathon Preliminary

Source of truth: problem statement business rules and API contract. The grader black-box tests the API behavior, so each fix below preserves the existing endpoint paths, JSON field names, and documented status/error codes.

## 1. `app/timeutils.py` — timezone offsets are dropped instead of converted to UTC
- **Location:** `parse_input_datetime`
- **Bug:** Offset-aware datetimes call `dt.replace(tzinfo=None)`, which removes the timezone without changing the clock time. Example: `2026-07-10T10:00:00+06:00` becomes `2026-07-10 10:00 UTC`, but it should become `2026-07-10 04:00 UTC`.
- **Why wrong:** The contract says offsets must be converted to UTC before storage/comparison; naive input is UTC.
- **Fix:** Use `dt.astimezone(timezone.utc).replace(tzinfo=None)` for aware datetimes; use naive UTC unchanged.

## 2. `app/auth.py` — access token expiry is 900 minutes, not 900 seconds
- **Location:** `create_access_token`
- **Bug:** `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)` makes a 15-hour lifetime.
- **Why wrong:** Access tokens must expire in exactly 900 seconds.
- **Fix:** Use `timedelta(seconds=ACCESS_TOKEN_EXPIRE_MINUTES * 60)` or `timedelta(seconds=900)`.

## 3. `app/auth.py` / `app/routers/auth.py` — logout does not invalidate access token
- **Location:** `revoke_access_token`, `get_token_payload`, `logout`
- **Bug:** Logout stores the token `jti`, but validation checks whether `sub` is in `_revoked_tokens`.
- **Why wrong:** The presented access token should immediately fail after logout.
- **Fix:** Check `payload.get("jti") in _revoked_tokens`.

## 4. `app/routers/auth.py` — refresh token is not single-use
- **Location:** `refresh`
- **Bug:** The refresh endpoint accepts the same refresh token repeatedly.
- **Why wrong:** Refresh tokens must rotate and the presented refresh token must be invalidated after one use.
- **Fix:** Maintain a used/blacklisted refresh `jti` set; reject reuse with 401.

## 5. `app/routers/auth.py` — duplicate username returns existing user instead of 409
- **Location:** `register`
- **Bug:** When a username already exists in an org, the code returns the existing user.
- **Why wrong:** Duplicate username within an org must return 409 `USERNAME_TAKEN`.
- **Fix:** Raise `AppError("USERNAME_TAKEN", 409, ...)` when duplicate user exists.

## 6. `app/routers/bookings.py` — start-time validation allows a 5-minute grace window
- **Location:** `create_booking`
- **Bug:** Uses `start <= now - timedelta(seconds=300)`.
- **Why wrong:** Start time must be strictly in the future at request time; no grace window.
- **Fix:** Reject when `start <= now`.

## 7. `app/routers/bookings.py` — invalid booking duration not fully rejected
- **Location:** `create_booking`
- **Bug:** It does not explicitly reject `end_time <= start_time` or durations under 1 hour.
- **Why wrong:** Duration must be whole hours, minimum 1, maximum 8, and end must be strictly after start.
- **Fix:** Reject if `duration_seconds <= 0`, `duration_seconds % 3600 != 0`, or hours not in `[1, 8]`.

## 8. `app/routers/bookings.py` — overlap logic rejects valid back-to-back bookings
- **Location:** `_has_conflict`
- **Bug:** Uses `existing.start <= new.end and new.start <= existing.end`.
- **Why wrong:** Back-to-back bookings are allowed. Overlap is only `existing.start < new.end and new.start < existing.end`.
- **Fix:** Use strict `<` comparisons.

## 9. `app/routers/bookings.py` — double-booking is not concurrency-safe
- **Location:** `create_booking`, `_has_conflict`
- **Bug:** Conflict check and insert are not atomic. Concurrent requests can both see no conflict and both commit.
- **Why wrong:** No double-booking must hold under concurrent requests.
- **Fix:** Guard booking creation with a process-wide lock around rate-limit/check/quota/reference/insert/commit, or enforce a stronger DB-level transaction strategy compatible with SQLite.

## 10. `app/routers/bookings.py` — quota is not concurrency-safe
- **Location:** `_check_quota`, `create_booking`
- **Bug:** Count is checked before insert without locking.
- **Why wrong:** A member may hold at most 3 confirmed bookings in `(now, now + 24h]`, even under concurrent requests.
- **Fix:** Protect quota check and booking insert with the same creation lock.

## 11. `app/services/ratelimit.py` — rate limiter is not concurrency-safe
- **Location:** `check_booking_rate_limit`
- **Bug:** The in-memory list is read/modified without a lock, with an artificial pause.
- **Why wrong:** POST `/bookings` must be limited to 20 rolling-60-second requests per user under concurrent requests.
- **Fix:** Wrap bucket read/write in a lock; count all create-booking attempts.

## 12. `app/services/reference.py` / `app/models.py` — reference codes can duplicate under concurrency
- **Location:** `next_reference_code`; `Booking.reference_code`
- **Bug:** Global counter read/sleep/increment has no lock, and DB column is not unique.
- **Why wrong:** Every booking reference code must be unique, including concurrent creation.
- **Fix:** Use a lock or UUID/DB-backed unique generation; add `unique=True` to `Booking.reference_code`.

## 13. `app/routers/bookings.py` — GET `/bookings` ordering is wrong
- **Location:** `list_bookings`
- **Bug:** Sorts by `start_time.desc()`.
- **Why wrong:** Must sort ascending by start time, ties by ascending id.
- **Fix:** `order_by(Booking.start_time.asc(), Booking.id.asc())`.

## 14. `app/routers/bookings.py` — GET `/bookings` pagination offset is wrong
- **Location:** `list_bookings`
- **Bug:** Uses `offset(page * limit)`.
- **Why wrong:** Page 1 skips the first `limit` items.
- **Fix:** Use `offset((page - 1) * limit)`.

## 15. `app/routers/bookings.py` — GET `/bookings` ignores requested limit
- **Location:** `list_bookings`
- **Bug:** Uses `.limit(10)` always.
- **Why wrong:** Endpoint must honor `limit`, default 10, max 100.
- **Fix:** Use `.limit(limit)` after validating/clamping via FastAPI.

## 16. `app/routers/bookings.py` — member can read another member’s same-org booking
- **Location:** `get_booking`
- **Bug:** Query scopes by room org, but does not enforce member ownership.
- **Why wrong:** Members may read only their own bookings; another member’s booking id behaves as 404 `BOOKING_NOT_FOUND`.
- **Fix:** After loading, if current user is member and `booking.user_id != current_user.id`, raise 404 `BOOKING_NOT_FOUND`.

## 17. `app/routers/bookings.py` — single booking response returns wrong `start_time`
- **Location:** `get_booking`
- **Bug:** Overwrites `response["start_time"]` with `booking.created_at`.
- **Why wrong:** Response must show the booking’s actual start time.
- **Fix:** Remove the overwrite and use serializer output.

## 18. `app/routers/bookings.py` — refund threshold logic is wrong
- **Location:** `cancel_booking`
- **Bug:** Uses `notice_hours > 48` for 100%; gives 50% for notices under 24 hours.
- **Why wrong:** `notice >= 48h` → 100%, `24h <= notice < 48h` → 50%, `<24h` → 0%.
- **Fix:** Compare timedeltas directly: `>= timedelta(hours=48)`, `>= timedelta(hours=24)`, else 0.

## 19. `app/routers/bookings.py` / `app/services/refunds.py` — refund rounding and response/log mismatch
- **Location:** `cancel_booking`, `log_refund`
- **Bug:** Response uses Python `round`, while log uses float dollars and `int()`, so half-cents may round incorrectly and response/log can differ.
- **Why wrong:** Half-cents must round up; response refund amount must equal the RefundLog amount.
- **Fix:** Compute integer cents once, using half-up integer math, and store/return the same value.

## 20. `app/routers/bookings.py` / `app/services/refunds.py` / `app/models.py` — concurrent cancellation can create duplicate refund logs
- **Location:** `cancel_booking`, `log_refund`, `RefundLog.booking_id`
- **Bug:** Status check, refund log creation, and cancellation update are not atomic; no unique constraint on `RefundLog.booking_id`.
- **Why wrong:** Already-cancelled booking must return 409, and a cancelled booking has exactly one RefundLog.
- **Fix:** Protect cancellation with a lock/transaction; set booking cancelled and add one refund log in one atomic section; optionally make `RefundLog.booking_id` unique.

## 21. `app/routers/rooms.py` / `app/services/stats.py` — room stats are stale/inconsistent
- **Location:** `room_stats`, `stats.record_create`, `stats.record_cancel`, `stats.get`
- **Bug:** Stats are kept in an unsynchronized in-memory dictionary; races lose updates and data may reset.
- **Why wrong:** Stats must always match confirmed bookings, including after concurrent bursts.
- **Fix:** Query confirmed bookings from DB on every `/rooms/{id}/stats` request, or lock and rebuild safely. DB query is simplest and most correct.

## 22. `app/routers/rooms.py` — availability cache can become stale after cancellation
- **Location:** `availability`; cancellation path
- **Bug:** Availability result is cached; cancellation does not invalidate the availability key.
- **Why wrong:** Availability must reflect current state immediately.
- **Fix:** Remove/bypass availability cache, or invalidate exact date key on booking creation and cancellation.

## 23. `app/routers/admin.py` / `app/routers/bookings.py` — usage report cache can become stale after creation
- **Location:** `usage_report`, `create_booking`, `cancel_booking`
- **Bug:** Usage report is cached. Cancellation invalidates report cache, but creation only invalidates availability.
- **Why wrong:** Usage report must reflect current state immediately.
- **Fix:** Remove/bypass report cache, or invalidate report cache on every create and cancel.

## 24. `app/services/export.py` — cross-org booking leak in admin export
- **Location:** `generate_export`
- **Bug:** With `include_all=true` and a `room_id`, it calls `fetch_bookings_raw(db, room_id)`, which filters only by `room_id`, not org.
- **Why wrong:** Admins may only read data in their own organization; cross-org IDs should behave as non-existent.
- **Fix:** Always use the org-scoped query path; do not call the raw unscoped fetcher from API code.

## 25. `app/services/notifications.py` — possible deadlock between create and cancel notifications
- **Location:** `notify_created`, `notify_cancelled`
- **Bug:** One function locks `email` then `audit`; the other locks `audit` then `email`.
- **Why wrong:** Concurrent valid requests may hang, violating liveness.
- **Fix:** Acquire locks in the same order in both functions, or avoid nested locks.

