# Fitbit Heart Rate Explorer (Google Health API)

A single-page local web app (`index.html`) that fetches your sampled heart rate
(beats per minute) from the **Google Health API** — the successor to the Fitbit
Web API — lets you download it as CSV, and renders it in interactive charts.

The extracted fields are: **device name**, **civil time** (your local wall-clock
time), and **sampled BPM**.

## 1. Run a local web server

Google OAuth requires the app to be served from a registered origin, so open the
page through a local HTTP server rather than as a `file://` URL.

From the directory containing `index.html`, run one of:

```bash
# Python 3
python3 -m http.server 8000

# or Node.js
npx serve -l 8000
```

Then browse to <http://localhost:8000/>. Keep the port stable — it must match
the origin you register with Google below.

## 2. Register with Google for API access

1. **Create a Google Cloud project.** Go to <https://console.cloud.google.com/>,
   sign in with the same Google account that owns your Fitbit data, and create a
   project (e.g. `heart-rate-explorer`).
2. **Enable the Health API.** In *APIs & Services → Library*, search for
   **Google Health API** (service name `health.googleapis.com`) and click
   **Enable**. If Google requires additional Health API developer enrollment or
   terms acceptance for your account, complete that flow when prompted — see
   <https://developers.google.com/health/get-started>.
3. **Configure the OAuth consent screen and register the scope.** In
   *APIs & Services → OAuth consent screen* (Google Auth Platform):
   - User type: **External**, then fill in the app name and your email.
   - **Register the scope (required!):** open the **Data Access** page and
     click **Add or remove scopes**. In the API column, search for
     **Google Health API**, check
     `https://www.googleapis.com/auth/googlehealth.health_metrics_and_measurements.readonly`
     (this is the scope that covers the `heart-rate` data type), then press
     **Update** and **Save**. Also add
     `https://www.googleapis.com/auth/googlehealth.activity_and_fitness.readonly`,
     which is the scope the **steps** data type lives under — the app needs
     both. All Health API scopes are classified as *Restricted*; if a scope is
     not registered here, the API rejects your token with
     `403 DISALLOWED_OAUTH_SCOPES` even though the OAuth consent popup
     succeeds. If no Health API scopes appear in the picker, the Health API
     isn't enabled yet — do step 2 first.
     Both scopes are in the `googlehealth` family, so they ride on a single
     token safely; what the API rejects is a token mixing in scopes from
     *outside* that family (such as the legacy `fitness.*` scopes).
   - **Test users:** while the app is in *Testing* mode, add your own Google
     account as a test user. (For personal use you can stay in Testing mode
     forever; you don't need Google verification.)
4. **Create an OAuth 2.0 Client ID.** In *APIs & Services → Credentials →
   Create credentials → OAuth client ID*:
   - Application type: **Web application**
   - Authorized JavaScript origins: `http://localhost:8000` (match your server
     port exactly; no trailing slash, no path)
   - No redirect URI is needed — the app uses the Google Identity Services
     token flow, which only checks the JavaScript origin.
5. Copy the generated **Client ID** (ends in `.apps.googleusercontent.com`).

## 3. Use the app

1. Open <http://localhost:8000/> and click **Grant access** (your Client ID is
   already built into the page). A single Google popup asks you to approve the
   heart rate scope. Optionally, type an email address into the **Email for CSV
   filename** box — it's used only to name the downloaded file and is never sent
   anywhere.
2. Pick your **time zone** (defaults to your browser's zone) and a **start/end**
   date-time. You enter civil (wall-clock) time; the app converts it to
   absolute physical time in that zone before querying, because the API filters
   on RFC 3339 physical timestamps.
3. Click **Fetch heart rate**. The app queries
   `GET https://health.googleapis.com/v4/users/me/dataTypes/heart-rate/dataPoints`
   with `pageSize=10000` (the API maximum) and follows `nextPageToken` until the
   range is complete, then does the same for
   `dataTypes/steps/dataPoints`. The API caps heart rate queries at **14 days**,
   which the app enforces before sending the request. If the steps call fails
   (usually because the activity scope hasn't been granted yet), the heart rate
   data is kept and only the activity classification is skipped.
4. **Download CSV** saves the data with columns
   `device_name, civil_time, beats_per_minute, steps_per_min, state`, where
   `state` is empty, `exercise`, or `stress`. If you typed an email into the
   filename box, it's folded into a filename-safe suffix (characters other than
   letters, digits, `.`, `_`, `-` become `_`), e.g.
   `heart_rate_jane.doe_gmail.com.csv`; otherwise it's just `heart_rate.csv`.
5. The **summary chart** shows a 10th–90th percentile band per 5-minute
   interval (red where the whole band is above 100 bpm, green where it is below
   90 bpm). A shaded indicator block marks the span shown in the **detail
   chart** (two hours by default). Drag inside the block to pan the window,
   drag either edge handle to move just that edge — widening or narrowing the
   window down to a 10-minute minimum — or click anywhere else to jump the
   window there. Date boundaries are marked by subtle background shading with
   a single label per date.

## Exercise vs. stress classification

Elevated heart rate has (at least) two very different causes, so the app uses
the steps stream to tell them apart.

1. **Cadence.** Each steps interval's count is spread across the minutes it
   covers, giving steps-per-minute.
2. **Bouts.** A minute at **50+ steps/min** counts as *active*. Consecutive
   active minutes form a bout; lulls of up to 2 minutes are bridged, and bouts
   shorter than 3 minutes are discarded so that walking to the kitchen doesn't
   register as exercise.
3. **Elevated runs.** Each minute's *median* bpm (robust against single
   spurious spikes) is compared to **100 bpm**. Consecutive elevated minutes
   form a run; dips of up to 2 minutes are bridged and runs under 2 minutes are
   dropped.
