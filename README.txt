Exporting SQLite to Git
=======================

Theoretically it's as simple as:

  fossil export --git | git fast-import

Unfortunately it doesn't work that way and that's what this project is all
about.

-----------
Quick Start
-----------

1. Run the script 'build' to fetch and build a suitable fossil tool and a
   git-export-filter tool.

2. Run the script 'import' to create a sqlite.git git clone of the
   http://sqlite.org/src fossil sources. (May take up to 30 minutes.)

3. See the 'Building' entry at the bottom of this README.

-------------
Fossil Issues
-------------

There are two problems with fossil export:

1) fossil versions after 1.18 produce a git fast-import data stream that
   causes git fast-import to fail with a fatal error.

2) fossil versions starting with 1.18 mangle export branch and tag names to
   avoid including characters git does not allow.  The problem is that many
   more characters are mangled than needed so that a tag like "version-1.18"
   is converted to "version_1_18" unnecessarily.

------------------
The Export Problem
------------------

The fossil change that introduces the export problem is here:

  http://www.fossil-scm.org/index.html/info/bc8d368b66

There is even a ticket about this export issue here:

  http://www.fossil-scm.org/index.html/info/4013b0a81a

This issue affects the sqlite, sqlite_docsrc and fossil repositories making it
impossible to export them from fossil and import them into git with a current
version of fossil.

---------------
The Tag Problem
---------------

The fossil change that introduces tag mangling is here:

  http://www.fossil-scm.org/index.html/info/b707622f29

It was a well-intentioned change as previously invalid git names would be
exported, but it went way, way too far.  In fact, the actual git rules about
allowable characters in names are:

  1. Characters with an ASCII value less than or equal to 32 are not allowed
  2. The character with an ASCII value of 0x7F is not allowed
  3. The characters '~', '^', ':', '\', '*', '?', and '[' are not allowed
  4. The character '/' is a separator and is not allowed within a name
  5. The name may not start with '.' or end with '.'
  6. The name may not end with '.lock'
  7. The name may not contain the '..' sequence
  8. The name may not contain the '@{' sequence
  9. If multiple components are used (separated by '/'), no empty '' components

A patch is included in the file patches/export_c_patch_diff.txt that allows
the full diversity of git names to be used and should be applied to the fossil
src/export.c file of fossil version 1.18 before building fossil.

A .tar.gz archive of the fossil 1.18 sources may be fetched from:

  http://fossil-scm.org/index.html/tarball/fossil-v1.18.tar.gz?uuid=df9da91ba8
  
The downloaded .tar.gz file should have these hash values:

  md5:  8f4eca8f7c0fc4a3d7566c54462fb725
  sha1: 449414cb6e6cff15ce670b32cddc34bb8857d160

----------------------
Git Fast Import Issues
----------------------

The git fast-import facility does not provide a means to filter the incoming
data stream to adjust user names (fossil export data only includes the user
login name as the email address) nor a means to adjust branch/tag names
(fossil exports a 'trunk' branch where git expects a 'master' branch and fossil
also exports what are essentially lightweight tags as annotated tags).

To deal with these issues, the git-export-filter utility is used.

It can be found at:

  http://repo.or.cz/w/git-export-filter.git

The included sqlite_authors file is used with the git-export-filter tool to
supply real user names and email addresses.  Also note that the sqlite_authors
file also works for the http://sqlite.org/docsrc fossil repository as well.

------------------
The Final Solution
------------------

After building a patched version of fossil 1.18 as described above and the
git-export-filter utility, a git repository of the SQLite sources can be
created like so (which is what the import script does):

  fossil clone http://sqlite.org/src sqlite.fsl
  git --git-dir=sqlite.git init --bare
  fossil export --git sqlite.fsl | \
  git-export-filter --authors-file sqlite_authors --require-authors \
    --trunk-is-master --convert-tagger tagger | \
  git --git-dir=sqlite.git fast-import

The above will create the sqlite.git git repository that is a clone of the
SQLite sources from the SQLite fossil respository http://sqlite.org/src
(note that only sources are cloned, not tickets or wiki pages or events).

The provided 'build' script will attempt to download the necessary sources,
patch them and build a suitable fossil and git-export-filter executable file.

The provided 'import' script will then attempt to clone the SQLite sources
and convert them into a sqlite.git repository.  It may be run again to update
the sqlite.git repository with new changes.

The initial run of the 'import' script may take up to 30 minutes on a fast
machine, but subsequent runs of 'import' on a fast machine will take more like
5 minutes but the CPU will be pounded in either case.

--------
Building
--------

Ideally, simply cloning from the new sqlite.git repository would allow one to
then build SQLite by simply using make (or configure and make).

Unfortunately, this is not the case, the make will fail with a message about
no rule to make the files 'manifest' and/or 'manifest.uuid'.

Both the SQLite sources and the Fossil sources require two fossil vcs specific
files to be created ('manifest' and 'manifest.uuid') in order for make to be
successful.

The 'manifest.uuid' file simply contains the SHA-1 hash of the current checkout
and while a real 'manifest' file contains a bunch of information, the only
thing that need be present is a line containing the UTC ISO date preceded
by 'D '.

The 'create-fossil-manifest' script takes care of creating these files and
should be run with the current working directory set to the top-level of the
git clone's working directory.

Any time the HEAD commit changes, the 'create-fossil-manifest' script should
be run to update the 'manifest' and 'manifest.uuid' files before next running
make or the output of the 'sqlite_source_id()' function will be incorrect.
