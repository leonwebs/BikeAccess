## Characterizing accessibility by bicycle
---

Denver has significant bike facilities: mixed use paths, bike lanes,
and separated bike lanes.  Business decision makers might like to
include bike accessibility as a soft benefit for their employees and
clients.  They might also like to know if bike access to their
location is limited in order to advocate for more facilities
there. This project seeks to quantify bike accessibility for given
locations in Denver.  

In order for a location to be considered highly accessible, it must
have a nearby bike facility.  That path must be highly connected
rather than isolated, otherwise it isn't useful as a transportation
option.  It must also reach places people want to go--even a lengthy
trail that is remotely situated wouldn't be useful for utility bike
trips.  To quantify this, we define a value that counts the number of
people reachable on the path at a site in question.  That number is
weighted as the inverse of the population's distance to the site.
Population is used as a proxy for accessibility, because it represents
the number of people who are able to get to the site as well as
loosely representing other businesses and points of interest that can
be reached from the site.  

Bike facility data come from DRCOG's bike facility dataset, and
population data are from census data.  
[Bike facility data][facilities]  
[Population data][census]

The code and analysis is available here.  
[BikeAccess-woven.md]


[facilities]: https://data.colorado.gov/resource/aevh-apr2.geojson?$limit=1300
[census]: https://gis.drcog.org/geoserver/DRCOGPUB/ows?service=WFS&version=1.0.0&request=GetFeature&typeName=DRCOGPUB:bicycle_facility_inventory&outputFormat=application%2Fjson
[BikeAccess-woven.md]: https://github.com/leonwebs/BikeAccess/blob/master/BikeAccess-woven.md
