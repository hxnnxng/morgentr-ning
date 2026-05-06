# Morgentræning

10-minutters morgenrutine-app: SI-led-mobilisering og core-stabilitet uden lyske-belastning.

## Architecture

Single self-contained `index.html` — no build step, no dependencies, no framework. Inline CSS and JS. Open the file directly or serve over HTTP for Strava OAuth to work.

The app has three screens (toggled via `.hidden` class):
1. **Start screen** (`#startScreen`) — overview + start button
2. **Workout screen** (`#workoutScreen`) — big circular timer, exercise name (links to YouTube), instructions, prev/pause/skip controls
3. **Done screen** (`#doneScreen`) — "Send til Strava" button

## Exercise data

`exercises` array near the top of the `<script>` block. Each entry:

```js
{
  name: '...',
  section: 'Del 1: ...' | 'Del 2: ...',
  videoId: 'YouTube ID',
  sides: ['Venstre side', 'Højre side'] | undefined, // present = asymmetric
  how: '...',  // execution instructions
  why: '...',  // rationale
  tip: '...' | null  // optional warning (e.g. lyske-care)
}
```

Asymmetric exercises (with `sides`) run **30 seconds per side back-to-back** (no rest between sides). Symmetric exercises run for `EXERCISE_TIME` (45s). All exercises are followed by `REST_TIME` (15s) before the next.

Currently asymmetric: Knee-to-Chest, Sideplanke på knæ, Ridderstræk.

## Timer behavior

- Single beeps at 3, 2, 1 seconds (660 Hz)
- Longer final beep at 0 (880 Hz, 250ms)
- Same beep pattern applies during rest countdowns
- Audio context resumes on user gesture (browser policy) — happens in `startWorkout()`
- Timer ring color: blue → orange (≤10s) → red pulsing (≤3s) → green (rest)

## Time tracking

`workoutStartedAt` is set when training starts. `togglePause()` accumulates `totalPausedMs`. `getElapsedSeconds()` returns wall-clock minus paused time, used as Strava `elapsed_time`.

`completedLog` is appended to whenever a side or full exercise completes (via timer or skip). `prevExercise()` pops the last entry. Used to build the Strava description.

## Strava integration

OAuth 2.0 with refresh-token flow, all client-side. Tokens stored in `localStorage`:
- `stravaClientId`, `stravaClientSecret` — user-supplied via modal
- `stravaRefreshToken`, `stravaAccessToken`, `stravaExpiresAt` — managed by app

Strava only allows **one API app per Strava account**. Callback domain in Strava settings should match the host (e.g. `up.railway.app` matches all subdomains).

OAuth flow:
1. User clicks "Send til Strava" on done screen
2. If no refresh token: modal opens for Client ID/Secret entry
3. `authorizeStrava()` saves credentials + persists workout state to `localStorage` (since redirect destroys in-memory state) and redirects to Strava authorize URL
4. Strava redirects back with `?code=...`
5. `handleOAuthCallback()` (run on every page load) detects code, exchanges for tokens, restores workout state, triggers `sendToStrava()`
6. Subsequent sends just use the refresh token — no redirect

Activity sent: `name: "Morgenyoga"`, `sport_type: "Yoga"`, `start_date_local`: workout start time, `elapsed_time`: actual seconds, `description`: generated list of exercises with durations + total time.

## Deployment

- **GitHub**: https://github.com/hxnnxng/morgentr-ning (branch: `master`)
- **Railway**: https://morgentr-ning-production.up.railway.app — auto-deploys from `master`

OAuth redirect won't work from `file://`. For local testing of Strava: `python3 -m http.server` and use `localhost` as Strava callback domain.

## Responsive design

Mobile-first with breakpoints at 640px and 360px, plus landscape mobile (max-height: 500px). Uses `dvh`, safe-area insets for iOS notch, and `viewport-fit=cover`.

## Conventions

- Danish UI text throughout
- No frameworks, no npm — keep the single-file architecture
- New exercises: add to the `exercises` array, mark asymmetric ones with `sides`
- New asymmetric exercises: append `sides: ['Venstre side', 'Højre side']` and mention "30 sekunder pr. side" in `how`
