# Tautulli → Last.fm Scrobbler (`scrobble_plex.py`) — Setup Guide

This document covers the complete setup of the `scrobble_plex.py` notification script for Tautulli, which scrobbles Plex music playback to Last.fm using `pylast`. Credentials are stored in the operating system credential store via the `keyring` library — no API keys or passwords live in the script itself.

Instructions are platform-agnostic; a dedicated section covers running Tautulli in a Docker Compose container, where the credential approach differs.

---

## 1. Overview

**How it works:**

| Plex event | Tautulli trigger | Script action |
|---|---|---|
| Track starts playing | Playback Start | Updates Last.fm "Now Playing" |
| Track resumes | Playback Resume | Updates Last.fm "Now Playing" |
| Track pauses | Playback Pause | Expires "Now Playing" immediately (resubmits with a 1s duration) |
| Track stops/finishes | Playback Stop | Scrobbles the track **if progress ≥ 50%**; otherwise skips |

Additional behavior:

- **Featured artist stripping:** Artist strings like `Artist feat. Other`, `Artist ft. Other`, `Artist, Other`, or `Artist & Other` are trimmed to the primary artist before submission, matching Last.fm's canonical artist naming.
- **Scrobble timestamp:** Calculated as *now minus elapsed listen time* (`duration × progress%`), so the scrobble reflects when the track actually started.
- **Unicode-safe:** stdout/stderr are reconfigured to UTF-8 so non-ASCII artist/track names don't crash the script when run under Tautulli.

---

## 2. Prerequisites

- **Tautulli** installed and connected to your Plex Media Server
- **Python 3.7+** available to the environment Tautulli runs in
- A **Last.fm account**
- A **Last.fm API account** (free — see below)

### 2.1 Install Python dependencies

In the environment where the script will execute (see the Docker section if containerized):

```
pip install pylast keyring
```

`keyring` automatically selects the native credential store for the platform (e.g., Windows Credential Manager, macOS Keychain, or SecretService/KWallet on desktop Linux). Headless or containerized environments typically lack a native backend — see §4.2 and §7.

### 2.2 Create a Last.fm API account

1. Go to <https://www.last.fm/api/account/create>
2. Fill in an application name (e.g., `Plex Scrobbler`). Callback URL and homepage can be left blank.
3. Submit and note the **API Key** and **Shared Secret**. These go into the credential store in the next step — they are never pasted into the script.

---

## 3. Store credentials with keyring

The script retrieves three secrets at runtime:

| Service name | Username field | Value stored |
|---|---|---|
| `lastfm` | `api_key` | Your Last.fm API key |
| `lastfm` | `api_secret` | Your Last.fm shared secret |
| `lastfm` | *your Last.fm username* | Your Last.fm account password |

Store them from a Python session run **as the same account (and in the same environment) that Tautulli executes scripts under**:

```python
import keyring

keyring.set_password("lastfm", "api_key", "YOUR_API_KEY")
keyring.set_password("lastfm", "api_secret", "YOUR_API_SECRET")
keyring.set_password("lastfm", "YOUR_LASTFM_USERNAME", "YOUR_LASTFM_PASSWORD")
```

Or as shell one-liners:

```
python -c "import keyring; keyring.set_password('lastfm','api_key','YOUR_API_KEY')"
python -c "import keyring; keyring.set_password('lastfm','api_secret','YOUR_API_SECRET')"
python -c "import keyring; keyring.set_password('lastfm','YOUR_LASTFM_USERNAME','YOUR_LASTFM_PASSWORD')"
```

**Verify** the values read back:

```
python -c "import keyring; print(keyring.get_password('lastfm','api_key'))"
```

If this prints your API key, storage worked.

### 3.1 Update the script with your username

Open `scrobble_plex.py` and set the username to match the one used when storing the password:

```python
lastfm_username = "YOUR_LASTFM_USERNAME"
```

This is the only edit the script requires.

---

## 4. Critical: credential stores are per-user and per-environment

`keyring` reads from the credential store of **the account and session the script executes under**, not the account you used to configure things.

### 4.1 Same host, different account

If Tautulli runs as a service/daemon under a dedicated service account, credentials stored from your interactive session will be invisible to it. Either:

- Run the `keyring.set_password()` commands as the service account, or
- Run Tautulli under the account that holds the credentials.

If the script authenticates when run manually but fails from Tautulli, this is almost always the cause.

### 4.2 Headless / no native backend

On headless systems there may be no unlocked credential store available at runtime. `keyring` will raise `NoKeyringError` or return `None`. Options:

- Configure a file-based backend (e.g., `keyrings.alt`, or `keyring_pass`, or an encrypted backend such as `keyrings.cryptfile`) — weigh the security tradeoffs, as some of these store secrets in plaintext or with a static passphrase.
- Use environment variables or Docker secrets with a small script modification (see §7.3) — the cleaner option in containers.

---

## 5. Script placement

Place `scrobble_plex.py` in a folder readable by the account Tautulli runs as, for example a `scripts` directory alongside Tautulli's configuration:

