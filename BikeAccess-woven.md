## Bike-ability index
---

This project is intended to assess the bikeability of locations based
on ease of access to population.  The characteristic I intend to
calculate is the cost weighted number of people accessible.  

#### Loading files

First we import our libararies and load the data files.  Bike paths
are given by DRCOG's bicycle facility data set.  


```python
import os
import sys
sys.path.append(os.getcwd()) # needed for pweave--why?
import matplotlib.pyplot as plt
import geopandas as gpd
#from ballpark import business as human
#from time import perf_counter as pfc

def loaddata(filename, url):
    if not(os.path.isfile('data/'+filename+'.geojson')):
        print('Retrieving {} from {} and storing to a file'.format(filename,url))
        geodat = gpd.read_file(url)
        geodat.to_file('data/'+filename + '.geojson', driver='GeoJSON')
        geodat.to_file('data/'+filename)
    else:
        print('Reading {} from disk'.format(filename)) 
        geodat = gpd.read_file('data/'+filename+'.geojson')
    #convert all to numeric where possible
    geodat = geodat.apply(gpd.pd.to_numeric, errors = 'ignore')
    #'pop' is not a good name for population
    if 'pop' in geodat.columns:
        geodat.rename({'pop':'population'}, axis = 'columns', inplace = True)

    return geodat
files = ['tracts', 'bikefacilities']
urls = ['https://data.colorado.gov/resource/aevh-apr2.geojson?$limit=1300',
        'https://gis.drcog.org/geoserver/DRCOGPUB/ows?service=WFS&version=1.0.0&request=GetFeature&typeName=DRCOGPUB:bicycle_facility_inventory&outputFormat=application%2Fjson'
        ]
urls = dict(zip(files, urls))
geodata = {eachfile: loaddata(eachfile, urls[eachfile]) for eachfile in files}

geodata['tracts'].crs = {'init': 'epsg:4326'}
geodata['bikefacilities'].crs = {'init': 'epsg:2877'}
```

```
Reading tracts from disk
Reading bikefacilities from disk
```



#### Preparing the data

Now that we have the data downloaded, let's start working with it.
This is my first strategy:
1. Find the population density of each tract, n.
2. Find the intersection of the paths with the census tracts. This
   creates a bunch of segments.
3. Establish an arbitrary finite width w0 around the path to be
   considered nearby.  
4. The number of people near each segment is w0 l n where, l is the
   length of the segment
5. Caluclate the characteristic for a given location by adding up the
   contribution from each successive contiguous piece.   
   

```python
ga = geodata['tracts'][['geometry', 'population']]
gb = geodata['bikefacilities'][['geometry']]
ga = ga.to_crs(gb.crs)

ga['area'] = ga.area
# area is in square feet
## convert to square miles
# ga['area'] = ga['area']/(5280**2)
ga['density'] = ga['population']/ga['area']
ga = ga[['geometry', 'density']]
w0 = 600 #ft, i.e. 100 yd on either side
```




```python
ga['geocopy'] = ga.geometry
segs = gpd.sjoin(gb, ga, how='inner', op='intersects')
segs.geometry  = segs.intersection(gpd.GeoSeries(segs.geocopy, crs = segs.crs))
#population per foot of distance to path
segs['n'] = segs.length*segs.density

segs = segs.drop(columns=['geocopy'])
segs['path_index'] = segs.index
segs = segs.rename(columns = {'index_right': 'tract_index'})
segs.reset_index(inplace = True, drop = True)
```



This doesn't quite do it, because a path could enter a tract, leave
it, and return.  That leaves disjoint segments, which must be
separated. 

```python
mults = segs.loc[segs['geometry'].type == 'MultiLineString' ]
mults = gpd.GeoDataFrame(([j,i] for i in mults.index for j in mults.geometry[i]),
                         columns = ['geometry', 'ind']
).join(mults, on = 'ind', rsuffix = '_multi'
).drop(columns = ['geometry_multi', 'ind'])
segs = (segs.loc[segs['geometry'].type == 'LineString' ]).append(
    mults, ignore_index = True)
```



#### Calculating the statistic

Now for a given piece add the contribution from each successive
contiguous piece.  Here I think I'll store each contiguous set so it
won't have to calculate it each time.  No, this won't work because the
distance to the point in question will change.  Oh and how to deal
with multiple ways of getting to the same place?  I must extend the
network only on the branch of shortest length.

Strategy: make a dictionary with keys being the length and values the
tail of the thread.  In case there are threads with identical lengths,
make the key (length, index).  Then pick the shortest available
non-dead end.  Sorting is by first member then second member, so that
works fine.  

Most of the work of growing the tree comes from finding the
intersections.  These I save in a data series for future use.  Some of
the stats seem really high.  I'm pretty sure that this is because the
metric is ill conditioned if the root segment is really short.  I
think I better roll it off.  The width is a nice way to do this, since
it represents an amount of distance that is essentially free.
... This isn't enough since there are many very short segments and
some are adjacent to each other.  See [future work].  


