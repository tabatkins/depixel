Depixelating Pixel Art
======================

For my future use, I distilled the [algorithm for depixelating pixel art](https://web.archive.org/web/20150914131222/http://research.microsoft.com/en-us/um/people/kopf/pixelart/index.html) into its component steps, minus the explanations and figures.

General Algorithm
-----------------

1.  Construct a graph where each pixel is a node, connected to its 8 neighbors.
2.  Remove all edges between nodes with "dissimilar" colors.  (In YUV, colors with dY>48, dU>7, or dV>6.)
3.  Examine all four-squares of nodes:
	1. If all four are fully connected, remove the diagonals. (It's a block of the same color.)
	2. If they're only connected via diagonals, use feature detection to tell which diagonal to keep:
		1. Compare the length of the curves (valence-2 nodes) that each diagonal is a part of. Cast vote in favor of the longer one, with weight equal to difference.
		2. Compare the size of the connected graphs each diagonal is a part of, in an 8x8 window.  Cast vote in favor of the smaller component, with weight equal to difference in size.
		3. If either diagonal connects directly to a lone pixel (valence-1 node) cast a vote in favor of it, with weight 5 (empirically determined).
4. You now have a fully planar set of disconnected graphs.
5. Chop each connection in half, associating each half with its closer point.  Construct a Voronoi diagram from each of these points+half-connections.  Quantize the Voronoi to a quarter-pixel grid.
6. Simplify the Voronoi diagram by collapsing valence-2 nodes into lines that connect their higher/lower-connected endpoints.  
7. You now have the basic depixelated shape.
8. Walk the graph of cell points (nodes).  Ignoring junctions separating similar (connected) cells, find all runs of valence-2 cell points and turn them into quadratic b-splines, with control points set to the nodes.
9. Simplify all 3-spline junctions.
    1. If one of the splines is a "shading spline" (separating two areas with YUV distance <= 100) and the other two are "contour spline" (YUV distance > 100), connect the two contour splines into one.
    2. Otherwise, find the two splines which connect with an angle closest to 180deg, and join them together.
    3. Keep in mind that the end-point of the non-connected spline of each triad will need to be adjusted to lie exactly on the curve defined by the connected splines.
10. You now have a relatively smooth depixelated shape.
11. Detect "corners" that shouldn't be smoothed away.  Corners are all runs of 4-5 points that are related by the following 5 relations (plus rotations/mirrors):

    1. (0,0) (.25, .75) (.75, .75) (1, 0)
    2. (-.25, .25) (.25, .75) (.75, .75) (1, 0)
    3. (-.25, .25) (.25, .75) (.75, .75) (1.25, .25)
    4. (0,0) (0,1) (1,1) (1,0)
    5. (.75, -.25) (.25, .25) (.25, .75) (.75, .75) (1.25, .25)
11. Optimize all remaining nodes by minimizing an energy function over them. Each node contributes the sum of:
    1. Curvature.  Integrate the curvature function over the segment influenced by the node. Can use simple numerical sampling here.
    2. Position deviation.  Take the fourth power (empirically determined) of the distance between its current location and its original location.

	The energy function is non-linear but smooth, so you can do local modifications by randomly selecting nodes, randomly jiggling them to find a lower-energy state, and repeating until satisfied. Paper doesn't specify how many iterations they used.
12. Rejigger the "corner" nodes to match up better with the energy-minimized nodes and minimize cell-shape distortion.  Paper is unclear here: use harmonic maps; it's solving a simple sparse linear system; [Hormann 2001] has a full explanation.
13. Remember to rejigger the endpoint of the non-connected spline in each triad connection!
14. Render the graph accordingly.  [Nehab and Hoppe 2008]  Each reshaped cell diffuses its color from its centroid, constrained by the spline boundaries?  Simpler if you assume that all color differences are significant; you get more contours, but they're all solid-color and look pretty good. (And can actually be done in SVG, which doesn't have diffusion-based paint servers yet.)

DONE.

Future work
-----------

1. Detect dithering patterns used in color-constrained graphics, and consider them part of a single solid-color region.
2. Detect "corners" (junctions where long straight lines meet) and increase the multiplicity of the knot vector to get a sharper corner.

Refs
----

NEHAB, D., AND HOPPE, H. 2008. Random-access rendering of general vector graphics. ACM Trans. Graph. 27, 5, 135:1–135:10.

HORMANN, K. 2001. Theory and Applications of Parameterizing Triangulations. PhD thesis, University of Erlangen.