```
/path/to/tautulli/scripts/scrobble_plex.py
```

Avoid network-mounted paths — Tautulli requires `allow_mounted_folders = 1` in its config file for those and flags it as a security risk. A local folder is preferred.

---

## 6. Tautulli configuration

### 6.1 Create the notification agent

1. Tautulli → **Settings → Notification Agents**
2. Click **Add a new notification agent** → choose **Script**

### 6.2 Configuration tab

| Setting | Value |
|---|---|
| Script Folder | Full path to the folder containing the script |
| Script File | `./scrobble_plex.py` |
| Script Timeout | `30` |
| Description | `Scrobble Plex` |

Click **Save** (you can reopen to configure the remaining tabs).

### 6.3 Triggers tab

Enable exactly these four:

- ☑ **Playback Start**
- ☑ **Playback Stop**
- ☑ **Playback Pause**
- ☑ **Playback Resume**

Leave everything else unchecked.

### 6.4 Conditions tab (recommended)

Add a condition so the script only fires for music, not movies/TV:

| Parameter | Operator | Value |
|---|---|---|
| Media Type | is | `track` |

Without this, video playback will invoke the script with video metadata and produce junk "scrobbles" or errors.

### 6.5 Arguments tab

Expand each trigger and enter the arguments **exactly as shown**, including the quotes (they protect artist/album/track names containing spaces):

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

Argument order maps to the script's positional parameters: `action, artist, album, title, duration_sec, progress_percent`. The trailing `0` on start/pause/resume is a placeholder for progress, which only matters on stop.

Click **Save**.

---

## 7. Running Tautulli in Docker Compose

Everything in §6 (agent, triggers, conditions, arguments) is identical in Docker. What changes is **where the script lives, where its dependencies are installed, and how credentials are provided** — because the script executes *inside the container*, not on the host.

### 7.1 Compose file and script volume

Example using the LinuxServer image, with a `scripts` directory bind-mounted into the container:

```yaml
services:
  tautulli:
    image: lscr.io/linuxserver/tautulli:latest
    container_name: tautulli
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      # Installs pip packages into the container at startup (survives image updates)
      - DOCKER_MODS=linuxserver/mods:universal-package-install
      - INSTALL_PIP_PACKAGES=pylast|keyring
    volumes:
      - ./tautulli/config:/config
      - ./tautulli/scripts:/config/scripts
    ports:
      - "8181:8181"
    restart: unless-stopped
```

Place `scrobble_plex.py` in `./tautulli/scripts` on the host. In the Tautulli UI, set:

| Setting | Value |
|---|---|
| Script Folder | `/config/scripts` |
| Script File | `./scrobble_plex.py` |

> **Note:** Use the *container-side* path (`/config/scripts`), not the host path. Tautulli only sees the container filesystem.

If you're using the official `tautulli/tautulli` image instead of LinuxServer, there is no mods mechanism — build a small derived image:

```dockerfile
FROM tautulli/tautulli:latest
RUN pip install --no-cache-dir pylast keyring
```

and reference it via `build:` in your compose file.

### 7.2 Why keyring's default backends don't work in a container

Containers have no native credential store — no desktop keyring daemon, no D-Bus session, no OS credential manager. `keyring.get_password()` inside the container will fail with `NoKeyringError` or silently return nothing. You have two workable approaches:

### 7.3 Recommended: Docker secrets (or env vars) with a script fallback

Provide the three secrets to the container and add a small fallback to the credential-retrieval section of the script. Docker Compose secrets mount each value as a file under `/run/secrets/`, keeping them out of `docker inspect` output and process environments:

```yaml
services:
  tautulli:
    # ... as above ...
    secrets:
      - lastfm_api_key
      - lastfm_api_secret
      - lastfm_password

secrets:
  lastfm_api_key:
    file: ./secrets/lastfm_api_key.txt
  lastfm_api_secret:
    file: ./secrets/lastfm_api_secret.txt
  lastfm_password:
    file: ./secrets/lastfm_password.txt
```

Create the three files (one value per file, no trailing whitespace), restrict their permissions (`chmod 600`), and keep the `secrets/` directory out of version control.

Then replace the credential-retrieval block in `scrobble_plex.py` with a version that tries keyring first and falls back to secret files / environment variables, so the same script works on a bare-metal host **and** in Docker:

```python
def get_secret(keyring_user, secret_file, env_var):
    # 1. Native credential store (bare-metal installs)
    try:
        import keyring
        value = keyring.get_password("lastfm", keyring_user)
        if value:
            return value
    except Exception:
        pass
    # 2. Docker secret file
    path = "/run/secrets/" + secret_file
    if os.path.exists(path):
        with open(path, "r", encoding="utf-8") as f:
            return f.read().strip()
    # 3. Environment variable
    return os.environ.get(env_var)

lastfm_username = "YOUR_LASTFM_USERNAME"

api_key = get_secret("api_key", "lastfm_api_key", "LASTFM_API_KEY")
api_secret = get_secret("api_secret", "lastfm_api_secret", "LASTFM_API_SECRET")
lastfm_password = get_secret(lastfm_username, "lastfm_password", "LASTFM_PASSWORD")
```

