# Vibe Coding Tethys Apps with Claude

A guide for building web apps with AI assistance and integrating them with a Tethys Portal.

1. **Part 1** — Install the Tethys Portal
2. **Part 2** — Build a traditional Tethys app (lives inside the portal framework)
3. **Part 3** — Build a standalone app and register it as a Proxy App (lives outside the portal, launched from it)

---

# Part 1: Install the Tethys Portal

Before anything, you need a running Tethys Portal. This only needs to be done
once.

**CLI commands:**
```bash
conda create -n tethys python=3.12 -y
conda activate tethys          # IMPORTANT: must activate BEFORE running tethys commands
pip install tethys-platform
tethys gen portal_config
tethys db configure
tethys start
```

**Or tell Claude:**
> "Install a Tethys Portal for me. Create a conda environment called 'tethys'
> with Python 3.12, activate it, install tethys-platform via pip, generate the
> portal config, configure the database, and start the server. Make sure the
> tethys conda environment is activated before running any tethys commands."

The portal will be running at `http://localhost:8000`. Default admin login is
`admin` / `pass`.

**Important notes:**
- **Use pip, not conda** to install `tethys-platform`. Conda's solver can take
  30+ minutes resolving dependencies. Pip finishes in under a minute.
- **Activate the `tethys` env first.** Tethys determines its home directory
  (`TETHYS_HOME`) from the `CONDA_DEFAULT_ENV` environment variable. If the env
  named `tethys` is active, `TETHYS_HOME` is `~/.tethys/`. If any other env is
  active (e.g., `base`), it becomes `~/.tethys/<env_name>/`. Running
  `tethys db configure` from the wrong env creates the database in the wrong
  place and causes `no such table` errors when you start the server.

---

# Part 2: Traditional Tethys App

Build an app that lives inside the Tethys framework — uses Django templates, Tethys routing, and runs as part of the portal process.

## How Traditional Tethys Apps Work

A Tethys app is a Python package installed under the `tethysapp` namespace. When the portal starts, it discovers all installed apps, syncs their metadata to the database, and registers their URL routes. Key components:

| File | Purpose |
|------|---------|
| `app.py` | App metadata class (name, icon, color, permissions, settings) |
| `controllers.py` | Route handlers decorated with `@controller` |
| `templates/<app>/` | Django HTML templates extending `tethys_apps/app_base.html` |
| `public/` | Static assets (CSS, JS, images) |
| `pyproject.toml` | Python package config (makes the app pip-installable) |
| `install.yml` | Dependency manifest for `tethys install` |

The critical contract: as long as these files exist with the right structure, Tethys will discover and serve the app.

## Step 1: Scaffold and Install an App

Before vibe coding, you need a scaffolded app installed in dev mode.

**CLI commands:**
```bash
conda activate tethys
tethys scaffold <app_name> /path/to/your/projects -d
cd /path/to/your/projects/tethysapp-<app_name>
tethys install -d
```

**Or tell Claude:**
> "Scaffold a new Tethys app called 'my_app' in /path/to/your/projects and
> install it in dev mode."

The scaffold creates the boilerplate package structure. The `-d` flag on install
does an editable install so code changes are reflected immediately. The app will
appear in the portal at `http://localhost:8000/apps/`.

**Why scaffold first?** The `tethysapp/` namespace package pattern (no
`__init__.py` in the parent) and the `pyproject.toml` entry points are easy to
get wrong. The scaffold gets this right in seconds.

## Step 2: Vibe Code the App

With the skeleton installed and the dev server running, iterate conversationally
with Claude. Just describe what you want and Claude modifies the generated files.

### Rules of the road (Tethys conventions Claude must follow)

