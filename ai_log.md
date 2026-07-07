AI tool: ChatGPT Desktop App
Date: 7/7/2026

Hey, for our R GIS lab, how should we fill out this table in our markdown? 
We have these variables: GDP per capita, Life expectancy, Region, and Population. 
We need to determine:
- Variable type
- Suitable as fill color? Why or why not?
- Would the result be a choropleth map? Why or why not?

Also, what R packages do we need to load for this lab? We are doing shapefile processing, plotting, simplification, etc.

Thought for a few seconds
Here is the completed table classifying the variables for choropleth mapping:

| Variable | Variable type | Suitable as fill color? Why or why not? | Would the result be a choropleth map? Why or why not? |
|---|---|---|---|
| GDP per capita | Quantitative, intensive | Yes. It is normalized per person, so countries can be compared by color. | Yes. A choropleth is appropriate because fill color shows an intensive country-level statistic. |
| Life expectancy | Quantitative, intensive | Yes. It is an average measure, so countries can be compared by color. | Yes. It is a choropleth because color represents an intensive country-level statistic. |
| Region | Categorical, nominal | Yes, but only with a qualitative palette because regions are categories. | No. It would be a categorical thematic map, not a choropleth, because the colors do not show statistical intensity. |
| Population | Quantitative, extensive | Not ideal. Total population depends on country size and can be misleading as polygon fill. | No, not strictly. Population density would be better for a choropleth because it normalizes population by area. |

For a spatial data pipeline in R that involves importing metadata, cleaning boundaries, simplifying polygons, and plotting, you will need the following libraries:
- tidyverse (specifically dplyr, readxl, ggplot2) for data wrangling and plotting.
- sf (Simple Features) as the core library for handling vector spatial data.
- mapview for interactive spatial data viewing and vertex counting (npts()).
- patchwork for combining multiple maps into a multi-panel figure.
- readxl to import World Bank Excel spreadsheets.
- rmapshaper for topology-preserving geometry simplification (ms_simplify()).
- scales for formatting chart breaks and labels.
- shadowtext for adding readable text labels with background outlines on the map.

Hey, I need to clean and merge some World Bank excel files in R. Here are the files:
- GDP per capita: API_NY.GDP.PCAP.PP.KD_DS2_en_excel_v2_417139.xls (sheet 'Data')
- Life expectancy: API_SP.DYN.LE00.IN_DS2_en_excel_v2_404463.xls (sheet 'Data')

I've uploaded a screenshot of how the xls file looks. The headers start from row 4 (row 1-3 is just junk text). The columns are Country Name, Country Code and columns for years like 2024.
There's also a sheet in the GDP file called 'Metadata - Countries' that has Country Code and Region.
How can I read these sheets, rename the 2024 columns to gdp_per_cap and life_exp, and merge them by code?
Also, I need to filter out aggregates (like 'World', 'Arab World') but KEEP countries that have missing values for one of the indicators (e.g. Afghanistan has NA for GDP but has life expectancy). How do I distinguish them?

Thought for a few seconds
To programmatically separate countries from regional or global aggregates, you can leverage the 'Metadata - Countries' sheet. 

In World Bank datasets, aggregate regions (like "Arab World", "World", "Latin America & Caribbean") do not have a geographic Region assigned in the metadata sheet (their Region column is left blank or NA), whereas individual countries always have a valid region name (e.g., "South Asia", "Europe & Central Asia").

Here is the approach:
1. Load the data from the spreadsheets, skipping the first 3 rows of metadata so that R reads row 4 as the header.
2. Load the metadata sheet.
3. Perform a sequential join.
4. Filter out rows where region is NA (this will drop the aggregates but keep actual countries that have missing indicator values).
5. Drop the region column at the end if it's only needed for filtering.

Here is the R code:

```r
gdp_file <- "API_NY.GDP.PCAP.PP.KD_DS2_en_excel_v2_417139.xls"
life_file <- "API_SP.DYN.LE00.IN_DS2_en_excel_v2_404463.xls"

gdp <- read_xls(gdp_file, sheet = "Data", skip = 3) |>
  transmute(
    name = `Country Name`,
    code = `Country Code`,
    gdp_per_cap = as.numeric(`2024`)
  )

life <- read_xls(life_file, sheet = "Data", skip = 3) |>
  transmute(
    name = `Country Name`,
    code = `Country Code`,
    life_exp = as.numeric(`2024`)
  )

regions <- read_xls(gdp_file, sheet = "Metadata - Countries") |>
  transmute(
    code = `Country Code`,
    region = Region
  )

country_stats <- gdp |>
  left_join(life, by = c("name", "code")) |>
  left_join(regions, by = "code") |>
  filter(!is.na(region)) |>
  select(name, code, gdp_per_cap, life_exp)
```

