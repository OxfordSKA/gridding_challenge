# Gridding Challenge Code

## [1. Background](#background)

This directory contains a simple, serial C implementation of the convolutional
gridding routine used by the W-projection imaging algorithm. The algorithm is
described in Cornwell, Golap & Bhatnagar (2008), IEEE Journal of Selected
Topics in Signal Processing, Vol. 2, Issue 5, p.647, and is available at the
following link:

<http://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=4703511>

The directory also contains wrapper code to read the input visibility data and
the complex convolution kernels.

The gridding function simply iterates the list of input visibilities and
convolves each one using the appropriate convolution kernel to update the
visibility grid in sequence. The image of the sky can then be generated by
taking the 2D FFT of the grid once all visibilities have been convolved onto it.

The challenge is to parallelise the convolution gridding step in an efficient
way to take advantage of modern multi-core and many-core processors and their
cache hierarchies.

## [2. Program structure](#progam-structure)

### `main.c`

The program reads visibility data from a file with the .vis extension. Each
visibility has coordinates (uu, vv, ww) in Fourier space, a complex amplitude,
and a scalar weight. The (uu, vv) coordinates determine where the visibility
falls on the 2D grid, and the (ww) coordinate determines which one of the
kernels in the stack will be used to do the convolution onto the grid.
Visibilities in the files used here are stored in blocks, with the data sets
provided in this example each containing a single block. The visibility
weights are generated by the program, but should be treated as input data for
the purposes of gridding.

Convolution kernels are read in from two FITS files containing the real and
imaginary components, using the function read_kernel. The convolution kernels
used for W-projection can only be defined numerically in the domain of the
grid, so they are stored in the 'kernels' array. This array is a stack of 2D
complex convolution kernels, so it could be thought of as a 3D complex cube.
The width and height of each 2D kernel in the stack is given by conv_size_half,
and the number of kernels in the stack is given by num_w_planes.
(Because the kernels are symmetric, only the first quadrant of each kernel
needs to be stored, hence the dimension is conv_size_half rather than
conv_size.)

The convolution kernel must always be centred on the visibility coordinates
(uu, vv). To achieve a more accurate convolution, the convolution kernels are
generated at a higher resolution than the grid. The number of kernel pixels
per grid cell is given by the 'oversample' parameter (which is 4 in all the
provided examples). By oversampling the kernel, we can allow for the distance
between the actual (uu, vv) coordinate of the visibility, and the coordinates
at the centre of the grid cell it falls into. The value of 'oversample' and
other kernel and grid parameters are read in from the header of the
convolution kernel FITS files.

### `oskar_grid_wproj.c`

The function that does the actual gridding is `oskar_grid_wproj_d` (in
double precision) or `oskar_grid_wproj_f` (in single precision). This
single-threaded function should be replaced to make the gridding run faster.

### How to enable checking of outputs

For verification, the output (the visibility grid and the normalisation value)
of the serial implementation can be compared with the output of the new
function.

The point at which the replacement gridding function should be called is
marked clearly in `main.c` (the section starting at line 290).
If a new gridding function is being tested, use:

```C
#define HAVE_NEW_VERSION 1
```

on line 46 of `main.c` to enable the check against the original grid.

The input data provided for this challenge are all in single precision.

## [3. Building and running the code](#building-and-running)

From this directory, the example code can be built using:

```bash
$ mkdir build
$ cd build
$ cmake ..
$ make -j8
```

Then you can create a link to the output executable ('main')
in the data directory.

From the data directory, run using:

```bash
$ ./main <test_data_root_name>
```

For example:

```bash
$ ./main SKA1-LOW_v5_EL82-EL70_100MHz
```

## [4. Test data](#test-data)

Data files to use with this challenge are available here:

<http://oskar.oerc.ox.ac.uk/gridding_challenge/gridding_data.zip>
*(Warning: 3.2 GB download!)*

Three sets of test data are provided, and their characteristics are
summarised in the table below.

| Data set        | # Visibilities | # W-planes   | Max. w_support value  |
|-----------------|---------------:|:------------:|:---------------------:|
| Synthetic       | 2152800        |      714     |           36          |
| SKA-LOW EL56-82 | 31395840       |      601     |           72          |
| SKA-LOW EL82-70 | 31395840       |      339     |           44          |

## [5. Current benchmarks](#benchmarks)

Recent work on the gridding problem was carried out by the Numerical Algorithms
Group (NAG) to port the C function to the NVIDIA Pascal (P100)
and Intel Xeon Phi/Knights Landing architectures.

A summary of the run-times obtained by NAG on these architectures using this
example data is given in the table below. Figures are in milliseconds.

|  Architecture              | Synthetic (ms) | EL56-82 (ms) | EL82-70 (ms) |
|----------------------------|---------------:|-------------:|-------------:|
| Xeon CPU (24 threads)      |   198          |    2927      |      2211    |
| Xeon Phi/KNL (136 threads) |   560          |    4948      |      3579    |
| P100                       |   74           |    781       |      553     |
