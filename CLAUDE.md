# SasTown Hub (static homepage)

GitHub Pages static site for `sastownhub.co.za` (CNAME → `www.sastownhub.co.za`).
This used to be the original multi-page SasTown Hub directory site (business
listings hardcoded directly in HTML across many pages); it's now reduced to a
single static homepage. The real product — ASP.NET Core + React + Postgres,
plus Android apps — lives in a separate repo, `c:\Sastown\SasTownHub`, under a
different GitHub account/org (`aethelst8`, not this repo's `KMoloi8`). Don't
confuse the two when committing/pushing.

## Current state (2026-10-28)

**The backend is down** (Oracle Cloud account/capacity problem, not a code
or deploy issue) **and this repo's GitHub Pages site is back to being what
`sastownhub.co.za` actually serves** — the reverse of the situation the
2026-08-20 entry below describes. That entry is now historical: the DNS
cutover it describes was reverted 2026-10-06 and hasn't been reinstated.
Full story in the 2026-10-06..10-28 session log below; short version:

- Oracle disabled the account shortly after a routine "Free Trial ended"
  email, then (after sitting unactioned ~2 weeks) terminated the compute
  instance outright. The boot volume and a manual backup of it survived
  and still exist in-region. Recreating an instance from that boot volume
  has been blocked for weeks by Ampere A1 capacity exhaustion in
  af-johannesburg-1's one and only AD — a hardware-availability problem,
  not something fixable from this end beyond retrying.
- `index.html` here now shows a maintenance notice with a Google Play link
  (not the old "in development" placeholder), and apex DNS was pointed
  back at GitHub Pages' IPs as the stopgap.
- A new static `/privacy` page was added here 2026-10-28, independent of
  backend status, to fix a Google Play Data Safety compliance deadline
  (the declared privacy/delete-data URL was 404ing because of the outage).

**Open, not yet decided**: how to actually fix the backend long-term — keep
manually retrying Oracle, pay for a stopgap VPS, or migrate off
Oracle+self-hosted-Postgres to Supabase + a free app host. Kabelo is
thinking this over during the week of 2026-10-28; don't assume a direction
or act on any of these without asking. Also still open from before:
what to do with this repo/the `www` subdomain long-term once the backend
situation resolves one way or another.

## History worth knowing

- All subpages (`businesses.html`, `taxi.html`, `schools.html`, every niche
  category page, etc.) were deleted 2026-07-24 — deliberate cleanup, not
  accidental loss. Trigger: `schools.html` had fabricated placeholder school
  data surfacing in a Google search snippet as if real.
- The original listing content isn't lost — it was extracted into
  `SasTownHub`'s `seed-data/categories.json` / `seed-data/listings.json`
  before deletion, filtered to exclude the fabricated/placeholder entries
  (schools, and several fake "business" cards using `via.placeholder.com`
  images). A separate untouched checkout of the old multi-page site is kept
  at `C:\Sastown\sastown-main` for reference if the extraction ever needs
  re-checking.
- The image files still sitting in this repo's root (`kumi.jpg`, `build.png`,
  etc.) are leftovers from the old multi-page site. Only `logo192.png` is
  still actually referenced (as the favicon) — the rest are orphaned but
  harmless; don't assume they're wired up to anything.
- `logo192admin.png` / `logo512admin.png` (untracked in this repo) are
  source assets for the separate `admin-mobile` app's launcher icon — dropped
  here at some point for convenience, not part of this site's content.

## Working conventions

- This repo's remote (`KMoloi8/sastown`) is a different GitHub account than
  `SasTownHub`'s (`aethelst8/sastown`) — same repo name, different owner,
  easy to mix up when both are open at once.
- Don't auto-commit or offer to commit; wait for an explicit ask (same
  convention as `SasTownHub`).

## Session log — 2026-08-08

Most of this session's work happened in the `SasTownHub` repo, not this one —
recorded here too since it's the context that explains the "don't re-add a
redirect" decision above and everything currently in progress.

