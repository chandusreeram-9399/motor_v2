# NEM Intermediary API — Frontend Integration Guide

**Base URLs:**

| Environment | URL |
|---|---|
| Staging | `https://test-api.masca.online:8735` |
| Production | `https://api.nemhmo.com` |

**Controller:** `NEMIntermediaryController` (+ `ApiUsers` login endpoint)
**Base route prefix:** `/api/NEMIntermediary` (login lives under `/api/ApiUsers`)
**Last generated:** 2026-10-07, from `NEMIntermediaryController.cs`

---

## Table of Contents

- [Executive Summary](#executive-summary)
- [Authentication](#authentication) — `POST /api/ApiUsers/Login`
- [1. Register Enrollee](#1-register-enrollee) — `POST /api/NEMIntermediary/Enrollement`
- [1b. Register Enrollees (Bulk)](#1b-register-enrollees-bulk) — `POST /api/NEMIntermediary/EnrollementBulk`
- [2. Renew Policy](#2-renew-policy) — `POST /api/NEMIntermediary/RenewPolicy`
- [3. Get Plan Providers](#3-get-plan-providers) — `GET /api/NEMIntermediary/planprovider`
- [4. Get Plans](#4-get-plans) — `GET /api/NEMIntermediary/Plans`
- [5. Get Plan Benefits](#5-get-plan-benefits) — `GET /api/NEMIntermediary/PlanBenefits`
- [6. Find Enrollee by Policy Number](#6-find-enrollee-by-policy-number) — `GET /api/NEMIntermediary/FindEnrolleeByPolicyNumber`
- [7. Find Enrollee by Phone Number](#7-find-enrollee-by-phone-number) — `GET /api/NEMIntermediary/FindEnrolleeByPhoneNumber`
- [8. Get Group Enrollees](#8-get-group-enrollees) — `GET /api/NEMIntermediary/GetgroupEnrollee`
- [9. Get Providers by Discipline](#9-get-providers-by-discipline) — `GET /api/NEMIntermediary/ProvidersByDiscipline`
- [10. Get Providers by Tier](#10-get-providers-by-tier) — `GET /api/NEMIntermediary/ProvidersByTier`
- [11. Lookup / Reference Data](#11-lookup--reference-data)
  - [11a. Get All Lookups](#11a-get-all-lookups) — `GET /api/NEMIntermediary/Lookups`
  - [11b. Get Genders](#11b-get-genders) — `GET /api/NEMIntermediary/Lookups/Genders`
  - [11c. Get Relationships](#11c-get-relationships) — `GET /api/NEMIntermediary/Lookups/Relationships`
  - [11d. Get States](#11d-get-states) — `GET /api/NEMIntermediary/Lookups/States`
  - [11e. Get Coverage Types](#11e-get-coverage-types) — `GET /api/NEMIntermediary/Lookups/CoverageTypes`
- [Quick Reference: All Error Envelopes](#quick-reference-all-error-envelopes)

---

## Executive Summary

This document is the API reference for the **NEM Intermediary** endpoints — the set of member-management operations exposed to intermediary/partner front ends: enrollee registration (single and bulk), policy renewal, plan/provider/benefit lookups, and enrollee search.

All 16 endpoints on `NEMIntermediaryController` are protected by a controller-wide `[JwtAuthFilter]` — every call **must** include a Bearer token obtained from the Login endpoint (`api/ApiUsers/Login`), which is the only endpoint in this reference that does not require one. There is no evidence in this controller of a stricter admin-only filter, so any valid, active user account can call every endpoint documented here.

Four things to know before integrating:

1. **Most identifiers on this API are strings, even when they represent numbers** (e.g. `deptypeid`, `genderid`, `stateaddressid`, `providerid`). Send them as JSON strings, not numeric literals.
2. **A few endpoints accept filter parameters they don't currently use for filtering.** These are called out explicitly in each endpoint's Notes so the frontend isn't built around behavior the API doesn't actually implement. See the ⚠️ callouts below.
3. **Don't hardcode the numeric IDs for gender, relationship, state, or coverage type.** Section 11 (Lookup / Reference Data) exposes the exact valid values for `genderid`, `deptypeid`, `stateaddressid`, and `covtypeid` — fetch and cache them instead of guessing.
4. **The bulk registration endpoint (§1b) returns HTTP `200` even when some or all records fail.** Each record is processed independently and its outcome is in `results[]` — always check the top-level `status` (`success` / `partial` / `failed`) and each record's `status`/`httpstatus`, not just the HTTP code.

### Endpoint Index

| # | Method | Route | Purpose |
|---|--------|-------|---------|
| — | `POST` | `/api/ApiUsers/Login` | Authenticate and obtain a JWT |
| 1 | `POST` | `/api/NEMIntermediary/Enrollement` | Register a new enrollee (principal or dependant) |
| 1b | `POST` | `/api/NEMIntermediary/EnrollementBulk` | Register up to 50 enrollees (principals + dependants) in one call |
| 2 | `POST` | `/api/NEMIntermediary/RenewPolicy` | Renew an existing enrollee's policy |
| 3 | `GET` | `/api/NEMIntermediary/planprovider` | List providers available on a plan |
| 4 | `GET` | `/api/NEMIntermediary/Plans` | List available plans |
| 5 | `GET` | `/api/NEMIntermediary/PlanBenefits` | List benefits covered by a plan |
| 6 | `GET` | `/api/NEMIntermediary/FindEnrolleeByPolicyNumber` | Look up one enrollee by policy/membership number |
| 7 | `GET` | `/api/NEMIntermediary/FindEnrolleeByPhoneNumber` | Look up one enrollee by phone number |
| 8 | `GET` | `/api/NEMIntermediary/GetgroupEnrollee` | List all enrollees under a group |
| 9 | `GET` | `/api/NEMIntermediary/ProvidersByDiscipline` | List providers filtered by discipline |
| 10 | `GET` | `/api/NEMIntermediary/ProvidersByTier` | List providers filtered by provider tier |
| 11a | `GET` | `/api/NEMIntermediary/Lookups` | All reference lists in one call (genders, relationships, states, coverage types) |
| 11b | `GET` | `/api/NEMIntermediary/Lookups/Genders` | Valid `genderid` values |
| 11c | `GET` | `/api/NEMIntermediary/Lookups/Relationships` | Valid `deptypeid` values |
| 11d | `GET` | `/api/NEMIntermediary/Lookups/States` | Valid `stateaddressid` values (scoped by country) |
| 11e | `GET` | `/api/NEMIntermediary/Lookups/CoverageTypes` | Valid `covtypeid` values |

---

## Authentication

### `POST /api/ApiUsers/Login`

Authenticates a username/password pair and returns a JWT to use as a Bearer token on every other endpoint in this guide.

- **Auth required:** No (`AllowAnonymous`)
- **Content-Type:** `application/json`

**Request payload**

| Field | Type | Required | Notes |
|---|---|---|---|
| `Username` | string | ✅ Yes | |
| `Password` | string | ✅ Yes | |

```json
{
  "Username": "your.username",
  "Password": "your-password"
}
```

**Success response — `200 OK`**

```json
{
  "status": "success",
  "message": "Login successful.",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires": "2026-09-24T02:00:00Z"
}
```

`expires` is 8 hours from the time of login (UTC).

**Error responses**

| Status | Body | Cause |
|---|---|---|
| `400 Bad Request` | `{ "status": "error", "message": "Username and password are required." }` | Missing username or password |
| `401 Unauthorized` | `{ "status": "error", "message": "Invalid username or password." }` | No matching active user, or password mismatch |
| `500 Internal Server Error` | `{ "status": "error", "message": "<exception message>" }` | Unhandled server error |

### Using the token

Every endpoint below requires this header:

```
Authorization: Bearer <token>
```

---

## 1. Register Enrollee

### `POST /api/NEMIntermediary/Enrollement`

Registers a new enrollee — either a **principal** member or a **dependant** attached to an existing principal. Also handles plan/scheme assignment for new principals and optional passport photo upload. To register several enrollees (e.g. a principal and their dependants) in one request, see [§1b](#1b-register-enrollees-bulk).

- **Auth required:** ✅ Yes

**Request payload** — `RegistrationPayload`

| Field | Type | Required | Notes |
|---|---|---|---|
| `plancode` | string | ✅ Yes | Matched against scheme code or scheme name. Invalid code → `400`. |
| `grouplegacyid` | string | ✅ Yes | Matched against `Group_id`. Invalid value → `400`. |
| `isprincipal` | string (`"true"`/`"false"`) | ✅ Yes | Determines the flow. Anything that doesn't parse as boolean is treated as `false`. |
| `principalid` | string | ✅ **only if `isprincipal` = `"false"`** | Membership no., membership no. + suffix, or CIF number of the existing principal. Not found → `400`. |
| `deptypeid` | string (numeric) | Recommended | `AFC_RELATIONSHIPS.keyId`. Stored as `Relationship_ID` on **every** registration (principal or dependant) — falls back to `0`/"PRINCIPAL" relation if missing or not found. Only affects suffix generation (via `Api_usp_udfNumberToLetter`) for dependants; principals always get suffix `"0"` regardless of this value. Valid values: [`GET /api/NEMIntermediary/Lookups/Relationships`](#11c-get-relationships) (§11c). |
| `firstname` | string | Recommended | Used in the duplicate-member check. |
| `lastname` | string | Recommended | Used in the duplicate-member check. |
| `othername` | string | Optional | |
| `dob` | string (date, e.g. `2026-09-23`) | Recommended | Used in the duplicate-member check; must be a parseable date if a `DateOfBirth` is to be stored. |
| `phone1` | string | Recommended | Stored as both `mobilePhone2` and `Postal_Phone`; used in duplicate check. |
| `email1` | string | Recommended | Used in duplicate check. |
| `genderid` | string (numeric) | Optional | `AFC_SEX.sex_id`. Must be a valid integer string if supplied — a non-numeric value throws a server error. Valid values: [`GET /api/NEMIntermediary/Lookups/Genders`](#11b-get-genders) (§11b). |
| `address1` | string | Optional | |
| `stateaddressid` | string (numeric) | Optional | `AFC_REGION.RegionID`. Must be a valid integer string if supplied. Valid values: [`GET /api/NEMIntermediary/Lookups/States`](#11d-get-states) (§11d), scoped to a country (defaults to Nigeria). |
| `providerid` | string (numeric) | Optional | Primary provider ID. Must be a valid integer string if supplied. |
| `effectivedate` | string (date) | Optional | Membership start date. Must be a parseable date if supplied. |
| `terminationdate` | string (date) | Optional | Must be a parseable date if supplied. |
| `renewaldate` | string (date) | Optional | Must be a parseable date if supplied. |
| `passportphoto` | string (Base64) | Optional | Raw Base64 or a data URI (`data:image/png;base64,...` — the prefix up to the first comma is stripped automatically). Invalid Base64 is logged server-side and silently skipped; it does not fail the request. |
| `idcard` | string | Optional | |
| `refcode` | string | Optional | |
| `referralname` | string | Optional | Stored as `Registrationsource`. |
| `alternatecode1`, `pixpath`, `username`, `isterminated` | string | Not used | Accepted by the model but not referenced anywhere in this endpoint's logic. |

**Sample request — new principal**

```json
{
  "plancode": "PLN001",
  "grouplegacyid": "GRP123",
  "isprincipal": "true",
  "deptypeid": "1",
  "firstname": "Ada",
  "lastname": "Okafor",
  "dob": "1990-05-14",
  "phone1": "08012345678",
  "email1": "ada.okafor@example.com",
  "genderid": "2",
  "address1": "12 Marina Road, Lagos",
  "stateaddressid": "25",
  "providerid": "104",
  "effectivedate": "2026-10-01",
  "terminationdate": "2027-09-30",
  "idcard": "A12345678"
}
```

**Sample request — dependant**

```json
{
  "plancode": "PLN001",
  "grouplegacyid": "GRP123",
  "isprincipal": "false",
  "principalid": "GRP123-00042",
  "deptypeid": "3",
  "firstname": "Chinedu",
  "lastname": "Okafor",
  "othername": "",
  "dob": "2015-03-02",
  "genderid": "1"
}
```

**Success response — `200 OK`**

```json
{
  "status": "success",
  "response": "Success|10456|",
  "Cif_Number": "10456",
  "MembershipNo": "GRP123-00042",
  "EffectiveDate": "2026-10-01",
  "TerminationDate": "2027-09-30"
}
```

For a dependant, `MembershipNo` is returned as `"<principal's membership no.>/<suffix>"`. `EffectiveDate`/`TerminationDate` reflect what was actually persisted in the database (empty string if not set).

**Error responses**

| Status | Body (`response` message) | Cause |
|---|---|---|
| `400 Bad Request` | `"Payload does not have information!"` | Empty/null payload |
| `400 Bad Request` | `"Invalid Plan Code."` | `plancode` doesn't match any scheme |
| `400 Bad Request` | `"Invalid Group Legacy ID."` | `grouplegacyid` doesn't match any group |
| `400 Bad Request` | `"Principal HMO Number not found."` | `isprincipal = "false"` and `principalid` doesn't match an existing member |
| `409 Conflict` | `"This member already exists in the database."` | A member with matching first name, surname, DOB, group, phone, and email already exists |
| `500 Internal Server Error` | `"<exception message>"` | Unhandled server error (e.g. non-numeric `genderid`/`stateaddressid`/`providerid`, unparseable date) |

---

## 1b. Register Enrollees (Bulk)

### `POST /api/NEMIntermediary/EnrollementBulk`

Registers a list of enrollees in one call — typically a principal plus their dependants. Each record goes through exactly the same logic and validation as the single [§1 Register Enrollee](#1-register-enrollee) endpoint, but **each record is processed independently**: there is no all-or-nothing transaction, and the response reports a result per record, in the same order as the request.

- **Auth required:** ✅ Yes

**How batching works**

- **Principals are processed before dependants**, regardless of their order in the array — so a dependant may appear before its principal in the request.
- A dependant can attach to a parent in one of two ways:
  - `principalid` — an **existing** member (membership no., membership no. + suffix, or CIF number), exactly as on the single endpoint; or
  - `principalref` — the `clientref` of a **principal in the same batch** that was registered successfully.
- If a dependant supplies both, **`principalid` wins** and `principalref` is ignored.
- A principal can only be referenced via `principalref` if it has a non-blank `clientref` **and** its own registration succeeded. If the principal fails (e.g. duplicate member), its dependants fail with a `principalref ... does not match` error.
- Records already saved are **not rolled back** if later records fail.

**Request payload**

| Field | Type | Required | Notes |
|---|---|---|---|
| `enrollees` | array of objects | ✅ Yes | 1 to **50** records. Empty, missing, or more than 50 → `400`. |

**Each object in `enrollees`** accepts every field from the single endpoint's `RegistrationPayload` ([§1 table](#1-register-enrollee) — same required/optional rules, same validation), plus two batch fields:

| Field | Type | Required | Notes |
|---|---|---|---|
| `clientref` | string | Recommended for all; ✅ **required on any principal that dependants reference** | Your own unique key for the record, echoed back in the result. Compared case-insensitively after trimming; duplicates within the batch → `400` for the whole request. Blank values aren't checked for uniqueness. |
| `principalref` | string | ✅ For a dependant when `principalid` is not supplied | `clientref` of a principal earlier/elsewhere in the same batch. |

Per-record minimum, as in §1: `plancode`, `grouplegacyid`, `isprincipal` — plus `principalid` or `principalref` when `isprincipal` is `"false"`. As in §1, send all IDs as strings.

**Sample request — one principal + one dependant**

```json
{
  "enrollees": [
    { "clientref": "P1", "plancode": "<plancode>", "grouplegacyid": "<groupid>", "isprincipal": "true", "deptypeid": "36",
      "firstname": "Ada", "lastname": "Okafor", "dob": "1990-05-14", "phone1": "08000000001", "email1": "ada@example.com",
      "genderid": "2", "stateaddressid": "25", "providerid": "<providerid>", "effectivedate": "2026-10-01", "terminationdate": "2027-09-30" },
    { "clientref": "D1", "principalref": "P1", "plancode": "<plancode>", "grouplegacyid": "<groupid>", "isprincipal": "false", "deptypeid": "8",
      "firstname": "Chinedu", "lastname": "Okafor", "dob": "2015-03-02", "genderid": "1" }
  ]
}
```

Replace `<plancode>` with a plan code from [§4 Plans](#4-get-plans), `<groupid>` with the group's ID (as on §1, `grouplegacyid` is matched against the group ID), and `<providerid>` with a provider ID from [§3](#3-get-plan-providers) / [§10](#10-get-providers-by-tier). `deptypeid`, `genderid` and `stateaddressid` values come from [§11](#11-lookup--reference-data).

**Success response — `200 OK`** (all records succeeded)

```json
{
  "status": "success",
  "total": 2,
  "succeeded": 2,
  "failed": 0,
  "results": [
    {
      "index": 0,
      "clientref": "P1",
      "status": "success",
      "httpstatus": 200,
      "response": "Success|10456|",
      "Cif_Number": "10456",
      "MembershipNo": "GRP123-00042",
      "EffectiveDate": "2026-10-01",
      "TerminationDate": "2027-09-30"
    },
    {
      "index": 1,
      "clientref": "D1",
      "status": "success",
      "httpstatus": 200,
      "response": "Success|10457|",
      "Cif_Number": "10457",
      "MembershipNo": "GRP123-00042/A",
      "EffectiveDate": "",
      "TerminationDate": ""
    }
  ]
}
```

(Values are illustrative.) As on §1, a dependant's `MembershipNo` is `"<principal's membership no.>/<suffix>"`, and the dates are what was actually saved (empty string if none).

**Partial-failure response — also `200 OK`**

```json
{
  "status": "partial",
  "total": 2,
  "succeeded": 1,
  "failed": 1,
  "results": [
    {
      "index": 0,
      "clientref": "P1",
      "status": "success",
      "httpstatus": 200,
      "response": "Success|10456|",
      "Cif_Number": "10456",
      "MembershipNo": "GRP123-00042",
      "EffectiveDate": "2026-10-01",
      "TerminationDate": "2027-09-30"
    },
    {
      "index": 1,
      "clientref": "D1",
      "status": "error",
      "httpstatus": 409,
      "response": "This member already exists in the database.",
      "Cif_Number": null,
      "MembershipNo": null,
      "EffectiveDate": null,
      "TerminationDate": null
    }
  ]
}
```

**Response fields**

| Field | Type | Notes |
|---|---|---|
| `status` | string | `"success"` (all succeeded), `"partial"` (some), or `"failed"` (none). **The HTTP status is `200` in all three cases.** |
| `total` / `succeeded` / `failed` | int | Record counts. |
| `results[]` | array | One entry per request record, **in request order**. |
| `results[].index` | int | 0-based position of the record in the request's `enrollees` array. |
| `results[].clientref` | string \| null | Echo of the request's `clientref`. |
| `results[].status` | string | `"success"` or `"error"`. |
| `results[].httpstatus` | int | The status code this record would have received on the single endpoint (`200`, `400`, `409`, `500`). |
| `results[].response` | string | `"Success\|<cif>\|"` on success; the error message otherwise. |
| `results[].Cif_Number`, `MembershipNo`, `EffectiveDate`, `TerminationDate` | string \| null | Populated on success; `null` on error. |

**Error responses — whole request rejected** (nothing processed)

| Status | Body (`response`) | Cause |
|---|---|---|
| `400 Bad Request` | `"enrollees list is required and cannot be empty."` | Missing/malformed body, `enrollees` missing, `null`, or empty |
| `400 Bad Request` | `"A maximum of 50 enrollees is allowed per request."` | More than 50 records |
| `400 Bad Request` | `"Duplicate clientref '<ref>' in the batch."` | Two records share a `clientref` (case-insensitive, trimmed) |
| `500 Internal Server Error` | `"<exception message>"` | Unhandled server error |

**Per-record errors** (appear inside `results[]` with `status: "error"`)

| `httpstatus` | `response` | Cause |
|---|---|---|
| `400` | `"Enrollee entry is null."` | A `null` element in the array |
| `400` | `"principalref '<ref>' does not match a principal registered successfully earlier in this batch."` | Unknown `principalref`, or that principal's own registration failed |
| `400` | `"Invalid Plan Code."` / `"Invalid Group Legacy ID."` / `"Principal HMO Number not found."` | Same causes as §1 |
| `409` | `"This member already exists in the database."` | Duplicate member, as on §1 |
| `500` | `"<exception message>"` | Unexpected error for that record (e.g. non-numeric `genderid`, unparseable date) — the rest of the batch still runs |

⚠️ **Retrying after a partial failure:** don't resubmit the whole batch — records that already succeeded would typically be rejected as duplicates (`409`). Resend only the failed records. If a dependant's principal was created in a *previous* call, use `principalid` (the principal's `MembershipNo` or `Cif_Number` from the earlier result) instead of `principalref`, since `principalref` only resolves within a single request.

---

## 2. Renew Policy

### `POST /api/NEMIntermediary/RenewPolicy`

Extends an existing enrollee's coverage and records a new scheme-membership period.

- **Auth required:** ✅ Yes

**Request payload** — `RenewalPayload`

| Field | Type | Required | Notes |
|---|---|---|---|
| `enrolleenumber` | string | ✅ Yes | Matched against membership no. or membership no. + suffix. |
| `groupid` | string (numeric) | ✅ Yes | Must parse to an integer; non-numeric values are silently treated as `0` (which will not match any enrollee, resulting in a `404`). |
| `effectivedate` | string (date) | Optional | Defaults to today if omitted or unparseable. |
| `terminationdate` | string (date) | Optional | Defaults to today + 365 days if omitted or unparseable. |

```json
{
  "enrolleenumber": "GRP123-00042",
  "groupid": "17",
  "effectivedate": "2026-10-01",
  "terminationdate": "2027-09-30"
}
```

**Success response — `200 OK`**

```json
{
  "status": "success",
  "response": "Renewal Successful for GRP123-00042"
}
```

**Error responses**

| Status | Body (`response`) | Cause |
|---|---|---|
| `400 Bad Request` | `"Invalid payload. Missing enrollee number or group ID."` | Missing `enrolleenumber` or `groupid` |
| `404 Not Found` | `"Enrollee not found."` | No enrollee matches `enrolleenumber` + `groupid` |
| `500 Internal Server Error` | `"<exception message>"` | Unhandled server error |

---

## 3. Get Plan Providers

### `GET /api/NEMIntermediary/planprovider`

Lists the providers attached to a plan, optionally filtered by coverage type.

- **Auth required:** ✅ Yes

**Query parameters**

| Field | Type | Required | Notes |
|---|---|---|---|
| `plancode` | string | ✅ Yes | Matched against scheme code or scheme name (trimmed, case handling per DB collation). |
| `grouplegacyid` | string | Accepted, not used | ⚠️ Accepted as a query parameter but not referenced in the filtering logic — passing it has no effect on the results. |
| `covtypeid` | string (numeric) | Optional | `AFC_DISCIPLINES.DISCIPLINE_ID` — despite the name, this filters on the **provider's discipline** (`afc_providers.discipline_id`), not a scheme/coverage type (`AFC_SCHEMETYPE`) as the name might suggest. Missing or non-numeric values are passed through as `0` to `Api_spGet_ProviderListByPlanCodeAndFilters`'s `@TypeID`. Valid values: [`GET /api/NEMIntermediary/Lookups/CoverageTypes`](#11e-get-coverage-types) (§11e). |

`GET /api/NEMIntermediary/planprovider?plancode=PLN001&covtypeid=2`

**Success response — `200 OK`** — array of provider objects:

```json
[
  {
    "providerid": 104,
    "providertin": null,
    "providerkey": "PRV-0104",
    "providername": "Lagoon Specialist Hospital",
    "provcatid": 0,
    "categoryname": "Hospital",
    "contactlastname": "Adeyemi",
    "contactfirstname": null,
    "contactothername": null,
    "contactphone1": "08011112222",
    "contactemail1": "contact@lagoonhospital.ng",
    "providerphone1": "08033334444",
    "provideremail1": "contact@lagoonhospital.ng",
    "address1": "5 Adeola Odeku St, Victoria Island",
    "regionid": 25,
    "region": "Lagos",
    "rcnumber": null
  }
]
```

`provcatid` and `rcnumber` are always `0`/`null` respectively — not populated by this endpoint. Returns `[]` (not an error) if the plan has no providers.

**Error responses**

| Status | Body (`response`) | Cause |
|---|---|---|
| `400 Bad Request` | `"plancode is required."` | Missing `plancode` |
| `404 Not Found` | `"Invalid Plan Code."` | `plancode` doesn't match any scheme |
| `500 Internal Server Error` | `"<exception message>"` | Unhandled server error |

---

## 4. Get Plans

### `GET /api/NEMIntermediary/Plans`

Lists available plans/schemes.

- **Auth required:** ✅ Yes

**Query parameters**

| Field | Type | Required | Notes |
|---|---|---|---|
| `grouplegacyid` | string | ✅ Yes | ⚠️ **Validated but not used to filter results.** The API confirms the group code exists (404 if not), but the plan list returned is the **full set of schemes with a non-null scheme code** — it is not scoped to that group. Don't build UI logic that expects a per-group plan list. |

`GET /api/NEMIntermediary/Plans?grouplegacyid=GRP123`

**Success response — `200 OK`** — array of plan objects:

```json
[
  { "planid": 1, "plancode": "PLN001", "planname": "NEM Silver Plan" },
  { "planid": 2, "plancode": "PLN002", "planname": "NEM Gold Plan" }
]
```

**Error responses**

| Status | Body (`response`) | Cause |
|---|---|---|
| `400 Bad Request` | `"grouplegacyid is required."` | Missing `grouplegacyid` |
| `404 Not Found` | `"Invalid Group ID."` | `grouplegacyid` doesn't match `GROUP_CODE` on any group |
| `500 Internal Server Error` | `"<exception message>"` | Unhandled server error |

---

## 5. Get Plan Benefits

### `GET /api/NEMIntermediary/PlanBenefits`

Lists benefits covered under a plan.

- **Auth required:** ✅ Yes

**Query parameters**

| Field | Type | Required | Notes |
|---|---|---|---|
| `plancode` | string | ✅ Yes | Matched against scheme code or scheme name. |

`GET /api/NEMIntermediary/PlanBenefits?plancode=PLN001`

**Success response — `200 OK`** — array of benefit objects:

```json
[
  {
    "benefitcodeid": 0,
    "benefitcodename": "OP001",
    "benefitcodedesc": "General Outpatient Consultation",
    "benefitlevelid": 1,
    "dateadded": "2026-09-23T14:05:32.118",
    "maxlimit": null
  }
]
```

`benefitcodeid` is always `0`, `benefitlevelid` is always `1`, `dateadded` is the time the API call was made (not a stored value), and `maxlimit` is always `null` — none of these four fields carry real per-benefit data in the current implementation.

**Error responses**

| Status | Body (`response`) | Cause |
|---|---|---|
| `400 Bad Request` | `"plancode is required."` | Missing `plancode` |
| `404 Not Found` | `"Invalid Plan Code."` | `plancode` doesn't match any scheme |
| `500 Internal Server Error` | `"<exception message>"` | Unhandled server error |

---

## 6. Find Enrollee by Policy Number

### `GET /api/NEMIntermediary/FindEnrolleeByPolicyNumber`

Looks up a single enrollee by their policy/membership number.

- **Auth required:** ✅ Yes

**Query parameters**

| Field | Type | Required | Notes |
|---|---|---|---|
| `policynumber` | string | ✅ Yes (no results without it) | No explicit empty-string validation in the controller — if omitted, the `@PolicyNumber` parameter is not passed to the stored procedure at all, and the result is driven by the stored procedure's own defaults (in practice, expect either no match → `404`, or unfiltered results if the SP treats a missing parameter as "no filter"; not verifiable from this controller alone). Always send a real value. |

`GET /api/NEMIntermediary/FindEnrolleeByPolicyNumber?policynumber=GRP123-00042`

**Success response — `200 OK`**

```json
{
  "status": "success",
  "response": "Success|10456",
  "Cif_Number": "10456",
  "EffectiveDate": "2026-10-01",
  "TerminationDate": "2027-09-30",
  "fullName": "Ada Okafor",
  "enrolleeId": "ENR-10456",
  "planType": "NEM Silver Plan",
  "address": "12 Marina Road, Lagos",
  "groupid": "GRP123",
  "groupName": "Acme Nigeria Ltd",
  "provider": "Lagoon Specialist Hospital",
  "memberStatus": "Active",
  "gender": "Female"
}
```

**Error responses**

| Status | Body (`response`) | Cause |
|---|---|---|
| `404 Not Found` | `"No enrollee(s) found."` | No match |
| `500 Internal Server Error` | `"<exception message>"` | Unhandled server error |

---

## 7. Find Enrollee by Phone Number

### `GET /api/NEMIntermediary/FindEnrolleeByPhoneNumber`

Same behavior and response shape as [§6](#6-find-enrollee-by-policy-number), keyed on phone number instead.

- **Auth required:** ✅ Yes

**Query parameters**

| Field | Type | Required |
|---|---|---|
| `phonenumber` | string | ✅ Yes |

`GET /api/NEMIntermediary/FindEnrolleeByPhoneNumber?phonenumber=08012345678`

Success/error responses are identical in shape to §6.

---

## 8. Get Group Enrollees

### `GET /api/NEMIntermediary/GetgroupEnrollee`

Lists all enrollees under a given group.

- **Auth required:** ✅ Yes

**Query parameters**

| Field | Type | Required | Notes |
|---|---|---|---|
| `groupid` | string (numeric) | ✅ Yes | Must parse as an integer, or the call returns `400`. ⚠️ A value of `"0"` passes this check but is then **not** sent to the stored procedure at all (only values `> 0` are passed as `@GroupID`) — send a real positive group ID. |

`GET /api/NEMIntermediary/GetgroupEnrollee?groupid=17`

**Success response — `200 OK`**

```json
{
  "status": "success",
  "data": [
    {
      "fullName": "Ada Okafor",
      "enrolleeId": "ENR-10456",
      "planType": "NEM Silver Plan",
      "address": "12 Marina Road, Lagos",
      "groupId": "GRP123",
      "groupName": "Acme Nigeria Ltd",
      "provider": "Lagoon Specialist Hospital",
      "memberStatus": "Active",
      "gender": "Female",
      "Cif_Number": "10456",
      "MembershipNo": null,
      "EffectiveDate": "2026-10-01",
      "TerminationDate": "2027-09-30"
    }
  ]
}
```

⚠️ Note the field naming difference from §6/§7: this response uses `groupId` (camelCase) instead of `groupid`, and it includes `MembershipNo`, which is **always `null`** — the field exists on the underlying model but is never populated by this endpoint.

**Error responses**

| Status | Body (`response`) | Cause |
|---|---|---|
| `400 Bad Request` | `"groupid must be valid."` | Missing or non-numeric `groupid` |
| `404 Not Found` | `"No enrollee(s) found."` | No enrollees under that group |
| `500 Internal Server Error` | `"<exception message>"` | Unhandled server error |

---

## 9. Get Providers by Discipline

### `GET /api/NEMIntermediary/ProvidersByDiscipline`

Lists providers, optionally filtered by discipline.

- **Auth required:** ✅ Yes

**Query parameters**

| Field | Type | Required | Notes |
|---|---|---|---|
| `discipline` | string | Optional | Omit (or send blank) to return providers across all disciplines. |

`GET /api/NEMIntermediary/ProvidersByDiscipline?discipline=Cardiology`

**Success response — `200 OK`**

```json
{
  "status": "success",
  "count": 2,
  "data": [
    {
      "Provider_Id": 104,
      "Provider": "Lagoon Specialist Hospital",
      "Discipline_Code": "CARD",
      "DISCIPLINE": "Cardiology"
    }
  ]
}
```

Note the field casing here is `Provider_Id` / `Provider` / `Discipline_Code` / `DISCIPLINE` — inconsistent with other endpoints' `providerid`/`providername`, since this DTO isn't shared with the rest of the controller.

**Error responses**

| Status | Body | Cause |
|---|---|---|
| `500 Internal Server Error` | `{ "status": "error", "response": "<exception message>" }` | Unhandled server error. There is no dedicated `400`/`404` path — an unmatched discipline just returns an empty `data` array with `count: 0`. |

---

## 10. Get Providers by Tier

### `GET /api/NEMIntermediary/ProvidersByTier`

Lists providers filtered by a provider grading tier (e.g. "Tier 1", "Tier 2", "International Tier").

- **Auth required:** ✅ Yes

**Query parameters**

| Field | Type | Required | Notes |
|---|---|---|---|
| `tier` | string | ✅ Yes | Matched against either `GradingCode` or `GradingDescription`. ⚠️ The tier's display text and its underlying `ProviderGradingId` are **not** the same number — e.g. "Tier 3" may resolve to grading ID `4`. Don't assume `tier=3` maps to ID `3`. |

`GET /api/NEMIntermediary/ProvidersByTier?tier=Tier%201`

**Success response — `200 OK`**

```json
{
  "status": "success",
  "tierid": 2,
  "gradingcode": "T1",
  "gradingdescription": "Tier 1",
  "count": 1,
  "data": [
    {
      "providerid": 104,
      "providertin": null,
      "providerkey": "PRV-0104",
      "providername": "Lagoon Specialist Hospital",
      "provcatid": 0,
      "categoryname": "Hospital",
      "contactlastname": "Adeyemi",
      "contactfirstname": null,
      "contactothername": null,
      "contactphone1": "08011112222",
      "contactemail1": "contact@lagoonhospital.ng",
      "providerphone1": "08033334444",
      "provideremail1": "contact@lagoonhospital.ng",
      "address1": "5 Adeola Odeku St, Victoria Island",
      "regionid": 25,
      "region": "Lagos",
      "city": "Lagos",
      "rcnumber": null,
      "providergradeid": 2,
      "gradingcode": "T1",
      "gradingdescription": "Tier 1"
    }
  ]
}
```

Always sanity-check the top-level `gradingcode`/`gradingdescription` echoed back against what you searched for — it confirms the tier text actually resolved to the grading you meant.

**Error responses**

| Status | Body (`response`) | Cause |
|---|---|---|
| `400 Bad Request` | `"tier is required."` | Missing `tier` |
| `404 Not Found` | `"Invalid Tier ('<tier>'). No matching GradingCode or GradingDescription in ProviderGrades."` | `tier` doesn't match any grading record |
| `500 Internal Server Error` | `"<exception message>"` | Unhandled server error |

---

## 11. Lookup / Reference Data

Five endpoints that expose the exact valid values for four fields used elsewhere in this API: `genderid` and `deptypeid` (Enrollment, §1), `stateaddressid` (Enrollment, §1), and `covtypeid` (Plan Providers, §3). Use these instead of hardcoding IDs — the underlying lookup tables can change.

All five share the same item shape:

```ts
{ id: number, name: string, code: string | null, extra: string | null }
```

`code` and `extra` are populated only where the underlying table has something to put there (see per-endpoint notes below) — otherwise they come back `null`.

- **Auth required:** ✅ Yes (all five)

### 11a. Get All Lookups

`GET /api/NEMIntermediary/Lookups`

Convenience endpoint that returns all four reference lists in a single call.

**Query parameters**

| Field | Type | Required | Notes |
|---|---|---|---|
| `countryid` | int | Optional | Filters the `states` list only. Defaults to `37` (Nigeria) if omitted. If supplied but non-numeric, standard Web API query-binding behavior applies — expect it to bind to `0` rather than the `37` default, which will return an empty `states` list; pass a real integer. |

`GET /api/NEMIntermediary/Lookups?countryid=37`

**Success response — `200 OK`**

```json
{
  "status": "success",
  "genders": [
    { "id": 1, "name": "Male", "code": null, "extra": null },
    { "id": 2, "name": "Female", "code": null, "extra": null }
  ],
  "relationships": [
    { "id": 1, "name": "PRINCIPAL", "code": null, "extra": "M" },
    { "id": 3, "name": "CHILD", "code": null, "extra": null }
  ],
  "states": [
    { "id": 25, "name": "Lagos", "code": "LA", "extra": null }
  ],
  "coveragetypes": [
    { "id": 2, "name": "Cardiology", "code": "CARD", "extra": null }
  ]
}
```

Note this combined response has **no `count` field** per list (unlike the four single-lookup endpoints below) — use `.length` client-side if you need a count.

**Error responses**

| Status | Body (`response`) | Cause |
|---|---|---|
| `500 Internal Server Error` | `"<exception message>"` | Unhandled server error |

### 11b. Get Genders

`GET /api/NEMIntermediary/Lookups/Genders`

Source: `AFC_SEX` (`sex_id` → `id`, `sex` → `name`). `code` and `extra` are always `null`.

`GET /api/NEMIntermediary/Lookups/Genders`

```json
{
  "status": "success",
  "count": 2,
  "data": [
    { "id": 1, "name": "Male", "code": null, "extra": null },
    { "id": 2, "name": "Female", "code": null, "extra": null }
  ]
}
```

### 11c. Get Relationships

`GET /api/NEMIntermediary/Lookups/Relationships`

Source: `AFC_RELATIONSHIPS` (`keyId` → `id`, `relationship` → `name`). `code` is always `null` (hardcoded in the query). `extra` carries the `Gender` the relationship implies (e.g. a "SON" relationship might imply Male) — still `null` where the table has no gender association.

`GET /api/NEMIntermediary/Lookups/Relationships`

```json
{
  "status": "success",
  "count": 5,
  "data": [
    { "id": 1, "name": "PRINCIPAL", "code": null, "extra": null },
    { "id": 2, "name": "SPOUSE", "code": null, "extra": null },
    { "id": 3, "name": "CHILD", "code": null, "extra": null }
  ]
}
```

### 11d. Get States

`GET /api/NEMIntermediary/Lookups/States`

Source: `AFC_REGION` (`RegionID` → `id`, `Region` → `name`, `region_code` → `code`), filtered by `CountryID`. `extra` is always `null`.

**Query parameters**

| Field | Type | Required | Notes |
|---|---|---|---|
| `countryid` | int | Optional | Defaults to `37` (Nigeria) if omitted. Same non-numeric-value caveat as §11a. |

`GET /api/NEMIntermediary/Lookups/States?countryid=37`

```json
{
  "status": "success",
  "count": 37,
  "data": [
    { "id": 25, "name": "Lagos", "code": "LA", "extra": null },
    { "id": 12, "name": "Abuja (FCT)", "code": "FC", "extra": null }
  ]
}
```

### 11e. Get Coverage Types

`GET /api/NEMIntermediary/Lookups/CoverageTypes`

Source: `AFC_DISCIPLINES` (`DISCIPLINE_ID` → `id`, `DISCIPLINE` → `name`, `CODE` → `code`). `extra` is always `null`. These are the values `covtypeid` accepts on the Plan Providers endpoint (§3) — see that endpoint's note on what this parameter actually filters.

`GET /api/NEMIntermediary/Lookups/CoverageTypes`

```json
{
  "status": "success",
  "count": 8,
  "data": [
    { "id": 2, "name": "Cardiology", "code": "CARD", "extra": null }
  ]
}
```

**Error responses (§11b–§11e, all four)**

| Status | Body (`response`) | Cause |
|---|---|---|
| `500 Internal Server Error` | `"<exception message>"` | Unhandled server error |

None of the five lookup endpoints has a `400`/`404` path — an unrecognized `countryid` simply returns an empty `states` array rather than an error.

---

## Quick Reference: All Error Envelopes

Every endpoint returns errors in one of two shapes, both JSON:

```json
{ "status": "error", "message": "..." }     // Login only
{ "status": "error", "response": "..." }    // Everything on NEMIntermediaryController
```

Frontend error handling should check for `status !== "success"` and read `message` (Login) or `response` (everything else) for the display text.

**Exception — bulk registration (§1b):** it reports per-record outcomes inside `results[]` and returns HTTP `200` with a top-level `status` of `success`, `partial`, or `failed`. Request-level problems (empty list, over 50, duplicate `clientref`) still use the standard `{ "status": "error", "response": "..." }` envelope with a `400`.
