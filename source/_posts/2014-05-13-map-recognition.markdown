---
layout: post
title: "Map Recognition"
date: 2014-05-13 15:27:53 -0400
comments: true
published: false
categories: 
---
I wanted to see if I could write a program to identify a solid-filled map[^solid map] of each of the US states. To make life a little more challenging, I wanted this program to work even if the states have been rotated and are no longer in the standard ["North is up" orientation](https://en.wikipedia.org/wiki/Map#Orientation_of_maps)[^projection]; handling rotations is in addition to the requirement to handle images that are shifted and re-scaled.

[^projection]: In reality, not only do I need to be concerned with [scale, rotation and translation](http://docs.opencv.org/doc/tutorials/imgproc/imgtrans/warp_affine/warp_affine.html), but I also should care about the [map projection](https://en.wikipedia.org/wiki/List_of_map_projections).
[^solid map]: Something like [this]({{ root_url }} /images/post_images/2014-05-13-map-recognition/texas.png").

####Data Set
The first challenge was getting a set of 50 solid-filled maps, one for each of the states. Some Google searching around led to [this page](http://www.50states.com/us.htm) which has outlined images for each states. Those images have not just the outline of the state, but also text with the name of the state, the website's URL, a star showing the state capital and dots indicating what I assume are major cities. In order to standardize the images, I want to remove the text and fill in the outlines. The fill should take care of the star and the dots.

First some Python to get the list of all state names:
```python
text = requests.get("http://www.50states.com/us.htm").text
doc = lxml.html.document_fromstring(text)

states = []
for a in doc.findall(".//ul[@class='bulletedList']/li/a"):
    url = a.get("href")
    state_name = posixpath.splitext(posixpath.split(urlparse.urlsplit(url).path)[-1])[0]
    states.append(state_name)
```

These images are then found at: `"url = http://www.50states.com/maps/%s.gif" % state`.

Next I tried to open one of these images using OpenCV's `imread` only to discover that OpenCV [does not handle gifs](http://docs.opencv.org/modules/highgui/doc/reading_and_writing_images_and_video.html#imread) so I built this utility method to convert the GIF to PNG and then to read them with `imread`:
{% gist 80e27feaef8eae0ed921 %}

With the image loaded, I believed the following operations should suffice to get my final, filled in image:

1.  Remove the text heading (strip off some number of rows from the top).
2.  Convert "color" image to binary image (1=black, 0=white) (this will be [required by `findContours`](http://docs.opencv.org/modules/imgproc/doc/structural_analysis_and_shape_descriptors.html?highlight=findcontours#findcontours))
3.  Find the "countours" that bound areas we will fill in (i.e. identify the state outlines).
4.  Convert contours to polygons.
5.  Fill in area bounded by "significant" polygons.

Here when I say significant polygons, I want to make sure the polygon bounds a non-trivial area and is not a stray mark.

I use the following methods to convert image and to display the image using matplotlib: 
{% gist aca55ff569cb790d7757 %}

This is my first attempt at the algorithm:
{% gist 6b4bf88c74bcacfa9d9f %}

Which seems to work for most states, but fails for Massachusetts:

{% img /images/post_images/2014-05-13-map-recognition/massachusetts-fail.png  Failure with Massachusetts %}

This seems to be due to a break somewhere in the outline. A simple `dilate` with a `3x3` kernel solved the problem.

The final algorithm is here:
{% gist cb6a22142fc5fc4d149c %}

which works for all states:

{% img /images/post_images/2014-05-13-map-recognition/states.png  Solid Fills US States %}

Now that I have some data, it is time to design some features that are invariant to rotation, scale and translation.