4. **Attribution.** Each bout projects an *influence window*: the bout itself,
   plus 2 minutes before (heart rate leads visible movement) and **15 minutes
   after** (it stays up during recovery). The part of an elevated run inside
   any influence window is **exercise**; any part outside every window is
   **stress** — the heart rate rose with no movement to explain it.

Because attribution is per-portion rather than per-run, a single run can split:
a workout that keeps you above 100 bpm for 40 minutes is *exercise* through the
recovery window and *stress* thereafter. That boundary is the main judgement
call in the scheme — if you'd rather attribute long cool-downs entirely to
exercise, raise `RECOVERY`.

All of the thresholds above are constants at the top of the `<script>` block
(`ACTIVE_CADENCE`, `MIN_BOUT`, `BOUT_BRIDGE`, `LEAD_IN`, `RECOVERY`,
`ELEVATED_BPM`, `ELEVATED_BRIDGE`, `MIN_ELEVATED`) and are meant to be tuned to
your own physiology.

### Reading the charts

- A **lane beneath both charts** shows movement as a pale blue underlay, with
  classified elevated episodes on top: solid blue for exercise, red for stress.
- The **detail chart** additionally washes the background blue or red across
  each episode, and draws steps-per-minute as faint bars along the bottom with
  a dashed line at the 50 steps/min activity threshold.
- The **daily summary** table reports, per civil day: total steps, *Active*
  time (time spent moving), *Elevated (exercise)*, and *Elevated (stress)*.

Note the distinction between *Active* and *Elevated (exercise)*: a gentle walk
can be active without ever pushing your heart rate past 100 bpm, so the two
columns are deliberately not the same number.

## Implementation notes

- **Protocol-buffer zero omission:** the API's JSON omits fields whose value is
  zero. A civil time of `07:00:00` arrives as `{"hours": 7}`, and midnight can
  arrive with no `time` object at all. The app fills in missing
  `hours`/`minutes`/`seconds` (and, if `civilTime` is entirely absent, derives
  it from `physicalTime` + `utcOffset`, where a missing `utcOffset` means UTC).
- **BPM** is transported as an int64 *string* (`"beatsPerMinute": "72"`); the
  app parses it to a number.
- Access tokens live only in page memory and expire after about an hour;
  re-grant if a fetch returns 401.

## Hosting it online (e.g. GitHub Pages)

Because the app is a single static file with no backend, you can host it so it's
usable from anywhere without a local server:

1. Push `index.html` to a repo and enable **GitHub Pages**. It'll be served at
   `https://<username>.github.io/<repo>/`.
2. In the Cloud console, add `https://<username>.github.io` as an **Authorized
   JavaScript origin**. Origins are scheme + host only (no path), so use the
   bare `github.io` origin — not the `/<repo>/` path.
3. Hardcode your client ID: set the `CLIENT_ID` constant near the top of
   `index.html`. When set, the app uses it automatically and hides the input
   field, so you never paste it again.
4. Keep the OAuth app in **Testing** mode with yourself as a test user. All
   Health scopes are *Restricted*; they work fine in Testing mode for listed
   test users. Moving to **Production** for public use would trigger Google
   verification, which for restricted scopes includes a paid annual third-party
   security assessment — unnecessary for a personal tool.
5. Once you're hosting for real, remove `http://localhost:8000` from the
   authorized origins (or use a separate client ID for local development) so
   nobody can run a copy under your project.

### Is it safe to put the client ID in the page?

Yes. The client **ID** is public by design and is meant to live in frontend
code; only the client **secret** must stay private, and this app has no secret
(it uses the browser-based public-client flow with PKCE). A copied client ID
can't be used from an origin you didn't authorize, and even from an authorized
origin every user only ever receives tokens for *their own* Google account —
your data is never exposed. The worst a leaked client ID enables is someone
consuming your project's API quota or showing your app's name on their consent
screen.

## Troubleshooting

- **`redirect_uri_mismatch` / `origin_mismatch`:** the page's origin (scheme +
  host + port) must exactly match an Authorized JavaScript origin.
- **400 `INVALID_DATA_POINT_FILTER_DATA_TYPE_RESTRICTION`:** the data type in
  the `filter` string must be snake_case (`heart_rate.sample_time.…`), even
  though the URL path uses kebab-case (`dataTypes/heart-rate`).
- **403 `DISALLOWED_OAUTH_SCOPES` ("Request contains disallowed OAuth
  scope(s)"):** the Health API rejects any token carrying scopes it doesn't
  allow. In order of likelihood:
  1. *Scope never registered:* register it on the *Data Access* page (setup
     step 3), then grant again for a fresh token.
  2. *Token contaminated with old scopes:* if this client ID was ever granted
     other scopes (legacy Google Fit `fitness.*`, or an earlier misspelled
     health scope), incremental authorization unions them into every new
     token and the API rejects the mix. The app disables
     `include_granted_scopes` and forces `prompt=consent`, and after granting
     it prints the token's actual scopes in the status line — if anything
     besides the single `googlehealth` scope appears, revoke the app entirely
     at myaccount.google.com → Security → Third-party apps, then grant again.
  3. *Cross-project mismatch:* the OAuth client ID must belong to the **same**
     Cloud project where the Health API is enabled and the scope is
     registered. Check the project selector at the top of the Credentials
     page against the one you configured.
- **403 "Could not mint UberMint from GaiaMint":** you signed in with a legacy
  Fitbit account. Sign in with a Google Account (migrate the Fitbit account
  first if needed).
- **Other 403 / permission denied:** confirm the Health API is enabled and
  your account is listed as a test user.
- **Empty results:** your Fitbit must be linked to this Google account and
  syncing; try a recent, small time range first.
