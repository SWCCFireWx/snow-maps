"""
Makes the NM/AZ snow maps from NOHRSC National Gridded Snowfall Analysis data.

  output/snow_map_season.png  - season total (Sep 30 12Z to latest 12Z)
  output/snow_map_72h.png     - 72-hour total ending at the latest 12Z

Run by GitHub Actions every morning (see .github/workflows/snow-maps.yml).
Can also be run by hand:  python make_snow_maps.py
"""
import os
import re
import sys
import datetime as dt

import requests
import numpy as np
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
import matplotlib.colors as mcolors
import xarray as xr
import cartopy.crs as ccrs
import cartopy.feature as cfeature
from metpy.plots import USCOUNTIES, USSTATES

# ---- Settings ----------------------------------------------------------------
MAPS_TO_MAKE = ['season', '72h']
OUTPUT_DIR = 'output'
OUTPUT_NAMES = {
    'season': 'snow_map_season.png',
    '72h':    'snow_map_72h.png',
}
MAX_DAYS_BACK = 5          # how far back to look if today's 12Z file isn't posted yet
BASE = 'https://www.nohrsc.noaa.gov/snowfall_v2/data'
DOWNSCALE_STEP = 0.005     # degrees; grid is linearly interpolated to this spacing for a smoother map

# Map area (NM + AZ)
lat_min, lat_max = 31.14, 37.34
lon_min, lon_max = -115.18, -102.57

locations = {
    'Winslow': (35.0242, -110.6974),
    'Payson': (34.2309, -111.3251),
    'Prescott': (34.5400, -112.4685),
    'Flagstaff': (35.1383, -111.6661),      # KFLG ASOS
    'Show Low': (34.2542, -110.0298),
    'Alpine': (33.8492, -109.1312),
    'Mount Lemmon': (32.4434, -110.7590),
    'Albuquerque': (35.0402, -106.6092),    # KABQ ASOS
    'Farmington': (36.7281, -108.2187),
    'Clovis': (34.4048, -103.2052),
    'Raton': (36.9034, -104.4391),
    'Santa Fe': (35.6870, -105.9378),
    'Red River': (36.7064, -105.4081),
    'Silver City': (32.7701, -108.2803),
    'Ruidoso': (33.3317, -105.6730),
}

# These points use the measured station total from ACIS instead of the grid
# (falls back to the grid if ACIS doesn't answer).
acis_targets = {
    'Flagstaff': 'USW00003103',    # Flagstaff Pulliam Airport (KFLG)
    'Albuquerque': 'USW00023050',  # Albuquerque International Sunport (KABQ)
}

# NWS color scale
hex_colors = [
    '#FFFFFF', '#C2D8E6', '#7AB1D3', '#247DB8', '#00328E',
    '#FFFFB1', '#FFC200', '#FF7000', '#E10000', '#900000',
    '#590000', '#C8B8E8', '#8A60B3', '#59177D', '#2D004F',
    '#120014', '#4CE3FF'
]
cmap = mcolors.ListedColormap(hex_colors)
boundaries = [0, 0.1, 1, 2, 6, 12, 24, 36, 48, 72, 96, 120, 180, 240, 360, 480, 600, 1000]
norm = mcolors.BoundaryNorm(boundaries, cmap.N)


# ---- Finding and downloading NOHRSC files ------------------------------------
def season_start_for(end):
    """NOHRSC seasons start Sep 30 12Z (e.g. 2025093012 for the 2025-2026 season)."""
    year = end.year if end >= dt.datetime(end.year, 9, 30, 12) else end.year - 1
    return dt.datetime(year, 9, 30, 12)


def expected_name(kind, end):
    stamp = end.strftime('%Y%m%d%H')
    if kind == 'season':
        return f'sfav2_CONUS_{season_start_for(end):%Y%m%d%H}_to_{stamp}.nc'
    return f'sfav2_CONUS_72h_{stamp}.nc'