(If you prefer plain environment variables over secrets, set `LASTFM_API_KEY`, `LASTFM_API_SECRET`, and `LASTFM_PASSWORD` under `environment:` — simpler, but the values are visible via `docker inspect` and in the compose file itself.)

### 7.4 Alternative: file-based keyring backend inside the container

If you want to keep the script unmodified, install a file-backed keyring inside the container (add `keyrings.alt` to `INSTALL_PIP_PACKAGES`), then exec in and store the credentials once:

```
docker exec -it tautulli python3 -c "import keyring; keyring.set_password('lastfm','api_key','YOUR_API_KEY')"
docker exec -it tautulli python3 -c "import keyring; keyring.set_password('lastfm','api_secret','YOUR_API_SECRET')"
docker exec -it tautulli python3 -c "import keyring; keyring.set_password('lastfm','YOUR_LASTFM_USERNAME','YOUR_LASTFM_PASSWORD')"
```

Caveats: the plaintext-file backend stores secrets **unencrypted** in the user's home directory inside the container, and that file must live on a persisted volume (e.g., under `/config`) or it disappears on container recreation. You may need a `keyringrc.cfg` pointing the backend's file at a persisted path. For containers, §7.3 is the cleaner and more conventional approach.

### 7.5 Docker testing

```
docker exec -it tautulli python3 /config/scripts/scrobble_plex.py "start" "Test Artist" "Test Album" "Test Track" 240 0
docker exec -it tautulli python3 /config/scripts/scrobble_plex.py "stop" "Test Artist" "Test Album" "Test Track" 240 80
```

The first should set Now Playing on your Last.fm profile; the second should produce a scrobble (delete it from your library afterward). Then run the same via the agent's **Test Notifications** tab and finally with real playback.

---

## 8. Testing (bare-metal)

1. **Manual test first**, as the account Tautulli runs under:
   ```
   python scrobble_plex.py "start" "Test Artist" "Test Album" "Test Track" 240 0
   ```
   You should see a `PLEX:` line, then `LAST.FM: Updated now playing...`, and the test track should appear as Now Playing on your Last.fm profile.

2. **Test a scrobble:**
   ```
   python scrobble_plex.py "stop" "Test Artist" "Test Album" "Test Track" 240 80
   ```
   80% ≥ 50%, so this scrobbles. Delete the test scrobble from your Last.fm library afterward.

3. **Test through Tautulli:** open the agent → **Test Notifications** tab → select a trigger → Test. Then play a real track in Plex past 50% and stop it.

4. **Check the logs:** Tautulli → **View Logs → Notification Logs**, plus the main Tautulli log for script stdout. The script's `print()` output appears there, showing exactly what arguments arrived and what pylast did.

---

## 9. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Credentials return `None` from Tautulli but work manually | Stored under a different account/environment than the one Tautulli executes scripts in. See §4 (bare-metal) or §7.2 (Docker). |
| `keyring.errors.NoKeyringError` | No native backend available (headless/container). Use §4.2 or §7.3. |
| `pylast.WSError: Invalid API key` | API key/secret mistyped when stored. Re-store and verify with `get_password` (or check the secret files). |
| Authentication failure | Password stored under a username that doesn't match `lastfm_username` in the script, or wrong password. |
| Script fires on movies/TV | Missing the Media Type = `track` condition (§6.4). |
| `ModuleNotFoundError: pylast` (Docker) | Dependencies not installed in the container — confirm the `universal-package-install` mod ran (check container logs at startup) or that your derived image built correctly. Packages must be reinstalled if you switch images. |
| Script works, then breaks after image update | Derived-image approach: rebuild. Mod approach: verify `INSTALL_PIP_PACKAGES` still present in compose. |
| `UnicodeEncodeError` in logs | Should be prevented by the `sys.stdout.reconfigure(encoding='utf-8')` lines — verify they're intact and Python is 3.7+. |
| Script times out | Increase Script Timeout beyond 30s; slow Last.fm API responses or blocked outbound traffic can stall pylast. |
| Nothing happens at all | Check Notification Logs to confirm the trigger fired; confirm Script Folder/File paths (container-side paths in Docker); confirm the correct Python interpreter runs `.py` files. |
| Wrong artist scrobbled on collab tracks | Expected: the script intentionally strips `feat./ft./,/&` and keeps the primary artist. Adjust the regex if you want different behavior. |

---

## 10. Behavior notes & tuning

- **Scrobble threshold:** hardcoded at `progress >= 50`. Last.fm's own rule is 50% *or* 4 minutes, whichever comes first; adjust the condition if you want the 4-minute rule for long tracks.
- **Pause handling:** rather than leaving a stale Now Playing for the track's full duration, pause resubmits Now Playing with `duration=1`, causing Last.fm to expire it almost immediately. Resume restores it.
- **Seek/skip behavior:** `{progress_percent}` reflects position at stop, not cumulative listen time, so seeking to 60% and stopping will still scrobble. This matches how most Plex scrobblers behave.
