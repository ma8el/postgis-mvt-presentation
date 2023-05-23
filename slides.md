---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://images.unsplash.com/photo-1489702932289-406b7782113c?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2072&q=80
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
#info: |
#  ## Slidev Starter Template
#  Presentation slides for developers.
#
#  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
# page transition
transition: slide-left
# use UnoCSS
css: unocss
---

# Mapbox Vector Tiles with PostGIS

Easy Tile Server on your PostgreSQL database

<div class="abs-br m-6 flex gap-2">
  <a href="https://github.com/ma8el/postgis-mvt" target="_blank" alt="GitHub"
    class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon-logo-github />
  </a>
</div>

---

# Overview

1. What are Tiles?
2. What are Vector Tiles?
3. Render Mapbox Vector Tiles in PostGIS
4. App architecture
5. Code
6. Demo

---
layout: two-cols
---

# What are Tiles?

- Map tiling is a way of constraining the digital mapping problem, just a little, to vastly increase the speed and efficiency of map display.
- Instead of supporting any scale, a tiled map only provides a limited collection of scales, where each scale is a factor of two more detailed than the previous one.
- Instead of rendering data for any region of interest, a tiled map only renders it over a fixed grid within the scale, and composes arbitrary regions by displaying appropriate collections of tiles.

Reference [Crunchydata](https://www.crunchydata.com/blog/dynamic-vector-tiles-from-postgis)

::right::
![Local Image](/tilePyramid.webp)

---
layout: two-cols
---

# Tile Coordinates

- Any tile in a tiled map can be addressed by referencing the zoom level it is on, and its position horizontally and vertically in the tile grid.

Can be queried via a webserver with the following scheme:

```
curl http://server/{z}/{x}/{y}.format
```

<br>

Reference [Crunchydata](https://www.crunchydata.com/blog/dynamic-vector-tiles-from-postgis)

::right::
![Local Image](/tileCoordinates.webp)

---

# What are Vector Tiles?

Vector tiles are ...

- a type of map tile that represent geographic data in a vector format rather than a raster (bitmap) format.
- not pre-rendered as an image of the map, vector tiles send the raw data of what's in the map. This data includes points, lines, and polygons, along with any associated metadata.
- typically smaller in size than raster tiles, which can lead to faster map load times and less bandwidth usage.

MVT are ...

- vector tiles that are used with Mapbox services, such as Mapbox GL JS for web mapping.
- open standard and have been adopted by many other platforms beyond Mapbox.
- encoded as Protocol Buffers (a compact binary format developed by Google), which allows them to be transferred quickly and efficiently over the network.

---
layout: image-right
image: https://images.unsplash.com/photo-1553532434-5ab5b6b84993?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=2148&q=80
---

# Render Mapbox Vector Tiles in PostGIS

- PostGIS extends PostgreSQL with advanced geospatial capabilities.
- It supports a wide array of spatial functions, including geometry processing and spatial analytics functions, which can be used to perform complex operations on the data before it's turned into vector tiles.
- This can be beneficial when dealing with large or complex spatial datasets.

---
layout: two-cols
---

# `ST_AsMVTGeom()`

- `ST_AsMVTGeom()` is a PostGIS function used for transforming geometry into a format suitable for encoding into a Mapbox Vector Tile.
- This function accepts a geometry and a bounding box (expressed as a BOX2D) and produces a geometry that's scaled and offset for use in a vector tile.
- `ST_AsMVTGeom()` effectively makes the geometry "tile-ready," meaning it's prepared for conversion into a vector tile format.

```sql {all}
SELECT 
  ST_AsMVTGeom(geom) AS geom, column1, column2
FROM myTable
```
::right::

# `ST_AsMVT()`

- `ST_AsMVT()` is a PostGIS function used for encoding a row of data into Mapbox Vector Tile format.
- The function can encode a number of attributes from the row into the tile, which means it's not just the geometry being encoded, but potentially other associated data as well.
- The output from `ST_AsMVT()` is a binary blob that represents a single vector tile, which can be served to a client for rendering.

```sql {all}
SELECT ST_AsMVT(mvtgeom.*)
FROM (
  SELECT ST_AsMVTGeom(geom) AS geom, column1, column2
  FROM myTable
) mvtgeom
```

---

# App architecture

- Client requests tiles from a webserver
- Client consumes the tiles via a `fastapi` webserver
- Webserver queries the database via `psycopg2`
- The webserver uses the `ST_AsMVT()` function to encode the data into a vector tile

<img src="/architecture.webp" class="m-5 h-50 rounded" />

Reference [Crunchydata](https://www.crunchydata.com/blog/dynamic-vector-tiles-from-postgis)

---

# Data

- For this showcase OSM building data (Hessen) is used
- Example:

| osm_id    | name                   | building    | geom |
|-----------|------------------------|-------------|------|
| 143258352 | Altes Amtsgericht      | yes         | ...  |
| 106938171 | Augustiner Kloster     | yes         | ...  |
| 161483967 | Aumühle (Gärtnerei)    | greenhouse  | ...  |

---
layout: two-cols
---
# Webserver

```python {all}
def get_mvt_query(table_name: str,
                  z: int,
                  x: int,
                  y: int) -> str:
    """
    Construct the SQL query to fetch the MVT data.
    """
    return f"""
        SELECT ST_AsMVT(q, '{table_name}', 4096, 'geom')
        FROM (
            SELECT osm_id, name, building, ST_AsMVTGeom(
                geom,
                ST_TileEnvelope({z}, {x}, {y}),
                4096,
                256,
                true
            ) AS geom
            FROM {table_name}
            WHERE ST_Intersects(
                geom,
                ST_TileEnvelope({z}, {x}, {y})
            )
        ) AS q
    """
```
::right::
# Endpoint

```python {all}
@app.get("/api/v1/mvt/{table_name}/{z}/{x}/{y}.mvt",
         response_class=Response)
async def get_mvt(table_name: str,
                  z: int,
                  x: int,
                  y: int) -> Response:

    conn = psycopg2.connect(
      ...
    )

    query = get_mvt_query(table_name, z, x, y)

    with conn.cursor() as cur:
        cur.execute(query)
        result = cur.fetchone()[0]
        if result is None:
            response = Response()
        else:
            response = Response(bytes(result),
                        media_type="application/x-protobuf")
        response.headers["Access-Control-Allow-Origin"] = "*"
        return response
```

[^1]: [Learn More](https://sli.dev/guide/syntax.html#line-highlighting)

---
layout: two-cols
---

# Frontend

``` javascript {all}
onMounted(() => {
  var vtLayer = new VectorTileLayer({
    declutter: false,
    source: new VectorTileSource({
      format: new MVT(),
      url: 'http://localhost:8000/api/v1
                /mvt/osm_buildings/{z}/{x}/{y}.mvt',
    }),
    renderMode: 'vector',
    style: new Style({
        stroke: new Stroke({
          color: 'blue',
          width: 1
        }),
    fill: new Fill({
      color: 'rgba(176, 196, 222, 0.4)'
    })
    })
  });
```
::right::
# -
``` javascript {all}
  const map = new Map({
    layers: [
      new TileLayer({
        source: new OSM(),
      }),
      vtLayer,
    ],
  })
})
```
---

# In action

<div grid="~ cols-1">
  <Map />
</div>

---
layout: iframe
url: http://localhost:5173/
---



---
layout: center
class: text-center
---

# Learn More

[Documentations](https://postgis.net/documentation/) · [GitHub](https://github.com/ma8el/postgis-mvt) · [Showcases](https://google.com)
