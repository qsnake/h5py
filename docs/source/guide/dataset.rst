.. _datasets:

========
Datasets
========

Datasets, like NumPy arrays, they are homogenous collections of data elements,
with an immutable datatype and (hyper)rectangular shape.  Unlike NumPy arrays,
they support a variety of transparent storage features such as compression,
error-detection, and chunked I/O.

They are represented in h5py by a thin proxy class which supports familiar
NumPy operations like slicing, along with a variety of descriptive attributes.

Datasets are created using either :meth:`Group.create_dataset` or
:meth:`Group.require_dataset`.  Existing datasets should be retrieved using
the group indexing syntax (``dset = group["name"]``).

NumPy compatibility
-------------------

Datasets implement the following parts of the NumPy-array user interface:

  - Slicing:  simple indexing and a subset of advanced indexing
  - **shape** attribute
  - **dtype** attribute

.. _dsetfeatures:

Special features
----------------

Unlike memory-resident NumPy arrays, HDF5 datasets support a number of optional
features.  These are enabled by the keywords provided to
:meth:`Group.create_dataset`.  Some of the more useful are:

Chunked storage
    HDF5 can store data in "chunks" indexed by B-trees, as well as in the
    traditional contiguous manner.  This can dramatically increase I/O
    performance for certain patterns of access; for example, reading every
    n-th element along the fastest-varying dimension.

Compression
    Transparent compression can substantially reduce
    the storage space needed for the dataset.  Beginning with h5py 1.1,
    three techniques are available, "gzip", "lzf" and "szip".  See the
    ``filters`` module for more information.

Error-Detection
    All versions of HDF5 include the *fletcher32* checksum filter, which enables
    read-time error detection for datasets.  If part of a dataset becomes
    corrupted, a read operation on that section will immediately fail with
    an exception.

Resizing
    When using HDF5 1.8,
    datasets can be resized, up to a maximum value provided at creation time.
    You can specify this maximum size via the *maxshape* argument to
    :meth:`create_dataset <Group.create_dataset>` or
    :meth:`require_dataset <Group.require_dataset>`. Shape elements with the
    value ``None`` indicate unlimited dimensions.

    Later calls to :meth:`Dataset.resize` will modify the shape in-place::

        >>> dset = grp.create_dataset("name", (10,10), '=f8', maxshape=(None, None))
        >>> dset.shape
        (10, 10)
        >>> dset.resize((20,20))
        >>> dset.shape
        (20, 20)

    Resizing an array with existing data works differently than in NumPy; if
    any axis shrinks, the data in the missing region is discarded.  Data does
    not "rearrange" itself as it does when resizing a NumPy array.

.. _slicing_access:

Slicing access
--------------

The best way to get at data is to use the traditional NumPy extended-slicing
syntax.   Slice specifications are translated directly to HDF5 *hyperslab*
selections, and are are a fast and efficient way to access data in the file.
The following slicing arguments are recognized:

    * Numbers: anything that can be converted to a Python long
    * Slice objects: please note negative values are not allowed
    * Field names, in the case of compound data
    * At most one ``Ellipsis`` (``...``) object

Here are a few examples (output omitted)

    >>> dset = f.create_dataset("MyDataset", (10,10,10), 'f')
    >>> dset[0,0,0]
    >>> dset[0,2:10,1:9:3]
    >>> dset[:,::2,5]
    >>> dset[0]
    >>> dset[1,5]
    >>> dset[0,...]
    >>> dset[...,6]

For compound data, you can specify multiple field names alongside the
numeric slices:

    >>> dset["FieldA"]
    >>> dset[0,:,4:5, "FieldA", "FieldB"]
    >>> dset[0, ..., "FieldC"]

Broadcasting
------------

For simple slicing, broadcasting is supported: 

    >>> dset[0,:,:] = np.arange(10)  # Broadcasts to (10,10)

Importantly, h5py does *not* use NumPy to do broadcasting before the write.
Broadcasting is implemented using repeated hyperslab selections, and is 
safe to use with very large target selections.  In the following example, a
write from a (1000, 1000) array is broadcast to a (1000, 1000, 1000) target
selection as a series of 1000 writes:

    >>> dset2 = f.create_dataset("MyDataset", (1000,1000,1000), 'f')
    >>> data = np.arange(1000*1000, dtype='f').reshape((1000,1000))
    >>> dset2[:] = data  # Does NOT allocate 3.8 G of memory

Broadcasting is supported for "simple" (integer, slice and ellipsis) slicing
only.


Coordinate lists
----------------

For any axis, you can provide an explicit list of points you want; for a
dataset with shape (10, 10)::

    >>> dset.shape
    (10, 10)
    >>> result = dset[0, [1,3,8]]
    >>> result.shape
    (3,)
    >>> result = dset[1:6, [5,8,9]]
    >>> result.shape
    (5, 3)

The following restrictions exist:

* List selections may not be empty
* Selection coordinates must be given in increasing order
* Duplicate selections are ignored

.. _sparse_selection:

Sparse selection
----------------

Additional mechanisms exist for the case of scattered and/or sparse selection,
for which slab or row-based techniques may not be appropriate.

NumPy boolean "mask" arrays can be used to specify a selection.  The result of
this operation is a 1-D array with elements arranged in the standard NumPy
(C-style) order:

    >>> arr = numpy.arange(100).reshape((10,10))
    >>> dset = f.create_dataset("MyDataset", data=arr)
    >>> result = dset[arr > 50]
    >>> result.shape
    (49,)

Additionally, the ``selections`` module contains additional classes which
provide access to native HDF5 dataspace selection techniques.  These include
explicit point-based selection and hyperslab selections combined with logical
operations (AND, OR, XOR, etc).  Any instance of a ``selections.Selection``
subclass can be used for indexing directly:

    >>> dset = f.create_dataset("MyDS2", (100,100), 'i')
    >>> dset[...] = np.arange(100*100).reshape((100,100))
    >>> sel = h5py.selections.PointSelection((100,100))
    >>> sel.append([(1,1), (57,82)])
    >>> dset[sel]
    array([ 101, 5782])

Length and iteration
--------------------

As with NumPy arrays, the ``len()`` of a dataset is the length of the first
axis, and iterating over a dataset iterates over the first axis.  However,
modifications to the yielded data are not recorded in the file.  Resizing a
dataset while iterating has undefined results.

.. note::

    Since Python's ``len`` is limited by the size of a C long, it's
    recommended you use the syntax ``dataset.len()`` instead of
    ``len(dataset)`` on 32-bit platforms, if you expect the length of the
    first row to exceed 2**32.

Reference
---------

.. autoclass:: h5py.Dataset

    **Dataset properties**

    .. autoattribute:: h5py.Dataset.shape
    .. autoattribute:: h5py.Dataset.dtype

    .. autoattribute:: h5py.Dataset.chunks
    .. autoattribute:: h5py.Dataset.maxshape
    .. autoattribute:: h5py.Dataset.compression
    .. autoattribute:: h5py.Dataset.compression_opts
    .. autoattribute:: h5py.Dataset.shuffle
    .. autoattribute:: h5py.Dataset.fletcher32

    .. autoattribute:: h5py.Dataset.regionref

    **Dataset methods**

    .. automethod:: h5py.Dataset.__getitem__
    .. automethod:: h5py.Dataset.__setitem__

    .. automethod:: h5py.Dataset.read_direct
    .. automethod:: h5py.Dataset.write_direct

    .. automethod:: h5py.Dataset.resize
    .. automethod:: h5py.Dataset.len






















