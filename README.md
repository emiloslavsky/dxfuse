# dxfuse: a FUSE filesystem for dnanexus

A filesystem that allows users access to the DNAnexus storage system.

[![Build Status](https://travis-ci.org/dnanexus/dxfuse.svg?branch=master)](https://travis-ci.org/dnanexus/dxfuse)

**NOTE: This is a project in its beta stage. We are using it on cloud workers, however, it may be run from any machine with a network connection and a DNAx account.**

The code uses the [FUSE](https://bazil.org/fuse/)
library, implemented in [golang](https://golang.org). The DNAnexus
storage system is not POSIX compilant. It holds not just files and
directories, but also records, databases, applets, and workflows. It
allows things that are not POSIX, for example:
1. Files in a directory can have the same name
2. A filename can include slashes
3. A file and a directory may share a name

To fit these names into a POSIX compliant filesystem, as FUSE and
Linux require, files are moved to avoid name collisions. For example,
if file `foo.txt` has three versions in directory `bar`, dxfuse will
present the following Unix directory structure:

```
bar/
    foo.txt
    1/foo.txt
    2/foo.txt
```

The directories `1` and `2` are new, and do not exist in the
project. If a file contains a slash, it is replaced with a triple
underscore. As a rule, directories are not moved, nor are their
characters modified. However, if a directory name contains a slash, it
is dropped, and a warning is emitted to the log.

dxfuse approximates a normal POSIX filesystem, but does not always have the same semantics. For example:
1. Metadata like last access time are not supported
2. Directories have approximate create/modify times. This is because DNAx does not keep such attributes for directories.
3. Files are immutable, which means that they cannot be overwritten.
4. A newly written file is located locally. When it is closed, it becomes read-only, and is uploaded to the cloud.

There are several limitations currently:
- Primarily intended for Linux, but can be used on OSX
- Intended to operate on platform workers
- Limits directories to 10,000 elements
- Updates to the project emanating from other machines are not reflected locally
- Rename does not allow removing the target file or directory. This is because this cannot be
  done automatically by dnanexus.

## Implementation

The implementation uses an [sqlite](https://www.sqlite.org/index.html)
database, located on `/var/dxuse/metadata.db`. It stores files and
directories in tables, indexed to speed up common queries.

Load on the DNAx API servers and the cloud object system is carefully controlled. Bulk calls
are used to describe data objects, and the number of parallel IO requests is bounded.

dxfuse operations can sometimes be slow, for example, if the server is
slow to respond, or has been temporarily shut down (503 mode). This
may cause the filesystem to lose its interactive feel. Running it on a
cloud worker reduces network latency significantly, and is the way it
is used in the product. Running on a local, non cloud machine, runs
the risk of network choppiness.

Bandwidth when streaming a file is close to the dx-toolkit, but may be a
little bit lower. The following table shows performance across several
instance types. The benchmark was *how many seconds does it take to
download a file of size X?* The lower the number, the better. The two
download methods were (1) `dx cat`, and (2) `cat` from a dxfuse mount point.

| instance type   | dx cat (seconds) | dxfuse cat (seconds) | file size |
| ----            | ----             | ---                  |  ----     |
| mem1\_ssd1\_x4  | 3                | 4                    | 285M |
| mem1\_ssd1\_x4  | 7                | 8                    | 705M |
| mem1\_ssd1\_x4  | 73               | 74                   | 5.9G |
|                 |                  |                      |      |
| mem1\_ssd1\_x16 | 2                | 2                    | 285M |
| mem1\_ssd1\_x16 | 4                | 5                    | 705M |
| mem1\_ssd1\_x16 | 27               | 28                   | 5.9G |
|                 |                  |                      |      |
| mem3\_ssd1\_x32 | 2                | 2                    | 285M |
| mem3\_ssd1\_x32 | 5                | 4                    | 705M |
| mem3\_ssd1\_x32 | 25               | 30                   | 5.9G |


# Building

To build the code from source, you'll need, at the very least, the `go` and `git` tools.
Assuming the go directory is `/go`, then, clone the code with:
```
git clone git@github.com:dnanexus/dxfuse.git
```

Build the code:
```
go build -o /go/bin/dxfuse /go/src/github.com/dnanexus/cmd/main.go
```

# Usage

To mount a dnanexus project `mammals` on local directory `/home/jonas/foo` do:
```
sudo dxfuse -uid $(id -u) -gid $(id -g) /home/jonas/foo mammals
```

The bootstrap process has some asynchrony, so it could take it a
second two to start up. It spawns a separate process for the filesystem
server, waits for it to start, and exits. To get more information, use
the `verbose` flag. Debugging output is written to the log, which is
placed at `/var/log/dxfuse.log`. The maximal verbosity level is 2.

```
sudo dxfuse -verbose 1 MOUNT-POINT PROJECT-NAME
```

Project ids can be used instead of project names. To mount several projects, say, `mammals`, `fish`, and `birds`, do:
```
sudo dxfuse /home/jonas/foo mammals fish birds
```

This will create the directory hierarchy:
```
/home/jonas/foo
              |_ mammals
              |_ fish
              |_ birds
```

Note that files may be hard linked from several projects. These will appear as a single inode with
a link count greater than one.

To stop the dxfuse process do:
```
sudo umount MOUNT-POINT
```

## Extended attributes (xattrs)

DNXa data objects have properties and tags, these are exposed as POSIX extended attributes. The package we use for testing is `xattr` which is native on MacOS (OSX), and can be installed with `sudo apt-get install xattr` on Linux. Xattrs can be written and removed. The examples here use `xattr`, although other tools will work just as well.

DNAx tags and properties are prefixed. For example, if `zebra.txt` is a file then `xattr -l zebra.txt` will print out all the tags, properties, and attributes that have no POSIX equivalent. These are split into three correspnding prefixes _tag_, _prop_, and _base_ all under the `user` Linux namespace.

Here `zebra.txt` has no properties or tags.
```
$ xattr -l zebra.txt

base.state: closed
base.archivalState: live
base.id: file-xxxx
```

Add a property named `family` with value `mammal`
```
$ xattr -w prop.family mammal zebra.txt
```

Add a tag `africa`
```
$ xattr -w tag.africa XXX zebra.txt
```

Remove the `family` property:
```
$ xattr -d prop.family zebra.txt
```

You cannot modify any _base.*_ attribute, these are read-only. Currently, setting and deleting xattrs can be done only for files that are closed on the platform.

## Mac OS (OSX)

For OSX you will need to install [OSXFUSE](http://osxfuse.github.com/). Note that Your Milage May Vary (YMMV) on this platform, we are focused on Linux currently.

# Common problems

If a project appears empty, or is missing files, it could be that the dnanexus token does not have permissions for it. Try to see if you can do `dx ls YOUR_PROJECT:`.

If you do not set the `uid` and `gid` options then creating hard links will fail on Linux. This is because it will fail the kernel's permissions check.

There is no natural match for DNAnexus applets and workflows, so they are presented as block devices. They do not behave like block devices, but the shell colors them differently from files and directories.
