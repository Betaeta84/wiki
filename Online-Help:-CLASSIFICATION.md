# Classification
The assignment of an element to a laser operation is called classification. It determines how this element will be burnt: as a vector, as an image, with a lot of power, fast / slow etc. All these properties are set by the [operations](https://github.com/meerk40t/meerk40t/wiki/Online-Help:-OPERATIONS).

There's no limit to the amount of operations a single element can be assigned to:
- a single operation, like engrave or cut
- multiple operations, a rastered image of the element as it has a solid fill and a vector cut around it's edges
- consequence: **if an element is not assigned to any operation at all, then this element will not be burned!**

You can assign an element manually to an operation if you drag that element and drop it on top of the operation:

![dragdrop](https://github.com/meerk40t/meerk40t/assets/2670784/9661da26-0e02-4e6f-a7ef-c99928134415)

Alternatively you can let MeerK40t do the assignment for you: this is done by looking at the color of the stroke and the fill of the element.

## Automatic classification
MeerK40t can assign an element to an operation based on the element type and the color of elements stroke and fill.
A simplified version of this logic:
- Elements that have a stroke color will be assigned to an engrave or cut operation of the same color.
- Elements that have a solid fill will be assigned to a raster operation that has the same color as the fill of the element. (Unless they are pure white then the will be joining the black elements - this allows white 'overpainting')
- All images will be assigned to the same image operation

You can influence this behaviour by a couple of options, set in Preferences.

<img src="https://github.com/meerk40t/meerk40t/assets/2670784/ce5b00ab-6af6-4e1d-8ee5-2353a9cf8208" width="300">

- Classify Reverse: Classify elements into operations in reverse order e.g. to match Inkscape's Object List 
- Fuzzy color-logic: Unticked: Classify elements into operations with an *exact* color match, Ticked: Allow a certain color-distance for classification.
- Treat "Black" as raster: Ticked: Classify will assign black elements to a raster operation, Unticked: Classify will assign black elements to an engrave operation
- Classify elements after creation: MeerK40t will immediately try to classify (automatically assign) an element as soon as it is created,
if you want to defer this to apply manual assignment, then untick this option.
- Classify after color-change: Whenever you change an elements color (stroke or fill), MeerK40t will then reclassify this element. You can turn this feature off by disabling this option.
- Don't autoload operations on empty set: MeerK40t feelds uncomfortable, if you don't have any operations defined (as this will not allow any elements to be burned). So it tries to come up with a default set of basic operations, if it recognises no operations. You can turn this feature off by ticking this box.
- Autogenerate Operations: If classification did not find a match, either with color matching (exact or fuzzy, see above) or by assigning to a default operation, then MeerK40t will create a matching operation for you, if you have this option set.
- Autogenerate both for fill and stroke: Active: for both stroke and fill we look for a corresponding hit, if none was found we generate a matching operation. Inactive: one hit of either stroke or fill is enough to prevent autogeneration

### Details

Don't read on, if you get easily confused or bored, MeerK40t does normally a great job of classifying elements.

<img src="https://github.com/meerk40t/meerk40t/assets/2670784/8e0c2bdd-65a3-46b0-ba09-94300bcf39c9" width="300">

The classification logic is as follows:

1. A given element will be handed over all operations in the tree (from top to bottom) who will decide if they will be able to classify it:
  1. is it the right type (eg an image op will only accept images while a cut op will refuse such a thing)
  2. if the checkmarks for stroke / color are set then it will compare it's own color to the stroke / fill of the node. If they match it will classify it   
  3. if there aren't any marks set, then it will accept such a node regardless of it's color
2. If there was no matching operation then it will look for all operations defined in the ``_default`` operations list (the one at the bottom of the screen) and will use one of them
3. If again such an op couldn't be found it will create a cut op if the elements stroke is red (with the color red and the checkmark for stroke set, so it will accept only similar colored elements), an engrave for all other colors (with the color of the element and the checkmark for stroke set, so it will only accept similar colored elements) and will look as well for a fill. If a fill is present then it will create a default raster op with the color "black" and no checkmarks set, so this raster will accept all elements with a fill.

So for a rectangle with stroke blue and fill cyan you would have two operations created:
- an engrave with a blue color
- a raster with a black color
If you don't want to have this then your shapes need to have no stroke color = transparent 

Additionally:
- Any successful classification for one op will not prevent more operations to classify the very same element too, this is very much intended (eg a pattern fill plus an outer engrave) - unless the "Stop" checkmark is set for the operation. This will prevent further classification.
- There is another checkmark: Default. If the first pass to find matching operations failed (no matching colors found), then in a second pass the operations with the default flag active will be used for classification. This step will be done before a check with the default material list would be done, so between 2. and 3. above.

Let's take this file:
![classification_test](https://github.com/meerk40t/meerk40t/assets/2670784/bc59c297-a476-41f8-ad01-e0e961f8be90)

It contains three raster operations marked yellow, cyan and black:
![grafik](https://github.com/meerk40t/meerk40t/assets/2670784/e32c4f18-e6b5-4137-8cb9-acfd85f1b03e)

Both the yellow and the cyan raster have 'fill' and 'stop' set:
![grafik](https://github.com/meerk40t/meerk40t/assets/2670784/c30b87bf-1802-4d4f-9e74-d9aff225c283)

That assigns the elements with a yellow fill to the yellow raster, the cyan ones to the cyan op and all others to the black one (as no color flag was set, which acts as an accept-all operation).
