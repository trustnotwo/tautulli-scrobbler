# Plex → Last.fm Scrobbler Setup

Simple setup guide for `scrobble_plex.py` with Tautulli in Docker. Follow the steps in order.

---

## Step 1 — Get a Last.fm API key

1. Go to **https://www.last.fm/api/account/create** (log in with your Last.fm account)
2. Application name: anything, e.g. `Plex Scrobbler`. Leave the other fields blank.
3. Submit. Copy the **API Key** and **Shared Secret** somewhere — you need them in Step 3.

---

## Step 2 — Create the folders

On the Docker host, run:

```bash
mkdir -p /mnt/NVME/apps/tautulli/scripts
mkdir -p /mnt/NVME/apps/tautulli/secrets
```

---

## Step 3 — Create the secret files

Create four text files, **one value per file, nothing else**:

```bash
echo -n "YourLastfmUsername"  > /mnt/NVME/apps/tautulli/secrets/lastfm_username.txt
echo -n "YourApiKeyHere"      > /mnt/NVME/apps/tautulli/secrets/lastfm_api_key.txt
echo -n "YourApiSecretHere"   > /mnt/NVME/apps/tautulli/secrets/lastfm_api_secret.txt
echo -n "YourLastfmPassword"  > /mnt/NVME/apps/tautulli/secrets/lastfm_password.txt
```

Then lock them down:

```bash
chmod 600 /mnt/NVME/apps/tautulli/secrets/*.txt
chown -R 568:568 /mnt/NVME/apps/tautulli/secrets
```

> If your password contains special characters like `$` or `!`, use single quotes:
> `echo -n 'p@$$word!' > .../lastfm_password.txt`

---

## Step 4 — Add the script

Copy `scrobble_plex.py` into the scripts folder:

```bash
cp scrobble_plex.py /mnt/NVME/apps/tautulli/scripts/
chown 568:568 /mnt/NVME/apps/tautulli/scripts/scrobble_plex.py
```

---

## Step 5 — Start Tautulli

From the folder containing `compose.yml`:

```bash
docker compose up -d tautulli
```

The compose file already handles everything else — it installs `pylast` automatically on startup and mounts the scripts and secrets into the container. First start may take a minute while pip installs.

---

## Step 6 — Quick test

Run this from the host:

```bash
docker exec -it tautulli python3 /config/scripts/scrobble_plex.py "start" "Test Artist" "Test Album" "Test Track" 240 0
```

✅ **Success:** you see `LAST.FM: Updated now playing: Test Artist - Test Track`, and "Test Artist" shows as *Now Playing* on your Last.fm profile.

❌ **If it errors:**
- `FileNotFoundError: /run/secrets/...` → a secret file is missing or misnamed (Step 3)
- `ModuleNotFoundError: pylast` → container is still installing, wait a minute and check `docker logs tautulli`
- `Invalid API key` / auth error → a value in one of the secret files is wrong

---

## Step 7 — Set up the Tautulli notification agent

Open Tautulli at `http://<your-server>:8181`, then:

**Settings → Notification Agents → Add a new notification agent → Script**

### Configuration tab
| Setting | Value |
|---|---|
| Script Folder | `/config/scripts` |
| Script File | `./scrobble_plex.py` |
| Script Timeout | `30` |
| Description | `Scrobble Plex` |

Click **Save**, then reopen the agent for the next tabs.

### Triggers tab
Check exactly these four (nothing else):
- ☑ Playback Start
- ☑ Playback Stop
- ☑ Playback Pause
- ☑ Playback Resume

### Conditions tab
Add one condition so it only fires on music:

| Parameter | Operator | Value |
|---|---|---|
| Media Type | is | `track` |

### Arguments tab
Expand each trigger and paste the matching line **exactly, including the quotes**:

**Playback Start**
```
"start" "{track_artist}" "{album_name}" "{track_name}" {duration_sec} 0
```

**Playback Stop**
```
"stop" "{track_artist}" "{album_name}" "{track_name}" {duration_sec} {progress_percent}
```

**Playback Pause**
```
"pause" "{track_artist}" "{album_name}" "{track_name}" {duration_sec} 0
```

**Playback Resume**
```
"resume" "{track_artist}" "{album_name}" "{track_name}" {duration_sec} 0
```

Click **Save**. Done.

---

## Step 8 — Verify with real playback

Play a song in Plex, let it get past **50%**, then stop it. Check:

- Your Last.fm profile shows the scrobble
- Tautulli → gear icon → **View Logs → Notification Logs** shows the script ran

That's it. From now on, anything played past 50% scrobbles automatically.

---

## Good to know

- **Tracks only scrobble past 50% progress** — skipping a song early won't scrobble it (on purpose).
- **Featured artists are stripped** — "Artist feat. Someone" scrobbles as just "Artist" (on purpose, matches Last.fm's artist pages).
- **Pausing clears your Now Playing** on Last.fm; resuming brings it back.
- If you change a secret file, restart the container: `docker compose restart tautulli`
