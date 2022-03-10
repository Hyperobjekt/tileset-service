# Hyperobjekt Tileset Service

## What this does?

This service will allow the creation of custom tilesets by uploading a CSV file.

## Why is this needed?

Generating custom tilesets is often tedious, time consuming, and requires knowledge of niche tools. This service will abstract away the complicated tasks and allow choropleth + bubble tilesets to be generated quickly and easily.

## How does it work?

When we have a dataset that requires a tileset, we shape a CSV file to have a GEOID column and any additional columns for data points we want to include in the tileset.  Once the CSV file is prepared, we deploy it to an S3 endpoint that has a lambda function that watches for new files.  When a new file is uploaded, the data is merged into an existing base tileset and output to a directory that is served by CloudFront.

**Example:**

- File containing census tracts data is uploaded to `s3://hyper-tiles/build/tracts/2010/eviction-lab-00.csv`
- The lambda function extracts `tracts/2010` from the key and uses that as the source geometry (located in `s3://hyper-tiles/source/tracts/2010/base.mbtiles`)
- The lambda function gets the source `.mbtiles` file from `s3://hyper-tiles/source/tracts/2010/base.mbtiles`
- The lambda function uses `tile-join` to join the source `.mbtiles` file with the uploaded `.csv` file and outputs it to a directory.
- The lambda function copies the tileset output directory to the tileset endpoint `s3://hyper-tiles/public/eviction-lab-00/tracts/`
- A Cloudfront endpoint is setup to serve any tilesets in `s3://hyper-tiles/public/` from `https://tiles.hypersite.io/`
  - in this case, the resulting tileset URL to use with mapboxgl would be `https://tiles.hypersite.io/eviction-lab-00/tracts/{z}/{x}/{y}.pbf`

## Work required to implement

- create an S3 bucket for storing source tilesets, build data, and public tilesets
- create base tilesets (with GEOID, name, and parent location) for the following geographies:
  - states: `s3://hyper-tiles/source/states/2010/base.mbtiles`
  - counties: `s3://hyper-tiles/source/counties/2010/base.mbtiles`
  - zip codes: `s3://hyper-tiles/source/zips/2010/base.mbtiles`
  - school districts: `s3://hyper-tiles/source/districts/2010/base.mbtiles`
  - census tracts: `s3://hyper-tiles/source/tracts/2010/base.mbtiles`
  - block groups: `s3://hyper-tiles/source/block-groups/2010/base.mbtiles`
- create a cloudfront distribution for serving tilesets
- create a lambda function that:
  - uses a docker container that has tippecanoe (and tile-join) available to it
  - watches any subdirectories of `s3://hyper-tiles/build/` for new CSV files
  - when a new file is uploaded:
    - extract the source geometry from the key (based on the folder)
    - get the `base.mbtiles` file from the source directory (fail if no matching source)
    - join  the `base.mbtiles` file with the uploaded CSV file and output to a directory (via `tile-join`)
    - copy the output directory to the tileset endpoint (via `aws s3 cp`)
    - invalidate the cloudfront distribution (via `aws cloudfront create-invalidation`)
