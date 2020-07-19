**Under Construction. Corrections and additional text coming soon.**

# Introduction
The G93 library implements a data compression technique that reduces many floating-point
raster data sets to about 50% of their uncompressed size while preserving the full
precision of the original data. The technique is relatively simple and requires
moderate computational resources. Because the G93 data compression technique is
_non-lossy_, it is suitable for various applications including data archival,
distribution, and real-time processing systems.

This wiki article provides information about the G93 technique. It includes performance test
data for representative data sets and provides implementation details for the data format
used by the software. The notes included in this article should be useful to developers
who wish to apply the G93 library to their own applications or who wish to implement
their own data compression logic for floating-point data.

# Performance Testing

The effectiveness of the G93 floating-point data compression algorithms was tested
using elevation and ocean bottom depth data taken from the U.S Geological Survey
(USGS) Cloud Optimized GeoTIFF (COG) data products and the GEBCO_2019 global
bathymetry data set. Although G93 is not limited to processing geophysical
applications, the availability of a large set of elevation data made these
products a convenient source of test data. Both products provide raster data
in grids with a uniform angular spacing (i.e fixed intervals of latitude and longitude). Each of the
high-resolution USGS products use a 1/3 second of arc spacing and cover a
1 degree square region. The lower-resolution GEBGO product covers the entire
Earth (land and water) and provides a raster data with a spacing of 15 seconds of arc.