def listing_pattern(kind, stamp):
    if kind == 'season':
        return rf'sfav2_CONUS_\d{{10}}_to_{stamp}\.nc'
    return rf'sfav2_CONUS_72h_{stamp}\.nc'


def url_exists(url):
    try:
        r = requests.get(url, stream=True, timeout=30)
        ok = r.status_code == 200
        r.close()
        return ok
    except requests.RequestException:
        return False


def find_latest(kind):
    """Try today's 12Z file first, then step back a day at a time."""
    now = dt.datetime.utcnow()
    end = now.replace(hour=12, minute=0, second=0, microsecond=0)
    if now < end:                      # before 12Z today -> start with yesterday
        end -= dt.timedelta(days=1)

    for _ in range(MAX_DAYS_BACK):
        stamp = end.strftime('%Y%m%d%H')
        folder = f'{BASE}/{end:%Y%m}'

        name = expected_name(kind, end)
        if url_exists(f'{folder}/{name}'):
            return f'{folder}/{name}', name, end

        # Fallback: read the folder listing in case the name differs slightly
        try:
            html = requests.get(folder + '/', timeout=30).text
            hits = sorted(set(re.findall(listing_pattern(kind, stamp), html)))
            if hits:
                return f'{folder}/{hits[0]}', hits[0], end
        except requests.RequestException:
            pass

        print(f'[{kind}] not posted yet for {stamp} - trying the day before')
        end -= dt.timedelta(days=1)

    raise RuntimeError(f'No {kind} file found in the last {MAX_DAYS_BACK} days.')


def download(url, name):
    with requests.get(url, stream=True, timeout=120) as r:
        r.raise_for_status()
        with open(name, 'wb') as f:
            for chunk in r.iter_content(1 << 20):
                f.write(chunk)


# ---- ACIS station totals --------------------------------------------------------
def query_direct_acis(sid, sdate, edate):
    """Sum daily snowfall at an ACIS station between two dates (inclusive).
    Returns None if the request fails or the station has no usable data."""
    try:
        payload = {'sid': sid, 'sdate': sdate, 'edate': edate, 'elems': [{'name': 'snow'}]}
        res = requests.post('https://data.rcc-acis.org/StnData', json=payload, timeout=10).json()
        if 'data' not in res:
            return None

        total = 0.0
        has_record = False
        for _, val in res['data']:
            if val == 'T':
                has_record = True
            elif val not in ['M', 'S', 'A', '']:
                try:
                    total += float(val)
                    has_record = True
                except ValueError:
                    pass
        return total if has_record else None
    except Exception:
        return None


def acis_dates(kind, valid_end):
    """Station days that line up with the grid's 12Z-to-12Z window.
    The day ending at 12Z isn't finished yet, so it stops at the day before."""
    last_day = (valid_end - dt.timedelta(days=1)).date()
    if kind == 'season':
        first_day = season_start_for(valid_end).date()
    else:
        first_day = (valid_end - dt.timedelta(days=3)).date()
    return first_day.isoformat(), last_day.isoformat()


# ---- Map drawing ---------------------------------------------------------------
def fmt(d):
    return f'{d.month}/{d.day}/{d.year}'


