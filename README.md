# Snow maps (NM / AZ)

Makes two maps from the NOHRSC National Gridded Snowfall Analysis every morning:

| Map | Link used by the Slides script |
|---|---|
| Season total | `https://raw.githubusercontent.com/<your-username>/snow-maps/maps/snow_map_season.png` |
| 72-hour total | `https://raw.githubusercontent.com/<your-username>/snow-maps/maps/snow_map_72h.png` |

## One-time setup

1. On GitHub, click **+ > New repository**. Name it `snow-maps`, set it to **Public**, and click **Create repository**.
2. Click **uploading an existing file**. Drag in `make_snow_maps.py`, `requirements.txt` and `README.md`, then **Commit changes**.
3. Click **Add file > Create new file**. In the name box type exactly
   `.github/workflows/snow-maps.yml`
   (typing the `/` makes the folders). Paste in the contents of `snow-maps.yml`, then **Commit changes**.
4. Go to the **Actions** tab. If asked, click the button to enable workflows.
5. Click **Daily snow maps > Run workflow > Run workflow** to make the first maps now.
   It takes about 3–5 minutes. When it shows a green check, a `maps` branch exists with both PNGs.

After that it runs on its own every morning. Nothing else to do.

## If something goes wrong

- **Red X on a run:** click it, then the failed step, to see the message. A run that couldn't find
  NOHRSC data keeps yesterday's maps in place.
- **Want to change the map** (points, colors, area): edit `make_snow_maps.py` on GitHub, then use
  **Run workflow** to remake it.
- GitHub pauses scheduled runs in a repo with no activity for 60 days. The daily publish should keep
  it active, but if runs ever stop, the Actions tab will show a button to turn them back on.
