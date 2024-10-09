[![GitHub release; latest by date](https://img.shields.io/github/v/release/SETI/rms-filecache)](https://github.com/SETI/rms-filecache/releases)
[![GitHub Release Date](https://img.shields.io/github/release-date/SETI/rms-filecache)](https://github.com/SETI/rms-filecache/releases)
[![Test Status](https://img.shields.io/github/actions/workflow/status/SETI/rms-filecache/run-tests.yml?branch=main)](https://github.com/SETI/rms-filecache/actions)
[![Documentation Status](https://readthedocs.org/projects/rms-filecache/badge/?version=latest)](https://rms-filecache.readthedocs.io/en/latest/?badge=latest)
[![Code coverage](https://img.shields.io/codecov/c/github/SETI/rms-filecache/main?logo=codecov)](https://codecov.io/gh/SETI/rms-filecache)
<br />
[![PyPI - Version](https://img.shields.io/pypi/v/rms-filecache)](https://pypi.org/project/rms-filecache)
[![PyPI - Format](https://img.shields.io/pypi/format/rms-filecache)](https://pypi.org/project/rms-filecache)
[![PyPI - Downloads](https://img.shields.io/pypi/dm/rms-filecache)](https://pypi.org/project/rms-filecache)
[![PyPI - Python Version](https://img.shields.io/pypi/pyversions/rms-filecache)](https://pypi.org/project/rms-filecache)
<br />
[![GitHub commits since latest release](https://img.shields.io/github/commits-since/SETI/rms-filecache/latest)](https://github.com/SETI/rms-filecache/commits/main/)
[![GitHub commit activity](https://img.shields.io/github/commit-activity/m/SETI/rms-filecache)](https://github.com/SETI/rms-filecache/commits/main/)
[![GitHub last commit](https://img.shields.io/github/last-commit/SETI/rms-filecache)](https://github.com/SETI/rms-filecache/commits/main/)
<br />
[![Number of GitHub open issues](https://img.shields.io/github/issues-raw/SETI/rms-filecache)](https://github.com/SETI/rms-filecache/issues)
[![Number of GitHub closed issues](https://img.shields.io/github/issues-closed-raw/SETI/rms-filecache)](https://github.com/SETI/rms-filecache/issues)
[![Number of GitHub open pull requests](https://img.shields.io/github/issues-pr-raw/SETI/rms-filecache)](https://github.com/SETI/rms-filecache/pulls)
[![Number of GitHub closed pull requests](https://img.shields.io/github/issues-pr-closed-raw/SETI/rms-filecache)](https://github.com/SETI/rms-filecache/pulls)
<br />
![GitHub License](https://img.shields.io/github/license/SETI/rms-filecache)
[![Number of GitHub stars](https://img.shields.io/github/stars/SETI/rms-filecache)](https://github.com/SETI/rms-filecache/stargazers)
![GitHub forks](https://img.shields.io/github/forks/SETI/rms-filecache)

# Introduction

`filecache` is a Python module that abstracts away the location where files used or
generated by a program are stored. Files can be on the local file system, in Google Cloud
Storage, on Amazon Web Services S3, or on a webserver. When files to be read are on the
local file system, they are simply accessed in-place. Otherwise, they are downloaded from
the remote source to a local temporary directory. When files to be written are on the
local file system, they are simply written in-place. Otherwise, they are written to a
local temporary directory and then uploaded to the remote location (it is not possible to
upload to a webserver). When a cache is no longer needed, it is deleted from the local
disk.

`filecache` is a product of the [PDS Ring-Moon Systems Node](https://pds-rings.seti.org).

# Installation

The `filecache` module is available via the `rms-filecache` package on PyPI and can be
installed with:

```sh
pip install rms-filecache
```

# Getting Started


The top-level file organization is provided by the `FileCache` class. A
`FileCache` instance is used to specify a particular **sharing policy** and
**lifetime**. For example, a cache could be private to the current process and group a set
of files that all have the same basic purpose. Once these files have been (downloaded and)
read, they are deleted as a group. Another cache could be shared among all processes on
the current machine and group a set of files that are needed by multiple processes, thus
allowing them to be downloaded from a remote source only one time, saving time and
bandwidth.

A `FileCache` can be instantiated either directly or as a context manager. When
instantiated directly, the programmer is responsible for calling
`FileCache.clean_up` directly to delete the cache when finished. In addition, a
non-shared cache will be deleted on program exit. When instantiated as a context manager,
a non-shared cache is deleted on exit from the context. See the class documentation for
full details.

Usage examples:

```python
from filecache import FileCache
with FileCache() as fc:  # Use as context manager
    # Also use open() as a context manager
    with fc.open('gs://rms-filecache-tests/subdir1/subdir2a/binary1.bin', 'rb',
                    anonymous=True) as fp:
        bin1 = fp.read()
    with fc.open('s3://rms-filecache-tests/subdir1/subdir2a/binary1.bin', 'rb',
                    anonymous=True) as fp:
        bin2 = fp.read()
    assert bin1 == bin2
# Cache automatically deleted here

fc = FileCache()  # Use without context manager
# Also retrieve file without using open context manager
path1 = fc.retrieve('gs://rms-filecache-tests/subdir1/subdir2a/binary1.bin',
                    anonymous=True)
with open(path1, 'rb') as fp:
    bin1 = fp.read()
path2 = fc.retrieve('s3://rms-filecache-tests/subdir1/subdir2a/binary1.bin',
                    anonymous=True)
with open(path2, 'rb') as fp:
    bin2 = fp.read()
fc.clean_up()  # Cache manually deleted here
assert bin1 == bin2

# Write a file to a bucket and read it back
with FileCache() as fc:
    with fc.open('gs://my-writable-bucket/output.txt', 'w') as fp:
        fp.write('A')
# The cache will be deleted here so the file will have to be downloaded
with FileCache() as fc:
    with fc.open('gs://my-writable-bucket/output.txt', 'r') as fp:
        print(fp.read())
```

A `FileCachePrefix` instance can be used to encapsulate the storage prefix string,
as well as any subdirectories, plus various arguments such as `anonymous` and `time_out`
that can be specified to each `exists`, `retrieve`, or `upload` method. Thus using one
of these instances can simplify the use of a `FileCache` by allowing the user to
only specify the relative part of the path to be operated on, and to not specify various
other parameters at each method call site.

Compare this examples to the one above:

```python
from filecache import FileCache
with FileCache() as fc:  # Use as context manager
    # Use GS by specifying the bucket name and one directory level
    pfx1 = fc.new_prefix('gs://rms-filecache-tests/subdir1', anonymous=True)
    # Use S3 by specifying the bucket name and two directory levels
    pfx2 = fc.new_prefix('s3://rms-filecache-tests/subdir1/subdir2a', anonymous=True)
    # Access GS using a directory + filename (since only one directory level
    # was specified by the prefix)
    # Also use open() as a context manager
    with pfx1.open('subdir2a/binary1.bin', 'rb') as fp:
        bin1 = fp.read()
    # Access S3 using a filename only (since two directory levels were already
    # specified by the prefix))
    with pfx2.open('binary1.bin', 'rb') as fp:
        bin2 = fp.read()
    assert bin1 == bin2
# Cache automatically deleted here
```

A benefit of the abstraction is that different environments can access the same files in
different ways without needing to change the program code. For example, consider a program
that needs to access the file ``COISS_2xxx/COISS_2001/voldesc.cat`` from the NASA PDS
archives. This file might be stored on the local disk in the user's home directory in a
subdirectory called ``pds3-holdings``. Or if the user does not have a local copy, it is
accessible from a webserver at
``https://pds-rings.seti.org/holdings/volumes/COISS_2xxx/COISS_2001/voldesc.cat``.
Finally, it could be accessible from Google Cloud Storage from the ``rms-node-holdings``
bucket at
``gs://rms-node-holdings/pds3-holdings/volumes/COISS_2xxx/COISS_2001/voldesc.cat``. Before
running the program, an environment variable could be set to one of these values::

```sh
$ export PDS3_HOLDINGS_DIR="~/pds3-holdings"
$ export PDS3_HOLDINGS_DIR="https://pds-rings.seti.org/holdings"
$ export PDS3_HOLDINGS_DIR="gs://rms-node-holdings/pds3-holdings"
```

Then the program could be written as:

```python
from filecache import FileCache
import os
with FileCache() as fc:
    pfx = fc.new_prefix(os.getenv('PDS3_HOLDINGS_SRC'))
    with pfx.open('volumes/COISS_2xxx/COISS_2001/voldesc.cat', 'r') as fp:
        contents = fp.read()
# Cache automatically deleted here
```

If the program was going to be run multiple times in a row, or multiple copies were going
to be run simultaneously, marking the cache as shared would allow all of the processes to
share the same copy, thus requiring only a single download no matter how many times the
program was run:

```python
from filecache import FileCache
import os
with FileCache(shared=True) as fc:
    pfx = fc.new_prefix(os.getenv('PDS3_HOLDINGS_DIR'))
    with pfx.open('volumes/COISS_2xxx/COISS_2001/voldesc.cat', 'r') as fp:
        contents = fp.read()
# Cache not deleted here; must be deleted manually using fc.clean_up(final=True)
# If not deleted manually, the shared cache will persist until the temporary
# directory is purged by the operating system (which may be never)
```

Finally, there are four classes that allow direct access to the four possible storage
locations without invoking any caching behavior: :class:`FileCacheSourceLocal`,
:class:`FileCacheSourceHTTP`, :class:`FileCacheSourceGS`, and :class:`FileSourceCacheS3`:

```python
    from filecache import FileCacheSourceGS
    src = FileCacheSourceGS('gs://rms-filecache-tests', anonymous=True)
    src.retrieve('subdir1/subdir2a/binary1.bin', 'local_file.bin')
```

Details of each class are available in the [module documentation](https://rms-filecache.readthedocs.io/en/latest/module.html).

# Contributing

Information on contributing to this package can be found in the
[Contributing Guide](https://github.com/SETI/rms-filecache/blob/main/CONTRIBUTING.md).

# Links

- [Documentation](https://rms-filecache.readthedocs.io)
- [Repository](https://github.com/SETI/rms-filecache)
- [Issue tracker](https://github.com/SETI/rms-filecache/issues)
- [PyPi](https://pypi.org/project/rms-filecache)

# Licensing

This code is licensed under the [Apache License v2.0](https://github.com/SETI/rms-filecache/blob/main/LICENSE).
