## A map of Massachusetts SMART program rates by region

We have at least two sources of data that we will use in generating a choropleth map of Massachusetts that visualizes the difference in SMART incentive values throughout the commonwealth.

- Geographic entities: [_Massachusetts Electricity Providers by Town_](http://maps-massgis.opendata.arcgis.com/datasets/1710ebf6cf614b5fa97c0a269cece375_0) (in Shapefile format)

- Incentive rates: [_SMART Program Guidelines_](https://www.mass.gov/info-details/solar-massachusetts-renewable-target-smart-program#smart-program-guidelines-) (in Excel format)

If we want to get really fancy, we can add a shaded relief bitmap image as a base layer. References:

- http://bl.ocks.org/mjhoy/5301594

- https://bl.ocks.org/ThomasThoren/550b2ce8b1e2470e75b2

The [last time](https://github.com/ptrfrncsmrph/mass-map) I tried this, the resulting image revealed an issue with the SRTM data, so I decided if I were to attempt it again, I'd use the more granular dataset that is available for US only. I also figured I could use Python3 to read the zipped DEM files directly. Another headache I ran into was conflicting Python environments and GDAL distributions, so maybe it would be nice to have the whole environment in a Docker container to have a reliably reproducible process. This will require learning Docker. All in all, this sounds like it could be a yak-shaving mission.

- 30m tiles (http://dwtkns.com/srtm30m/)

These have to be downloaded manually:

- http://e4ftl01.cr.usgs.gov/MEASURES/SRTMGL1.003/2000.02.11/N41W070.SRTMGL1.hgt.zip
- http://e4ftl01.cr.usgs.gov/MEASURES/SRTMGL1.003/2000.02.11/N41W071.SRTMGL1.hgt.zip
- http://e4ftl01.cr.usgs.gov/MEASURES/SRTMGL1.003/2000.02.11/N41W072.SRTMGL1.hgt.zip
- http://e4ftl01.cr.usgs.gov/MEASURES/SRTMGL1.003/2000.02.11/N42W071.SRTMGL1.hgt.zip
- http://e4ftl01.cr.usgs.gov/MEASURES/SRTMGL1.003/2000.02.11/N42W072.SRTMGL1.hgt.zip
- http://e4ftl01.cr.usgs.gov/MEASURES/SRTMGL1.003/2000.02.11/N42W073.SRTMGL1.hgt.zip
- http://e4ftl01.cr.usgs.gov/MEASURES/SRTMGL1.003/2000.02.11/N42W074.SRTMGL1.hgt.zip

## Steps for creating SVG

I was hoping to have this all in a Makefile, but there are a number of things that, as far as I can tell, can't be automated. 

1. Download the Shapefile of MA towns and utility providers from [this URL](http://maps-massgis.opendata.arcgis.com/datasets/1710ebf6cf614b5fa97c0a269cece375_0) and convert it to GeoJSON using GDAL (I tried Mike Bostock's `shp2json` but I ended up with invalid output).
  ```bash
  $ mkdir -p shp
  $ curl http://maps-massgis.opendata.arcgis.com/datasets/1710ebf6cf614b5fa97c0a269cece375_0.zip \
    -o shp/ma-towns-util.zip
  $ unzip $_ -d shp
  $ mkdir -p json
  $ ogr2ogr -f GeoJSON json/ma-towns-util.json shp/Electric_Utility_Providers_in_Massachusetts.shp
  ```
2. Use [`geoproject`](https://github.com/d3/d3-geo-projection/blob/master/README.md#geoproject) (from `d3-geo-projection`) to project our map. According to [`d3-stateplane`](https://github.com/veltman/d3-stateplane#nad83--massachusetts-mainland-epsg26986) the optimal projection for Massachusetts mainland is `EPSG:26986`. I had some dependency issues trying to call this; `geoproject` is expected to be installed globally, and it required `resolve` to be installed globally as well. I don't know.
  ```bash
  $ geoproject "d3.geoConicConformal().parallels([41 + 43 / 60, 42 + 41 / 60]).rotate([71 + 30 / 60, 0]).fitSize([960, 960], d)" \
    < json/ma-towns-util.json > json/ma-projected.json
  ```
3. Convert to TopoJSON.
  ```bash
  $ geo2topo towns=json/ma-projected.json > json/ma-projected-topo.json
  ```
4. Simplify. I can use `-p 1` to simplify by pixels. This is the benefit of pre-projecting the image; no need to do any guess-and-check work with simplifying by steradians. The file size is reduced from 19.4 MB to 352 KB, and quantizing further reduces to 176 KB 👍.
  ```bash
  $ toposimplify -p 1 -f \
    < json/ma-projected-topo.json \
    > json/ma-simple-topo.json
  $ topoquantize 1e5 \
    < json/ma-simple-topo.json \
    > json/ma-quantized-topo.json
  ```
5. Merge by `ELEC_LABEL`. **Note** that `regions` field name must be in sync with the lens path in the `insertSMARTRate` script.
  ```
  $ topomerge -k "d.properties.ELEC_LABEL" regions=towns \
    < json/ma-quantized-topo.json \
    > json/ma-merge-topo.json
  ```
5. Now for the non-automated stuff. For the SMART rate data, there's no easy way of getting it; I just manually downloaded the Excel file, opened it in Numbers and exported a CSV of the section I cared about. I was then parsing that CSV using `d3.csvParse` and then scraping the relevant bits with vanilla JS `map` and `filter`. **And this is the ugly bit:** the Node script. It expects the CSV first followed by the TopoJSON. Not sure how brittle this is. **To do:** re-write in Haskell.
  ```bash
  $ node lib/insertSMARTRate.js csv/Block1.csv json/ma-merge-topo.json \
    > json/ma-rates-topo.json
  ```