In the table, the regions of coverage for the USGS data are indicated by named locations.
All data values are given in bits-per-symbol (bits for elevation point)[&lsqb;1&rsqb;](#note1). The data sources
used for this test series were taken from files in either the GeoTIFF or NetCDF data format,
both of which implement their own techniques for
data compression. Thus the bits-per-symbol rates for the source files were
already less than the 32 bits that would be expected for the single-precision
floating-point data that they provided. Again, all three of the data formats cited
in the table are based on _non-lossy_ compression techniques that preserve
the full precision of the original data. Better compression ratios are available
using _lossy_ methods, but only at the cost of losing some information from the source data. 

As the table shows, the G93 generally outperforms the better established data formats.
However, there are other, commercially available data compression implementations
that outperform G93. Improving the G93 data compression ratios is an area of continuing research.



| Data Set           | Format | Entropy (bps) | Source sz (bps) | G93 Compressed (bps)|
| ------------------ | ------ |-------------- | --------------- | ------------------- |
| State College, PA  | TIFF   |  23.9         |   24.4          |  17.2               |
| Mt. Washington, NH | TIFF   |  23.8         |   23.8          |  16.7               |
| Newport News, VA   | TIFF   |  17.7         |   17.4          |  12.6               |
| GEBCO_2019         | NetCDF |  24.7         |   25.1          |  15.4               |

The table above refers to a metric called _entropy_. For purposes of this article,
entropy can be thought of a measure of the amount of variability or complexity
in a data set. Data with a lot of variability will have a high entropy value.
Data with little variability will have a low entropy value. In practice, the
amount of entropy in a data source is a good indicator of how well it should compress.
It is used here because it provides an objective way of characterizing the variability
of an input data set and rating the effectiveness of the data compression algorithm.
Entropy values are essentially the idealized average number of bits per symbol required
to encode the symbols (elevations, depths, etc.) in the source data based on certain
formal definitions.

What we see in the table confirms that entropy is a rough predictor of the ability of the various
data compression schemes used by the three implementations (TIFF, NetCDF, and G93) to reduce the
size of the original 32-bit floating-point values. In relatively flat regions, such as
Newport News, the entropy is lower and the data compression results in small outputs.
On the other hand, the State College PA sample features a series of alternating ridges and
valleys that lead to a much larger variability in the source data, a larger entropy value,
and a larger output size.  In the case of GEBCO_2019, the higher entropy is due both to
the variability of the terrain it covers and the wider spacing of sample points.
In practice, the larger the raster cell spacing, the more opportunity there is for a large change
in value from sample to sample.

Entropy is an important consideration in data compression and information theory in general.
Readers who are interested in learning more about the concept of entropy (and Claude Shannon’s elegant
formulation of the idea) may refer to the Wikipedia article
[Entropy (information theory)](https://en.wikipedia.org/wiki/Entropy_(information_theory)).
The entropy values shown in the table above were computed using the EntropyTabulator class which is
provided in the Gridfour project software distribution. 

# The G93 Technique
Floating-point data presents special problems for data compression implementations. Many of the
techniques that produce good results for integer forms exhibit lack-luster performance for
real-valued raster products. This characteristic led the G93 project to implement special procedures
for compressing floating-point data.

The G93 does not actually break new ground, but is a refinement of ideas that were
used in the TIFF image-file format. So as an introduction to the G93 floating-point format,
we will begin with a brief look at its predecessor.

## Differencing
Before processing data using a conventional compression algorithm, raster data formats such as TIFF
often apply a preprocessing technique known as differencing. To see how the technique works,
consider a raster consisting of two rows of sample values given as:

    A, B, C,
    D, E, F

A typical differencing scheme would transform the data into a series of differences:

    A, B-A, C-B,
    D, E-D, F-E

For integer-based raster data, the differences between subsequent samples
tend to be of smaller magnitude than the source data. They also tend to have reduced
variability (less entropy) and therefore compress more efficiently than the untransformed
representation of the data.

Unfortunately, differencing techniques do not work well for floating-point formats.
While they do reduce the magnitude of the data, they do not reduce the complexity
of the mantissa part of the floating-point representations. The TIFF data format
addresses this limitation by splitting the floating-point data into separate
bytes and performing differencing on the components (Adobe, 2005). 

As we discuss bits and byte layouts, we will use the following conventions:

1. Bits are numbered according to their power of 2.  So the low-order
   bit in a 32-bit integer or floating-point value is bit 0.  The
   high order bit is bit 31.
2. Bytes are numbered starting with 0 for the low-order byte and 3
   for the high-order byte.
3. The numbering schemes have nothing to do with byte ordering (little-endian
   versus big-endian orders). They refer to the numeric positions of the
   bits or bytes within the value being discussed.  Byte ordering is a necessary
   consideration in implementation, but is not considered in this discussion of general idea.


The TIFF differencing proceeds as follows. For each row, the set of high-order bytes
(order-3 bytes) are ganged together, followed by the set of order-2 bytes,
order-1, and finally order-0. Within each gang, differencing is performed on a
byte-by-byte basis. For example, let A0 be the low order byte of floating-point
value A, A1 be the next higher order byte, A2 the one after that, and finally A3
be the highest order byte. The TIFF floating-point differencing would encode the data as:

    A3, B3-A3, C3-B3, (A2-C3), B2-A2, C2-B2, (A1-C2), etc.
    D3, E3-D3, F3-D3, (D2-F3), E2-D2, F2-E2, (F1-F2), etc.

Note that at the transition between rows, the literal (non-differenced) value is stored
as the first symbol. And, at the transition between byte sets, TIFF just continues with the previous value
(the transition pairs are marked by parentheses). Although these transition values are perhaps
not as optimal as they could be, the rows in actual raster data sets tend to be long enough
that their overall contribution to the entropy in the encoded bytes is small.

The motivation for the TIFF approach is that floating-point data formats consist of three
parts -- the sign bit, the exponent, and the mantissa -- and each of these parts has slightly
different statistical behavior. In general, the sign bits and exponents tend to have low
variability, while the mantissa components have high variability. By partitioning the source data
into separate sets of bytes, the TIFF approach arranges them in groups that have more uniform
behavior and can compress more readily.

Once the bytes are separated into groups, the TIFF encoding scheme uses the
general-purpose LZW compression scheme to reduce the final size of the data.  LZW works
by recognizing patterns and redundant elements in the data stream and replacing them with more
compact representations.  The differencing transformation converts the original floating-point representations
(which have high variability, and thus low redundancy) to a form that yields more readily to
conventional compression. 

The TIFF approach has one important weakness.  By using byte boundaries as the division between
pieces, it combines some dissimilar components into the same group.  

Recall that the IEEE-754 single-precision floating-point format lays out the 32 bit format as follows:

1.	The sign bit (bit 31, the highest order bit)
2.	The exponent (8 bits, bits 23 through 30)
3.	The mantissa (23 bits, bits 0 through 27)

In the TIFF approach, the sign bit and the 7 high bits of the exponent
are combined in the high-order byte group. The low bit from the exponent is combined with the 7 high bits
of the mantissa.  But because these separate components have different statistical properties,
combining them diminishes the ability of the compressor to reduce their size.

### G93 Refinements
The G93 implementation refines the TIFF approach by dividing the components into 5 separate groups:  

1.	The set of sign bits (packed into bytes, 8 bits per byte), no differencing applied
2.	The set of exponents (bits 23 through 30) one byte per value, no differencing applied
3.	The set of 7 high-order bits from the mantissa (bits 16 through 22), differencing applied
4.	The set of 8 middle-order bits from the mantissa (bits 8 through 15), differencing applied
5.	The set of 8 low-order bits from the mantissa (bits 0 through 7), differencing applied

Once the floating-point data is separated into groups as described above, G93 uses the general-purpose Deflate
data compression API to reduce the data to a more compact form. Deflate is the compressor
used in the well-known Zip data compression utilities. Like the LZW compressor used by TIFF,
Deflate looks for repetition and redundancy in the data and uses compact representations
to achieve data compression. By partitioning the data into groups that are based on the nature
of the components, G93 improves the amount of repetition in the input byte sequence and thus
yields better results from the compressor.

Each of the 5 sequences of bytes is processed through Deflate separately. Differencing is applied
only to some of the sequences. Where differencing is applied, the G93 floating-point encoder follows the
pattern that is used for its integer-based counterparts ( see [G93 Compression Algorithms](https://github.com/gwlucastrig/gridfour/wiki/G93-Compression-Algorithms) ).
In particular, differencing is used for all values sequence in the data and is not
limited to the length of a single row as it is in TIFF. Instead, the transition from row to row is handled
by referring the first value in the row to the previous row.

The text below shows the general pattern of differences uses in all three mantissa sequences. In
this text, the letters A, B, C, etc. represent single bytes from within a particular sequence.

	A,   B-A, C-B
	D-A, E-D, F-E

The text block below shows an example of statistics collected while processing the GEBCO_2019 data set.
The counts are the average number of bits used for each "tile" in the data set. A tile
is a subset of the overall data grid that consists of 90 rows and 120 columns of samples.
So before compression, the groups of Exponent and Mantissa components contain 90x120 = 10800 elements
(the sign bit group contains 10800/8 = 1350 elements). So in each case except for the sign bits,
the group started with a size of 10800 bytes and was reduced to something smaller by the
Deflate compressor.  The text shows both variations (with and without
differencing). As the data shows, had differencing been used for the exponents, it would have degraded the data
compression ratios. But for the mantissa components (M2, M1, and M0), it has a small advantage. The magnitude of the component sizes also reflect the differences in variability for the different components.

	Sign bits               26.90
	Exponent               222.22
	Exponent differencing  244.65
	M2                    3364.05    (high-order 7 bits of mantissa)
	M2 differencing       2810.71
	M1                    9978.12
	M1 differencing       8870.84
	M0                    8955.62    (low-order 8 bits of mantissa)
	M0 differencing       8857.40

The choice of whether a component group is processed using differencing is based on experimentation with a number
of data sets including elevation sources and artificial test data. So it is worth emphasizing that the
choice is not based on any formal theory of data compression, but was established through plain-old
trial-and-error. In view of that, the choices made for G93 may not be the optimal solution for all data sets.

## Implementation Details
Details of the implementation including byte order, bit packing, and byte-sequence encoding can be
found by referring to the CodecFloat.java class included in the Gridfour source code distribution
under the _core_ module (see [The Gridfour Project Page)](https://github.com/gwlucastrig/gridfour) ).

## References
Adobe, Inc. (2005). _Adobe Photoshop&reg; TIFF Technical Note 3_ April 8, 2005. Accessed April 2020 from
[http://chriscox.org/TIFFTN3d1.pdf](http://chriscox.org/TIFFTN3d1.pdf).

General Bathymetric Chart of the Oceans [GEBCO], 2019. _GEBCO Gridded Bathymetry Data_.
Accessed April 2020 from [https://www.gebco.net/data_and_products/gridded_bathymetry_data/](https://www.gebco.net/data_and_products/gridded_bathymetry_data/)

U.S. Geological Survey [USGS] (2019). _USGS Digital Elevation Models (DEM) Switching to New Distribution Format_.
Accessed April 2020 from [https://www.usgs.gov/news/usgs-digital-elevation-models-dem-switching-new-distribution-format](https://www.usgs.gov/news/usgs-digital-elevation-models-dem-switching-new-distribution-format)

## Notes
<a name="note1">&lsqb;1&rsqb;</a>The USGS Cloud Optimized GeoTIFF files contain multiple
raster data products including a full-resolution source data set and 4 or 5 lower-resolution
"overview" data sets.  The bit counts cited above are based solely on the full-resolution product.

