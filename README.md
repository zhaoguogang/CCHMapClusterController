CCHMapClusterController
=======================

`CCHMapClusterController` solves the problem of displaying many annotations on an `MKMapView` and is available under the MIT license.

[![Build Status](https://travis-ci.org/choefele/CCHMapClusterController.png?branch=master)](https://travis-ci.org/choefele/CCHMapClusterController)&nbsp;![Version](https://cocoapod-badges.herokuapp.com/v/CCHMapClusterController/badge.png)&nbsp;![Platform](https://cocoapod-badges.herokuapp.com/p/CCHMapClusterController/badge.png)&nbsp;[![Issues in Progress](https://badge.waffle.io/choefele/cchmapclustercontroller.png?label=in%20progress&title=In%20Progress)](https://waffle.io/choefele/cchmapclustercontroller)

## Getting started

If you have your project set up with an `MKMapView`, integrating clustering will take 4 lines of code:

<pre>
<b>#import "CCHMapClusterController.h"</b>
  
@interface ViewController()

<b>@property (strong, nonatomic) CCHMapClusterController *mapClusterController;</b>

@end

- (void)viewDidLoad
{
  [super viewDidLoad]
    
  NSArray annotations = ...
  <b>self.mapClusterController = [[CCHMapClusterController alloc] initWithMapView:self.mapView];
  [self.mapClusterController addAnnotations:annotations withCompletionHandler:NULL];</b>
}
</pre>

<p align="center" >
  <img src="MapClustering.png" alt="Map Clustering" title="Map Clustering">
</p>

Don't worry about manually updating the clusters; `CCHMapClusterController` automatically knows when changes have occurred that require the clusters to regroup.

To try out the clustering, experiment with the examples included in this project, or download the app “Stolpersteine in Berlin” [![Download on the App Store](https://linkmaker.itunes.apple.com/htmlResources/assets//images/web/linkmaker/badge_appstore-sm.png)](https://itunes.apple.com/de/app/stolpersteine-in-berlin/id640731757?mt=8&uo=4).

## Developer's guide

- [Recent changes](#recent-changes)
- [Installation](#installation)
- [Performance](#performance)
- [Cell size and margin factor](#cell-size-and-margin-factor)
- [Customizing cluster annotations](#customizing-cluster-annotations)
- [Positioning cluster annotations](#positioning-cluster-annotations)
- [Animations](#animations)
- [Code recipes](#code-recipes)
 - [Finding a clustered annotation](#finding-a-clustered-annotation)
 - [Centering the map without changing the zoom level](#centering-the-map-without-changing-the-zoom-level)
 - [Zooming in to a cluster](#zooming-in-to-a-cluster)
- [Additional reading](#additional-reading)
- [License (MIT)](#license-mit)

### Recent changes

See [Changes](https://github.com/choefele/CCHMapClusterController/blob/master/CHANGES.md) for a high-level overview of recent updates.

### Installation

Use [CocoaPods](http://cocoapods.org) to easily integrate `CCHMapClusterController` into your project. Minimum deployment targets are 6.0 for iOS and 10.9 for OS X.

```ruby
platform :ios, '6.0'
pod "CCHMapClusterController"
```

```ruby
platform :osx, '10.9'
pod "CCHMapClusterController"
```

### Performance

The clustering algorithm splits a rectangular area of the map into a grid of square cells. For each cell, a representation for the annotations in this cell is selected and displayed. 

The quad tree implementation used to gather annotations for a cell is based on [TBQuadTree](https://github.com/thoughtbot/TBAnnotationClustering/blob/master/TBAnnotationClustering/TBQuadTree.h) and is very fast. For this reason, performance is less dependent on the number of clustered annotations, but rather on the number of visible clusters on the map. This number can be configured with the cell size and the margin factor (see below). 

Other factors are the density of the clustered annotations (annotations spread over a large area cluster faster) and the way annotation views are implemented (if possible, use images instead of `drawRect:`).

The examples in this project contain two data sets for testing: 5000+ annotations in a small area around Berlin and 80000+ annotations spread over the entire US. Both data sets perform fine on an iPhone 4S.

### Cell size and margin factor

`CCHMapClusterController` has a property `cellSize` to configure the size of the cell in points (1 point = 2 pixels on Retina displays). This way, you can select a cell size that is large enough to display map icons with minimal overlap. More likely, however, you will choose the cell size to optimize clustering performance (the larger the size, the better the performance). The actual cell size used for clustering will be adjusted so that the map's width is a multiple of the cell size. This avoids realignment of cells when panning across the 180th meridian.

The `marginFactor` property configures the additional map area around the visible area that's included for clustering. This avoids sudden changes at the edges of the visible area when the user pans the map. Ideally, you would set this value to 1.0 (100% additional map area on each side), as this is the maximum scroll area a user can achieve with a panning gesture. However, this affects performance as this will cover 9x the map area for clustering. The default is 0.5 (50% additional area on each side).

To debug these settings, set the `debugEnabled` property to `YES`. This will display the clustering grid over the map.

### Customizing cluster annotations

Cluster annotations are of type `CCHMapClusterAnnotation`. You can customize their titles and subtitles by registering as a `CCHMapClusterControllerDelegate` with `CCHMapClusterController` and implementing two delegate methods.

In these methods, `CCHMapClusterAnnotation` gives you access to the annotations contained in the cluster through the property `annotations`. An annotation in this array will always implement `MKAnnotation`, but is otherwise of same type as the instances you added to `CCHMapClusterController` when calling `addAnnotations:withCompletionHandler:`.

Here is an example:

```Objective-C
- (NSString *)mapClusterController:(CCHMapClusterController *)mapClusterController titleForMapClusterAnnotation:(CCHMapClusterAnnotation *)mapClusterAnnotation
{
    NSUInteger numAnnotations = mapClusterAnnotation.annotations.count;
    NSString *unit = numAnnotations > 1 ? @"annotations" : @"annotation";
    return [NSString stringWithFormat:@"%tu %@", numAnnotations, unit];
}

- (NSString *)mapClusterController:(CCHMapClusterController *)mapClusterController subtitleForMapClusterAnnotation:(CCHMapClusterAnnotation *)mapClusterAnnotation
{
    NSUInteger numAnnotations = MIN(mapClusterAnnotation.annotations.count, 5);
    NSArray *annotations = [mapClusterAnnotation.annotations.allObjects subarrayWithRange:NSMakeRange(0, numAnnotations)];
    NSArray *titles = [annotations valueForKey:@"title"];
    return [titles componentsJoinedByString:@", "];
}
```

Customizing the look of clustered annotations is possible via the standard `mapView:viewForAnnotation:` method that's part of `MKMapViewDelegate`. In addition, the delegate method `mapClusterController:willReuseMapClusterAnnotation:` is called when a cluster annotation is reused for a cell (see property `reuseExistingClusterAnnotations` below). The iOS example contains code that demonstrates how to display the current cluster size as part of the annotation view.

### Positioning cluster annotations

For aesthetic reasons, cluster annotations are not lined up evenly as this would make the underlying grid obvious. This library comes with two implementations to configure the position of cluster annotations:

- `CCHCenterOfMassMapClusterer` (default): computes the average of the coordinates of all annotations in a cluster
- `CCHNearCenterMapClusterer`: uses the position of the annotation in a cluster that's closest to the center

Instances of these classes can be assigned to `CCHMapClusterController`'s property `clusterer`. By implementing the protocol `CCHMapClusterer`, you can provide your own strategy for positioning cluster annotations.

In addition, `CCHMapClusterController` by default reuses cluster annotations for a cell. This is beneficial for incrementally adding more annotations to the clustering (e.g. when downloading batches of data) because you want to avoid the cluster annotation jumping around during updates. Set `reuseExistingClusterAnnotations` to `NO` if you don't want this behavior.

### Animations

By default, annotation views for cluster annotations receive an animation that fades the view in when added and out when removed (`CCHFadeInOutAnimator`). You can provide your own animation code by implementing the protocol `CCHMapAnimator` and changing `CCHMapClusterController`'s property `animator`.

### Code recipes

#### Finding a clustered annotation

A common use case is to have a search field where the user can make a choice from a list of matching annotations. Selecting an annotation would then zoom to its position on the map.

For this to work, you have to figure out which cluster contains the selected annotation. In addition, the clustering changes while zooming thus requiring an incremental approach to finding the cluster that contains the annotation the user is looking for.

`CCHMapClusterController` contains an easy to use interface to help you with this. Note that you have to use an annotation that has previously been added to the clustering:

```Objective-C
    id<MKAnnotation> clusteredAnnotation = ...
    [self.mapClusterController addAnnotations:@[clusteredAnnotation] withCompletionHandler:NULL];
    
    ...
    
    [self.mapClusterController selectAnnotation:clusteredAnnotation andZoomToRegionWithLatitudinalMeters:1000 longitudinalMeters:1000];
``` 

#### Centering the map without changing the zoom level

`MKMapView` offers the method `setCenterCoordinate:animated:` to center the map on a new coordinate without changing the current zoom level. Unfortunately, this method doesn't work as advertised on iOS 7. Instead, calling it will zoom the map slightly thus causing the clusters to regroup with a different zoom level.

The following code avoids this problem:

```Objective-C
- (void)mapView:(MKMapView *)mapView didSelectAnnotationView:(MKAnnotationView *)view
{
    MKMapPoint point = MKMapPointForCoordinate(view.annotation.coordinate);
    MKMapRect rect = [mapView visibleMapRect];
    rect.origin.x = point.x - rect.size.width * 0.5;
    rect.origin.y = point.y - rect.size.height * 0.5;
    [mapView setVisibleMapRect:rect animated:YES];
}
```

#### Zooming in to a cluster

On iOS 7, you could use `showAnnotations:animated:`, but this will also add the given annotations to the `MKMapView`. Thus you will end up with all the clustered annotations on the screen _in addition_ to the clusters.

Instead, `CCHMapClusterAnnotation` offers the method `mapRect` that manually calculates an `MKMapRect` that includes all clustered annotations:

```Objective-C
- (void)mapView:(MKMapView *)mapView didSelectAnnotationView:(MKAnnotationView *)view
{
    if ([view.annotation isKindOfClass:CCHMapClusterAnnotation.class]) {
        CCHMapClusterAnnotation *clusterAnnotation = (CCHMapClusterAnnotation *)view.annotation;
        MKMapRect mapRect = [clusterAnnotation mapRect];
        UIEdgeInsets edgeInsets = UIEdgeInsetsMake(20, 20, 20, 20);
        [mapView setVisibleMapRect:mapRect edgePadding:edgeInsets animated:YES];
    }
}
```

### Additional reading

- [Theodore Calmes' in-depth explanation](http://robots.thoughtbot.com/how-to-handle-large-amounts-of-data-on-maps) of how to implement a clustering algorithm
- [Video of Claus' presentation at Macoun 2013](http://www.macoun.de/video2013tsso1.php) that explains some of the techniques used to implement `CCHMapClusterController` (German)
- [Blog article](http://www.technology-ebay.de/the-teams/ebay-kleinanzeigen/blog/ios-mapkit-clustering.html) covering Claus' Macoun presentation

### License (MIT)

Copyright (C) 2013 Claus Höfele

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
