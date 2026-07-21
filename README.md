# Tautulli Scrobbler for Plex

Scrobbles Plex music playback to Last.fm using Tautulli's script notification agent. Tracks scrobble once they pass 50% progress. Now Playing status updates on start/resume and clears on pause.

Written for Tautulli running in Docker, but any setup works as long as the script can find its secrets and `pylast` is installed.

## What you need

- Tautulli (Docker) already watching your Plex server
- A Last.fm account
- A Last.fm API key — create one at https://www.last.fm/api/account/create (name it whatever, leave the optional fields blank, save the API key and shared secret)

## 1. Credentials

The script reads four files from `/run/secrets/` inside the container:

```
lastfm_username
lastfm_api_key
lastfm_api_secret
lastfm_password
```

Each file contains just the value, nothing else. Make a folder on your host, create the files, then mount that folder to `/run/secrets` in your compose file:

```bash
mkdir -p /path/to/tautulli/secrets
echo -n "YourUsername"   > /path/to/tautulli/secrets/lastfm_username
echo -n "YourApiKey"     > /path/to/tautulli/secrets/lastfm_api_key
echo -n "YourApiSecret"  > /path/to/tautulli/secrets/lastfm_api_secret
echo -n "YourPassword"   > /path/to/tautulli/secrets/lastfm_password
chmod 600 /path/to/tautulli/secrets/*
```

```yaml
    volumes:
      - /path/to/tautulli/secrets:/run/secrets:ro
```

If your password has special characters like `$` or `!`, wrap it in single quotes so the shell doesn't mangle it.

(If you're on Docker Swarm you can use real Docker secrets instead — they land in the same place.)

## 2. Install pylast in the container

The script needs the `pylast` library. Quick way:

```bash
docker exec tautulli pip install pylast
```

This won't survive the container being recreated, though. If you use the linuxserver.io Tautulli image, add this to your compose file so it installs on every startup:

```yaml
    environment:
      - DOCKER_MODS=linuxserver/mods:universal-package-install
      - INSTALL_PIP_PACKAGES=pylast
```

Any other approach that gets pylast into the container permanently is fine too.

## 3. Add the script

Drop `scrobble_plex.py` somewhere Tautulli can see it — a `scripts` folder inside your Tautulli config mount is the usual spot:

```bash
cp scrobble_plex.py /path/to/tautulli/config/scripts/
```

## 4. Test it

```bash
docker exec -it tautulli python3 /config/scripts/scrobble_plex.py "start" "Test Artist" "Test Album" "Test Track" 240 0
```

You should see `LAST.FM: Updated now playing: Test Artist - Test Track`, and it'll show as Now Playing on your Last.fm profile for a few minutes.

Common errors:

- `FileNotFoundError: /run/secrets/...` — secrets aren't mounted, or a file is misnamed
- `ModuleNotFoundError: pylast` — pylast isn't installed (step 2)
- Auth / invalid API key errors — one of the secret values is wrong

## 5. Set up the notification agent

In Tautulli: **Settings → Notification Agents → Add a new notification agent → Script**

**Configuration:**

| Setting | Value |
|---|---|
| Script Folder | `/config/scripts` |
| Script File | `./scrobble_plex.py` |
| Script Timeout | `30` |
| Description | `Scrobble to Last.fm` |

Save, then reopen the agent to edit the rest.

**Triggers** — check these four only:

- Playback Start
- Playback Stop
- Playback Pause
- Playback Resume

**Conditions** — add one so it only fires on music:

| Parameter | Operator | Value |
|---|---|---|
| Media Type | is | `track` |

**Arguments** — expand each trigger and paste the matching line exactly, quotes included:

Playback Start:
```
"start" "{track_artist}" "{album_name}" "{track_name}" {duration_sec} 0
```

Playback Stop:
```
"stop" "{track_artist}" "{album_name}" "{track_name}" {duration_sec} {progress_percent}
```

Playback Pause:
```
"pause" "{track_artist}" "{album_name}" "{track_name}" {duration_sec} 0
```

Playback Resume:
```
"resume" "{track_artist}" "{album_name}" "{track_name}" {duration_sec} 0
```

Save. Play a song in Plex, let it get past 50%, stop it, and check your Last.fm profile. Tautulli's Notification Logs (gear icon → View Logs) show every time the script fires if you need to debug.

## Editing the artist allowlist

To match Last.fm's artist pages, the script strips featured artists by cutting the name at the first `feat.`, `ft.`, `featuring`, comma, or ampersand. That works for the common `Artist feat. Someone` case, but it would also mangle artists whose names legitimately contain a comma or ampersand — `Tyler, The Creator` would scrobble as just `Tyler`, and `Earth, Wind & Fire` as just `Earth`.

To prevent that, the script keeps an allowlist called `KNOWN_ARTISTS` near the top of `scrobble_plex.py`. Any name on it is left intact. It's already seeded with a batch of common comma/ampersand acts, and adding your own is a one-line edit:

```python
KNOWN_ARTISTS = {
    "tyler, the creator",
    "earth, wind & fire",
    # ...add your artist here, lowercase, on its own line:
    "florence + the machine",
}
```

A few things to keep in mind:

- **Lowercase it.** Matching is case-insensitive, but every entry in the set must be lowercase to work.
- **Match how Plex tags it.** The comparison is otherwise exact, so the entry has to match the artist name exactly as it appears in your library — right down to spacing and punctuation. If an artist still gets chopped, check the actual tag in Plex and copy it from there.
- **Trailing features still get stripped.** A listed artist with a real feature — `Tyler, The Creator feat. Kali Uchis` — still scrobbles as `Tyler, The Creator`. You only need the base name on the list.
- **No restart needed.** Tautulli runs the script fresh on every playback event, so edits take effect on the next track — unlike secret changes, which need a container restart.

Names that use other separators — like `+` in `Florence + the Machine`, `/` in `AC/DC`, or `!` in `Panic! at the Disco` — are never split in the first place, so they don't need to be on the list.

## Behavior notes

- Nothing scrobbles below 50% progress — skipped tracks stay off your profile
- Featured artists get stripped, so "Artist feat. Someone" scrobbles as "Artist" (matches Last.fm's artist pages) — except for names on the allowlist above
- Pausing clears Now Playing; resuming brings it back
- If you change a secret, restart the container