def make_map(kind, file_name, valid_end):
    ds = xr.open_dataset(file_name)
    snow_var = 'Data'

    if ds.lon.min() >= 0 and lon_min < 0:
        lon_min_slice, lon_max_slice = lon_min + 360, lon_max + 360
    else:
        lon_min_slice, lon_max_slice = lon_min, lon_max
    lat_descending = bool(ds.lat[0] > ds.lat[-1])
    lat_slice = slice(lat_max, lat_min) if lat_descending else slice(lat_min, lat_max)

    snow_data = ds.sel(lat=lat_slice, lon=slice(lon_min_slice, lon_max_slice))[snow_var] * 39.3701  # m -> in

    # Downscale with linear interpolation for a smoother-looking map
    new_lon = np.arange(lon_min_slice, lon_max_slice, DOWNSCALE_STEP)
    if lat_descending:
        new_lat = np.arange(lat_max, lat_min, -DOWNSCALE_STEP)
    else:
        new_lat = np.arange(lat_min, lat_max, DOWNSCALE_STEP)
    snow_data = snow_data.interp(lat=new_lat, lon=new_lon, method='linear')

    # Point values: ACIS for the listed stations, downscaled grid for the rest
    # (no-data cells show as 0.0")
    sdate, edate = acis_dates(kind, valid_end)
    vals = {}
    for name, (lat, lon) in locations.items():
        if name in acis_targets:
            acis_val = query_direct_acis(acis_targets[name], sdate, edate)
            if acis_val is not None:
                vals[name] = float(acis_val)
                print(f'[{kind}] {name}: {acis_val:.1f}" (ACIS {acis_targets[name]}, {sdate} to {edate})')
                continue
            print(f'[{kind}] {name}: ACIS request failed, using the grid')

        q_lon = lon + 360 if (ds.lon.min() >= 0 and lon < 0) else lon
        v = float(np.squeeze(snow_data.sel(lat=lat, lon=q_lon, method='nearest').values))
        vals[name] = 0.0 if np.isnan(v) else v

    fig = plt.figure(figsize=(14, 8))
    ax = plt.axes(projection=ccrs.Mercator())
    ax.set_extent([lon_min, lon_max, lat_min, lat_max], crs=ccrs.PlateCarree())

    ax.add_feature(USSTATES.with_scale('500k'), edgecolor='black', linewidth=1.5, zorder=3)
    ax.add_feature(cfeature.BORDERS.with_scale('10m'), edgecolor='black', linewidth=1.5, zorder=3)
    ax.add_feature(USCOUNTIES.with_scale('500k'), edgecolor='gray', linewidth=0.5, zorder=2)

    snow_data.plot(
        ax=ax, transform=ccrs.PlateCarree(), cmap=cmap, norm=norm, zorder=1, alpha=0.75,
        cbar_kwargs={'label': 'Snowfall (inches)', 'ticks': boundaries[:-1],
                     'spacing': 'uniform', 'fraction': 0.046, 'pad': 0.04}
    )

    for name, (lat, lon) in locations.items():
        ax.plot(lon, lat, marker='o', color='white', markeredgecolor='black', markersize=6,
                transform=ccrs.PlateCarree(), zorder=4)
        ax.text(lon + 0.05, lat + 0.05, f"{name}\n({vals[name]:.1f}\")",
                transform=ccrs.PlateCarree(), fontsize=8, fontweight='bold', color='black',
                bbox=dict(facecolor='white', alpha=0.8, edgecolor='black', boxstyle='round,pad=0.2'),
                zorder=5)

    if kind == 'season':
        title = f'Total Season Snowfall ({fmt(season_start_for(valid_end))} to {fmt(valid_end)} 12Z)'
    else:
        start = valid_end - dt.timedelta(hours=72)
        title = f'72-Hour Snowfall ({fmt(start)} 12Z to {fmt(valid_end)} 12Z)'
    ax.set_title(title)

    out_path = os.path.join(OUTPUT_DIR, OUTPUT_NAMES[kind])
    fig.savefig(out_path, dpi=150, bbox_inches='tight', facecolor='white')
    plt.close(fig)
    ds.close()
    print(f'[{kind}] saved {out_path}')


def main():
    os.makedirs(OUTPUT_DIR, exist_ok=True)
    made = 0
    for kind in MAPS_TO_MAKE:
        try:
            url, name, valid_end = find_latest(kind)
            print(f'[{kind}] downloading {url}')
            download(url, name)
            make_map(kind, name, valid_end)
            made += 1
        except Exception as e:          # keep going so one failure doesn't block the other map
            print(f'[{kind}] FAILED: {e}')
    if made == 0:
        sys.exit('No maps were made.')


if __name__ == '__main__':
    main()