I wrote this code based on that:
```r
country_stats <- gdp |>
  left_join(life, by = c("name", "code")) |>
  left_join(regions, by = "code") |>
  filter(!is.na(region)) |>
  select(name, code, gdp_per_cap, life_exp)
```
I got 217 rows from nrow(country_stats). Does 217 sound correct for World Bank economies? Also, did I do the filter correctly to remove aggregates without dropping countries with NA values?

Thought for a few seconds
Yes, 217 is the correct number of economies. The World Bank currently tracks 217 economies (comprising member states and other temporary territories or administrative regions) alongside various regional aggregates.

Your code is correct:
- filter(!is.na(region)) is executed before select(name, code, gdp_per_cap, life_exp), which is crucial. If you selected the columns first, the region column would be lost, making the filter impossible.
- By doing left_join and then filtering out rows where the metadata region is missing, you have successfully removed regional groups (like "Arab World" and "World") while preserving countries that have partially missing values in either gdp_per_cap or life_exp (such as Aruba or Afghanistan).

I need to read a geojson file `WB_GAD_ADM0_complete.geojson` in R. It's about 174MB.
I only need the columns `ISO_A3` (as `code`) and `WB_STATUS` (as `status`).
How do I import it and keep only these columns using `sf`? Does `select` or `transmute` drop the geometry column?
Also, how do I inspect the details like the geometry type, row count, and if it's using long/lat?
Lastly, rendering Quarto takes forever because of this 174MB load. How can I cache this cell so it doesn't reload on every render?

Thought for a few seconds
Here are the solutions for importing, checking metadata, and optimizing render performance:

Yes, the geometry column in simple feature (sf) objects is "sticky." When you use select() or transmute(), the geometry column is retained automatically unless you explicitly drop it.

You can import and inspect the spatial data like this:

```r
library(sf)

country_boundaries <- st_read("WB_GAD_ADM0_complete.geojson", quiet = TRUE) |>
  transmute(
    code = ISO_A3,
    status = WB_STATUS
  )

# Print details
country_boundaries

# Get features count (should be 284 rows)
nrow(country_boundaries)

# Get geometry types (should be MULTIPOLYGON)
unique(st_geometry_type(country_boundaries))

# Check if coordinates are in longitude-latitude (returns TRUE if longlat/WGS84)
st_is_longlat(country_boundaries)
```

To prevent Quarto from re-running this slow cell on every render, you can add the `#| cache: true` option inside your code chunk:

```r
#| label: import-boundaries
#| cache: true
#| message: false

country_boundaries <- st_read("WB_GAD_ADM0_complete.geojson", quiet = TRUE) |>
  transmute(
    code = ISO_A3,
    status = WB_STATUS
  )
```

During the first render, Quarto will execute the cell and save the resulting objects to a cache directory. Subsequent renders will skip the code and load the cached object, speeding up your render times significantly.

I imported the boundaries. It has 284 rows and is MULTIPOLYGON in longlat.
Before doing any simplification, how do I check if there are any invalid geometries in the data, and how do I fix them if they're invalid?
And does fixing them change the row count or types?

Thought for a few seconds
Here is the workflow using the sf validation and repair functions:

```r
# Count invalid geometries
sum(!st_is_valid(country_boundaries))

# Repair geometries and overwrite the object
country_boundaries <- st_make_valid(country_boundaries)

# Verify no invalid geometries remain
sum(!st_is_valid(country_boundaries))

# Check if row count or geometry types changed
nrow(country_boundaries)
unique(st_geometry_type(country_boundaries))
```

How it works:
- st_is_valid() returns a logical vector. !st_is_valid() converts valid to FALSE and invalid to TRUE. Summing this vector counts the invalid geometries (you should see 5 invalid shapes initially).
- st_make_valid() automatically resolves topology errors (such as self-intersections or duplicate vertices) and yields valid geometries.
- The row count should remain exactly 284 and geometry types should stay MULTIPOLYGON after the repair.