**The problem**: `sastownhub.azurewebsites.net` started showing a generic
error page, and every closed tester's app showed "Something went wrong on
our end." Root cause, confirmed via Azure logs and a live SSMS test: Azure
SQL's free serverless tier hit its monthly compute allowance and
hard-paused the database for the rest of August (renews 2026-09-01 UTC).
Unlike a normal idle auto-pause, it doesn't resume on reconnect — only
Azure Portal's "continue with additional charges" toggle or the September
reset restores access. Kabelo didn't want to pay, and didn't want this
class of billing-cliff risk recurring.

**Decisions made**:
- Drop Azure entirely (App Service + Azure SQL) rather than patch around it.
- Self-host on an Oracle Cloud "Always Free" VM, Docker Compose stack, with
  `sastownhub.co.za` pointed directly at it instead of the `azurewebsites.net`
  URL — this is why this repo's redirect was removed (see above).
- Swap SQL Server → Postgres (Npgsql) in `SasTownHub`, discovered mid-plan:
  Microsoft's SQL Server Linux container is x86-64 only (unsupported under
  emulation), Azure SQL Edge (the ARM option) was retired 2025-09-30, and
  Oracle's free compute is ARM64-only. Postgres has full native ARM64 support.
- Data recovery solved itself: the ~27 real listings don't need pulling from
  the paused Azure DB at all — `SasTownHub`'s `seed-data/` already has them
  (extracted from this repo's old HTML before the 2026-07-24 deletion), and
  the existing `SeedImporter`'s placeholder-exclusion list was cross-checked
  against the raw HTML and confirmed correct. Accepted gap: anything
  submitted live through the app *after* launch (e.g. "electrical
  Enterprise") isn't in that data and won't come back this way - Kabelo chose
  to accept that rather than wait for the Azure reset or pay to unblock early.

**What's been done** (full detail in `SasTownHub`'s own `CLAUDE.md`):
EF Core provider swapped to Npgsql with a fresh Postgres-native migration;
`Dockerfile`/`docker-compose.yml`/`nginx/` config written for the VM;
CI/CD reworked to SSH-deploy instead of Azure zip-deploy;
`tools/SasTownHub.IpWhitelist` deleted (obsolete); `mobile`/`admin-mobile`
API host changed to `sastownhub.co.za`; a `MAINTENANCE_MODE` gate added to
the Android app (shows only a yellow "in development" screen matching this
site's homepage, no navigation, regardless of cached data) since devices
with previously-cached data were still showing a normal-looking but
non-functional home screen; this repo's `index.html` rewritten to match.

**Outstanding**:
1. Blocked on Oracle - tenancy still initializing (support said 48 hours).
2. Once the VM exists: Docker setup, bring up the stack, run migrations +
   SeedImporter for real (first genuine live-apply test), point DNS at it,
   get the Let's Encrypt cert, verify in a real browser.
3. Set the GitHub Actions secrets so `SasTownHub`'s CI/CD stops failing.
4. Flip `MAINTENANCE_MODE` back to `false` once the new hosting is verified
   live, rebuild and redistribute the Android app.
5. Decommission the Azure resources (App Service, SQL server, resource
   group) as an explicit separate step - not done yet, still sitting there.
6. Upload the current AAB (`versionCode 5`/"1.4") to Play Console.
7. Some `SasTownHub` changes (the maintenance-mode gate, message wording,
   version bumps) and this file were still uncommitted as of this log entry.

## Session log — 2026-08-20 (migration completed)

Closes out everything the 2026-08-08 log left outstanding. Full step-by-step
detail (OCI console gotchas, the account/deploy-key mix-ups, the Dockerfile
restore bug, etc.) is in `SasTownHub`'s own project memory, not repeated
here — this is the summary relevant to this repo specifically.

- Oracle "Always Free" VM created (Ampere A1, Ubuntu 24.04 ARM64), Docker +
  Compose stack (Postgres/app/nginx) brought up, migrations applied and
  `SeedImporter` run for real (24 categories, 26 listings, 32 placeholders
  correctly excluded) — the first genuine live-apply test of the whole
  Postgres rebuild.
- DNS cutover done: apex `A` record repointed from GitHub Pages' three IPs
  to the VM's `145.241.99.203` (this is the change that makes this repo's
  own `index.html` unreachable at the bare domain now — see "Current state"
  above). Let's Encrypt TLS obtained and verified; HTTP redirects to HTTPS.
- `SasTownHub`'s GitHub Actions secrets set, CI/CD confirmed working
  end-to-end via a real triggered deploy.
- Azure fully decommissioned: `sastownhub-rg` (App Service, Plan, SQL
  Server + both databases) deleted, confirmed gone; the now-dead
  `AZURE_WEBAPP_PUBLISH_PROFILE` GitHub secret removed too.
- Android `MAINTENANCE_MODE` flipped back to `false` (versionCode 6/"1.5"),
  since the API host it already pointed at (`https://sastownhub.co.za`) is
  now actually live. Built, committed, pushed. Kabelo still needs to upload
  the new AAB to Play Console's closed-testing track himself and post a
  follow-up to the tester Google Group (drafted and sent by Kabelo this
  session, mirroring the tone of the original "we're migrating" message).
- **Nothing outstanding from the original migration punch list** as of this
  entry. The only open question is the "what happens to this repo/`www`
  now" one noted in "Current state" above.

## Session log — 2026-08-24 (Play Store production application)

Also entirely `SasTownHub`-side work (the Android app), recorded here for
continuity since it closes out the last item the 2026-08-20 log left open
(uploading the AAB). Full detail lives in `SasTownHub`'s own memory.

- The versionCode 6/"1.5" AAB (the maintenance-mode-lifted build from
  2026-08-20) was uploaded to the Play Console closed-testing track.
- Kabelo submitted Google Play's "Apply for access to production" form the
  same day (08:23, status "reviewing your application form" — Google's
  stated turnaround is ~7 days or less). Answers were grounded in real
  project history rather than invented: closed-test recruitment via
  family/friends/classmates (3 real business signups), and the real
  2026-08-07 WhatsApp/Facebook bug (a tester emailed a screenshot; root
  cause was SA phone numbers not formatted for `wa.me` links plus one
  listing with bad Facebook data that crashed the Android app on tap — both
  fixed) used for the engagement/feedback/readiness questions.
- **Outstanding**: wait for Google's decision email. No other action
  pending on this application.

## Session log — 2026-08-25 (Play Console crash triage + Android white-screen bug)

Also entirely `SasTownHub`-side work (the Android app), recorded here for
continuity with the other Android session logs above.

**Play Console `ActivityNotFoundException` (CategoryScreen.kt:180)**: Kabelo
saw this in Play Console's Crashes & ANRs and wasn't sure if it was already
fixed. Confirmed via `git log` it's the same bad-Facebook-data crash from the
2026-08-07 WhatsApp/Facebook fix (commit `6407f39`, referenced in the
2026-08-24 log above) — `openExternalLink()` in `CategoryScreen.kt` already
catches it. The report's "last occurred 17 days ago" lines up with right
after the fix landed, before affected testers updated. No new occurrences
since; closed out, no action needed.

**New bug reported**: on the Home screen, tapping the back arrow to exit a
category page and then immediately fast-double-tapping near the top of the
screen (aiming at the "SasTown Hub" title) produces a full white screen
(status bar included) that needs a force-close to clear. Reproducible on
Kabelo's own phone (Samsung Galaxy S21) and on Honor devices; not
reproducible when the same steps are done slowly.

Investigated live over `adb` (phone connected via USB) rather than blind
code review, since a static read of `HomeScreen.kt`/`CategoryScreen.kt`/
`SasTownTopBar.kt`/`MainActivity.kt` turned up nothing suspicious — no
click handler exists on the title at all, no shared mutable state between
category screens, no custom exception handling. Root cause confirmed via
logcat captured *during* a live frozen white screen: it's the OEM's own
Notification Shade, not the app. `com.aethelst8.sastownhub`'s process
never dies (same pid throughout, no `FATAL EXCEPTION`/ANR/kill in any
capture) — the fast tap sequence lands on the real OS status bar (which
sits flush against the app's edge-to-edge black header, so it's visually
indistinguishable from the app's own title bar), and that confuses the
OEM SystemUI's status-bar-panel touch state machine into reporting
`fraction = 1.0` / `SHADE_STATE` from a tap instead of an actual
swipe-down, which never actually renders the shade's content — leaving a
blank full-screen overlay. Confirmed on both Samsung (`SecPanelSplitHelper`,
One UI-specific) and Honor (MagicOS, a completely different SystemUI
codebase) — two unrelated OEM skins misbehaving the same way rules out a
vendor-specific bug and points to a generic Android windowing race
triggered by fast taps right at the status-bar/app-window boundary.

**Mitigation attempted and reverted**: added 20dp of extra top padding to
`SasTownTopBar.kt` (via `windowInsets`) to push the back arrow and title
further from the true status bar edge, on the theory that more buffer
would reduce how often a fast tap overshoots into OS territory. Compiled
clean, installed on Kabelo's phone via `adb` (required uninstalling the
existing production-signed build first — debug and release signatures
don't match), verified no navigation regression. **Kabelo tested manually
and reported no improvement** — change was reverted, repo back to clean
`HEAD`. Consistent with the diagnosis: the bug is confirmed to live
entirely in the OEM's SystemUI process, so nothing inside this app's own
window/touch-handling code can prevent the second tap from being
misinterpreted once it lands past the status bar boundary — no further
app-side code fix identified as of this session. Still open: whether to
just accept/document this as a known OS quirk, or explore other angles
later (e.g. whether Android's `WindowInsetsController` offers any way to
influence the OS panel's own gesture disambiguation — untested, may not
be exposed to apps at all).

**Tooling notes for future on-device sessions** (this dev environment has
no Android emulator, so live debugging goes through the physical phone
over `adb`):
- `adb.exe` isn't on `PATH` here; it's at
  `C:\Users\Moloi\AppData\Local\Android\Sdk\platform-tools\adb.exe`.
- Building `SasTownHub/mobile` via `gradlew` needs `JAVA_HOME` set — no
  system JDK/`java` on `PATH` either. Android Studio's bundled JDK works:
  `JAVA_HOME="C:\Program Files\Android\Android Studio\jbr"`.
- `gradlew :app:assembleDebug --offline` works from a warm Gradle cache
  without network access.

## Session log — 2026-08-25 (Firebase notifications, production outage, Play Store launch)

Same day as the log above, but a separate and much larger chunk of work -
entirely `SasTownHub`-side again. Ends with SasTownHub live in Google Play
production for the first time. Full blow-by-blow (exact commits, logcat/
Actions-log excerpts) is in the conversation transcript; this is the
durable summary.

**Admin push notifications - the whole pipeline already existed, just
never finished**: Kabelo asked whether admin-side "new listing" push
notifications could work now that Oracle hosting is live. Traced it end
to end and found the entire thing already built (Firebase project
registered, `AdminFirebaseMessagingService.kt` receiving pushes, backend
`PushNotificationService.cs` sending them, `docker-compose.yml` already
passing a `FIREBASE_SERVICE_ACCOUNT_JSON` env var through) - the only gap
was Kabelo never doing the one-time Firebase Console step (generating a
service-account key) and pasting it into the VM's `.env`, both blocked on
Kabelo no longer having the VM's SSH private key (see below).

**Solved via CI/CD instead of SSH**: since GitHub Actions' `deploy.yml`
already had working SSH access via its own key, added a `.env`-syncing
step there driven by a new `FIREBASE_SERVICE_ACCOUNT_JSON` GitHub secret,
so every deploy keeps `.env` in sync without anyone needing raw SSH. This
pattern (route VM changes through the CI/CD channel instead of direct
SSH) became the theme of the rest of this session until SSH was restored.

**That deploy caused a real production outage** (~30-45 min, full site
502, not just admin): three compounding bugs, found live via `adb`
logcat, GitHub Actions log output, and `WebFetch` polling the live site
(no other observability existed) rather than guessing:
1. `Program.cs` parsed the Firebase credential unguarded at startup - a
   malformed value threw before the host could bind to a port, taking
   the *entire* site down, not just push notifications. Fixed: wrapped
   in try/catch, logs and no-ops instead (matches the no-op-on-failure
   philosophy `PushNotificationService.cs` already had).
2. A stray uncommitted local diff to `docker-compose.yml` on the VM (root
   cause never identified) made `git pull` abort with a merge conflict on
   *every* deploy that day, and the script didn't stop on that error - so
   even the fix above wasn't actually landing. Fixed: deploy now does
   `git fetch` + `git reset --hard origin/main` instead of a plain pull,
   permanently immune to this class of drift (nothing on the VM is ever
   meant to hand-edit tracked files directly).
3. Once the app container was healthy again, nginx kept forwarding to a
   dead container IP and still 502'd - it resolves its static
   `proxy_pass http://app:8080;` hostname once at its own startup/reload
   and never again, and the `nginx` container itself hadn't restarted in
   5 days across all of the `app` container's recreations that day.
   Fixed: `docker compose restart nginx` added after every deploy.

Also added permanent `docker compose ps` + `docker compose logs app
--tail 100` output to every deploy run, since diagnosing this outage live
was only possible through that same SSH channel - a silent successful
deploy hiding a crash-looping container is exactly what let this drag on.

**Two more real bugs found once the outage was resolved and admin login
was retried**, both confirmed live via `adb`/logcat rather than guessed:
- Admin app login said "Couldn't reach the server" - turned out to be a
  genuinely stale installed APK still pointing at the decommissioned
  `sastownhub.azurewebsites.net` (confirmed via `UnknownHostException` in
  logcat). Rebuilt from current source (already correctly pointed at
  `sastownhub.co.za`) and reinstalled - no code fix needed for this part.
- Separately, a real client/server contract bug was also fixed while
  chasing the same symptom: the admin API's no-body endpoints (`Login`,
  `Logout`, `Approve`, `Reject`, `RegisterDevice`) returned HTTP 200 with
  an empty body, but the Android side declares them `Response<Unit>` -
  Retrofit still runs a 200's empty body through the kotlinx.serialization
  converter and throws (nothing to decode), while it skips conversion
  entirely for 204/205 regardless of declared type. Changed all five to
  return `NoContent()`. This wasn't the actual cause of the "couldn't
  reach server" report (that was the stale-APK issue above), but it was a
  real latent bug worth fixing anyway.

**Listing submission was also broken end-to-end**, found by actually
submitting a real test listing rather than assuming the earlier fixes
were sufficient:
- nginx's default `client_max_body_size` (1M) rejected a normal phone
  photo (~4MB) with a bare 413 before it ever reached the app, even
  though the app's own `[RequestSizeLimit(5_000_000)]` already allows up
  to 5MB - nginx just never had a matching limit set. Fixed in both
  `nginx/conf.d/app.conf` and `app-ssl.conf.disabled` (unclear which is
  actually active on the VM vs. what's tracked in git - see below).
- Uploaded logos are saved to `/app/wwwroot/uploads/logos/` inside the
  container's own filesystem, and `docker-compose.yml`'s `app` service
  had no volume for it - every deploy (which always recreates the
  container) silently wiped any photo uploaded since the last build,
  confirmed live when a just-submitted test listing's logo vanished
  right after the very next deploy. Fixed with a new `listing-uploads`
  named volume, same pattern as `postgres-data`.
- The nginx fix above initially didn't deploy at all: `nginx/**` wasn't
  in `deploy.yml`'s trigger `paths:` filter (only `backend/**`,
  `frontend/**`, `docker-compose.yml` were), so pushing an nginx-only
  change never fired the workflow. Added `nginx/**` to the filter.
- End state confirmed working: admin login succeeds, a real test listing
  with a photo submitted from the public app shows up correctly in the
  admin app's Pending Listings with its image intact, push notification
  fires, approve/reject both work.

**Also shipped in this session**: the "add one" empty-category-listings
link (added earlier the same day, see previous log entry's context) got
committed and built into a new release AAB - bumped `versionCode` 6→7 /
`versionName` "1.5"→"1.6" since 6 was already consumed by the prior
closed-testing upload and Play Console requires strictly ascending codes
across every track. Signing credentials for this were already present in
Kabelo's global `~/.gradle/gradle.properties` (not asked for/handled
in-session).

**Kabelo lost the VM's original SSH private key** partway through this
session (confirmed - no local copy, only ever existed in the
`VM_SSH_PRIVATE_KEY` GitHub secret, which can't be extracted back out).
This is what forced the whole "route everything through CI/CD" pattern
above. Restored properly rather than left as a permanent workaround: had
Kabelo generate a fresh local ed25519 key pair himself (private key never
touched Claude - only the public key, which was pasted into the
conversation and used in a one-off deploy-pipeline step to append it to
the VM's `~/.ssh/authorized_keys`), confirmed a real `ssh -i ...` login
worked, then immediately reverted that bootstrap step out of the
permanent deploy script. Kabelo has full personal SSH access again as of
this session.

**Play Store production launch**: separately, Google's "Apply for access
to production" review (submitted 2026-08-24) came back **approved** this
session. Kabelo went through Play Console's release flow himself
(countries/regions, review the third-party-app-stores setting - opted to
not publish to any third-party stores since there was no reason to for a
small local-business app - create the release with the new versionCode
7/"1.6" AAB) and published. **SasTownHub is now live in Google Play
production** - first real public release, not just closed testing.

**Left open / worth knowing for next time**:
- The nginx active-config mystery: `nginx/conf.d/app.conf` (HTTP-only) is
  tracked as the "active" file and `app-ssl.conf.disabled` as inactive,
  matching the README's pre-TLS-cutover naming - but HTTPS demonstrably
  works live (confirmed repeatedly via `WebFetch` all session, including
  after the `git reset --hard` change). Something about how TLS actually
  got activated on the VM doesn't match what's in git/the README. Not
  investigated further since the `client_max_body_size` fix was applied
  to both files defensively either way - but if nginx config ever needs
  touching again, don't trust the `.conf`/`.conf.disabled` naming at face
  value without checking `nginx -T` on the actual VM first.
- Root cause of the stray `docker-compose.yml` diff that broke `git pull`
  never identified - moot now (force-reset makes deploys immune to it
  either way, and it's presumably been overwritten by now), but worth
  keeping in mind if similar drift resurfaces.
- pgAdmin access to Postgres (via an SSH tunnel to the VM's
  loopback-only `127.0.0.1:5432`) was blocked for the same lost-SSH-key
  reason and not yet confirmed working now that the key is restored -
  Kabelo was mid-testing the tunnel command when this session's work
  wrapped up.

**Also after launch**: the homepage's hero "Get the App" button had been a
disabled `javascript:void(0)` placeholder since before the app existed
(`Views/Home/Index.cshtml`, a leftover `TODO: wire up the real Play Store
link once the Android app is submitted (Phase 6)`). Wired it to the real
listing (`https://play.google.com/store/apps/details?id=com.aethelst8.sastownhub`),
then swapped the initial Bootstrap-button version for Google's own hosted
"Get it on Google Play" badge image, matching Play Store brand guidelines,
per Kabelo's preference after seeing an example.

**pgAdmin confirmed working** with the restored SSH key - tunnel command:
```
ssh -i "$env:USERPROFILE\.ssh\sastownhub_vm" -L 5433:127.0.0.1:5432 -N ubuntu@145.241.99.203
```
then connect pgAdmin to `localhost:5433` as before. Also used it to clean
up this session's test listings (`Listings` table, Postgres, table/column
names are PascalCase and need double-quoting in raw SQL, e.g. `"Listings"`,
`"Id"`) - `Id`s 27-31 ("test23", "testweee", "abaa", "as", "rere") were all
created 2026-08-25 during today's testing and clearly distinguishable from
the legitimate 2026-08-20 seed-import data by both date and content; `Id 29`
("abaa") was sitting `Approved` and briefly live on the public site before
being deleted.

## Session log — 2026-09-11 (taxi fares, Services page, app update prompt, text selection)

Started as a request against this repo (update the old static `taxi.html`'s
fare prices) but `taxi.html` no longer exists here - all subpages were
deleted 2026-07-24 (see "History worth knowing" above). Kabelo confirmed the
real destination was `SasTownHub`'s live `/taxi` page, so all of this
session's actual work happened there instead, recorded here for continuity
per this file's established convention. Full technical detail lives in
`SasTownHub`'s own `CLAUDE.md`/memory (currently stale - see note below).

**Taxi fares updated**: `Views/Info/Taxi.cshtml` (website, live instantly)
and `StaticInfoScreen.kt` (Android, hardcodes its own mirrored copy of the
same table and needed a separate fix + rebuild) - Parys R50, Heilbron split
out to its own row at the old combined R40, Kroonstad added at R150.

**Found and fixed a broken image while verifying the taxi change in a
browser**: the homepage's "Get it on Google Play" badge was hotlinking a
`play.google.com` badge URL that now 404s - Google appears to have moved or
retired it. Fixed by downloading the badge and self-hosting it at
`wwwroot/images/google-play-badge.png` instead of hotlinking any external
URL, so it can't break the same way again.

**Added a new "Our Services" static info page** (website `/services` +
Android `static/services`), following the same static-info-page pattern
already used for taxi/schools/places (`InfoController`, `StaticInfoScreen.kt`).
Lists Kabelo's own freelance services - website creation, CV revamp, Android
app development, business plans, job and university/college applications,
posters/flyers, QR code generation. Subtitle wording corrected from "things
Kabelo can help you with directly" to "things we can help you with directly"
per Kabelo's explicit request, on both platforms.

**Added an "Update App" button to the Android home top bar**, right-aligned
next to the "SasTown Hub" title, small enough to sit in the bar, yellow
outline with a yellow-tinted press ripple, shown only when Play Core's
`AppUpdateManager` reports a newer version is available (which only ever
returns true for a real Play Store install, never a sideloaded/debug build).
**Why**: static content (taxi fares, the new Services page, etc.) is bundled
into the app binary, not fetched live - users on an older installed version
have no way to know new content shipped until they update, so this closes
that gap. Clicking it opens the Play Store listing. New dependency added:
`com.google.android.play:app-update-ktx`.

**Fixed app text not being selectable/copyable** - a real user reported being
unable to highlight or copy any text in the app. Root cause: Compose `Text`
isn't selectable by default. Fixed by wrapping `MainActivity`'s `NavHost` in
a single `SelectionContainer`, which applies to every screen at once without
touching individual composables.

**versionCode churn worth knowing about**: bumped 8 -> 9 -> 10
(versionName 1.6.1 -> 1.6.2 -> 1.6.3) over the course of this session.
versionCode 9 turned out to already be consumed by a prior upload attempt
outside this session's knowledge - Play Console rejected the versionCode-9
AAB outright ("Version code 9 has already been used"), discovered only at
actual upload time. **Lesson for next time**: don't trust `build.gradle.kts`
as the source of truth for "next free versionCode" - check Play Console's
actual last-used version first, since it can be ahead of what's committed
here (e.g. from an upload Kabelo made directly through the console).

**Final AAB** (versionCode 10 / "1.6.3") built and handed to Kabelo to
upload himself, as always - signing credentials live only in his global
Gradle properties, never touched by an assistant. Release notes drafted for
Play Console's `en-GB` locale format.

**Also noted, not acted on this session**: `SasTownHub`'s own `CLAUDE.md` is
significantly stale - it still describes the Azure-to-Oracle migration as
in-progress and doesn't mention the Play Store production launch, the
Firebase/nginx/outage work, or anything from the 2026-08-20 through
2026-08-25 session logs above. Worth a dedicated pass to bring it current,
separate from this entry.

## Session log — 2026-10-06 through 2026-10-28 (Oracle outage, Play Store compliance fire)

Spans several separate check-ins over three weeks as the same incident
dragged on. Recorded here (not `SasTownHub`) since nearly all the actual
work ended up happening in this repo - the backend itself couldn't be
touched for most of this period.

**How it started**: Kabelo got a routine "Your Oracle Cloud Free Trial has
ended" email on 2026-10-06. Shortly after, the live site went fully
unreachable (confirmed via direct TCP probes to `145.241.99.203` - port
80/443/22 and even ICMP all got no response, not just a refused
connection). OCI Console showed the `sastownhub-vm` instance as `Stopped`.
Clicking Start returned "Instance is disabled and will not accept any
action requests. Please contact customer support to reenable" - an
account-level hold, not just an idle-instance auto-stop. Account
Management/Billing pages in the Console also errored out at the time,
consistent with that.

**Immediate mitigation (2026-10-06)**: rather than wait on Oracle support
(no ticket was submitted at this point - see below), apex DNS was pointed
back at GitHub Pages' four IPs (`185.199.108/109/110/111.153`), and this
repo's `index.html` was rewritten from the old "in development" copy to a
maintenance notice with a Google Play badge linking to
`com.aethelst8.sastownhub`, so visitors saw something useful instead of a
dead connection. Propagated within minutes. (Caution for next time: this
domain's DNS is on GoDaddy, not Namecheap - Kabelo has `aethelst8.com` on
Namecheap and the two consoles got mixed up once mid-session; always
confirm the domain name in the DNS panel's URL before editing records.)

**The support ticket was never actually sent.** Account Management started
loading again on its own by 2026-10-20, suggesting the account-level hold
had lifted - but by then the instance itself was gone entirely from the
Instances list (not just stopped/terminated-but-listed). Lesson: an
Oracle "contact support to reenable" message is not something to just wait
out - submit the ticket promptly, since leaving a disabled instance
unactioned for ~2 weeks appears to be what triggered Oracle reclaiming it
outright.

**What survived**: the boot volume (`sastownhub-vm (Boot Volume)`, 47GB,
region af-johannesburg-1, compartment `kmoloi928` (root)) was still
`Available` after the instance's termination, and the VCN (`sastownhub-vcn`
+ `sastownhub-public-subnet`) was untouched. A manual full backup of the
boot volume (`sastownhubBootVolBackUp`) was taken before touching anything
further, as a safety net.

**Attempting to recreate the instance from that boot volume hit two
tenancy-level surprises**, both worth remembering if this ever recurs:
1. The tenancy's Always Free Ampere A1 service limit had been reduced from
   the original 4 OCPU/24GB to **2 OCPU/12GB** at some point during the
   disable/reclaim cycle - confirmed via Compute → the "Service limits
   status" panel shown when sizing the new instance (showed "0 of 2" cores,
   "0 of 12" GB, i.e. a lowered ceiling, not just prior usage). An instance
   sized at the old 4/24 fails outright at creation time for exceeding this
   new limit. There's a "Request Increase" link on that same limits page to
   ask Oracle to restore the original allowance - not yet done as of this
   log entry.
2. Even sized correctly, instance creation has been failing since
   2026-10-20 (and still failing as of 2026-10-28, i.e. 8+ days and
   counting) with `Out of capacity for shape VM.Standard.A1.Flex in
   availability domain AD-1` - genuine Ampere hardware contention in
   Oracle's Johannesburg region, which has only the one AD (no
   alternate-AD workaround exists here). Boot source (existing boot volume
   vs. a fresh image) makes no difference to this - it's shape/AD capacity,
   unrelated to boot source. Dropping the request size to the minimum
   (1 OCPU/6GB) to improve placement odds, then resizing up once capacity
   allows, is the practical workaround being tried; even that has not
   succeeded yet. Also: Oracle rate-limits repeated `LaunchInstance` calls
   per user ("Too many requests for the user") if retried too quickly -
   retries need to be paced (minutes apart), not hammered.
   Kabelo opted to keep retrying this manually day-to-day rather than
   script it, and is also weighing a bigger move away from Oracle
   entirely (Supabase for Postgres + a free app host - Render's free web
   service tier was the most viable candidate found, with two caveats:
   it cold-starts after 15min idle, and its filesystem is ephemeral across
   deploys/restarts so uploaded listing photos would need to move to
   Supabase Storage rather than local disk, same class of bug already
   fixed once on Oracle via a named Docker volume). No decision made as of
   this entry - Kabelo is thinking it over during the week of 2026-10-28.

**Separate fire that landed mid-outage**: a 2026-10-15 Google Play
"Action Required" email (re-surfaced 2026-10-28, deadline **2026-10-29**)
flagged the app's Data Safety declaration because its privacy-policy /
delete-data URL (`https://sastownhub.co.za/privacy`, declared correctly -
the URL itself was never wrong) was 404ing as a direct consequence of the
backend outage. Fixed independently of the backend situation: added a
static `privacy/index.html` to this repo (content mirrors the backend's
`Views/Info/Privacy.cshtml`, with the stale "hosted on Microsoft Azure"
lines corrected and the contact email set to `aethelst8@gmail.com`),
linked from the homepage footer, deployed via the same GitHub Pages
pipeline already serving the maintenance notice. Submitted via Play
Console's "App content" flow same day (submission #14) and published
before the deadline - confirms this repo's GitHub Pages hosting is a
reliable place to park Play Store compliance pages regardless of whatever
is happening with the actual backend. Note: GitHub Pages serves this at
`/privacy/` and 301-redirects the bare `/privacy` path to it - normal
behavior, Google's checker follows it fine.
