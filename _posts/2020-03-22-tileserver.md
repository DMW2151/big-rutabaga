---
layout: default
title:  Today We're Writing a Tileserver
date:   2020-03-24 22:47:53 -0500
permalink: /posts/tileserver-1
---


## Intro

![Watch the video](https://i.gyazo.com/21ba6028e901123ac3b5257b2a78a691.gif)

A few weeks ago I attended an OpenData NYC event that asked participants to hack together an app that could be used to map community trash cleanup events. This is a project I hope to continue working on, and something that I'm actually quite motivated to continue with. At it's core, this is a simple application, serve a map, allow users to interact with the map. Allow them to create points for litter reports, group meeting points, an select areas or linestrings for coverage areas. In addition, we had some other goals such as making it mobile friendly (i.e. minimize data transfer) and perhaps making it possible to share events on social media.

## Serving Tiles! - Weekend 1

This is all great, but my interest (and what has consumed a good chunk of my last few evenings) is in how to serve slippy map tiles in a way that makes a really enjoyable map experience. During the event, we elected to display two layers (three if `*.png` basemaps count).

The first layer would serve all NYC sidewalks (downloaded from: [NYC Open Data Portal](https://data.cityofnewyork.us/City-Government/NYC-Street-Centerline-CSCL-/exjm-f27b),see also [metadata](https://github.com/CityOfNewYork/nyc-geo-metadata/blob/master/Metadata/Metadata_StreetCenterline.md)) and we'd take a traditional approach for this layer. Typically, when one creates a tileserver they either serve tiles from a directory system or from an `*.mbtiles` file that contains all pre-calculated tiles. In either case, any tile can be accessed by looking up a unique `x`, `y`, `z` value where `z`, represents a zoom level, and `x`, and `y` represent a location on Earth. The advantage of having these tiles pre-generated is that when user's make a request, the server simply need to access the tile stored at the corresponding `XYZ` value rather than doing any calculation on the fly.

I wrote a short script that used [Tippecanoe](https://github.com/mapbox/tippecanoe) to convert files from `*.geojson` to `*.mbtiles`. This is great, but there is the upfront cost of storing the same feature at different zoom levels, finding the right Tippecanoe configuration for each layer or datasource, and then running the tile generation before being able to serve tiles.

For reference, the configuration we ended up using is available [here](https://github.com/Litter-Coalition/Litter-Coalition/blob/fix/tippecanoe/geo_dataproc/dataproc.sh#L29).

With that complete, we turned to the matter of how to serve these tiles. [Tileserver-GL](https://github.com/maptiler/tileserver-gl) is a tool that can be used to create a data catalog from a directory of multiple `*.mbtiles`, and worked quite well for our use case, but I still found that working with the containerized version of the server was quite frustrating.

For user generated events, we'd serve these directly from a database. This is a bit of an unusual choice. Typically there is no reason to serve tiles from the DB, in fact, [it is discouraged](https://news.ycombinator.com/item?id=21938455). The linked comment (quoted below) explains the issue is excellent detail.

>If you are attempting to serve this data on demand each time you need a tile you have to simplify and clip a potentially massive polygon....
>
>Therefore, if your data does not very change quickly I would almost always suggest using preprocessing over dynamic rendering of tiles. You might spend more effort >maintaining something than you expect if you start using PostGIS to create tiles on demand over existing tiling tools.
>

However, in our case the data did change quite often, we didn't expect that much traffic, and the rendered polygons would be a few points, not complex catchment areas, so Taking inspiration from this [toy tileserver](https://blog.crunchydata.com/blog/dynamic-vector-tiles-from-postgis), we ran a flask api that simply queried `PostGIS` to generate tiles on the fly.

## Serving Tiles! - Beyond

Although explicitly discouraged, I was curious how much I could push a tiny server when it comes to serving tiles. I set out to make the following improvements on the tileserver implementation linked above. For an added challenge, I wanted to see if I could make something passable run on a DB hosted on a `t2.micro` instance.

1. Using Nginx:
   - upgrade the server to HTTP 2
   - gzip tiles in transit
   - cache recently generated tiles to disk.
  
2. Port the server to Go to easily make use of:
   - The underlying `net/http` package's concurrency model
   - The underlying connection pooling in `libpq`

3. Come up with some clever PostGIS or geometry tricks to optimize tiles before sending them over the wire

The code for this project is [here](), but I'll briefly go into some of the tricks used to accomplish the third goal listed above. It seems that at least in this setup, the database is the limiting factor (there's also not much room to change the knobs given the size of the instance, but perhaps on a more robust machine the DB config could be modified to improve this performance without tricks).

I loaded in data from a sample file (all blocks in New York State) into the DB with the following script. Next, I stood up a simple frontend using [OpenlayersJS](https://www.google.com/search?client=firefox-b-1-d&q=openlayers) and sent some requests to my webserver. Querying `blocks` was feasible, but the performance was a bit clunky.

```bash
# Download 190MB -> Unzip 600MB
wget https://www2.census.gov/geo/tiger/TIGER2020/TABBLOCK20/tl_2020_36_tabblock20.zip &&\
    unzip tl_2020_36_tabblock20.zip -d /data/layers/nyblocks &&\
    rm tl_2020_36_tabblock20.zip

# Use OGR2OGR to write into the DB
ogr2ogr -overwrite \
        -f "PostgreSQL" PG:"dbname=${POSTGRES_DB} user=${POSTGRES_USER} host=localhost" \
        /data/layers/blocks/ \
        -nln blocks \
        -lco OVERWRITE=yes \
        -t_srs "EPSG:4326" \
        -nlt PROMOTE_TO_MULTI
```

There was room to improve the performance of the query I was using. I was routinely seeing request times of ~200ms on the db to generate a tile. This could be improved with an instance with better CPU and working memory, or a simpler query. I elected to create a pseudo-topological model of each geometry to help keep query speed high and tile size low.

Each block/blockgroup in the data contains a polygon, adjacent polygons intersect on their shared edge, but the server returns the common edge twice. If I decompose the shape to edges, drop shared edges, and then only return relevant edges instead of entire geometries, the responses could be up to 50% smaller. Furthermore, this procedure allows for simplifying lines in a way that does not cause gaps. In effect, it is a layman's implementation of the PostGIS Topology module's [recommendation](https://trac.osgeo.org/postgis/wiki/UsersWikiSimplifyPreserveTopology) for simplifying adjacent shapes.

To get this effect, I ran the following on each table in the DB. Note that the below ignores exterior edges or data associated with each edge (this can be easily included w. small modifications)

```SQL
-- Return the intersection of shapes if & only if they touch (i.e overlap, but have no shared interior points)
create table bglines as (
    SELECT
        a.geoid20,
        b.geoid20 as neighbor_geoid,
        st_intersection(
            a.wkb_geometry,
            b.wkb_geometry
        ) as wkb_geometry
    FROM blocks a
    left join blocks b
on ST_Touches(a.wkb_geometry, b.wkb_geometry)::bool);
```

This is an expensive query and it got a bit rough on the `t2.micro`, running `docker stats` showed the CPU was hurting for a good 10 minutes while this query ran. I'd be interested to see how this compares to a better instance.

```bash
NAME                CPU %
src_db_1            89.56%
```

From there, stuck the following query behind my only API endpoint and modified the query to simplify shapes slightly where possible. The logic in the endpoint converted the `XYZ` coordinates to the bounds of an envelope (args: $1, $2, $3, $4)
and I modified the `st_segmentize` and `st_simplify` params a bit until I was happy with the values.

```SQL
WITH mvtgeom as (
        SELECT
            ST_AsMVTGeom(
                st_segmentize(
                    st_simplify(c2.wkb_geometry, $6), $5
                ),
                st_transform(
                    ST_MakeEnvelope($1, $2, $3, $4, 3857),
                    4326
                ),
                extent => 4096,
                buffer => 64
            ) AS geom
        FROM bglines c2
        WHERE c2.wkb_geometry &&
            st_transform(
                ST_MakeEnvelope($1, $2, $3, $4, 3857),
                4326
            )
    )
    SELECT ST_AsMVT(mvtgeom.*) as pbf FROM mvtgeom;
```

With proper indexing, caching, gzipping, etc. I was able to get performance I was quite happy with. Even without the cache activated, with I was back down to seeing very satisfactory <100ms responses from the DB. There are a few outstanding issues.

1. Tile size is unbounded, unless controlled for by the front end, zooming too far out will return tiles >500kb (the recommended tile size max is ~50kb) and the request will take >1s.

2. Something looks a bit off in the projection. The standard for Web-Mercator and general WGS is not the same coordinate system, perhaps I missed a transform somewhere.

3. Returning edges is cheating, I should find a way to re-join shapes together so they represent meaningful objects/areas on the front end.

4. Compression I'm getting isn't great (~40% ??), maybe there's some trick I can employ to help gzip out a bit. I know Tippecanoe's `--reverse` flag does some compression magic when generating tiles.

I'm shocked with how well this turned out, quite motivated to download the whole US by block and see how well this scales.