Okay, 5 invalid geometries were found and repaired.
Now I need to simplify the boundaries to around 10,000 total vertices using `ms_simplify` from `rmapshaper`.
- How do I count the current total vertices?
- How do I count how many polygons make up Singapore (code `SGP`) before simplifying?
- How do I calculate the keep ratio for `ms_simplify` to get 10,000 vertices, and make sure smaller islands don't get deleted entirely?
- How do I check the Singapore polygon count again after?

Thought for a few seconds
Here is how to set up the simplification pipeline and the checks:

To count the total vertices in the dataset, use the npts() function from the mapview package:

```r
points_before <- npts(country_boundaries)
points_before
```

To count Singapore's individual polygons, filter for Singapore ("SGP") and cast the MULTIPOLYGON geometry to POLYGON features:

```r
sgp_polygons_before <- country_boundaries |>
  filter(code == "SGP") |>
  st_cast("POLYGON") |>
  nrow()
```

To simplify the boundaries to roughly 10,000 points, compute the proportion of vertices to keep:

```
keep_target = 10000 / points_before
```

To prevent smaller island nations or features from being completely deleted when their geometry simplifies past the default threshold, set keep_shapes = TRUE in ms_simplify().

Here is the complete implementation:

```r
# Calculate target keep ratio
keep_target <- 10000 / points_before

# Simplify polygons
country_boundaries <- country_boundaries |>
  ms_simplify(
    keep = keep_target,
    keep_shapes = TRUE
  )
```

I ran the code and checked Singapore:
- npts went from 318,347 to 10,071
- Singapore polygons went from 20 to 1
Is it normal for Singapore to drop from 20 polygons to just 1? Did I lose data or did the islands just get smoothed out?

Thought for a few seconds
Yes, this is completely normal and expected!

Singapore's geometry originally contains 20 polygons because it includes the main island along with 19 smaller offshore islands/islets (like Sentosa, Pulau Ubin, Pulau Tekong, etc.). 

When you simplify the total dataset down to ~10,000 vertices (keeping only about 3% of the original detail):
- The smaller offshore islands fall below the scale resolution and are smoothed away.
- The main island remains as a single, coarse boundary representation.

Since you specified keep_shapes = TRUE, the row for Singapore itself is not dropped from the dataset (meaning nrow(country_boundaries) remains 284), but its geometry is simplified to a single polygon representing the main island.

I am trying to add outer lines to my choropleth map to represent the world edges and an ocean background color for the globe. How would I do so? Here's my current implementation:

```{r}
#| label: final-map
#| fig-width: 10
#| fig-height: 12
#| out-width: 100%



largest_40 <- countries |>
  filter(!is.na(code)) |>
  mutate(area = st_area(geometry)) |>
  arrange(desc(area)) |>
  slice_head(n = 40) |>
  st_centroid() |>
  st_transform(8857)

largest_40_coords <- largest_40 |>
  mutate(
    x = st_coordinates(geometry)[, 1],
    y = st_coordinates(geometry)[, 2]
  ) |>
  st_drop_geometry()


create_map <- function(var_name, fill_palette, breaks, legend_title, label_func) {
  
  main_data <- countries |>
    filter(!is.na(code), !is.na(.data[[var_name]]))
    
  gray_data <- countries |> 
    filter(is.na(code) | is.na(.data[[var_name]])) |>
    mutate(
      map_category = if_else(is.na(code), "Undetermined", "Missing"),
      fill_color = if_else(is.na(code), "grey50", "grey80")
    )
    
  ggplot() +
    geom_sf(data = main_data, aes(fill = .data[[var_name]]), color = "grey30", linewidth = 0.15) +
    geom_sf(data = gray_data, aes(color = map_category), fill = gray_data$fill_color, linewidth = 0.15) +
    geom_shadowtext(
      data = largest_40_coords,
      aes(x = x, y = y, label = code),
      size = 2.5,
      color = "black",
      bg.color = "white",
      bg.r = 0.15
    ) +
    scale_fill_fermenter(
      palette = fill_palette,
      direction = 1,
      breaks = breaks,
      labels = label_func,
      guide = guide_colorsteps(
        title = legend_title,
        title.position = "top",
        title.hjust = 0.5,
        barwidth = 20,
        barheight = 0.7
      )
    ) +
    scale_color_manual(
      name = NULL,
      values = c("Missing" = "grey30", "Undetermined" = "grey30"),
      guide = "none" 
    ) +
    coord_sf(crs = "EPSG:8857", expand = FALSE) +
    theme_void() +
    theme(
      legend.position = "top",
      legend.title = element_text(face = "bold", size = 11),
      legend.text = element_text(size = 9),
      plot.margin = margin(5, 5, 5, 5)
    )
}

p_life <- create_map(
  "life_exp", 
  "Greens", 
  c(60, 65, 70, 75, 80), 
  "Life Expectancy (Years)",
  label_number()
)

p_gdp <- create_map(
  "gdp_per_cap", 
  "Oranges", 
  c(2000, 5000, 10000, 20000, 50000, 100000), 
  "GDP per Capita (Current USD)",
  label_number(scale_cut = cut_short_scale())
)

dummy_gray <- data.frame(
  map_category = c("Missing", "Undetermined"),
  fill_color = c("grey80", "grey50")
)

p_legend <- ggplot(dummy_gray) +
  geom_rect(aes(xmin = 0, xmax = 0, ymin = 0, ymax = 0, color = map_category), fill = dummy_gray$fill_color) +
  scale_color_manual(
    name = NULL,
    values = c("Missing" = "grey30", "Undetermined" = "grey30"),
    guide = guide_legend(override.aes = list(fill = c("grey80", "grey50")), nrow = 1)
  ) +
  theme_void() +
  theme(
    legend.position = "bottom",
    legend.text = element_text(size = 11),
    plot.margin = margin(t = 10, b = 0)
  )

final_plot <- p_life / p_gdp / p_legend +
  plot_layout(heights = c(1, 1, 0.05)) +
  plot_annotation(
    title = "Global Development in 2024",
    caption = "\"Undetermined\" refers to an area's legal status. \n Source: World bank (indicators, boundaries, and classification of legal status)",
    theme = theme(
      plot.title = element_text(hjust = 0.5, face = "bold", size = 18, margin = margin(b = 15)),
      plot.caption = element_text(hjust = 1, size = 10, color = "grey40")
    )
  )

final_plot
```

To achieve that beautiful curved globe effect with an ocean background, we need to create custom `sf` objects for the background polygon and the left/right border lines. We also need to calculate the precise clipping limits so the top and bottom crop cleanly at 56°S and 84°N.

Here is your implementation updated with the ocean background and edges:

