# TimeZoneLookup

A timezone lookup library taking as parameters a latitude and a longitude and returning a timezone as string (e.g. Europe/Paris).

The goal of the library is to retrieve as fast as possible a timezone into a dataset by giving a latitude and a longitude.

# How it works

## Dataset 

The library does a simple lookup in an array dataset. The most difficult part was to generate this dataset. At the end, the library directly uses the generated dataset, but for a matter of transparency let's see how it was generated. 

```
 ____________________________      _______________      ________________      _____________
|                            |    |               |    |                |    |             |
| lat-long-points-generator  | -> | apply-tzwhere | -> | kdtree-islands | -> | reduce file |
|____________________________|    |_______________|    |________________|    |_____________|
```

### [lat-long-points-generator](https://github.com/databerries/lat-long-points-generator)
Generate a csv with lat/long points that cover the entire earth at a regular step in degrees. Sample of data (latitude, longitude, index): 

```
51.15,2.2,1
51.15,2.25,2
51.15,2.3,3
51.15,2.35,4
51.15,2.4,5
51.15,2.45,6
51.15,2.5,7
51.2,2.35,8
51.2,2.4,9
51.2,2.45,10
```
The dataset generated at this stage contains 25'920'000 points with an accuracy of 0.005 degree.

### [apply-tzwhere](https://github.com/databerries/apply-tzwhere)

Associate each lat/long point with the corresponding timezone using the library [java-tzwhere](https://github.com/sensoranalytics/java-tzwhere/)

You might wandering why not simply using java-tzwhere? Here is the reason:
java-tzwhere is a very accurate lib, which is really great! It's also relatively fast, it takes few milliseconds. However to process billion of rows it remains too slow.
This library is faster (about few nanoseconds), however it's less accurate.

This process step generates a new csv. Here is a sample of data (latitude, longitude, string timezone, index):

```
51.15,2.2,Europe/Paris,1
51.15,2.25,Europe/Paris,2
51.15,2.3,Europe/Paris,3
51.15,2.35,Europe/Paris,4
51.15,2.4,Europe/Paris,5
51.15,2.45,Europe/Paris,6
51.15,2.5,Europe/Paris,7
51.2,2.35,Europe/Paris,8
51.2,2.4,Europe/Paris,9
51.2,2.45,Europe/Paris,10
```

### [kdtree-islands](https://github.com/databerries/neareast-tz)

At this step there are timezones only for points on land. We decided to compute timezones for the points in the seas but near the coasts by 20km. To do so, we used an implementation of a [KdTree](https://github.com/phishman3579/java-algorithms-implementation/blob/master/src/com/jwetherell/algorithms/data_structures/KdTree.java).

### reduce file

At this step dataset is already usable but quite heavy. To reduce it we applied transformations on it:
* use timezone ids instead of timezone strings. Keep a mapping file to do the conversion. 
* remove the index (the index was useful to process steps in parallel and reorder the result)
* remove latitude and the longitude, because only the position of the points in the file matters (see later the hash function).

Here a sample:
```
328
328
328
328
```

# How to use it 

```Java
    TimeZoneLookup tz = new TimeZoneLookup();
    double latitude = 48.3904;
    double longitude = -4.4861;
    ZoneId result = tz.getZoneId(latitude, longitude);
    System.out.println(result);
```

* result

```
Europe/Paris
```

# Why is it useful ?
When you want a timezone without any in call to external API and to be accurate (the error margin is 5km).
Before writing this library we have benchmarked different libraries and none of them gave us good results for our application

# TODO
- Include international seas timezones
