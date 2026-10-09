# BILGE — Backend API Specification

This document defines the REST API the BILGE frontend needs from its backend.

**Current state:** the frontend has no backend yet. Everything below is
currently served from static files or `localStorage`:

| Feature | Today (frontend only) | Replaced by |
|---|---|---|
| Universities catalog | `src/data/universities.js` | [Universities](#3-universities) |
| Scholarships catalog | `src/data/scholarships.js` | [Scholarships](#4-scholarships) |
| Sign up / sign in | `bilge.accounts`, `bilge.session` in localStorage (`src/auth/AuthProvider.jsx`) | [Auth](#2-authentication) |
| Saved universities / scholarships | `bilge.saved.universities`, `bilge.saved.scholarships` | [Saved items](#6-saved-items) |
| Document checklists | `bilge.checklist.<uniId>`, `bilge.checklist.sch.<schId>` | [Checklists](#7-document-checklists) |
| ORT practice questions + results | `src/data/ortPractice.js`, `bilge.ort.results` | [ORT practice](#5-ort-practice) |
| Language / theme | `bilge.lang`, `bilge.theme` | [Preferences](#8-user-preferences) |
| Contact form | `mailto:` link | [Contact](#9-contact) |
| Terms / Privacy / Cookies | `src/data/legal.js` | [Legal pages](#10-legal-pages) |

The field names in this spec match the objects the React components already
use, so the frontend only needs to swap data sources, not rewrite components.

---

## 1. Conventions

### Base URL

```
https://api.bilge.kg/v1
```

### Format

- Request and response bodies are JSON (`Content-Type: application/json`).
- Field names are `camelCase`.
- Timestamps are ISO 8601 UTC strings: `"2026-10-09T08:15:00Z"`.
- Yearly recurring dates (admission windows) are `"MM-DD"` strings: `"08-15"`.
- Money is a number in USD (`tuitionUsd: 6500`); `null` means unknown/varies.

### Language

Content endpoints accept the UI language and return translated text where
available, falling back to English.

- Query param `?lang=en|ru|ky`, **or** header `Accept-Language: ru`.
- The frontend's language codes are `EN`, `RU`, `KG`. Map them to
  `en`, `ru`, `ky`.

### Authentication

Endpoints marked 🔒 require an access token:

```
Authorization: Bearer <accessToken>
```

Endpoints marked 🔓 work without a token but return extra fields (e.g.
`isSaved`) when one is sent.

### Pagination

List endpoints that paginate take `page` (1-based, default `1`) and
`pageSize` (default `6` for universities, matching `ITEMS_PER_PAGE`) and
return:

```json
{
  "items": [],
  "pagination": { "page": 1, "pageSize": 6, "totalItems": 18, "totalPages": 3 }
}
```

### Errors

Every error response has the same shape:

```json
{
  "error": {
    "code": "wrongPassword",
    "message": "The password is incorrect.",
    "field": "password"
  }
}
```

- `code`: stable machine-readable string. The frontend maps it to the
  translation key `auth.errors.<code>`, so keep the existing codes.
- `message`: human-readable fallback text (English).
- `field`: optional; the form field to highlight.

| HTTP | `code` | When |
|---|---|---|
| 400 | `validation` | Body or query failed validation (`field` is set) |
| 401 | `unauthorized` | Missing, invalid or expired access token |
| 401 | `noAccount` | Sign in: no account with this email |
| 401 | `wrongPassword` | Sign in: password does not match |
| 403 | `forbidden` | Authenticated but not allowed |
| 404 | `notFound` | Resource id does not exist |
| 409 | `exists` | Sign up: email already registered |
| 429 | `rateLimited` | Too many requests (send a `Retry-After` header) |
| 500 | `generic` | Unexpected server error |

> **Security note:** `noAccount` vs `wrongPassword` tells an attacker which
> emails are registered. The current UI uses both codes. Consider returning a
> single `invalidCredentials` code (and adding `auth.errors.invalidCredentials`
> to the translations) when the backend goes live.

---

## 2. Authentication

### Shared object: `User`

```json
{
  "id": "usr_8f3a2c",
  "name": "Aigerim Toktosunova",
  "email": "aigerim@example.com",
  "createdAt": "2026-10-09T08:15:00Z",
  "preferences": { "language": "ru", "theme": "dark" }
}
```

The frontend reads `user.name` (first word shown as "Hello, …") and
`user.email` on the account page.

### Shared object: `AuthTokens`

```json
{
  "accessToken": "eyJhbGciOi...",
  "refreshToken": "rt_2b1c...",
  "expiresIn": 900
}
```

`expiresIn` is the access token's lifetime in seconds. The refresh token can
instead be set as an `HttpOnly` cookie. If you do that, leave it out of the body.

---

### `POST /auth/register`

Create an account (page `/signup`). Signs the user in immediately.

**Request**

```json
{
  "name": "Aigerim Toktosunova",
  "email": "aigerim@example.com",
  "password": "correct-horse-42"
}
```

| Field | Type | Rules |
|---|---|---|
| `name` | string | required, trimmed, 1–100 chars |
| `email` | string | required, valid email, trimmed + lowercased by the server |
| `password` | string | required, min 8 chars (matches `auth.errors.passwordShort`) |

**Response `201 Created`**

```json
{
  "user": {
    "id": "usr_8f3a2c",
    "name": "Aigerim Toktosunova",
    "email": "aigerim@example.com",
    "createdAt": "2026-10-09T08:15:00Z",
    "preferences": { "language": "en", "theme": null }
  },
  "tokens": {
    "accessToken": "eyJhbGciOi...",
    "refreshToken": "rt_2b1c...",
    "expiresIn": 900
  }
}
```

**Errors:** `400 validation` (`field`: `name` | `email` | `password`), `409 exists` (`field`: `email`).

Store passwords with a slow hash (bcrypt or argon2), not the SHA-256 digest the
local-only version uses.

---

### `POST /auth/login`

Sign in (page `/login`).

**Request**

```json
{
  "email": "aigerim@example.com",
  "password": "correct-horse-42"
}
```

**Response `200 OK`**: same shape as `/auth/register` (`user` + `tokens`).

**Errors:** `400 validation`, `401 noAccount` (`field`: `email`), `401 wrongPassword` (`field`: `password`), `429 rateLimited`.

---

### `POST /auth/refresh`

Swap a refresh token for a new access token.

**Request**

```json
{ "refreshToken": "rt_2b1c..." }
```

(Leave the body empty if the refresh token is an `HttpOnly` cookie.)

**Response `200 OK`**

```json
{
  "accessToken": "eyJhbGciOi...",
  "refreshToken": "rt_9d4e...",
  "expiresIn": 900
}
```

**Errors:** `401 unauthorized` (refresh token invalid, expired or revoked).

---

### `POST /auth/logout` 🔒

Revoke the current refresh token ("Sign out" on `/account`).

**Request**

```json
{ "refreshToken": "rt_2b1c..." }
```

**Response `204 No Content`** (empty body)

---

### `GET /auth/me` 🔒

Return the signed-in user. Called on app load to restore the session.

**Request:** no body.

**Response `200 OK`**

```json
{
  "user": {
    "id": "usr_8f3a2c",
    "name": "Aigerim Toktosunova",
    "email": "aigerim@example.com",
    "createdAt": "2026-10-09T08:15:00Z",
    "preferences": { "language": "ru", "theme": "dark" }
  }
}
```

**Errors:** `401 unauthorized`.

---

### `POST /auth/forgot-password`

Start a password reset. The "Forgot password?" link currently points to
`/contact?topic=account`.

**Request**

```json
{ "email": "aigerim@example.com" }
```

**Response `202 Accepted`**: always the same, whether or not the email
exists:

```json
{ "message": "If an account exists for this email, a reset link has been sent." }
```

---

### `POST /auth/reset-password`

Finish a password reset with the token from the email.

**Request**

```json
{
  "token": "pr_51ab...",
  "password": "new-password-123"
}
```

**Response `200 OK`**: same shape as `/auth/login` (`user` + `tokens`).

**Errors:** `400 validation` (`field`: `password`), `400 invalidToken` (expired or already used).

---

### `POST /auth/oauth/{provider}` (planned)

The sign-in page shows Google and Apple buttons that currently display
"coming soon". `provider` is `google` or `apple`.

**Request**

```json
{ "idToken": "<ID token from Google Identity Services / Sign in with Apple>" }
```

**Response `200 OK`**: same shape as `/auth/login`. If this provider
created the account, the status is `201`.

---

## 3. Universities

### Shared object: `UniversitySummary`

The data used by the cards on `/universities` and in lists elsewhere.

```json
{
  "id": "auca",
  "name": "American University of Central Asia",
  "shortName": "AUCA",
  "city": "Bishkek",
  "country": "Kyrgyzstan",
  "type": "Private",
  "founded": 1993,
  "logo": "/images/universities/auca-logo.png",
  "photo": "/images/universities/auca-photo.jpg",
  "color": "#8B1E2D",
  "rank": 1,
  "languages": ["English"],
  "tuitionUsd": 6500,
  "tuitionNote": null,
  "majors": ["Business Administration", "Economics", "Software Engineering"],
  "ortScore": 150,
  "admissionWindow": { "open": "02-01", "close": "08-15" },
  "status": { "state": "open", "daysLeft": 12 },
  "isSaved": false
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | string | URL slug, used in `/universities/:id` |
| `country` | string | one of `Kyrgyzstan`, `Kazakhstan`, `Russia`, `Turkey`, `Hungary` |
| `type` | string | `Public` \| `Private` \| `Intergovernmental` |
| `logo`, `photo` | string \| null | absolute or site-relative image URL |
| `color` | string | hex brand color, used as the fallback logo background |
| `rank` | integer | sort order for "Top ranking" |
| `languages` | string[] | values from `English`, `Kyrgyz`, `Russian`, `Turkish` |
| `tuitionUsd` | number \| null | approximate yearly fee in USD; `0` = free, `null` = varies |
| `tuitionNote` | string \| null | optional caveat shown under the price |
| `ortScore` | integer \| null | competitive ORT estimate; `null` = ORT not used |
| `admissionWindow` | object \| null | yearly window; may wrap the new year (`"11-01"` → `"01-15"`) |
| `status` | object | **computed by the server** for today, see below |
| `isSaved` | boolean | only when a token is sent; otherwise omit or `false` |

`status` matches what `getAdmissionStatus()` in `src/utils/admission.js` returns:

```json
{ "state": "open", "daysLeft": 12 }
{ "state": "upcoming", "opensOn": "Feb 1, 2027", "daysUntil": 115 }
{ "state": "unknown" }
```

Compute it in the Asia/Bishkek time zone. It can also stay client-side,
since the frontend already has the function.

---

### `GET /universities` 🔓

Paginated, filtered, sorted list for `/universities`.

**Query parameters** (they match the page's URL params):

| Param | Type | Example | Meaning |
|---|---|---|---|
| `q` | string | `bishkek` | Case-insensitive search across `name`, `shortName`, `city`, `country`, `majors` |
| `country` | comma list | `Kyrgyzstan,Turkey` | Match any |
| `abroad` | boolean | `true` | Shortcut for every country except `Kyrgyzstan` (ignored if `country` is sent) |
| `lang` | comma list | `English,Russian` | Teaches in **any** of these languages |
| `type` | comma list | `Public,Private` | Match any |
| `min` | number | `1000` | `tuitionUsd >= min`. With `min` or `max` set, `null` tuition is excluded |
| `max` | number | `3000` | `tuitionUsd <= max` (`max=0` = free only) |
| `open` | `1` | `1` | Only universities whose window is open today |
| `saved` | `1` | `1` | 🔒 Only universities the user saved |
| `sort` | enum | `lowestTuition` | `topRanking` (default, `rank` asc) \| `lowestTuition` \| `highestTuition` \| `nameAZ` |
| `page` | integer | `2` | default `1` |
| `pageSize` | integer | `6` | default `6`, max `50` |

When sorting by tuition, `null` tuition always comes last, and ties fall back
to `rank`.

**Request:** no body.
`GET /universities?country=Kyrgyzstan&lang=English&sort=lowestTuition&page=1`

**Response `200 OK`**

```json
{
  "items": [
    {
      "id": "uca",
      "name": "University of Central Asia",
      "shortName": "UCA",
      "city": "Naryn",
      "country": "Kyrgyzstan",
      "type": "Intergovernmental",
      "founded": 2000,
      "logo": "/images/universities/uca-logo.jpg",
      "photo": "/images/universities/uca-photo.jpg",
      "color": "#1E3A5F",
      "rank": 2,
      "languages": ["English"],
      "tuitionUsd": 4500,
      "tuitionNote": "Most students receive need-based aid; many pay far less.",
      "majors": ["Computer Science", "Economics", "Communications & Media"],
      "ortScore": null,
      "admissionWindow": { "open": "10-01", "close": "03-31" },
      "status": { "state": "open", "daysLeft": 173 },
      "isSaved": true
    }
  ],
  "pagination": { "page": 1, "pageSize": 6, "totalItems": 9, "totalPages": 2 }
}
```

**Errors:** `400 validation` (unknown `sort`, non-numeric `min`/`max`), `401 unauthorized` (`saved=1` without a token).

---

### `GET /universities/filters`

Filter options and counts for the sidebar (the per-country numbers).

**Response `200 OK`**

```json
{
  "countries": [
    { "value": "Kyrgyzstan", "count": 14 },
    { "value": "Kazakhstan", "count": 1 },
    { "value": "Russia", "count": 1 },
    { "value": "Turkey", "count": 1 },
    { "value": "Hungary", "count": 1 }
  ],
  "languages": ["English", "Kyrgyz", "Russian", "Turkish"],
  "types": ["Public", "Private", "Intergovernmental"],
  "sortOptions": ["topRanking", "lowestTuition", "highestTuition", "nameAZ"],
  "tuitionRange": { "min": 0, "max": 6500 }
}
```

---

### `GET /universities/{id}` 🔓

Full detail for `/universities/:id`.

**Path:** `id`, the university slug, e.g. `auca`.

**Response `200 OK`**: all `UniversitySummary` fields, plus:

```json
{
  "id": "auca",
  "name": "American University of Central Asia",
  "shortName": "AUCA",
  "city": "Bishkek",
  "country": "Kyrgyzstan",
  "type": "Private",
  "founded": 1993,
  "website": "https://auca.kg",
  "logo": "/images/universities/auca-logo.png",
  "photo": "/images/universities/auca-photo.jpg",
  "color": "#8B1E2D",
  "rank": 1,
  "languages": ["English"],
  "tuitionUsd": 6500,
  "tuitionNote": null,
  "majors": ["Business Administration", "Economics", "Software Engineering", "Psychology"],
  "ortScore": 150,
  "admissionWindow": { "open": "02-01", "close": "08-15" },
  "status": { "state": "open", "daysLeft": 12 },
  "isSaved": false,

  "partnership": null,
  "about": "AUCA is a liberal arts university in Bishkek ...",
  "highlights": [
    "English-medium liberal arts education",
    "Dual-degree partnership with Bard College (USA)"
  ],
  "requirements": [
    { "label": "ORT", "value": "Required for Kyrgyz citizens" },
    { "label": "English", "value": "AUCA English test, or TOEFL / IELTS / Duolingo" }
  ],
  "steps": [
    { "title": "Create an online application", "text": "Apply on the AUCA admissions site ..." },
    { "title": "Take the English test", "text": "Sit the AUCA English Language Test ..." }
  ],
  "documents": [
    "ORT certificate",
    "Attestat (school leaving certificate) with transcript",
    "Passport copy"
  ],
  "aid": "Need-based financial aid and merit scholarships cover part of tuition ...",
  "faqs": [
    { "q": "Is there a dormitory?", "a": "AUCA offers limited dormitory places ..." }
  ],
  "contact": { "email": "admissions@auca.kg", "address": "7/6 Aaly Tokombaev St, Bishkek" },

  "scholarships": [
    {
      "id": "ugrad",
      "name": "Global UGRAD",
      "shortName": "UGRAD",
      "logo": "/images/scholarships/ugrad-logo.png",
      "color": "#0F766E",
      "fundingKey": "Fully Funded",
      "countryKey": "USA"
    }
  ],
  "similar": [
    { "id": "uca", "name": "University of Central Asia", "shortName": "UCA", "city": "Naryn", "logo": "/images/universities/uca-logo.jpg", "color": "#1E3A5F", "tuitionUsd": 4500 }
  ],
  "checklist": { "checked": ["ORT certificate", "Passport copy"] }
}
```

| Field | Notes |
|---|---|
| `partnership` | string \| null, shown for abroad universities with a Kyrgyzstan agreement |
| `contact` | `email` and `phone` are optional, `address` is usually present |
| `scholarships` | related scholarships, expanded from the id list in the data file |
| `similar` | up to 3 other universities in the **same country** |
| `checklist` | only with a token: the documents the user has ticked ([§7](#7-document-checklists)) |

**Errors:** `404 notFound`.

---

## 4. Scholarships

### Shared object: `ScholarshipSummary`

```json
{
  "id": "flex",
  "name": "FLEX Program",
  "shortName": "FLEX",
  "provider": "U.S. Department of State · American Councils",
  "typeKey": "High School",
  "fundingKey": "Fully Funded",
  "fundingColor": "success",
  "countryKey": "USA",
  "countryFlag": "🇺🇸",
  "meta": { "labelKey": "AGE", "value": "15–17" },
  "window": { "open": "08-15", "close": "10-15" },
  "status": { "state": "open", "daysLeft": 6 },
  "logo": null,
  "photo": "/images/scholarships/flex-photo.jpg",
  "color": "#1D4ED8",
  "summary": "Spend one school year in the United States ...",
  "isSaved": false
}
```

| Field | Type | Values |
|---|---|---|
| `typeKey` | enum | `High School` \| `Undergraduate` \| `Master's` \| `Bachelor/Master` |
| `fundingKey` | enum | `Fully Funded` \| `Stipend + Tuition` \| `Partial Funding` |
| `fundingColor` | enum | `success` \| `warning` \| `info` (badge color) |
| `countryKey` | enum | `USA` \| `Germany` \| `Europe` \| `Hungary` \| `UK` |
| `meta` | object | `labelKey`: `AGE` \| `LANGUAGE` \| `EXPERIENCE` \| `TYPE` \| `REQUIREMENT`; plus **either** `value` (literal text) **or** `valueKey` (translation key, e.g. `"2 Years Min"`) |
| `window` | object \| null | yearly application window, `"MM-DD"` |
| `status` | object | same shape as university `status` |

The `*Key` fields are translation keys (`scholarships.types.*`,
`scholarships.countries.*`, …). Keep them as stable English keys and do not
translate them on the server.

---

### `GET /scholarships` 🔓

List for `/scholarships`. Not paginated (the catalog is small); add
`page`/`pageSize` if it grows.

**Query parameters**

| Param | Type | Meaning |
|---|---|---|
| `q` | string | Search `name`, `shortName`, `provider`, `summary`, and the **translated** country and type labels in the request language |
| `destination` | enum | exact `countryKey` |
| `level` | enum | `typeKey`. `Bachelor/Master` programs also match `level=Undergraduate` and `level=Master's` |
| `funding` | enum | exact `fundingKey` |
| `saved` | `1` | 🔒 only saved scholarships |

**Request:** no body. `GET /scholarships?level=Master's&funding=Fully%20Funded`

**Response `200 OK`**

```json
{
  "items": [
    {
      "id": "chevening",
      "name": "Chevening Scholarships",
      "shortName": "Chevening",
      "provider": "UK Foreign, Commonwealth & Development Office",
      "typeKey": "Master's",
      "fundingKey": "Fully Funded",
      "fundingColor": "success",
      "countryKey": "UK",
      "countryFlag": "🇬🇧",
      "meta": { "labelKey": "EXPERIENCE", "valueKey": "2 Years Min" },
      "window": { "open": "08-05", "close": "11-04" },
      "status": { "state": "upcoming", "opensOn": "Aug 5, 2027", "daysUntil": 300 },
      "logo": "/images/scholarships/chevening-logo.png",
      "photo": "/images/scholarships/chevening-photo.jpg",
      "color": "#7F1D1D",
      "summary": "The UK government's fully funded one-year master's scholarship for future leaders.",
      "isSaved": false
    }
  ],
  "total": 1
}
```

---

### `GET /scholarships/filters`

**Response `200 OK`**

```json
{
  "destinations": ["USA", "Germany", "Europe", "Hungary", "UK"],
  "levels": ["High School", "Undergraduate", "Master's", "Bachelor/Master"],
  "funding": ["Fully Funded", "Stipend + Tuition", "Partial Funding"]
}
```

---

### `GET /scholarships/{id}` 🔓

Full detail for `/scholarships/:id`.

**Response `200 OK`**: all `ScholarshipSummary` fields, plus:

```json
{
  "id": "flex",
  "name": "FLEX Program",
  "...": "all ScholarshipSummary fields",

  "officialUrl": "https://flex.americancouncils.org",
  "about": "The Future Leaders Exchange (FLEX) program is funded by ...",
  "covers": ["Round-trip travel to the USA", "Living with a host family", "Monthly allowance"],
  "eligibility": ["Citizen of Kyrgyzstan (or another participating country)", "Currently enrolled in secondary school"],
  "steps": [
    { "title": "Round 1 — short English test", "text": "Open testing sessions are held in cities across Kyrgyzstan in autumn." }
  ],
  "documents": ["Passport or birth certificate", "School transcripts (last 3 years)", "Parent/guardian permission"],
  "tips": ["Testing is free — never pay anyone for \"guaranteed\" selection."],
  "faqs": [
    { "q": "Is there an application fee?", "a": "No. FLEX is fully funded and applying is free." }
  ],
  "universities": [
    { "id": "auca", "name": "American University of Central Asia", "shortName": "AUCA", "logo": "/images/universities/auca-logo.png", "color": "#8B1E2D", "city": "Bishkek", "country": "Kyrgyzstan" }
  ],
  "checklist": { "checked": ["Passport or birth certificate"] }
}
```

`universities` is the expanded list of partner universities (it can be empty).
`checklist` is included only when a token is sent.

**Errors:** `404 notFound`.

---

## 5. ORT practice

### `GET /ort/questions`

Questions for the diagnostic quiz on `/ort-prep?tab=practice`.
**Do not send `answer` or `explanation`**, otherwise the client can see the
answers. The server reveals them when the attempt is submitted.

**Query parameters**

| Param | Type | Meaning |
|---|---|---|
| `subject` | enum | `all` (default) \| `math` \| `verbal` |

**Response `200 OK`**

```json
{
  "subject": "math",
  "items": [
    {
      "id": "m1",
      "subject": "math",
      "prompt": "If 3x − 7 = 11, what is the value of x?",
      "options": ["4", "5", "6", "7"]
    },
    {
      "id": "m2",
      "subject": "math",
      "prompt": "A jacket costs 2,400 som after a 20% discount. What was the original price?",
      "options": ["2,880 som", "3,000 som", "2,600 som", "3,200 som"]
    }
  ],
  "counts": { "all": 10, "math": 5, "verbal": 5 }
}
```

`counts` provides the "N questions" label shown next to each subject.

---

### `POST /ort/attempts` 🔓

Submit answers. The server grades them, stores the result for a signed-in
user, and returns the review screen data. Without a token the attempt is
graded but not stored.

**Request**

```json
{
  "subject": "math",
  "answers": {
    "m1": 2,
    "m2": 0,
    "m3": 3,
    "m4": 0,
    "m5": 2
  }
}
```

| Field | Type | Rules |
|---|---|---|
| `subject` | enum | `all` \| `math` \| `verbal` |
| `answers` | object | question id → chosen option index (0-based). Every question in the subject must be answered |

**Response `201 Created`**

```json
{
  "attempt": {
    "id": "att_71c0",
    "date": "2026-10-09T08:40:12Z",
    "subject": "math",
    "correct": 4,
    "total": 5,
    "percent": 80
  },
  "review": [
    {
      "id": "m1",
      "prompt": "If 3x − 7 = 11, what is the value of x?",
      "options": ["4", "5", "6", "7"],
      "answer": 2,
      "chosen": 2,
      "isCorrect": true,
      "explanation": "3x = 11 + 7 = 18, so x = 18 ÷ 3 = 6."
    },
    {
      "id": "m2",
      "prompt": "A jacket costs 2,400 som after a 20% discount. What was the original price?",
      "options": ["2,880 som", "3,000 som", "2,600 som", "3,200 som"],
      "answer": 1,
      "chosen": 0,
      "isCorrect": false,
      "explanation": "2,400 is 80% of the original price: 2,400 ÷ 0.8 = 3,000 som."
    }
  ],
  "feedback": "high"
}
```

`feedback` is `high` (≥ 80%), `mid` (≥ 50%) or `low`, and maps to
`ortPractice.feedbackHigh|Mid|Low`.

**Errors:** `400 validation` (unknown question id, index out of range, missing answers).

---

### `GET /me/ort/attempts` 🔒

History for the Progress tab and the account page. The frontend keeps the
last 20 attempts; the server can keep more and return them paginated.

**Query parameters:** `limit` (default `20`, max `100`), `subject` (optional filter).

**Response `200 OK`**

```json
{
  "items": [
    { "id": "att_71c0", "date": "2026-10-09T08:40:12Z", "subject": "math", "correct": 4, "total": 5, "percent": 80 },
    { "id": "att_6b22", "date": "2026-10-02T17:03:44Z", "subject": "all", "correct": 6, "total": 10, "percent": 60 }
  ],
  "stats": {
    "count": 2,
    "bestPercent": 80,
    "lastAttemptAt": "2026-10-09T08:40:12Z"
  }
}
```

`stats.bestPercent` is `null` when there are no attempts. The account page
shows "Start diagnostic" in that case.

---

## 6. Saved items

`kind` is `universities` or `scholarships`.

### `GET /me/saved` 🔒

Everything the user has saved. Used by `/account`.

**Response `200 OK`**

```json
{
  "universities": [
    { "id": "auca", "name": "American University of Central Asia", "shortName": "AUCA", "logo": "/images/universities/auca-logo.png", "color": "#8B1E2D", "savedAt": "2026-10-01T12:00:00Z" }
  ],
  "scholarships": [
    { "id": "flex", "name": "FLEX Program", "shortName": "FLEX", "logo": null, "color": "#1D4ED8", "savedAt": "2026-10-03T09:30:00Z" }
  ]
}
```

---

### `GET /me/saved/{kind}` 🔒

Just the ids (cheap, used for the heart/bookmark state and the
"Saved only" count).

**Response `200 OK`**

```json
{ "ids": ["auca", "uca"] }
```

---

### `PUT /me/saved/{kind}/{id}` 🔒

Save an item. Idempotent: saving an item twice is not an error.

**Request:** no body.

**Response `200 OK`**

```json
{ "id": "auca", "saved": true, "savedAt": "2026-10-09T08:45:00Z" }
```

**Errors:** `404 notFound` (no such university or scholarship).

---

### `DELETE /me/saved/{kind}/{id}` 🔒

Unsave an item. Idempotent.

**Response `200 OK`**

```json
{ "id": "auca", "saved": false }
```

---

## 7. Document checklists

The "Documents" section on detail pages lets users tick off documents.
`kind` is `universities` or `scholarships`. Items are identified by the
document's text, as they are today. If documents get edited often, switch to
stable document ids.

### `GET /me/checklists/{kind}/{id}` 🔒

**Response `200 OK`**

```json
{
  "kind": "universities",
  "id": "auca",
  "checked": ["ORT certificate", "Passport copy"],
  "done": 2,
  "total": 6,
  "updatedAt": "2026-10-08T19:22:10Z"
}
```

`done` only counts checked items that are still in the current `documents`
list (same rule as the UI).

---

### `PUT /me/checklists/{kind}/{id}` 🔒

Replace the full checked list. The UI toggles one checkbox at a time; sending
the whole array keeps it simple and idempotent.

**Request**

```json
{ "checked": ["ORT certificate", "Passport copy", "Photos 3×4 cm"] }
```

| Field | Rules |
|---|---|
| `checked` | string[]; every value must be in the item's `documents` list; duplicates removed |

**Response `200 OK`**: same shape as `GET`.

**Errors:** `400 validation` (unknown document), `404 notFound`.

---

## 8. User preferences

### `PATCH /me/preferences` 🔒

Sync language and theme across devices. Guests keep using localStorage.

**Request** (send only the fields that change)

```json
{
  "language": "ru",
  "theme": "dark"
}
```

| Field | Values |
|---|---|
| `language` | `en` \| `ru` \| `ky` |
| `theme` | `light` \| `dark` \| `null` (follow the system setting) |

**Response `200 OK`**

```json
{ "preferences": { "language": "ru", "theme": "dark" } }
```

---

### `PATCH /me` 🔒

Update profile.

**Request**

```json
{ "name": "Aigerim T." }
```

**Response `200 OK`**: `{ "user": { ...User } }`

---

### `POST /me/password` 🔒

Change password.

**Request**

```json
{ "currentPassword": "correct-horse-42", "newPassword": "even-better-pass-7" }
```

**Response `204 No Content`**

**Errors:** `401 wrongPassword` (`field`: `currentPassword`), `400 validation` (`field`: `newPassword`).

---

### `DELETE /me` 🔒

Delete the account and all its data (saved items, checklists, ORT attempts).

**Request**

```json
{ "password": "correct-horse-42" }
```

**Response `204 No Content`**

---

### `POST /me/import` 🔒 (one-time migration)

Moves data a user built up in localStorage before the backend existed. Call
it once after the first sign-in on a device that has local data, then clear
those keys.

**Request**

```json
{
  "saved": {
    "universities": ["auca", "uca"],
    "scholarships": ["flex"]
  },
  "checklists": {
    "universities": { "auca": ["ORT certificate", "Passport copy"] },
    "scholarships": { "flex": ["Passport or birth certificate"] }
  },
  "ortResults": [
    { "date": "2026-09-20T10:00:00.000Z", "subject": "all", "correct": 7, "total": 10 }
  ],
  "preferences": { "language": "ky", "theme": "light" }
}
```

The fields map to these localStorage keys: `bilge.saved.*`,
`bilge.checklist.<id>` / `bilge.checklist.sch.<id>`, `bilge.ort.results`,
`bilge.lang`, `bilge.theme`.

**Response `200 OK`**

```json
{
  "imported": { "savedUniversities": 2, "savedScholarships": 1, "checklists": 2, "ortAttempts": 1 },
  "skipped": { "unknownIds": [] }
}
```

The import merges with existing data. Unknown ids are skipped, not treated as errors.

---

## 9. Contact

### `POST /contact` 🔓

Replaces the `mailto:` handoff on `/contact`. If a token is sent, the server
links the message to the user.

**Request**

```json
{
  "name": "Aigerim Toktosunova",
  "email": "aigerim@example.com",
  "topic": "account",
  "message": "I forgot my password and cannot sign in."
}
```

| Field | Rules |
|---|---|
| `name` | required, 1–100 chars |
| `email` | required, valid email |
| `topic` | `general` \| `account` \| `universities` \| `feedback` |
| `message` | required, 10–5000 chars |

**Response `201 Created`**

```json
{
  "id": "msg_a91f",
  "receivedAt": "2026-10-09T08:50:00Z",
  "message": "Thanks — we'll reply to aigerim@example.com."
}
```

**Errors:** `400 validation`, `429 rateLimited`. Add a captcha or honeypot field if spam becomes a problem.

---

## 10. Legal pages

### `GET /legal/{doc}`

`doc` is `terms`, `privacy` or `cookies`. Supports `?lang=`.

**Response `200 OK`**

```json
{
  "doc": "privacy",
  "title": "Privacy Policy",
  "updated": "September 2026",
  "sections": [
    { "heading": "What we store", "body": "..." },
    { "heading": "Passwords", "body": "..." }
  ]
}
```

**Errors:** `404 notFound`.

> The current privacy, terms and cookie texts say "BILGE does not run a
> server" and "data never leaves your device". **Rewrite them before
> launching the backend.**

---

## 11. Endpoint summary

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/auth/register` | — | Create account |
| POST | `/auth/login` | — | Sign in |
| POST | `/auth/refresh` | — | New access token |
| POST | `/auth/logout` | 🔒 | Sign out |
| GET | `/auth/me` | 🔒 | Current user |
| POST | `/auth/forgot-password` | — | Send reset email |
| POST | `/auth/reset-password` | — | Set new password with token |
| POST | `/auth/oauth/{provider}` | — | Google / Apple sign-in (planned) |
| GET | `/universities` | 🔓 | List + filter + sort + paginate |
| GET | `/universities/filters` | — | Filter options and counts |
| GET | `/universities/{id}` | 🔓 | University detail |
| GET | `/scholarships` | 🔓 | List + filter |
| GET | `/scholarships/filters` | — | Filter options |
| GET | `/scholarships/{id}` | 🔓 | Scholarship detail |
| GET | `/ort/questions` | — | Practice questions (no answers) |
| POST | `/ort/attempts` | 🔓 | Grade (and save) an attempt |
| GET | `/me/ort/attempts` | 🔒 | Attempt history + best score |
| GET | `/me/saved` | 🔒 | All saved items (expanded) |
| GET | `/me/saved/{kind}` | 🔒 | Saved ids |
| PUT | `/me/saved/{kind}/{id}` | 🔒 | Save |
| DELETE | `/me/saved/{kind}/{id}` | 🔒 | Unsave |
| GET | `/me/checklists/{kind}/{id}` | 🔒 | Document checklist |
| PUT | `/me/checklists/{kind}/{id}` | 🔒 | Update checklist |
| PATCH | `/me` | 🔒 | Update profile |
| PATCH | `/me/preferences` | 🔒 | Language / theme |
| POST | `/me/password` | 🔒 | Change password |
| DELETE | `/me` | 🔒 | Delete account |
| POST | `/me/import` | 🔒 | Migrate localStorage data |
| POST | `/contact` | 🔓 | Contact form |
| GET | `/legal/{doc}` | — | Terms / privacy / cookies |

🔒 token required · 🔓 token optional (adds user-specific fields) · — public
