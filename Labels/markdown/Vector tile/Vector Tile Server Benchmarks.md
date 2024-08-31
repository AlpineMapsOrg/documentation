## Vector Tile Server Benchmarks
While developing the vector tile generation and serving procedure, we improved the process in multiple iterations. This next section documents some of the improvements the individual versions underwent, explains the reasoning behind them and shows the time improvements of the creation and serving of vector tiles.
### Serving
Initially we concentrated on fast serving speeds for the vector tiles. Since our AlpineMaps application is requesting a large amount of vector tiles simultaneously at the first start-up without any client side caching and should also scale well with increased traffic demands, the serving time is an important metric that should be optimized as best we can.

For testing purposes we created two test data sets of URLs. Those URLs will be requested simultaneously from the tile server using the Linux command *wget*.

**simple\_sample.txt**
Contains 4 URLs around Hohe Tauern with the highest zoom level.
We chose such a simple data set since the first iteration took far too long even for this subset.

**hohe\_tauern\_sample.txt**
Contains 90 URLs around Hohe Tauern with highest 3 zoom levels.
30 URLs per zoom level.

We chose Hohe Tauern for our sample set due to the high number of mountain peaks located in a small area.

|                           | **simple\_sample** | **hohe\_tauern\_sample** |
| ------------------------- | ------------------ | ------------------------ |
| **simple view**           | 0.25s              | 2.7s                     |
| **initial**               | 10s                | 3m 41s                   |
| **min distance features** | 0.27s              | 5.3s                     |
| **final version**         | 0.14s              | 1.39s                    |
#### Simple view
As explained before, the Martin server is able to provide the data by only defining a view with the main disadvantage being that it doesn't allow us to better configure what exactly is being served at which zoom levels. So it is very possible that two peaks, which are located quite near to each other will be served in one tile.

Nevertheless we used the simple view here for the benchmark in order to compare the times with an already optimized function provided by the creators of the Martin server application.
#### Initial
The goal of the initial version was to distribute the POI of a tile uniformly within it. The idea was to divide the tile in 4 sub-tiles and request the POI with the highest importance within this region (e.g. highest peak). Unfortunately, as can be seen in the table above, this resulted in quite long serving times of up to 4 minutes for 90 different simultaneous requests. 
#### Min distance features
This version already contained a simplified algorithm of the final version. The requested tile precomputes the distances to other peaks with higher importance metrics and shows a limited amount of a preordered list to the user. As you can see this version already performs in a similar time frame to the optimized simple view from the Martin server and was initially a good stopping point for serving optimizations. The main demerit of this version was that it dramatically increased the duration needed to precompute the tiles. Which will be discussed in the next [[Vector Tile Server Benchmarks#Creation|section]].
#### Final Version
This final version is the version at this moment in writing. Meaning that also the improvements, that are discussed in the next section, are done. The main improvement that should have contributed to the increased serving time was the introduction of indices that greatly improved the querying times.

### Creation
Similar to the serving time improvements the procedure to precompute the features for quick and ideal serving of the tiles underwent multiple versions. For testing purposes we only measured the time it took for the peaks creation for every single improvement step. The reason for this is, that it had a manageable initial time and all the other types (like cities) correlate in a similar fashion with the improvements done.

All times are calculated on Ubuntu using the following command:
```
/usr/bin/time -f "%E" psql alpinemaps -U alpine -f peaks.sql 
```

|                                                      | Peak   | City    |
| ---------------------------------------------------- | ------ | ------- |
| Initial version                                      | 2m 18s | 16m 33s |
| Materialized view and indices                        | 57s    | -       |
| Join on b.importance\_metric >= a.importance\_metric | 1m 4s  | -       |
| Removed improved ordering of intermediate table      | 1m 1s  | -       |
| Remove nested query                                  | 22s    | 1m 0s   |
| Final                                                | 22s    | 56s     |

#### Initial version
As explained before, this version correlates with the last version of the serving improvements (before final version). It calculates the minimum distance to peaks with higher importance and orders this from the highest distance to the shortest distance and orders it accordingly.
#### Materialized view and indices
In this version we changed the initial peaks view to a materialized view. This allows indices for faster filtering and comparisons. We added those indices for the geom and importance\_metric fields.
#### Join on b.importance\_metric >= a.importance\_metric
In this version we introduced a *join on* clause in the distance\_\<feature\> materialized view. The *join on* clause initially only selected the features within a certain radius around each feature and further tries to reduce the amount of features by only adding entries with higher importance\_metrics than itself to the join clause. This should in theory have improved the time by minimizing the amount of comparisons needed, but worsened it by a slight amount instead. Nevertheless we left it in there as it theoretically should provide improvements for future features types that are packed more densely. 
#### Removed improved ordering of intermediate table
We found that there was an ```order by id asc, importance_metric asc``` clause in a temporary table. Since the id is irrelevant for ordering we removed it. Additionally ordering by importance\_metric in ascending order was false, we therefore changed it to a descending order. This was done because it is more important to show features with higher importance than lower importance when two calculated min distances are similar.

#### Remove nested query
Initially we used two nested select queries for the calculation of distance\_\<feature\>. The inner query calculated a temporary table that contained every single distance to other features, while the outer query only selected the min distance of the nested query. In this step the min distance calculation was moved from the outer query to the inner query. 

After this step was done we realized that having a nested select query was not needed anymore. Those two improvements not only provided us with a cleaner code basis for future maintenance and better code understandability but also provided a significant improvement to the generation speed.

#### Final Version
As mentioned above the final version is the version at this moment in writing. There were still a couple of minor improvements and changes that happened since the previously timed version (without the focus on generation time). But in the end no major time improvements happened.


