```python

class Forest(gpd.geodataframe.GeoDataFrame):#, gpd.geodataframe.GeoDataFrame):
    def __init__(self, fromfile, gdf=segs, width=w0):
        if not fromfile==None:
            gdf = gpd.read_file(
                'data/{}.geojson'.format(fromfile))
            gdf['sectsets'] = gpd.pd.read_pickle(
                'data/{}sectsets'.format(fromfile))
        elif 'gstat' not in gdf.columns:
            gdf = gdf.assign(sectsets =
                        gpd.pd.DataFrame(data = [{i: set() for i in gdf.index}]).T)
            gdf['gstat'] = gpd.np.NaN
            gdf['treeindex'] = -1
        gpd.geodataframe.GeoDataFrame.__init__(self, gdf)
        self.w = width
        
    def root(self, iroot):
        l = self.geometry[iroot].length/2 + self.w
        #this better conditions the statistic
        self.g = self.dg(iroot,l)
        self.tipdict = {(l,0): iroot}
        self.icand = set(self.index)
        self.loc[iroot, 'treeindex'] = self.workingtree
        
    def extend(self, itiptup, meth):
        l = itiptup[0]
        i0 = self.tipdict.pop(itiptup)
        self.icand -= {i0}
        self.loc[i0, 'treeindex'] = self.workingtree 
        isect = meth(i0)
        for i in isect:
            dl = self.geometry[i].length
            self.g += self.dg(i,l+dl)
            self.tipdict[self.keyassign(l+dl)] = i

    def sectfind(self, i0):
        sects =  self.geometry[list(self.icand)].intersects(self.geometry[i0])
        isects =  sects[sects].index
        #remember what intersections were
        self.at[i0,'sectsets'] |= set(isects)
        for i in isects:
            self.at[i, 'sectsets']  |= {i0,}
        return isects
    def sectlookup(self, i0):
        return self.icand & self.at[i0,'sectsets']
        
    def keyassign(self, length):
        aux=0
        while (length, aux) in self.tipdict.keys():
            aux+=1
        return (length, aux)

        # k = [j[1] for j in self.tipdict.keys() if j[0]==length]
        # if len(k)==0:
        #     k = (length, 0)
        # else:
        #     k = (length, 1+max(k))
        # return k
    
    def dg(self, itip, l):
        "delta g"
        return (self.geometry[itip].length *
                self.w *
                self.density[itip] *
                (1/l)
        )

    def growfromroot(self, iroot):
        self.workingtree = self.treeindex[iroot]
        if not gpd.np.isnan(self.gstat[iroot]):
            return
        elif self.workingtree < 0:
            meth = self.sectfind
            self.workingtree = max(self.treeindex) + 1
            print('Planting new tree...')
        else:
            meth = self.sectlookup
            print('Growth on  old tree')
        self.root(iroot)
        while len(self.tipdict)>0:
            #choose the branch of min length so we don't
            #accidently reach some population by a longer route
            #than necessary
            imin = min(self.tipdict.keys())
            self.extend(imin, meth)
        self.loc[iroot, 'gstat'] = self.g

    def save(self, filename = 'bikeforest'):
        self.drop(columns = ['sectsets']).to_file(
            'data/{}.geojson'.format(filename),
            driver = 'GeoJSON')
        self[['sectsets']].to_pickle(
            'data/{}sectsets'.format(filename))

filename = None
if os.path.isfile('data/bikeforest.geojson'):
    filename='bikeforest'
    print('Retrieving bikeforest from file')
bikeforest = Forest(fromfile = filename)
```

```
Retrieving bikeforest from file
```



This works.  It does take a while, depending on how large the path
network is.   The following would run them all, but would take a long
time.

```python
for j in range(29):
    l = 500*j
    for i in range(l,l+500):
        print(i)
        bikeforest.growfromroot(i)
    bikeforest.save()
```


Instead let's pick a thousand at random and do them.  

```python
for i in gpd.np.random.choice(len(bikeforest), size = 1000, replace = False):
    print(i)
    bikeforest.growfromroot(i)
bikeforest.save()
```



#### Plotting

Now let's make a few plots to see how it does.  First we'll make a map
and highlight the most connected places.  I'd like it to be laid over
a street map, but let's pick up that piece later.  Then we'll plot the
distribution of the statistic.  

```python
f = []
ax = []
def appendnewfig(*args, **kwargs):
    f[len(f):], ax[len(ax):] = tuple(zip(
        plt.subplots()
        ))

appendnewfig()
bikeforest.plot(ax = ax[-1])
bikeforest[bikeforest.gstat.notna()].plot(column='gstat', ax = ax[-1])
f[-1].savefig('figures/map.pdf')
appendnewfig()
g = bikeforest.gstat[~gpd.np.isnan(bikeforest.gstat)]
g.sort_values(inplace=True)
g.reset_index(inplace = True, drop = True)
g.plot(ax =ax[-1])
f[-1].savefig('figures/cdf.pdf')
```

![](figures/BikeAccess_Plotting_1.png){#Plotting }\
![](figures/BikeAccess_Plotting_2.png){#Plotting }\





### Future Work
This is a first pass at doing this.  There is a big problem that there
are many extremely short segments because of the random nature of
dividing the paths.  These short segments ill-condition the statistic.
In addition there are some very long segments, so using a single
length for the whole thing isn't quite right.  I must overhaul the
process to break each path up into regularly sized segments.  In this
case, it would take the population density of each segments as a
weighted mean of the census tracts it passes through.
