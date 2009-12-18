s3fslite
========

s3fslite is a fork of Randy Rizun's s3fs, a file system that stores
all data in an Amazon S3 bucket:

* <http://code.google.com/p/s3fs/wiki/FuseOverAmazon>

This fork is intended to work better when using `rsync` to copy data
to an S3 mount.

This fork has the following changes:

*   File metadata is cached in a SQLite database for faster access.
    File systems do lots of `getattr` calls, and each one normally
    requires a HEAD request to S3. Caching them locally improves
    performance a lot and reduces the number (and hence cost) of
    requests to Amazon.

    The original s3fs has the beginnings of in-memory stat caching,
    but it does not persist across mounts. For large file systems,
    losing the entire cache on a restart is costly.

*   `readdir` requests do *not* send off file attribute requests.
    The original code effectively issues a `getattr` request to S3
    for each file when directories are listed. The cache is not
    consulted, but the results are put in the cache.

    This behavior made listing directories ridiculously slow. It
    appears to have been an attempt to optimize (by priming the
    cache) that backfired. It wouldn't be the first time that a
    cache optimization has made things slower overall.

*   The MIME type of files is reset when files are renamed. This
    fixes a bug in s3fs that is particularly devastating for `rsync`
    users. `rsync` always writes to a temporary file, then renames
    it to the target name. Without this fix, MIME types were rarely
    correct, which confused browsers when looking at static content
    on an S3 archive.

*   By default, ACLs are set based on the file permission. If the
    file is publicly readable, the "public-read" ACL is used, which
    permits anyone to read the file (including web browsers). If
    not, it defaults to "private", which denies access to public
    browsers. Setting the "default_acl" option overrides this, and
    sets everything to the specified ACL.

Everything is started as before. By default, the database is
created/opened in the current directory (you can change this using
the `-o attr_cache=` option), and is named "`<bucketname>.sqlite`".

My usual command to mount a system is:

    s3fs <bucket> <mountpoint> -o attr_cache=/var/cache/s3fs -o use_cache=/tmp -o allow_other

This mounts the file system with a file cache, makes stored files
publicly browsable, and allows all users of the local machine to use
the mount.

I also put my S3 access key and secret access key in
`/etc/passwd-s3fs` with the form:

    ACCESSKEY:SECRETACCESSKEY

s3fs knows to look for it there, so you do not have to provide it on
the command line.

The complete list of supported options is:

*   `accessKeyId=` specify the Amazon AWS access key (no default)

*   `secretAccessKey=` specify the Amazon AWS secret access key (no
    default)

*   `default_acl=` specify the access control level for files
    (default `public-read` for files with public read permissions,
    `private` for everything else).

*   `retries=` specify the maximum number of times a failed/timed
    out request should be retried (default `2`)

*   `use_cache=` specify the directory for (and enable) a file cache
    (default no cache)

*   `connect_timeout=` specify the timeout interval for request
    connections (default `2`)

*   `readwrite_timeout=` specify the timeout interval for read and
    write operations (default `10`)

*   `url=` specify the host to connect to (default
    `http://s3.amazonaws.com`)

*   `attr_cache=` specify the directory where the attribute cache
    database should be created and accessed (default current
    directory)

And now for the README that was included with the original code
(with a few updates):


S3fs-FUSE
=========

S3FS is FUSE (File System in User Space) based solution to
mount/unmount an Amazon S3 storage buckets and use system commands
with S3 just like it was another Hard Disk.

In order to compile s3fs, You'll need the following requirements:

*   Kernel-devel packages (or kernel source) installed that is the
    SAME version of your running kernel

*   LibXML2-devel packages

*   CURL-devel packages (or compile curl from sources at:
    curl.haxx.se/ use 7.15.X)

*   GCC, GCC-C++

*   pkgconfig

*   FUSE (2.7.x)

*   FUSE Kernel module installed and running (RHEL 4.x/CentOS 4.x
    users read below)

*   OpenSSL-devel (0.9.8)

*   SQLite 3

If you're using YUM or APT to install those packages, then it might
require additional packaging, allow it to be installed.

I think this list has everything you need for Ubuntu. I already have
many of the development packages on my system, so this list may not
be complete. Please let me know if you discover something that is
missing:

*   build-essential

*   pkg-config

*   libxml2-dev

*   libcurl4-openssl-dev

*   libsqlite3-dev

*   libfuse2

*   libfuse-dev

*   fuse-utils

To install them all, type these commands from a shell:

    sudo apt-get install build-essential pkg-config libxml2-dev
    sudo apt-get install libcurl4-openssl-dev libsqlite3-dev
    sudo apt-get install libfuse2 libfuse-dev fuse-utils

If you do this and it does not compile from Ubuntu, please let me
know so I can update the list with any missing packages.


Downloading and compiling:
--------------------------

In order to download s3fs, user the following command:

    git clone git://github.com/russross/s3fslite.git

Go inside the directory that has been created (`s3fslite`) and run:

    make

This should compile the code. If everything goes OK, you'll be
greated with "ok!" at the end and you'll have a binary file called
"`s3fs`".

As root (you can use `su`, `su -`, `sudo`) do:

    make install

this will copy the "`s3fs`" binary to `/usr/bin`.

Congratulations. S3fs is now compiled and installed.


Usage:
------

In order to use `s3fs`, make sure you have the Access Key and the
Secret Key handy.

First, create a directory where to mount the S3 bucket you want to
use.  Example (as root):

    mkdir -p /mnt/s3

Then run:

    s3fs mybucket -o accessKeyId=aaa -o secretAccessKey=bbb /mnt/s3

This will mount your bucket to `/mnt/s3`. You can do a simple
"`ls -l /mnt/s3`" to see the content of your bucket.

If you want to allow other people access the same bucket in the same
machine, you can add "`-o allow_other`" to read/write/delete content
of the bucket.

You can add a fixed mount point in `/etc/fstab`, here's an example:

    s3fs#mybucket /mnt/s3 fuse allow_other,accessKeyId=XXX ,secretAccessKey=YYY 0 0

This will mount upon reboot (or by launching: `mount -a`) your bucket
on your machine.

All other options can be read at:

* <http://code.google.com/p/s3fs/wiki/FuseOverAmazon>


Known Issues:
-------------

s3fs should be working fine with S3 storage. However, There are
couple of limitations:

*   There is no full UID/GID support yet, everything looks as
    "`root`" and if you allow others to access the bucket, others
    can erase files. There is, however, permissions support built
    in.

*   Currently s3fs could hang the CPU if you have lots of time-outs.
    This is *not* a fault of s3fs but rather `libcurl`. This
    happends when you try to copy thousands of files in 1 session,
    it doesn't happend when you upload hundreds of files or less.

*   CentOS 4.x/RHEL 4.x users: if you use the kernel that shipped
    with your distribution and didn't upgrade to the latest kernel
    RedHat/CentOS gives, you might have a problem loading the
    "`fuse`" kernel. Please upgrade to the latest kernel (2.6.16 or
    above) and make sure "`fuse`" kernel module is compiled and
    loadable since FUSE requires this kernel module and s3fs
    requires it as well.

*   Moving/renaming/erasing files takes time since the whole file
    needs to be accessed first. A workaround could be to use s3fs's
    cache support with the `-o use_cache` option.


License:
--------

s3fslite retains the original GPL v2 license that s3fs uses. See the
file `COPYING` for details.