```r
#| label: final-map
#| fig-width: 10
#| fig-height: 12
#| out-width: 100%

# 1. Create the ocean background and side borders
# Globe background (Light blue fill, no border)
lon <- c(seq(-180, 180, by = 1), seq(180, -180, by = -1), -180)
lat <- c(rep(84, 361), rep(-56, 361), 84)
ocean_bg <- st_polygon(list(cbind(lon, lat))) |> st_sfc(crs = 4326)

# Left and right antimeridian edges (Thin black line)
left_edge <- st_linestring(cbind(rep(-180, 141), seq(-56, 84, by = 1))) |> st_sfc(crs = 4326)
right_edge <- st_linestring(cbind(rep(180, 141), seq(-56, 84, by = 1))) |> st_sfc(crs = 4326)
side_edges <- c(left_edge, right_edge)

# 2. Get top 40 largest countries for labels
largest_40 <- countries |>
  filter(!is.na(code)) |>
  mutate(area = st_area(geometry)) |>
  arrange(desc(area)) |>
  slice_head(n = 40) |>
  st_centroid() |>
  st_transform(8857)

largest_40_coords <- largest_40 |>
  mutate(
    x = st_coordinates(geometry)[, 1],
    y = st_coordinates(geometry)[, 2]
  ) |>
  st_drop_geometry()

# 3. Y-limits in projected space (EPSG:8857)
lims <- st_sfc(st_point(c(0, -56)), st_point(c(0, 84)), crs = 4326) |> 
  st_transform(8857) |> 
  st_coordinates()
y_limits <- lims[, "Y"]

create_map <- function(var_name, fill_palette, breaks, legend_title, label_func) {
  
  main_data <- countries |>
    filter(!is.na(code), !is.na(.data[[var_name]]))
    
  gray_data <- countries |> 
    filter(is.na(code) | is.na(.data[[var_name]])) |>
    mutate(
      map_category = if_else(is.na(code), "Undetermined", "Missing"),
      fill_color = if_else(is.na(code), "grey50", "grey80")
    )
    
  ggplot() +
    # Add our new ocean background and side edges BEFORE drawing the countries
    geom_sf(data = ocean_bg, fill = "#e0f3f8", color = NA) +
    geom_sf(data = side_edges, color = "black", linewidth = 0.3) +
    geom_sf(data = main_data, aes(fill = .data[[var_name]]), color = "grey30", linewidth = 0.15) +
    geom_sf(data = gray_data, aes(color = map_category), fill = gray_data$fill_color, linewidth = 0.15) +
    geom_shadowtext(
      data = largest_40_coords,
      aes(x = x, y = y, label = code),
      size = 2.5,
      color = "black",
      bg.color = "white",
      bg.r = 0.15
    ) +
    scale_fill_fermenter(
      palette = fill_palette,
      direction = 1,
      breaks = breaks,
      labels = label_func,
      guide = guide_colorsteps(
        title = legend_title,
        title.position = "top",
        title.hjust = 0.5,
        barwidth = 20,
        barheight = 0.7
      )
    ) +
    scale_color_manual(
      name = NULL,
      values = c("Missing" = "grey30", "Undetermined" = "grey30"),
      guide = "none" 
    ) +
    # Apply our calculated y-limits here
    coord_sf(crs = "EPSG:8857", ylim = y_limits, expand = FALSE) +
    theme_void() +
    theme(
      legend.position = "top",
      legend.title = element_text(face = "bold", size = 11),
      legend.text = element_text(size = 9),
      plot.margin = margin(5, 5, 5, 5)
    )
}

p_life <- create_map(
  "life_exp", 
  "Greens", 
  c(60, 65, 70, 75, 80), 
  "Life Expectancy (Years)",
  label_number()
)

p_gdp <- create_map(
  "gdp_per_cap", 
  "Oranges", 
  c(2000, 5000, 10000, 20000, 50000, 100000), 
  "GDP per Capita (Current USD)",
  label_number(scale_cut = cut_short_scale())
)

dummy_gray <- data.frame(
  map_category = c("Missing", "Undetermined"),
  fill_color = c("grey80", "grey50")
)

p_legend <- ggplot(dummy_gray) +
  geom_rect(aes(xmin = 0, xmax = 0, ymin = 0, ymax = 0, color = map_category), fill = dummy_gray$fill_color) +
  scale_color_manual(
    name = NULL,
    values = c("Missing" = "grey30", "Undetermined" = "grey30"),
    guide = guide_legend(override.aes = list(fill = c("grey80", "grey50")), nrow = 1)
  ) +
  theme_void() +
  theme(
    legend.position = "bottom",
    legend.text = element_text(size = 11),
    plot.margin = margin(t = 10, b = 0)
  )

final_plot <- p_life / p_gdp / p_legend +
  plot_layout(heights = c(1, 1, 0.05)) +
  plot_annotation(
    title = "Global Development in 2024",
    caption = "\"Undetermined\" refers to an area's legal status. \n Source: World bank (indicators, boundaries, and classification of legal status)",
    theme = theme(
      plot.title = element_text(hjust = 0.5, face = "bold", size = 18, margin = margin(b = 15)),
      plot.caption = element_text(hjust = 1, size = 10, color = "grey40")
    )
  )

final_plot
```

### Code Walkthrough

Here is a breakdown of how the background and edges were created and applied:

#### 1. Creating the Ocean Polygon (`ocean_bg`)
```r
lon <- c(seq(-180, 180, by = 1), seq(180, -180, by = -1), -180)
lat <- c(rep(84, 361), rep(-56, 361), 84)
ocean_bg <- st_polygon(list(cbind(lon, lat))) |> st_sfc(crs = 4326)
```
Instead of relying on `ggplot2`'s panel background—which would fill the entire rectangular plot area—we create a precise spherical polygon representing the entire world bounded by our crop coordinates. 
*   **`lon` & `lat`:** We generate hundreds of individual vertices by taking steps of `1` degree across the map. We trace from longitude -180 to 180 at latitude 84°N, and then trace backwards from 180 to -180 at latitude 56°S, closing the loop.
*   **Why so many points?** Because when this simple rectangle is later transformed into the Equal Earth projection (`EPSG:8857`), having vertices every 1 degree allows the left and right edges to mathematically bend and trace the curved antimeridian accurately. If we only used the 4 corners, `sf` would draw straight lines between them!