- **Controllers** use the `@controller` decorator from `tethys_sdk.routing`
- **Templates** must extend `tethys_apps/app_base.html` (or the app's `base.html` which extends it)
- **Static files** go in `public/` and are referenced with `{% static 'app_name/...' %}`
- **Authentication** is on by default — `@controller(login_required=True)` is the default
- **The `index` controller** in `app.py` defines which controller serves the app's home page
- **URL patterns** are auto-generated from function names: `add_dam` becomes `/apps/app-name/add-dam/`
- **Use `App.render()`** instead of Django's `render()` for proper context injection
- **Template blocks**: `app_content`, `app_navigation_items`, `app_actions`, `scripts`, `styles`

### Example vibe coding session

```
You: "Add a page that shows a Leaflet map where users can click to add points"
Claude: [modifies controllers.py, creates/updates template, adds JS]
You: [refresh browser, see result]
You: "The points should be saved to a database. Add a model and a form."
Claude: [adds persistent store setting to app.py, creates model, updates controller]
You: "Now add a table below the map showing all saved points"
Claude: [updates template and controller to query and display data]
```

## Step 3: Restart and Verify

After modifying `app.py` (especially settings, permissions, or the app class),
restart the server.

**CLI commands:**
```bash
# Ctrl+C to stop, then:
tethys start
```

If you added persistent store settings or new permissions:
```bash
tethys db sync
```

**Or tell Claude:**
> "Restart the Tethys server and sync the database."

## Step 4: Polish and Package

**Tell Claude:**
> "Update the app icon, description, and tags in app.py. Update install.yml with
> all the pip dependencies you added."

Or do it manually:
1. Update `install.yml` with any pip/conda dependencies Claude added
2. Update the app icon (`public/images/icon.gif`)
3. Update metadata in `app.py` (description, tags, color)
4. Test with a non-admin user to verify permissions work

## Tips for Traditional Apps

- **Use Tethys Gizmos** — Built-in UI components (maps, plots, tables, forms). Tell Claude: "Use the Tethys MapView gizmo for the map instead of raw Leaflet." Docs: http://docs.tethysplatform.org/en/stable/tethys_sdk/gizmos.html
- **Let Claude read error pages** — Tethys runs in DEBUG mode, so error pages are detailed. Paste them to Claude.
- **Don't fight the framework** — If Claude suggests Flask-style routes or a standalone Django project, redirect it. The scaffolded structure is the contract.

---

# Part 3: Standalone App as a Tethys Proxy App

Build any kind of web app (React, Streamlit, Dash, Flask, etc.) using AI, then register it with the Tethys Portal so users can launch it from the app library. This is the **Proxy App** approach.

## What Is a Proxy App?

A Proxy App is a link registered in the Tethys Portal that points to an external web application. It:

- **Appears in the app library** alongside native Tethys apps (with an icon, name, description, and tags)
- **Launches the external app** when clicked (opens in a new tab by default, or same tab)
- **Can restrict access** via Tethys user/group permissions (controls who sees the app in the library)
- **Does NOT embed the app** in the portal — it's a direct link to wherever the app is running

This is how you'd integrate something like [AquiferX](https://github.com/njones61/aquiferx) (a React/Vite SPA) with a Tethys Portal.

## Step 1: Vibe Code the Standalone App

Build your app with whatever framework you want. There are no Tethys conventions
to follow — it's a completely independent web application.

**Tell Claude:**
> "Build me a Streamlit app that visualizes USGS streamflow data on a map. Put it
> in a folder called my-streamlit-app. Install any needed packages."

Or start from an existing app like AquiferX.

### Good frameworks for vibe coding

| Framework | Good for | Launch command |
|-----------|----------|----------------|
| **React + Vite** | Rich interactive SPAs | `npm run dev` (port 3000) |
| **Streamlit** | Quick data dashboards | `streamlit run app.py` (port 8501) |
| **Dash / Plotly** | Data-heavy dashboards | `python app.py` (port 8050) |
| **Flask** | Lightweight APIs + pages | `flask run` (port 5000) |
| **FastAPI** | APIs with auto-docs | `uvicorn main:app` (port 8000) |
| **Panel / Bokeh** | Scientific visualizations | `panel serve app.py` (port 5006) |

## Step 2: Run the App

Start the standalone app on its own port.

**CLI commands:**
```bash
# React/Vite app
cd aquiferx && npm install && npm run dev      # http://localhost:3000

# Streamlit app
cd my-streamlit-app && streamlit run app.py    # http://localhost:8501
```

**Or tell Claude:**
> "Start the app for me."

Claude will run the appropriate command. The app must be accessible at a URL
(localhost for dev, a real URL for production).

## Step 3: Register as a Proxy App in Tethys

### Option A: CLI

```bash
conda activate tethys
tethys proxyapp add \
  "AquiferX" \
  "http://localhost:3000" \
  "Interactive groundwater data visualization and analysis" \
  "https://raw.githubusercontent.com/njones61/aquiferx/main/public/icon.png" \
  "groundwater,hydrology,visualization"
```

Full argument list:
```
tethys proxyapp add <name> <endpoint> [description] [icon_url] [tags] \
  [enabled] [show_in_apps_library] [back_url] [open_new_tab] \
  [display_external_icon] [order]
```

### Option B: Django Admin

1. Go to `http://localhost:8000/admin/`
2. Log in as admin
3. Navigate to **Tethys Apps > Proxy Apps > Add**
4. Fill in the fields and save

### Option C: Tell Claude

> "Register my app as a proxy app in the Tethys Portal. The app is called
> 'AquiferX', it's running at http://localhost:3000, and it's a groundwater
> visualization tool. Use the tethys proxyapp add command."

### Managing Proxy Apps

**CLI commands:**
```bash
tethys proxyapp list                           # List all proxy apps
tethys proxyapp list -v                        # Verbose listing
tethys proxyapp update "AquiferX" -s endpoint "http://newurl:3000"
tethys proxyapp update "AquiferX" -s enabled False
```

**Or tell Claude:**
> "List all proxy apps registered in Tethys."
> "Update the AquiferX proxy app endpoint to http://localhost:5000."

## Step 4: Control Access (Optional)

By default, all logged-in users can see and launch proxy apps. To restrict access:

1. Enable restricted app access in `portal_config.yml`:
   ```yaml
   settings:
     ENABLE_RESTRICTED_APP_ACCESS: True
   ```

2. Assign permissions in Django Admin:
   - Go to **Admin > Tethys Apps > Proxy Apps**
   - Select the app
   - Use the **Object permissions** section (powered by Django Guardian)
   - Grant `access_app` permission to specific users or groups

**Or tell Claude:**
> "Enable restricted app access in the Tethys portal config. Then show me how to
> assign permissions to specific users in the Django admin."

With this enabled, only users/groups with the `access_app` permission will see
the app in the library.

## Authentication Considerations

Tethys controls **who can see and click the link**, but it does **not** forward
authentication to the standalone app. The standalone app is responsible for its
own auth. Options:

| Approach | Complexity | Description |
|----------|------------|-------------|
| **No app auth** | Simple | The standalone app is open. Tethys just controls who sees the link. Good for internal/trusted networks. |
| **Shared OAuth/SSO** | Medium | Configure the standalone app to use the same OAuth2 provider (e.g., Google, GitHub) as Tethys. Users log in once via SSO. |
| **Reverse proxy with auth** | Medium | Put both Tethys and the app behind nginx. Use nginx to check the Tethys session cookie before forwarding requests to the standalone app. |
| **API token passthrough** | Advanced | Tethys generates a JWT/token and passes it as a URL parameter. The standalone app validates it against Tethys. |

For a workshop setting, **"no app auth"** is the simplest — Tethys controls visibility, and the app itself is open on localhost.

---

# Workshop Outline

## Session 1: Setup (~15 min)
- Install Tethys Portal (Part 1)
- Start the portal, log in, explore the app library
- Brief overview of the two app approaches (Parts 2 and 3)

## Session 2: Traditional Tethys App (~30 min)
- Scaffold a new app with `tethys scaffold`
- Install in dev mode with `tethys install -d`
- Vibe code with Claude:
  - "Add a Leaflet map to the home page"
  - "Add a form to submit data"
  - "Show the submitted data in a table"
- See results live in the portal

## Session 3: Standalone + Proxy App (~45 min)
- Vibe code a standalone app from scratch (e.g., React, Streamlit, or Dash)
- Or clone an existing app (e.g., AquiferX)
- Run it on a separate port
- Register it as a Proxy App in Tethys
- Launch it from the portal
- Demonstrate access control with permissions

## Session 4: Discussion (~15 min)
- Compare the two approaches: when to use which
- What Claude got right/wrong
- How this workflow scales to real projects
- Packaging and sharing apps

---

# Quick Comparison

| | Traditional Tethys App | Proxy App |
|---|---|---|
| **Framework** | Django (Tethys SDK) | Anything (React, Streamlit, Flask, etc.) |
| **Runs as** | Part of the Tethys portal process | Separate process on its own port |
| **UI** | Tethys templates, header, nav sidebar | Completely custom |
| **Authentication** | Built-in (Django auth) | Must handle separately |
| **Portal integration** | Deep (same URL space, shared DB) | Shallow (link in app library) |
| **Vibe coding freedom** | Must follow Tethys conventions | No constraints |
| **Best for** | Apps that need tight portal integration | Existing apps, complex UIs, non-Python apps |

---

# Workshop Prompts

Each app idea is presented as a pair: one prompt for the **Traditional Tethys** approach and one for the **Standalone + Proxy App** approach. These are complete, copy-pasteable prompts to give directly to Claude.

---

## Prompt Pair 1: Bear River Streamflow Explorer

### Traditional Tethys Version

```
First, scaffold and install a new Tethys app:
1. Run: tethys scaffold streamflow_explorer /Users/njones/python_projects/tethys -d
2. Run: cd /Users/njones/python_projects/tethys/tethysapp-streamflow_explorer && tethys install -d

Then build an app called "Bear River Streamflow Explorer" that displays USGS stream
gages in the Bear River basin on a map. When the user clicks a gage, the app
fetches real streamflow data from the USGS and displays a hydrologic analysis.

Use the Tethys MapView gizmo for the map and Plotly JS for the plots.

STEP 1 - Discover gages (do this at app startup or on first page load):
- Use the USGS Water Services site service
  (https://waterservices.usgs.gov/nwis/site/) to find all active stream gages in
  the Bear River basin
- Use HUC code 16010 (Bear River basin) with siteType=ST and siteStatus=active
  and hasDataTypeCd=dv and parameterCd=00060
- Parse the response to get station ID, name, latitude, and longitude for each gage

STEP 2 - Map page (the app's home/index controller):
- Use the Tethys MapView gizmo to display a map centered on the Bear River basin
  (~41.8°N, -111.5°W)
- Show each gage as a clickable point on the map with the station name as a popup
- Use a clean basemap
- When the user clicks a gage, navigate to an analysis page for that station

STEP 3 - Station selector in the app navigation sidebar:
- Show a dropdown or list of available stations so users can also select by name
- Use a Tethys SelectInput gizmo if appropriate

STEP 4 - Analysis page (a second controller, URL like
/apps/streamflow-explorer/analysis/{station_id}/):
- Fetch daily mean discharge (parameter 00060) for the past 3 years from the USGS
  daily values service (https://waterservices.usgs.gov/nwis/dv/)
- Parse the JSON response directly (don't use the dataretrieval package)
- Do the data processing in the controller and pass results to the template

STEP 5 - Create a 3-panel analysis display on the analysis page:

Panel 1 - Hydrograph:
- Daily mean discharge (cfs) vs date using Plotly JS
- 30-day rolling average as a smooth overlay line
- Station name and ID in the title

Panel 2 - Flow Duration Curve:
- Discharge (y-axis, log scale) vs exceedance probability (x-axis, 0-100%)
- Mark and label Q10, Q50, and Q90 with horizontal dashed lines

Panel 3 - Monthly Box Plots:
- Box-and-whisker plots showing daily flow distributions by calendar month
  (Jan-Dec) to show the snowmelt seasonal pattern

STEP 6 - Summary statistics:
- Below the plots, display a summary table using a Tethys TableView gizmo or a
  simple HTML table: period of record, mean/median/min/max discharge, Q10/Q50/Q90

Use the @controller decorator for routing. Templates should extend the app's
base.html. Use App.render() for rendering. Handle API errors gracefully with a
user-friendly message.
```

**After the prompt completes**, restart the Tethys server so the portal picks up
the changes:
```bash
tethys start
```
Or tell Claude: *"Restart the Tethys server."*
Then visit `http://localhost:8000/apps/` to launch the app.

### Standalone Version (Streamlit + Proxy App)

```
Build an interactive web application called "Bear River Streamflow Explorer" that
displays USGS stream gages in the Bear River basin on a map. When the user clicks
a gage, the app fetches real streamflow data from the USGS and displays a
hydrologic analysis.

Use Streamlit with Folium for the map and Matplotlib for the plots. Install any
needed packages automatically.

STEP 1 - Discover gages:
- Use the USGS Water Services site service
  (https://waterservices.usgs.gov/nwis/site/) to find all active stream gages in
  the Bear River basin
- Use HUC code 16010 (Bear River basin) with siteType=ST and siteStatus=active
  and hasDataTypeCd=dv and parameterCd=00060
- Parse the response to get station ID, name, latitude, and longitude for each gage

STEP 2 - Interactive map:
- Display a Folium map centered on the Bear River basin (~41.8°N, -111.5°W)
- Show each gage as a clickable marker with the station name as a tooltip
- Use a clean basemap (CartoDB positron or similar)
- When the user clicks a gage marker, store the selected station ID

STEP 3 - Sidebar or panel with station selector:
- Show a list or dropdown of available stations so users can also select by name
- Display the station name and ID for the currently selected gage

STEP 4 - Data retrieval and analysis (triggered when a gage is selected):
- Fetch daily mean discharge (parameter 00060) for the past 3 years from the USGS
  daily values service (https://waterservices.usgs.gov/nwis/dv/)
- Parse the JSON response directly (don't use the dataretrieval package)

STEP 5 - Create a 3-panel analysis figure for the selected gage:

Panel 1 - Hydrograph:
- Daily mean discharge (cfs) vs date
- 30-day rolling average as a smooth overlay line
- Station name and ID in the panel title

Panel 2 - Flow Duration Curve:
- Discharge (y-axis, log scale) vs exceedance probability (x-axis, 0-100%)
- Mark and label Q10, Q50, and Q90 with horizontal dashed lines and annotations

Panel 3 - Monthly Box Plots:
- Box-and-whisker plots showing daily flow distributions by calendar month
  (Jan-Dec)
- This should clearly show the snowmelt seasonal pattern

STEP 6 - Summary statistics:
- Below the plots, display a clean summary table: period of record,
  mean/median/min/max discharge, and the Q10/Q50/Q90 values

Formatting:
- Professional styling throughout — use a clean matplotlib style
- The map should take roughly the top half of the page, plots below
- Figure size appropriate for the layout (~12x10 inches)
- Handle edge cases: if a gage has no data or the API is slow, show a friendly
  message instead of crashing

Use only standard packages: streamlit, streamlit-folium, folium, numpy, pandas,
matplotlib, and requests.
```

**After building, register as a proxy app:**
```bash
conda activate tethys
tethys proxyapp add "Bear River Streamflow Explorer" "http://localhost:8501" \
  "Interactive USGS streamflow analysis for the Bear River basin" \
  "" "streamflow,USGS,hydrology,Bear River"
```

**Or tell Claude:**
> "Register this app as a proxy app in my Tethys Portal. It's called 'Bear River
> Streamflow Explorer', it's running at http://localhost:8501, and it's an
> interactive USGS streamflow analysis tool for the Bear River basin. Tag it with
> streamflow, USGS, hydrology, Bear River."

---

## Prompt Pair 2: Reservoir Storage Monitor

### Traditional Tethys Version

```
First, scaffold and install a new Tethys app:
1. Run: tethys scaffold reservoir_monitor /Users/njones/python_projects/tethys -d
2. Run: cd /Users/njones/python_projects/tethys/tethysapp-reservoir_monitor && tethys install -d

Then build an app called "Reservoir Storage Monitor" that tracks reservoir storage
levels for major reservoirs in Utah using data from the US Bureau of Reclamation.

STEP 1 - Reservoir data:
- Use the Bureau of Reclamation's Rise API
  (https://data.usbr.gov/rise/api/) to fetch reservoir data
- Target these Utah reservoirs: Flaming Gorge, Strawberry, Deer Creek, Jordanelle,
  and Echo
- For each reservoir, fetch daily storage (acre-feet) for the past 5 years
- If the USBR API is difficult, fall back to hardcoded sample data with realistic
  seasonal patterns for demonstration purposes

STEP 2 - Map page (home/index controller):
- Use the Tethys MapView gizmo to show Utah with markers for each reservoir
- Color-code markers by current storage level relative to capacity:
  - Green: > 70% full
  - Yellow: 40-70% full
  - Red: < 40% full
- Clicking a marker navigates to a detail page for that reservoir

STEP 3 - Dashboard page (a second controller):
- Show a summary dashboard with all reservoirs in a Tethys TableView gizmo
- Columns: reservoir name, current storage, capacity, percent full, trend
  (up/down arrow based on last 30 days)

STEP 4 - Detail page (a third controller at /apps/reservoir-monitor/detail/{id}/):
- Show a time-series chart of storage over the period of record using Plotly JS
- Overlay the historical average as a dashed line
- Show a "rule curve" or capacity line as a horizontal reference
- Display a seasonal comparison: current year vs previous years as overlaid traces

STEP 5 - Navigation:
- App navigation sidebar with links to Map, Dashboard, and an About page
- About page should briefly describe the app and data sources

Use @controller decorators. Templates extend the app's base.html. Use
App.render() for rendering. Handle missing data gracefully.
```

**After the prompt completes**, restart the Tethys server:
```bash
tethys start
```
Or tell Claude: *"Restart the Tethys server."*
Then visit `http://localhost:8000/apps/` to launch the app.

### Standalone Version (Dash + Proxy App)

```
Build an interactive web application called "Reservoir Storage Monitor" using
Plotly Dash. The app tracks reservoir storage levels for major reservoirs in Utah.

STEP 1 - Reservoir data:
- Use the Bureau of Reclamation's Rise API
  (https://data.usbr.gov/rise/api/) to fetch reservoir data
- Target these Utah reservoirs: Flaming Gorge, Strawberry, Deer Creek, Jordanelle,
  and Echo
- For each reservoir, fetch daily storage (acre-feet) for the past 5 years
- If the USBR API is difficult, fall back to hardcoded sample data with realistic
  seasonal patterns for demonstration purposes

STEP 2 - Layout:
- Use a multi-page Dash app with dash-bootstrap-components for clean styling
- Top of page: a Plotly Mapbox or Scattermap showing Utah with markers for each
  reservoir
- Color-code markers by current percent full (green > 70%, yellow 40-70%,
  red < 40%)
- Clicking a marker updates the charts below (use Dash callbacks)

STEP 3 - Dashboard panel:
- Below the map, show a summary table of all reservoirs using a Dash DataTable
- Columns: reservoir name, current storage, capacity, percent full, 30-day trend

STEP 4 - Detail charts (update when a reservoir is selected):
- Time-series chart of storage over the period of record
- Overlay the historical average as a dashed line
- Show capacity as a horizontal reference line
- A second chart: seasonal comparison showing current year vs previous years as
  overlaid traces (x-axis = day of year, y-axis = storage)

STEP 5 - Styling:
- Professional dark or light theme using dash-bootstrap-components
- Responsive layout that works on different screen sizes
- Loading spinners while data is being fetched

Use only: dash, dash-bootstrap-components, plotly, pandas, numpy, and requests.
Run the app on port 8050.
```

**After building, register as a proxy app:**
```bash
conda activate tethys
tethys proxyapp add "Reservoir Storage Monitor" "http://localhost:8050" \
  "Track Utah reservoir storage levels with live USBR data" \
  "" "reservoirs,water-supply,Utah,USBR"
```

**Or tell Claude:**
> "Register this app as a proxy app in my Tethys Portal. It's called 'Reservoir
> Storage Monitor', it's running at http://localhost:8050, and it tracks Utah
> reservoir storage levels with live USBR data. Tag it with reservoirs,
> water-supply, Utah, USBR."

---

## Prompt Pair 3: Flood Frequency Analyzer

### Traditional Tethys Version

```
First, scaffold and install a new Tethys app:
1. Run: tethys scaffold flood_frequency /Users/njones/python_projects/tethys -d
2. Run: cd /Users/njones/python_projects/tethys/tethysapp-flood_frequency && tethys install -d

Then build an app called "Flood Frequency Analyzer" that performs Log-Pearson Type III
flood frequency analysis on USGS annual peak streamflow data for any gage in the
US.

STEP 1 - Station search page (home/index controller):
- Provide a text input where the user can enter a USGS station ID (e.g., 10126000)
- Also provide a state dropdown — when a state is selected, fetch the list of
  gages in that state from the USGS site service
  (https://waterservices.usgs.gov/nwis/site/) with siteType=ST and
  siteStatus=active and hasDataTypeCd=pk
- Display results in a table so the user can click to select a gage

STEP 2 - Fetch peak flow data:
- Use the USGS peak streamflow service
  (https://nwis.waterdata.usgs.gov/nwis/peak?site_no={station_id}&agency_cd=USGS&format=rdb)
- Parse the RDB (tab-delimited) format to extract water year and peak discharge
- Filter out any years with missing or qualified data

STEP 3 - Log-Pearson Type III analysis:
- Compute log-transformed peak flows
- Calculate mean, standard deviation, and skew coefficient of the log values
- Use Bulletin 17C method: apply the frequency factors for the LP3 distribution
  at return periods: 2, 5, 10, 25, 50, 100, 200, and 500 years
- Compute the corresponding discharge for each return period
- Also compute 90% confidence intervals for each estimate

STEP 4 - Results page (second controller):
- Plot 1: Flood frequency curve
  - X-axis: return period (log scale) or exceedance probability
  - Y-axis: peak discharge (log scale)
  - Plot the observed data as points (use Weibull plotting positions)
  - Plot the fitted LP3 curve as a smooth line
  - Show confidence interval bands as shaded regions
  - Title with station name and ID

- Plot 2: Time series of annual peaks
  - Bar chart of annual peak flows by water year
  - Horizontal lines for the 10-year, 50-year, and 100-year flood levels

- Table: flood quantile estimates with return period, exceedance probability,
  discharge, and confidence bounds

Use Plotly JS for the charts. Use @controller decorators. Templates extend the
app's base.html. Use App.render() for rendering. Use numpy/scipy for the
statistical calculations (scipy.stats for the LP3 distribution).
```

**After the prompt completes**, restart the Tethys server:
```bash
tethys start
```
Or tell Claude: *"Restart the Tethys server."*
Then visit `http://localhost:8000/apps/` to launch the app.

### Standalone Version (Streamlit + Proxy App)

```
Build an interactive web application called "Flood Frequency Analyzer" using
Streamlit. The app performs Log-Pearson Type III flood frequency analysis on USGS
annual peak streamflow data for any gage in the US.

STEP 1 - Station search (sidebar):
- Provide a text input in the sidebar where the user can enter a USGS station ID
  (e.g., 10126000)
- Also provide a state dropdown — when a state is selected, fetch the list of
  gages in that state from the USGS site service
  (https://waterservices.usgs.gov/nwis/site/) with siteType=ST and
  siteStatus=active and hasDataTypeCd=pk
- Show results in a selectbox so the user can pick a gage by name

STEP 2 - Fetch peak flow data:
- Use the USGS peak streamflow service
  (https://nwis.waterdata.usgs.gov/nwis/peak?site_no={station_id}&agency_cd=USGS&format=rdb)
- Parse the RDB (tab-delimited) format to extract water year and peak discharge
- Filter out any years with missing or qualified data
- Show a preview of the raw data in an expandable section

STEP 3 - Log-Pearson Type III analysis:
- Compute log-transformed peak flows
- Calculate mean, standard deviation, and skew coefficient of the log values
- Use Bulletin 17C method: apply the frequency factors for the LP3 distribution
  at return periods: 2, 5, 10, 25, 50, 100, 200, and 500 years
- Compute the corresponding discharge for each return period
- Also compute 90% confidence intervals for each estimate

STEP 4 - Visualizations:
- Plot 1: Flood frequency curve (Matplotlib)
  - X-axis: return period (log scale) or exceedance probability
  - Y-axis: peak discharge (log scale)
  - Plot the observed data as points (use Weibull plotting positions)
  - Plot the fitted LP3 curve as a smooth line
  - Show confidence interval bands as shaded regions
  - Title with station name and ID

- Plot 2: Time series of annual peaks (Matplotlib)
  - Bar chart of annual peak flows by water year
  - Horizontal lines for the 10-year, 50-year, and 100-year flood levels

- Table: flood quantile estimates with return period, exceedance probability,
  discharge, and confidence bounds displayed with st.dataframe

STEP 5 - Additional features:
- Allow the user to download the results table as CSV
- Show the station location on a small Folium map at the top of the page
- Display the number of years of record and any data quality notes

Use only: streamlit, streamlit-folium, folium, numpy, scipy, pandas, matplotlib,
and requests.
```

**After building, register as a proxy app:**
```bash
conda activate tethys
tethys proxyapp add "Flood Frequency Analyzer" "http://localhost:8501" \
  "Log-Pearson Type III flood frequency analysis with live USGS data" \
  "" "flood-frequency,hydrology,USGS,LP3"
```

**Or tell Claude:**
> "Register this app as a proxy app in my Tethys Portal. It's called 'Flood
> Frequency Analyzer', it's running at http://localhost:8501, and it does
> Log-Pearson Type III flood frequency analysis with live USGS data. Tag it with
> flood-frequency, hydrology, USGS, LP3."

---

## Prompt Pair 4: Water Quality Dashboard

### Traditional Tethys Version

```
First, scaffold and install a new Tethys app:
1. Run: tethys scaffold water_quality /Users/njones/python_projects/tethys -d
2. Run: cd /Users/njones/python_projects/tethys/tethysapp-water_quality && tethys install -d

Then build an app called "Water Quality Dashboard" that lets users explore water
quality monitoring data from the EPA's Water Quality Portal (WQP).

STEP 1 - Search page (home/index controller):
- Show a Tethys MapView gizmo with a drawing tool that lets the user draw a
  bounding box on the map
- Also provide inputs for: state, county (optional), parameter group dropdown
  (nutrients, metals, physical, bacteria), and date range
- A "Search" button that queries the Water Quality Portal

STEP 2 - Query the Water Quality Portal:
- Use the WQP REST API (https://www.waterqualitydata.us/data/Station/search) to
  find monitoring stations matching the user's criteria
- Then fetch results from
  (https://www.waterqualitydata.us/data/Result/search) for those stations
- Use parameters like statecode, countycode, characteristicName, startDateLo,
  startDateHi, mimeType=csv
- Parse the CSV response into a usable format

STEP 3 - Results map (second controller):
- Show all matching stations on a MapView gizmo
- Color-code by the most recent measurement value relative to EPA thresholds
  (e.g., green = good, yellow = marginal, red = exceeds limit)
- Clicking a station shows a popup with station name and latest readings

STEP 4 - Station detail page (third controller at
/apps/water-quality/station/{station_id}/):
- Time series chart of the selected parameter at that station using Plotly JS
- Show EPA threshold as a horizontal reference line
- Box plots by season (Winter/Spring/Summer/Fall)
- Summary statistics table

STEP 5 - Navigation sidebar:
- Links to Search, Results, and an About page
- The About page describes the data source and EPA threshold references

Use @controller decorators. Templates extend the app's base.html. Use
App.render() for rendering. Handle slow API responses with loading indicators.
```

**After the prompt completes**, restart the Tethys server:
```bash
tethys start
```
Or tell Claude: *"Restart the Tethys server."*
Then visit `http://localhost:8000/apps/` to launch the app.

### Standalone Version (Streamlit + Proxy App)

```
Build an interactive web application called "Water Quality Dashboard" using
Streamlit. The app lets users explore water quality monitoring data from the EPA's
Water Quality Portal (WQP).

STEP 1 - Search controls (sidebar):
- State dropdown (all US states)
- County text input (optional)
- Parameter group dropdown: Nutrients (Nitrogen, Phosphorus), Metals (Lead,
  Arsenic, Mercury), Physical (Temperature, pH, Dissolved Oxygen), Bacteria
  (E. coli, Fecal Coliform)
- Date range picker (default: last 3 years)
- "Search" button

STEP 2 - Query the Water Quality Portal:
- Use the WQP REST API (https://www.waterqualitydata.us/data/Station/search) to
  find monitoring stations matching the user's criteria
- Then fetch results from
  (https://www.waterqualitydata.us/data/Result/search) for those stations
- Use parameters like statecode, countycode, characteristicName, startDateLo,
  startDateHi, mimeType=csv
- Parse the CSV response with pandas
- Cache results using @st.cache_data to avoid repeated API calls

STEP 3 - Map display:
- Show all matching stations on a Folium map
- Color-code markers by the most recent measurement value relative to EPA
  thresholds (green = good, yellow = marginal, red = exceeds limit)
- Popups with station name and latest readings
- Display station count and summary metrics above the map

STEP 4 - Station analysis (when user selects a station):
- Provide a selectbox of stations found in the search
- Time series chart of measurements at that station (Matplotlib or Plotly)
- EPA threshold shown as a horizontal reference line
- Box plots by season (Winter/Spring/Summer/Fall)
- Summary statistics displayed in a clean table

STEP 5 - Export:
- Allow the user to download the filtered dataset as CSV
- Show data quality notes (detection limits, qualifier codes)

Use only: streamlit, streamlit-folium, folium, pandas, numpy, matplotlib, plotly,
and requests.
```

**After building, register as a proxy app:**
```bash
conda activate tethys
tethys proxyapp add "Water Quality Dashboard" "http://localhost:8501" \
  "Explore EPA water quality monitoring data interactively" \
  "" "water-quality,EPA,WQP,monitoring"
```

**Or tell Claude:**
> "Register this app as a proxy app in my Tethys Portal. It's called 'Water
> Quality Dashboard', it's running at http://localhost:8501, and it lets users
> explore EPA water quality monitoring data. Tag it with water-quality, EPA, WQP,
> monitoring."

---

## Prompt Pair 5: Watershed Delineation Tool

### Traditional Tethys Version

```
First, scaffold and install a new Tethys app:
1. Run: tethys scaffold watershed_tool /Users/njones/python_projects/tethys -d
2. Run: cd /Users/njones/python_projects/tethys/tethysapp-watershed_tool && tethys install -d

Then build an app called "Watershed Delineation Tool" that lets users click a point on
a map and delineates the upstream watershed using the USGS StreamStats API or the
EPA WATERS GeoViewer services.

STEP 1 - Map page (home/index controller):
- Use the Tethys MapView gizmo to display a map of the US
- The user clicks a point on a stream to define a pour point
- Capture the latitude/longitude of the click

STEP 2 - Delineate the watershed:
- Use the USGS StreamStats API
  (https://streamstats.usgs.gov/streamstatsservices/) to delineate the watershed
  for the clicked point
- Endpoint:
  https://streamstats.usgs.gov/streamstatsservices/watershed.geojson?rcode={state}&xlocation={lon}&ylocation={lat}&includeparameters=true&includeflowtypes=false&includefeatures=true&crs=4326
- The state code (rcode) can be determined from the clicked coordinates using a
  simple lookup or reverse geocode
- Parse the GeoJSON response to get the watershed boundary polygon and basin
  characteristics

STEP 3 - Display results on the map:
- Draw the delineated watershed boundary as a polygon overlay on the map
- Show the pour point as a marker
- Display the stream network within the watershed if available in the response
- Zoom the map to fit the watershed extent

STEP 4 - Basin characteristics panel:
- Display the basin characteristics returned by StreamStats in a table:
  drainage area, mean basin elevation, mean annual precipitation, basin slope,
  percent forest cover, etc.
- Show the watershed area prominently

STEP 5 - Flow statistics (if available):
- If StreamStats returns flow statistics, display them in a second table
- Show peak flow estimates for various return periods if available

Use @controller decorators with AJAX endpoints for the delineation request (since
it can take 30+ seconds). Show a loading spinner while waiting. Templates extend
the app's base.html.
```

**After the prompt completes**, restart the Tethys server:
```bash
tethys start
```
Or tell Claude: *"Restart the Tethys server."*
Then visit `http://localhost:8000/apps/` to launch the app.

### Standalone Version (Streamlit + Proxy App)

```
Build an interactive web application called "Watershed Delineation Tool" using
Streamlit. The app lets users specify a point on a stream and delineates the
upstream watershed using the USGS StreamStats API.

STEP 1 - Map interface:
- Display a Folium map centered on the US
- Let the user input latitude and longitude (text inputs or by clicking the map
  with streamlit-folium's st_folium click capture)
- Also provide a dropdown to select the state (needed for the StreamStats API)

STEP 2 - Delineate the watershed:
- When the user clicks "Delineate", call the USGS StreamStats API:
  https://streamstats.usgs.gov/streamstatsservices/watershed.geojson?rcode={state}&xlocation={lon}&ylocation={lat}&includeparameters=true&includeflowtypes=false&includefeatures=true&crs=4326
- Show a spinner while waiting (this can take 30+ seconds)
- Parse the GeoJSON response to get the watershed boundary and basin
  characteristics

STEP 3 - Display results:
- Draw the delineated watershed boundary on the Folium map as a GeoJSON overlay
- Show the pour point as a marker
- Zoom the map to fit the watershed
- Use streamlit-folium to render the updated map

STEP 4 - Basin characteristics:
- Display the basin characteristics in a clean table using st.dataframe or
  st.table: drainage area, mean elevation, mean precipitation, slope, etc.
- Highlight the drainage area prominently with st.metric

STEP 5 - Export and sharing:
- Allow the user to download the watershed boundary as a GeoJSON file
- Allow the user to download the basin characteristics as CSV
- Show the raw API response in an expandable section for transparency

Handle errors gracefully — StreamStats doesn't cover all locations and can be
slow. Show friendly messages for unsupported areas or timeouts.

Use only: streamlit, streamlit-folium, folium, pandas, numpy, requests, and json.
```

**After building, register as a proxy app:**
```bash
conda activate tethys
tethys proxyapp add "Watershed Delineation Tool" "http://localhost:8501" \
  "Click a point to delineate the upstream watershed via USGS StreamStats" \
  "" "watershed,delineation,StreamStats,hydrology"
```

**Or tell Claude:**
> "Register this app as a proxy app in my Tethys Portal. It's called 'Watershed
> Delineation Tool', it's running at http://localhost:8501, and it delineates
> upstream watersheds via USGS StreamStats. Tag it with watershed, delineation,
> StreamStats, hydrology."

---

# Key References

- Tethys Docs: http://docs.tethysplatform.org/
- App Development Tutorial: http://docs.tethysplatform.org/en/latest/tutorials/key_concepts.html
- Gizmos (built-in UI components): http://docs.tethysplatform.org/en/stable/tethys_sdk/gizmos.html
- Proxy Apps CLI: `tethys proxyapp --help`
- AquiferX (example standalone app): https://github.com/njones61/aquiferx
- USGS Water Services: https://waterservices.usgs.gov/
- Water Quality Portal: https://www.waterqualitydata.us/
- USGS StreamStats: https://streamstats.usgs.gov/
