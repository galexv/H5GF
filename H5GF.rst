# Objective
Our goal is to define a versatile Green's function format, H5GF,  that is portable and interoperable across different types of codes, algorithms, and research groups. It should be applicable to most types of correlation functions, and extendable in future versions. 

# File format
Green's functions are structures are stored in the HDF5 file format. We recommend using [hdf5 version 2](https://www.hdfgroup.org/HDF5/doc/H5.format.html) or later.

# Notation and Naming
HDF5 files are organized into groups and datasets, summarized as objects, which form a tree structure with the datasets as leaves. Attributes can be attached to each object. The H5GF Green's function file format specification adopts this naming and uses the following notation to depict the tree or its subtrees:


`\-- item`
    An object within a group, that is either a dataset or a group. If it is a group itself, the objects within the group are indented by five spaces with respect to the group name.

`+-- attribute`
    An attribute, that relates either to a group or a dataset.

`\-- data: <type>[dim1][dim2]`
    A dataset with array dimensions dim1 by dim2 and of type <type>. The type is taken from Enumeration, Integer, Float or String and follows the HDF5 Datatype classes. If the type is not mandated by H5GF, <type> is indicated. A scalar dataspace is indicated by [].

`(identifier)`
    An optional item.

`<identifier>`
    An optional item with unspecified name. 

`#comment`
   A comment briefly descibing the group or dataset.

# General organization

H5GF defines an organization of the HDF5 file or a part thereof into groups, datasets, and attributes. The root level of the H5GF structure may coincide with the root of the HDF5 file or be an arbitrary group inside the HDF5 tree. A number of groups are defined at the H5GF root level. Several levels of subgroups will exist inside the H5GF structure.

The H5GF structure is allowed to possess non-specified groups, datasets, or attributes that contain additional information such as application-specific parameters or data structures, leaving scope for future extensions or application specific features.


#H5GF file entries description

##H5GF root level

The root level of an H5GF structure holds a number of groups and is organized as follows:

H5GF root

    \-- mesh
    \-- data
    \-- (tail)
    \-- version

## data dataset
data is a multi-dimensional array with scalar (real or complex) values.
If the Green's function is tensor-valued (e.g., matrix-valued), it is represented as a scalar-valued Green's function with additional indices: [G1(k),G2(k)] --> G(k,i) with i=1..2.

## mesh group
mesh contains the grids/meshes on which the Green's function is stored. There is one mesh per dimension of the Green's function data, stored at `/mesh/1`, `/mesh/2`, etc. The number of meshes corresponds to the number of dimensions of data; mesh `<n>` corresponds to the n-th index of the `data` dataset. Examples for grids or meshes are detailed below.

    \-- mesh
         \-- 1
         \-- (2)
         \-- (3)
         \-- ...

Every mesh is itself a group containing the following entries:

    \-- kind: char[??]
    \-- (label: char[??])
    \-- <domain-specific dataset>
    \-- <domain-specific dataset>
    \-- ...

###kind
A string (character array) uniquely identifying the domain of the mesh and the meaning of the mesh-specific datasets that define the parameters of the mesh.

###label
An optional string labeling the index.

## Domain-specific mesh data
The domain-specific mesh data comprises datasets that describe the domain on which this particular index of the green's function is defined. Examples are:  k-meshes defined in the Brillouin zone; frequency meshes defined on bosonic or fermionic Matsubara frequencies; Legendre meshes; time (real or imaginary) meshes; and index meshes (to represent tensor-valued functions).

### Index mesh

    \---kind="INDEX"
    \---N :int[] # dimension

Index meshes describe simple indices (like spin or orbital indices).

### Matsubara frequency mesh

    \---kind="MATSUBARA"
    \---N :int[] # max. Matsubara frequency index
    \---statistics :int[] # 0:Bosonic 1:Fermionic
    \---beta :double[] # inverse temperature
    \---positive_only :int[] # 0 if both positive and negative frequencies are stored, 1 otherwise
    \---(points :double[N]) # location of points on the Matsubara axis

This defines the grid $ \omega_n = (2n+1)\pi/\beta $ for fermions, $ \Omega_n = 2n\pi/\beta $ for bosons.

For fermions: $n=0..(N-1)$ (N grid points) if only positive frequencies are stored; $n=-N, -(N-1), .., -1, 0,..(N-1)$ (2N grid points) if both frequencies are stored. 

For bosons:  $n=0..(N-1)$ (N grid points) if only positive frequencies are stored; $n=-(N-1), .., 0,..(N-1)$ (2N-1 grid points) if both frequencies are stored.

