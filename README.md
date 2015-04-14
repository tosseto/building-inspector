## NYPL Labs Map Polygon fixer

Authors: [Mauricio Giraldo Arteaga] / NYPL Labs

- [Data ingest](#ingest)
- [Consensus generation](#consensus)
- [API querying](#api)

### Environment variables and buildpacks

#### Heroku buildpacks

This project makes use of several environment variables and buildpacks to work. The RGeo gem in Heroku does not get built properly so some tasks do not work. These are the environment variables you need to add in order for it to work:

- `BUILDPACK_URL`: https://github.com/heroku/heroku-buildpack-multi.git

This is related to the [`.buildpacks` file](.buildpacks) which contains the list of additional gem packs to install when deploying the application. This should happen automatically. In order to verify that the right gems are installed, enter the Rails console in the application and type:

`RGeo::Geos.supported?`

This should return `=> true`. If this is not the case you may want to visit the [issues list for `heroku-geo-buildpack`](https://github.com/cyberdelia/heroku-geo-buildpack/issues).

#### Social media login integration

This application makes use of [OmniAuth](https://github.com/intridea/omniauth) which requires that you set up the necessary API keys for [Google](https://developers.google.com/+/api/oauth#apikey), [Twitter](https://dev.twitter.com/oauth) and [Facebook](https://developers.facebook.com/docs/facebook-login/v2.3) (links take you to the applicable area for each platform). Once procured, you need to set the following environment variables to their proper values:

- `FACEBOOK_KEY`
- `FACEBOOK_SECRET`
- `GOOGLE_KEY`
- `GOOGLE_SECRET`
- `TWITTER_KEY`
- `TWITTER_SECRET`

##### Google URIs

**IMPORTANT:** Make sure under "APIs" that you have the "Contacts API" and "Google+ API" enabled for this application.

In the `APIs & auth > Credentials` section of the [Google Developers Console](https://console.developers.google.com/) you need to create a **Client ID for OAuth** and set these values (replace `{APPLICATION_URL}` with the proper URL of the application):

- `Redirect URIs`: `http://{APPLICATION_URL}/users/auth/google_oauth2/callback`
- `Javascript Origins`: `http://{APPLICATION_URL}`

### <a name="ingest"></a>Data ingest

This project works with GeoJSON files generated by the [NYPL Map Vectorizer]. The specific files for the NYPL use case are found in the [/public/files](public/files/) folder.

#### First ingest

After downloading, and running the proper `rake db:migrate` you need to do a base ingest of data using the included rake task.

####All sheet data ingest

`rake data_import:ingest_bulk id=LAYERID force=true`

This assumes the presence of `public/files/config-ingest-LAYERID.json` with a list of IDs and bounding boxes to import for the layer `LAYERID`. This **erases all sheet/polygon/flag data** for those IDs in the config file.

####About centroids

Polygon centroids are used in some tasks to determine where to put markers in case the flag value is none/blank/etc. There is an included Python script -- `ingestor_config_builder.py` -- to easily calculate centroids for the sheets to be used in the inspector. It assumes a folder full of GeoTIFF files.

Usage:

`python ingestor_config_builder.py /path/to/folder/with/geotiffs`

It creates a config file in the application root folder with the name `config-ingest-FOLDERNAME` where `FOLDERNAME` is the name of the folder where the GeoTIFFs were found.

####Add bulk centroids

The original GeoJSON files do not have centroids (they were added and processed later). To create the centroids of the polygons in the database you need to run:

`rake data_import:ingest_centroid_bulk id=LAYERID force=true`

####Single sheet data ingest

`rake data_import:ingest_geojson id=SOMEID layer_id=SOMELAYERID bbox=SOMEBOUNDINGBOX force=true`

This imports polygons from a file `public/files/SOMEID-traced.json` into the database **replacing** any polygons (and its corresponding flags) that are associated to ID `SOMEID`.

**NOTE:** So far only layers 859 and 860 are provided. Layer 859 has separate GeoJSON for centroids and polygons. Layer 860 sheets have a single file with both fields. Ingesting 859 requires a separate `data_import:ingest_centroid_bulk` process for centroids.

####Single sheet centroid updating

`rake data_import:ingest_centroids_for_sheet id=SOMEID force=true`

This updates the polygon centroids for a given `sheet_id` from a file `public/files/SOMEID-traced.json` **replacing** any existing polygon centroids (not the polygons themselves).

### <a name="consensus"></a>Consensus generation

As people inspect polygons according to the task (eg. "YES/NO/FIX" for the geometry task) they are essentially casting a vote alongside fellow inspectors. We show the same polygon and task to several people and tally up those votes to decide whether they agree. If they do, that polygon is removed from the pool for that given task.

Each task has a different means to come to consensus. So far, consensus for `geometry`, `color` and basic `address` (value being `NONE`) are implemented. A rake task is available for this in `lib/tasks/flag_processing.rake`:

````
rake db:calculate_consensus
````

This task should be scheduled to execute regularly (say, every ten minutes). New consensus generation for other tasks is being implemented.

### <a name="api"></a>API querying

The following API endpoints have been added to export the inspection consensus process (paginated by groups of 500). Consensus is defined by **agreement of 75% or more of three or more votes**:

#### Polygon export → /api/polygons/…

Active endpoints:
````
/api/polygons/:task/page/:page
/api/polygons/:task/:consensus/page/:page
````

Accept tasks:
`geometry`, `color`, `polygonfix`, `address` with numeric paging (500 records per page).

`consensus` values depend on the task and perhaps only useful in the `geometry` and `color` tasks.

- `/api/polygons/geometry/page/PAGENUMBER` returns all polygons in `PAGENUMBER` regardless of their consensus value.
- `/api/polygons/geometry/yes/page/PAGENUMBER` returns all polygons in `PAGENUMBER` that have been marked as *correct*.
- `/api/polygons/geometry/no/page/PAGENUMBER` returns all polygons in `PAGENUMBER` that have been marked as *not buildings*.
- `/api/polygons/geometry/fix/page/PAGENUMBER` returns all polygons in `PAGENUMBER` that need to be *fixed*.

### Version notes

- 2.0 Second release with `address`, `polygonfix` and `color` tasks. Refactored and improved code. New API endpoints.
- 1.1 Added API endpoints
- 1.0 First release with single `geometry` task

### Copyright 2013-2014 The New York Public Library

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.


[Mauricio Giraldo Arteaga]: https://twitter.com/mgiraldo
[NYPL Map Vectorizer]: https://github.com/NYPL/map-vectorizer