#### 2. Creating the Side Borders (`side_edges`)
```r
left_edge <- st_linestring(cbind(rep(-180, 141), seq(-56, 84, by = 1))) |> st_sfc(crs = 4326)
right_edge <- st_linestring(cbind(rep(180, 141), seq(-56, 84, by = 1))) |> st_sfc(crs = 4326)
```
The prompt specifically asked to "leave the top and bottom edges open". If we added a black border to our `ocean_bg` polygon, it would draw a box. Instead, we generate two independent `st_linestring` objects corresponding strictly to the left (-180° longitude) and right (180° longitude) edges.

#### 3. Calculating the Clipping Limits (`y_limits`)
```r
lims <- st_sfc(st_point(c(0, -56)), st_point(c(0, 84)), crs = 4326) |> 
  st_transform(8857) |> 
  st_coordinates()
y_limits <- lims[, "Y"]
```
You can't pass latitude limits like `ylim = c(-56, 84)` directly into `coord_sf` when you've specified a projected CRS like `EPSG:8857` (because Equal Earth expects coordinates in meters, not degrees). 
*   Here, we create two dummy points at the south and north clipping latitudes (0° lon, 56°S and 0° lon, 84°N). 
*   We use `st_transform(8857)` to convert them into meters to see exactly where those latitudes end up on the unrolled map.
*   We extract those projected "Y" (vertical) coordinates to pass into `coord_sf(ylim = y_limits)`.

#### 4. Layering in `ggplot2`
```r
    geom_sf(data = ocean_bg, fill = "#e0f3f8", color = NA) +
    geom_sf(data = side_edges, color = "black", linewidth = 0.3) +
```
Finally, within the `create_map` function, we ensure these components are drawn **first**, underneath the main and gray country data. We set `color = NA` on the ocean to keep its top and bottom borders invisible, and then stack the black `side_edges` over it.


the ocean background doesnt cover the whole globe and gets cut off on the side, how would i troubleshoot and patch this?

Edited grpAssgmt10.qmd

Ah, I see exactly what happened! Good catch.

### The Cause: Projecting Straight Lines
When `st_polygon` connects points, it draws straight lines between them. My initial code only provided dense coordinate points for the **top** (84°N) and **bottom** (-56°S) edges of the polygon. 

However, it connected the top-right corner `(180, 84)` directly to the bottom-right corner `(180, -56)` with no intermediate points in between. When `ggplot2` projected this polygon into the Equal Earth CRS (`EPSG:8857`), it projected those two corners and then drew a straight line between them. But Equal Earth's meridians (like the edge at 180°) are heavily curved! Because our polygon edge lacked intermediate vertices, it stayed stubbornly straight, cutting inward and chopping off the natural "bulge" of the globe on the left and right sides.

### The Fix
To ensure the polygon bends properly along the antimeridian when projected, we must explicitly provide points along the left and right edges so that `ggplot2` has enough data to trace the curve.

Here is the updated logic I just patched into your file:

```r
# 1. Top edge: Sweep from -180 to 180 longitude at latitude 84
lon_top <- seq(-180, 180, by = 1)
lat_top <- rep(84, length(lon_top))

# 2. Right edge: Sweep from 83 down to -56 latitude at longitude 180
lon_right <- rep(180, length(seq(83, -56, by = -1)))
lat_right <- seq(83, -56, by = -1)

# 3. Bottom edge: Sweep backwards from 180 to -180 longitude at latitude -56
lon_bottom <- seq(180, -180, by = -1)
lat_bottom <- rep(-56, length(lon_bottom))

# 4. Left edge: Sweep from -55 back up to 84 latitude at longitude -180
lon_left <- rep(-180, length(seq(-55, 84, by = 1)))
lat_left <- seq(-55, 84, by = 1)

# Combine all 4 edges into a single closed loop
lon <- c(lon_top, lon_right, lon_bottom, lon_left)
lat <- c(lat_top, lat_right, lat_bottom, lat_left)

# Create the full background polygon
ocean_bg <- st_polygon(list(cbind(lon, lat))) |> st_sfc(crs = 4326)
```

Now, `ocean_bg` is composed of 1002 points tracing a complete, dense rectangle around the clipped world. When passed into the `EPSG:8857` projection, the left and right edges will perfectly contour to the shape of the globe, filling the whole map up to the thin black border lines!

Try running the chunk again, the ocean background should now perfectly envelop the continents!