If the optional parameter `points' is specified, they need to be verified upon reading.

> Suggestion: split this mesh into for disjoint meshes, one for fermions, one for bosons, each for specifying both positive and negative or only positive frequencies.


> Andrey: [Q] There seem to be a number of conditionals  - this is error-susceptible for future implementations.
How about removing positive_only and replacing it with N_min (and N correspondingly with N_max), and defining grid as [N_min, N_max)? This would simplify conventions.

### Imaginary time mesh

    \---kind="IMAGINARY_TIME"
    \---N :int[] # number of time slices
    \---statistics :int[] # 0:Bosonic 1:Fermionic
    \---beta :double[] # inverse temperature
    \---last_point_included :int[] # 0 if the last point is at $\beta$, 1 otherwise (i.e. $\beta/N*(N-1)$)     
    \---half_point_mesh :int[] # 0 if points are at 0, \beta/N*0.5, \beta/N*1.5, ... \beta/N*(N-0.5), \beta. 1 if points are at 0, \beta/N, 2\beta/N, ...
    \---(points :double[N]) # location of points on the imaginary time axis

If the optional parameter `points' is specified, they need to be verified upon reading.

### Real frequency mesh

    \---kind="REAL_FREQUENCY"
    \---points :double[N] # location of points on the real frequency axis

### Legendre mesh

    \---kind="LEGENDRE"
    \---N :int[] # number of legendre points
    \---beta: double[] #inverse temperature
    \---statistics :int[] # 0:Bosonic 1:Fermionic

### momentum index mesh

    \---kind="MOMENTUM_INDEX"
    \---points : double[N][spatial_dimension] # location of the k-points, for N k-points in spatial_dimension dimensions. The entries of this matrix specify the location of the points in the Brillouin zone.


### real space index mesh

    \---kind="REAL_SPACE_INDEX"
    \---points : double[N][spatial_dimension] # location of the real space points, for N real space points in spatial_dimension dimensions. The entries of this matrix specify the location of the points in the Brillouin zone.

### Placeholder for other meshes, define if needed

 1. Non-equidistant frequency meshes
 2. Non-equidistant imaginary time meshes
 3. Power meshes

> // Andrey I would like to strongly advocate to dump points in the grids. The pros : 

> 1. It will reduce the dependency on domain specific parameters. Real world example : TRIQS-1.0 had 'half-bin' grids in and TRIQS-1.2 does not. The old data therefore is not protected to be loaded/saved automatically. So this will guarantee the compatibility of future versions of the format with the very first one (5 years down the road we won't need to look at how the specific parameter was defined if our goal is just to compare with the data).

> 2. It is more human-readable and guarantees hardcoded mapping between physical points and data in the file and not in the code (without converters from codes that may suffer from human error). The data is directly plottable - important for benchmarks.  

> Therefore the grid might look like
```
grid
     \---(points)
     \---kind 
     \---(label)
     \---<domain-specific property1> : statistics, beta, ...
     \---<domain-specific property2> : statistics, beta, ...
```

> For defined kind - the domain specific attributes should primarily be used for reading and writing. 
Main cons : redundancy for some grids, for example the Matsubara one. 
</pre>

## tail group
The tail group contains the expansion of the Green's function around Matsubara frequency infinity, written as
math::
G(i\omega_n) = c_0 + c_1/(i\omega_n) + c_2/(i\omega_n)^2+...

High frequency tails are only defined if there is only one Matsubara/imaginary time/ real time/ real frequency axis. They are not defined for multiple-frequency vertex functions.

For single frequency Green's functions, the tails are stored as matrices with dimensionality equal to the number of non-frequency indices.
 
    \-- (tail)
         \-- descriptor: string="INFINITY_POLE"
         \-- (0) # c_0 matrix
         \-- (1) # c_1 matrix
         \-- (2) # c_2 matrix
         \-- (3) # c_3 matrix
         \-- (...)

For Green's functions which are not stored in Matsubara frequencies, these coefficients describe the high frequency tails of the function transformed to Matsubara frequencies.

The descriptor specifies the type of high frequency expansion. For the numerically known high frequency behavior described here, it should be "INFINITY_POLE"

## version

    \-- version
        \-- major: int[]
        \-- minor: int[]
        \-- reference: string
        \-- originator: string

Version of the hdf5 specification this data file adheres to, with minor and major version. Current minor version is 1, current major version is 0. reference contains a string pointing to the URL of this document. Originator is a program specific string that describes the program that wrote this file.

# Future extensions
Future versions of this document may introduce new meshes and tail formats. Existing meshes and tail formats will only be changed at each major release version. 
Backward compatibility is maintained between minor versions. 