# Adobe API gotchas

Things about Adobe's APIs that cost real debugging time, written down so they
only cost it once. Each entry says what the behaviour is, how it shows up, and
what to do instead.

Nothing here is in Adobe's documentation. All of it was established against a
live tenant.

---

## Audit events API

`https://platform.adobe.io/data/foundation/audit/events`

### The 90-day window drifts. Recompute it for every request.

Adobe measures "the minimum start date cannot be less than 90 days ago" from
the clock **at the moment each request lands** — not from when your run
started.

Calculate `now - 90 days` once at the top of a long job and that fixed start
date slowly slides out of range. Partway through, every remaining request
starts returning:

```
HTTP 422  {"detail":"The minimum start date cannot be less than 90 days ago."}
```

In `external_access_audit.py` this hit at **request 86 of 2,286**. Everything
before it succeeded; everything after it failed.

**Do this instead** — recompute per item, with a margin:

```python
WINDOW_MARGIN = timedelta(hours=2)
for account in accounts:
    b = datetime.now(timezone.utc)
    a = b - timedelta(days=days) + WINDOW_MARGIN
    ...
```

Verified over 219 consecutive requests with no drift.

### The user filter is `user==`, and it is CASE-SENSITIVE

Two traps in one field.

* The parameter is `user`, **not** `userEmail`. Using `userEmail==` returns a
  thoroughly misleading `422 "More than 2 timestamp filters are not allowed."`
* The audit log stores addresses **lower-cased**, while the Admin Console
  preserves whatever case an account was created with. So a roster address
  like `Firstname.Lastname@example.com` returns **zero events** for somebody
  who is in the platform every day.

```python
params = [("property", f"user=={email.lower()}")]   # always .lower()
```

Proven: the same address returned 1,000 events lower-cased and 0 upper-cased.

### Per-user `totalElements` is exact, even though pages cap at 1,000

A page returns at most 1,000 events, but `page.totalElements` on a
single-user query is the server's own full count for that user.

That makes **one query per email** the accurate way to answer "has this person
used the platform?" — it cannot be fooled by the page cap, unlike sweeping the
whole log and tallying locally, which silently loses the overflow of any
window still at the cap after being split.

Too slow for a whole org. Right for any list a decision rests on.

### Why these three matter more than they look

All three failure modes return **"no activity"** rather than an error the
caller notices. On an access-review tool that silently becomes *"delete this
person"*.

The guard worth building: an account only counts as unused when **every**
sandbox actually answered. Anything less is a distinct `UNKNOWN` verdict that
is explicitly not safe to act on. That distinction caught 242 wrongly-listed
accounts while the window-drift bug above was still live — including a daily
user of the platform.

### Other practicalities

* **Retention is 90 days, hard.** Any wider window is refused outright, so
  "no usage" can only ever mean "none in 90 days", never "never".
* **A sandbox that 403s is not clean.** Sandboxes exist that a given
  credential cannot read at all. Report them as unreadable; silence from a
  door you could not open is not an all-clear.
* **CJA keeps its own separate audit trail.** It is not in this API. A user
  whose work is entirely in Customer Journey Analytics shows zero activity
  here and may be perfectly active.

---

## User Management API (Admin Console)

`https://usermanagement.adobe.io/v2/usermanagement`

* **Org-level: send no sandbox header.** `x-sandbox-name` must be omitted, not
  sent empty.
* `/users/{orgId}/{page}` gives every account with `groups` (its product
  profiles), `type` (`federatedID` / `adobeID` / `enterpriseID`), `status`,
  `domain` and `adminRoles`. Page until `lastPage` is true.
* `/groups/{orgId}/{page}` gives the profile catalogue, and crucially
  `productName` — which is the only way to tell which product a profile like
  `"Read Access"` actually belongs to.
* **`type` is a security signal.** `adobeID` is a personal Adobe account the
  corporate directory does not control, as opposed to a federated SSO
  identity. Worth surfacing on any access review.
* **A user's `groups` are DIRECT memberships only.** Access inherited by
  nesting a user group inside a product profile is not resolved by this API,
  so any entitlement count derived from it is a **floor, not a total**.

---

## Schema Registry

* **Adobe hex-ids every tenant-created custom class**, so a class id ending in
  a long hex blob says nothing about whether a schema was hand-modelled or
  auto-generated. Use `schemaType` (`standard` vs `adhoc`) and the namespace
  key (`meta:tenantNamespace` vs `meta:datasetNamespace`) instead.
  See `RELEASE_NOTES.md` under data_dictionary v3.4.8 and v3.6.
* **The `?unifiedProfile=enabled:true` filter on Catalog `/dataSets` is
  silently ignored** and returns everything. Filter client-side on
  `tags.unifiedProfile`.
* **The Query Service table name lives in `tags['adobe/pqs/table']`**, and is
  not the friendly dataset name. SQL needs the former.

---

## A general rule, learned the hard way

When an API can fail in a way that looks identical to a legitimate negative
answer, the two **must** be tracked separately all the way to the output. Any
tool whose result removes access, deletes data, or closes something down
should be able to say, per row, *"we asked and the answer was no"* versus
*"we could not ask"*.

If it cannot make that distinction, it is not safe to act on, however tidy the
spreadsheet looks.
