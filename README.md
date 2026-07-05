# SpotiPi

Shows the current Spotify album cover full-screen on a 32x32 RGB LED matrix. The cover fills the entire panel and updates whenever the track changes; when nothing is playing, the matrix shows a dim idle frame.

This uses Spotify's Web API `currently-playing` endpoint, not the browser-only Web Playback SDK. The first run opens Spotify OAuth, then the script stores a refresh token in `.cache/spotify_token.json`.

## What you need

- A Raspberry Pi (a Pi 3/4/Zero 2 is more comfortable than a Pi Zero — see the memory note below).
- A 32x32 RGB LED matrix panel wired to the Pi, typically through an Adafruit RGB Matrix HAT/Bonnet.
- A Spotify account (the account whose playback you want to display).

## Files

- `spotify_matrix.py` - Pi runtime script.
- `.env` - local Spotify credentials, ignored by Git.
- `.env.example` - template for recreating local config.
- `requirements.txt` - Python dependencies, excluding the hardware-specific RGB matrix bindings.

## Step 1: Create a Spotify app

1. Go to the [Spotify developer dashboard](https://developer.spotify.com/dashboard) and log in with your Spotify account.
2. Click **Create app**. Name and description can be anything (e.g. "SpotiPi").
3. Under **Redirect URIs**, add exactly:

   ```text
   http://127.0.0.1:8888/callback
   ```

4. Under **Which API/SDKs are you planning to use?**, select **Web API**, then save.
5. Open the app's **Settings** and copy the **Client ID** and **Client secret** — you'll need them in Step 4.

## Step 2: Install the RGB matrix bindings (on the Pi)

The Python bindings come from the [hzeller/rpi-rgb-led-matrix](https://github.com/hzeller/rpi-rgb-led-matrix) project:

```bash
sudo apt-get update && sudo apt-get install -y git make g++ python3-dev python3-venv cython3
git clone https://github.com/hzeller/rpi-rgb-led-matrix.git
cd rpi-rgb-led-matrix
make build-python PYTHON=$(which python3)
sudo make install-python PYTHON=$(which python3)
```

If you're using the Adafruit HAT/Bonnet, follow that project's README notes for the `adafruit-hat` wiring; no code changes are needed here — it's selected with the `--hardware-mapping adafruit-hat` flag at runtime.

## Step 3: Install this project (on the Pi)

```bash
git clone https://github.com/Cyburban/spotipi.git
cd spotipi
python3 -m venv .venv --system-site-packages
source .venv/bin/activate
pip install -r requirements.txt
```

The `--system-site-packages` flag lets the venv see the system-wide `rgbmatrix` bindings installed in Step 2.

This install sometimes crashes the raspberry pi zero, I had to do some fancy workarounds. Might be easier to use a pi with more memory!

## Step 4: Configure credentials

Copy the template and fill in the values from Step 1:

```bash
cp .env.example .env
nano .env
```

`.env` needs all three values (the script exits immediately if any are missing):

```text
SPOTIFY_CLIENT_ID=your_client_id
SPOTIFY_CLIENT_SECRET=your_client_secret
SPOTIFY_REDIRECT_URI=http://127.0.0.1:8888/callback
```

The redirect URI must match the one you allowlisted in the Spotify dashboard exactly. `.env` is listed in `.gitignore`, so it stays out of Git.

## Step 5: Authorize Spotify (first run only)

Run the one-time OAuth flow before involving the matrix:

```bash
.venv/bin/python spotify_matrix.py --auth-only --no-browser
```

The script prints an authorization URL and waits for Spotify to redirect back to `127.0.0.1:8888`. Because the Pi is usually headless, forward that port from your computer first, then open the printed URL in your local browser:

```bash
ssh -L 8888:127.0.0.1:8888 pi@raspberrypi.local
```

After you approve, the token is cached in `.cache/spotify_token.json` (also Git-ignored) and refreshed automatically from then on — you won't need the browser again.

## Step 6: Run

This is the working command to run the script on your raspberry pi with a 32x32 panel on an Adafruit HAT:

```bash
sudo -E .venv/bin/python spotify_matrix.py \
  --gpio-slowdown 4 \
  --no-hardware-pulse \
  --hardware-mapping adafruit-hat
```

`sudo` is required for the matrix hardware's GPIO timing; `-E` keeps your environment so `.env` and the token cache are found. Rows and columns default to 32, so no size flags are needed.

Useful hardware options:

```bash
sudo -E .venv/bin/python spotify_matrix.py \
  --hardware-mapping regular \
  --gpio-slowdown 2 \
  --brightness 65
```

If the panel stays dark or flickers, first verify the wiring independently of Spotify with the built-in test pattern:

```bash
sudo -E .venv/bin/python spotify_matrix.py --test-pattern --hardware-mapping adafruit-hat
```

## Run at boot (optional)

To start the display automatically on power-up, create a systemd unit. Adjust the paths and flags to match your setup (this example assumes the project lives at `/home/pi/spotipi`):

```bash
sudo tee /etc/systemd/system/spotipi.service > /dev/null <<'EOF'
[Unit]
Description=Spotify album art on RGB LED matrix
Wants=network-online.target
After=network-online.target

[Service]
WorkingDirectory=/home/pi/spotipi
ExecStart=/home/pi/spotipi/.venv/bin/python spotify_matrix.py --gpio-slowdown 4 --no-hardware-pulse --hardware-mapping adafruit-hat
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now spotipi.service
```

The service runs as root (matrix GPIO needs it), and `WorkingDirectory` makes the script find `.env` and `.cache/spotify_token.json`. Complete Step 5 once before enabling the service so the cached token already exists. Check on it with `systemctl status spotipi` or `journalctl -u spotipi -f`.

## Testing without a Pi

For a non-Pi test that writes one PNG frame instead of using matrix hardware:

```bash
python spotify_matrix.py --mock-output /tmp/spotify-matrix-frame.png --once
```

To see what a full-screen cover looks like at 32x32 without Spotify or hardware, render local preview frames:

```bash
python spotify_matrix.py --preview-frames /tmp/spotify-matrix-preview
```
