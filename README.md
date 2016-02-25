# Consistent Archive Tree

## motivation

Despite it's considerable age, Tape ARchive format (tar) is still in widespread use.
This is unfortunate because has as several features that are footguns in a modern context.

Firstly, because it contains timestamps (ctime, mtime) then archiving the same files twice
will give a tar file that has a few bytes different, and thus will have a different hash.
creating a archive deterministically was not a priority when tar was designed.
cryptographic hashes had not been discovered yet, and the internet was just a research project!
now days, every system is a distributed system, and security is paramount.
determinism enhances both reliability and security.

Secondly, tar does weird things about file locations. it's possible to create a tar file that contains
things in absolute directories, say `/bin` and when you unpack it, and it writes to that directory!
some tar flies contain everything neatly packed into a directory, and that unpacks to a subdirectory,
this is what you want. Other tar files can contain many files that extract straight into the current directory,
or worse, into parent directories. To be safe, you must use `tar -tf` to list the files in a tar file before
extracting them, or else files may be overridden.

Of course, most tarballs don't do that, so most people don't check, even if they know they are ment to,
so bad tarballs remain an possible attack vector. This means that applications utilizing tarballs need
to be audited, etc. A new archive format that didn't have this bug would be safe though.

Thirdly, tar format has a bunch of weird things that made sense historically (when you where actually using it
archive files to a magnetic tape) like a file can end in an arbitary number of empty blocks. (this was to make
it easy to update a file inplace, without writing out the whole archive (which takes a long time to do with
a reel of magnetic tape!)

## goals

* deterministic: the same files should produce an archive with the same hash then archived a second time.
* files should be hashed, so that tampering is detected, and so that identical files are not stored twice.
* random access: it must be possible to read a file out of the middle of a archive without reading the entire archive.
* simple: since we are inventing a new protocol, we need people to implement it, so the easier the better.

## rough spec

Archive files must contain file content, as well as metadata about that content, such as directory structures.

### file + framing.

a file is written as:

``` js
file <file.length as ascii decimal>\n
<file.content>\n
<file.length as ascii decimal>\n
<hash(file.content) as ascii hex>\n\n
```
file hash is written after the content, so it is possible to stream the file into the format without hashing it first.
(but the length can be written first because it's already known by `fs.stat(file.name)`). length is written at both ends
so it's possible to scan the file forwards and reverse, because that is necessary for tree object (next section) and
doing both the same way means it's possible to use the same routines for writing and reading both types of object.

the hash is followed by a double newline, so that separations between files are visually obvious when inspecting a raw file.

by putting the length and hash in a readable format it means it's possible to view archive text files in a text
editor/viewer which makes implementation much more accessable.
Implementing the protocol would be accessable even in a language as simple as bash.

### directory tree

A "directory" is literally a set of references that are used to look up file,
although it's commonly thought of as a containment (i.e. like a "folder" containing papers),
but that is merely an appearance. folders are really collections of references.

a folder is also represented like a file
``` js
tree <length as ascii>\n
<tree content as json literal array with one key per line and indentation as follows*:
[
  {
    "hash": <hash>,
    "mode": <mode as integer>,
    "name": <filename as string>,
    "size": <bytes as integer>
    "total": <size of all files written out inside this directory, _including_ framing> 
    //directory size?
  },
  ...
]>\n
<length as ascii decimal>\n 
<hash>\n\n
```

> \* i.e. stringify with JSON.stringify(tree, null, 2)

by using json format for the trees then it is not necessary to implement a more complex parser than something to handle
`<type> <length>\n<content>\n<length>\n<hash>\n\n`

the <size> field should be the size of the content of the files, the same numbers as where written out in the framing
of those files.

including the total length of the files inside a directory makes it more efficient to scan through the file.
as will be covered in the next section.

## order

objects are written to the file in topological order, leaves (files) before directories that contain them.
this has advantages when both reading and writing. first the directory is read to get file names,
and then the files processed in aphabetical order. Call stat on the file to get the mode and length,
write the length then `\n`, then stream the file to the archive (hashing it as it goes past),
then write another `\n`, the hash, then `\n\n`. keep track of how many bytes are written,
including if a file is a subdirectory.

write out all the files in the directory in this way, and then write the data you have collected about that directory,
the directory is written out with the same framing, but containing indented JSON format.

## seeking

to pull a single file out of a large archive, start by reading the last part of the file.
read the hash, and the length, then read back until you have the last tree object.
then look at the lengths and calculate the relative positions of those objects, whether they are another tree,
or a file themselves. This means the time required to read out a single file is proportional to how deep it is nested.

## streaming

Expanding an archive can be done is a single pass. each file is written to a temporary location (and the hash checked),
then when the tree is read and verified a directory is created and the temporary files are moved into to.
Finally, the archive comes to a single root directory, this is created with the same name as the archive file name
(or as set by the user if the file is read over standard input), and then finally moved into place.
The best part about extracting a the content of a file into a temporary location and then moving it is that
extraction is atomic, and if the process fails part way through, it will not create a partially extracted archive
in the target location, instead that will be littered in the /tmp directory, which will be cleaned up by the
operating system later.

## bikesheds

* would the performance of a binary format outweigh the cognitive ease of a text format?
* would fixed length framing (with binary lengths, or padded decimals) make parsing much easier?
* does json actually add more overhead, or would it be easier to have a simple, say, comma separated format?
  (this would mean you'd have to implement escaping of weird characters on the file names)
* should the last tree include a file name.
* since this depends on hashes, which may need to be upgraded at some point, should be have a first header
  that specifies the hash algorithm? could just be `hash <alg, for example sha256> <length in bytes>\n`
  at the start of the archive.
