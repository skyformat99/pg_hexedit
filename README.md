# pg_hexedit - Open PostgreSQL relation files in a hex editor with tags and annotations

Copyright (c) 2017, VMware, Inc.
Copyright (c) 2002-2010 Red Hat, Inc.
Copyright (c) 2011-2016, PostgreSQL Global Development Group

Author: Peter Geoghegan [`<pg@bowt.ie>`](mailto:pg@bowt.ie)
pg_filedump author: Patrick Macdonald [`<patrickm@redhat.com>`](mailto:patrickm@redhat.com)

License: [GNU General Public License version 2](https://opensource.org/licenses/GPL-2.0)

Supported versions: PostgreSQL 11 (earlier versions are untested)

Supported platforms: Linux + libwxgtk (though MacOS probably also works)

## Overview

pg_hexedit is an experimental toolkit to format PostgreSQL heap and index files
(B-Trees indexes only) when opened within the open source GUI hex editor
[wxHexEditor](https://github.com/EUA/wxHexEditor).  It makes viewing and
editing PostgreSQL relation files *significantly* easier.

![Image of wxHexEditor with pg_hexedit tags](./screenshot1.png)

__CAUTION:__ Do not use pg_hexedit with a PostgreSQL data directory if you are
not prepared to have it __corrupt data__! pg_hexedit is made available for
educational purposes only.  It is an *experimental* tool, originally used for
simulating corruption/corruption analysis.

The type of file (heap/index) is determined automatically by the content of the
blocks within the file, using pg_filedump-style hueristics.  The default is to
format the entire file using the block size listed in block 0 as wxHexEdit tag
XML.  These defaults can be modified using run-time options.  However,
pg_hexedit is typically invoked using the packaged convenience scripts.


## Initial setup

### Building pg_hexedit

Note that pg_hexedit is a fork of
[pg_filedump](https://wiki.postgresql.org/wiki/Pg_filedump).  The pg_hexedit
executable, which is what actually generates wxHexEditor format XML, must be
built from source as a PostgreSQL frontend utility.

To compile pg_hexedit, you will need to have a properly configured
PostgreSQL source tree or complete install tree (with include files)
of the appropriate PostgreSQL major version.

There are two makefiles included in this package.  Makefile is a standalone
makefile for pg_hexedit.  Alter its `PGSQL_INCLUDE_DIR` variable to point to
the PostgreSQL include files.  Makefile.contrib can be used if this package
was untarred in the contrib directory of a PostgreSQL build tree:

```shell
  $ make
  $ # if using Makefile.contrib:
  $ make install
```

It is also possible to use Makefile.contrib without being in the contrib
directory:

```shell
  $ make -f Makefile.contrib USE_PGXS=1
```

This method requires that the `pg_config` program be in your $PATH, but should
not require any manual adjustments of the `Makefile`.

### Obtaining wxHexEditor

It is recommended that you build wxHexEditor's master branch from source.
There are general stability issues with wxHexEditor, especially with the tag
feature that pg_hexedit targets.  It's worth having all recent bug fixes. See:

https://github.com/EUA/wxHexEditor

It's generally only mildly inconvenient to do this on a modern desktop Linux
system.

While wxHexEditor does have noticeable stability issues, these seem to be worth
working around, given the lack of any better alternative that is open source
and cross platform.

### Caret GTK+ bug

There appears to be a tendency for wxHexEditor's caret to fail to appear on a
mouse click event.  If this happens, you can work around it by changing the
Window that is highlighted within your desktop environment.

## Quickstart guide - Using the convenience scripts

### Overview

pg_hexedit and wxHexEditor can be invoked using convenience scripts that take
care of everything.  These are designed to be run on a PostgreSQL backend
hacker's laptop, while the target PostgreSQL server is actually running.  The
server is queried to locate the relevant relation files. The scripts also take
care of adding convenience offsets to the wxHexEditor cache, which can be used
to quickly locate internal pages of a B-Tree, for example.

`psql` should be within your $PATH when the scripts are invoked. libpq
environment variables like $PGDATABASE can be set within the `hexedit.cfg`
file.  These control what database is opened by wxHexEditor, and other such
standard details.  Note that just like pg_filedump, pg_hexeditor has no
dependency on a running server, and is generally safer to use offline, despite
the fact that it is typically used online.  It is convenient to invoke
wxHexEditor using the scripts provided during analysis of *in situ* issues, or
when learning about PostgreSQL internals.

Having a PostgreSQL relfilenode file open in a hex editor __risks data
corruption__, especially when the PostgreSQL server is actually running
throughout.  The scripts were designed with backend development convenience in
mind, where __the database should only contain disposable test data__.

Convenience script dependencies:

* The convenience scripts rely on `CREATE EXTENSION IF NOT EXISTS pageinspect`
  running.

[`contrib/pageinspect`](https://www.postgresql.org/docs/current/static/pageinspect.html)
must be installed.

* The scripts are built on the assumption that they're invoked by a user that
  has the operating system level permissions needed to open PostgreSQL relation
  files, and Postgres superuser permissions.  The user invoking the script should
  have access to the same filesystem, through the same absolute paths. (Be very
  careful if the Postgres data directory is containarized; that's untested and
  unsupported.)

To open the Postgres table `pg_type` with tags and annotations:

```shell
  # Should be invoked with CWD that finds pg_hexedit executable:
  $ pwd
  /home/pg/code/pg_hexedit
  # Confirm configuration:
  $ $EDITOR hexedit.cfg
  # Invoke script:
  $ ./table_hexedit pg_type
Replacing /home/pg/code/pg_hexedit/.wxHexEditor with pg_hexedit optimized settings...
Determined that data directory is /home/pg/pgbuild/data/root
Running pg_hexedit against /home/pg/pgbuild/data/root/base/13042/1247, the first segment in relation pg_type...
Note: Only blocks 0 - 500 will be annotated, to keep overhead low
Opening /home/pg/pgbuild/data/root/base/13042/1247 with ../wxHexEditor/wxHexEditor...
Tip: 'Go to Offset' dialog (shortcut: Ctrl + G) has decile splitter block start positions cached
```

To open the Postgres B-Tree `pg_type_typname_nsp_index` with tags and annotations:

```shell
  $ ./btree_hexedit pg_type_typname_nsp_index
Replacing /home/pg/code/pg_hexedit/.wxHexEditor with pg_hexedit optimized settings...
Determined that data directory is /home/pg/pgbuild/data/root
Running pg_hexedit against /home/pg/pgbuild/data/root/base/13042/2704, the first segment in relation pg_type_typname_nsp_index...
Note: Leaf pages will have single tag to keep overhead low (-l flag does this)
Note: Only blocks 0 - 500 will be annotated, to keep overhead low
Opening /home/pg/pgbuild/data/root/base/13042/2704 with ../wxHexEditor/wxHexEditor...
Tip: 'Go to Offset' dialog (shortcut: Ctrl + G) has children of root and root offsets cached
```

The scripts will only open the first 1GB segment file in the relation.  Note
also that these convenience scripts limit the range of blocks that are
summarized, to keep the overhead acceptable.  (This can be changed by modifying
hexedit.cfg.)

If there is concurrent write activity by Postgres, the process of building XML
tags may error out before finishing.  In practice there is
unlikely to be trouble.  The scripts perform a `CHECKPOINT` before opening
relation files.

### Getting acceptable performance

While wxHexEditor compares favorably with other hex editors when tasked with
editing very large files, it appears to be far more likely to become
unresponsive when there are many tags.  It may be necessary to work around this
limitation at some point.

Generalize from the example of the convenience scripts for guidance on this.
Limiting the range that is summarized can be very effective.

## Direct invocation

pg_hexedit retains a minority of the flags that appear in pg_filedump:

```shell
  pg_hexedit [-hkl] [-R startblock [endblock]] [-s segsize] [-n segnumber] file
```

A new flag, `-l`, has been added.

Invoking it directly might be more useful when you want to work on a copy of
the database that is not under the control of a running PostgreSQL server.

See `pg_hexedit -h` for full details.

## Interpreting tuple contents with pageinspect

Because the pg_hexedit executable is a frontend utility that doesn't have
direct access to catalog metadata, tuple contents are not broken up into
multiple attribute/column tags.  The frontend utility has no way of determining
what the "shape" of tuples ought to be.

[`contrib/pageinspect`](https://www.postgresql.org/docs/current/static/pageinspect.html)
provides a solution -- it can split up the contents of the tuple further.

Suppose that you want to interpret the contents of the tuple for the `pg_type`
`int4` type's entry. This tuple (most columns omitted for brevity):

```sql
postgres=# select ctid, oid from  pg_type where typname = 'int4';
 ctid  | oid
-------+-----
 (0,8) |  23
(1 row)
```

We need to build a set of arguments to the pageinspect function
`tuple_data_split()`.  There are some subtleties that we go over now.

Copy all 140 bytes of the tuple contents within wxHexEditor into the system
clipboard (Ctrl + C). These are colored off-white or light gray.  Do not copy
header bytes, and do not copy the NUL bytes between the next tuple (alignment
padding), if any, which are *plain* white.

You should now be able to paste something close to the raw tuple contents
into a scratch buffer in your text editor.

`69 6E 74 34 00 00 ...` (Truncated for brevity.)

Copy and paste the "DataInterpreter" values for both `t_infomask2` and
`t_infomask` in plain decimal in the same scratch buffer file (N.B.: the
physical order of fields on the page *is* `t_infomask2` followed by
`t_infomask`, the opposite order to the order of corresponding
`tuple_data_split()` arguments).

You're probably using a little-endian machine (x86 is little-endian), and
that's what "DataInterpreter" will show as decimal by default.  Don't bother
trying to use bitstrings and integer casts (e.g. `SELECT x'1E00'::int4`),
because Postgres interprets that as having big-endian byte order, regardless of
the system's actual byte order.  It's best to just use interpreted decimal
values everywhere that an int4 argument is required.

Finally, we'll need to figure out a `t_bits` argument to give to
`tuple_data_split()`, which is a bit tricky with alignment considerations.
Putting it all together:

```sql
postgres=# SELECT tuple_data_split(
    rel_oid => 'pg_type'::regclass,
    t_data => E'\\x69 6E 74 34 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 0B 00 00 00 0A 00 00 00 04 00 01 62 4E 00 01 2C 00 00 00 00 00 00 00 00 EF 03 00 00 2A 00 00 00 2B 00 00 00 66 09 00 00 67 09 00 00 00 00 00 00 00 00 00 00 00 00 00 00 69 70 00 00 00 00 00 00 FF FF FF FF 00 00 00 00 00 00 00 00',
    t_infomask2 => 30,
    t_infomask => 2313,
    t_bits => '11111111111111111111111111100000');
```

This will return a bytea array, with one elemnent per tuple.  Note that this
doesn't count the Oid value as an attribute, because it's a system column.
The first element returned in our `bytea` array is the name of the type,
`int4`.

## Areas that might be improved someday

* Support additional index AMs: GIN, GiST, SP-GiST, and BRIN.

* Support sequence relation files.

* Support control files.

* Support MultiXact and CLOG/pg_xact SLRUs.

* Support full-page images from a WAL stream.

Possibly, this could be built on top of the `wal_consistency_checking` server
parameter that appeared in Postgres 10.  It looks like it wouldn't be very hard
to combine that with a hacked `pg_waldump`, whose `--bkp-details` option ouputs
different versions of the same block over time, for consumption by wxHexEditor.
(pg_waldump would only need to be customized to output raw data, rather than
generating the usual textual output; the rest could be scripted fairly easily).

Note that wxHexEditor has a "compare file" option that this could make use of.

With a bit more work, an abstraction could be built that allows the user to
travel back and forth in time to an arbitrary LSN, and to see a guaranteed
consistent image of the entire relation at that point.
