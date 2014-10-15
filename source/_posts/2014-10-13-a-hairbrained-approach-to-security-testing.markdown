---
layout: post
title: "A Hairbrained Approach to Security Testing"
date: 2014-10-13 23:56:06 -0700
comments: true
categories: fuzzing format-grammar
---

Let's go back to a structure defined in the introductory post:

```
struct pascal_t
{
  unsigned 8 big length;
  unsigned 8 big string[length];
}
```

If you were testing an application that read a `pascal_t`, what kind of data would you feed it in an attempt to break it? One strategy might be to throw random data at the program: this is called _fuzzing_. Because reading `string` is dependent upon the value of `length`, it would follow that fuzzing `length` is more likely to surface vulnerabilities than fuzzing `string`. (If you think this problem is trivially easy, remember that the Heartbleed bug was this kind of exploitation.)

Binspector knows a lot about a file being analyzed. As it turns out, the knowledge it collects makes this kind of intelligent fuzzing heuristic pretty straightforward to implement.

<!-- more -->

Consider the following change to the sample binary:

{% img left /images/binfile_f.png %}

According to the format grammar, the binary file is no longer _well-formed_ because the `length` does not match the amount of ensuing data. At this point, what happens during reading depends entirely on the application. For example, binspector will produce the following:

```
$ binspector -i file.bin -t format.bfft -s user_name_t
error: EOF reached. Consider using the eof slot.
in file: format.bfft:3
$main$
```

Binspector (and any other application reading a `pascal_t`) needs `length` to derive the contents of `string`. Intelligent fuzzing is based on the observation that the more interesting values are the ones used to drive further reading of the file. With a well-formed binary and an associated format grammar, Binspector can produce a series of derivative files that have been strategically altered. (The fuzzing engine used to be a separate tool I called Hairbrain. Despite my love of the name, it was easier to maintain the tools as one codebase than keep them apart.)

## Integral Attacks

The first attack type starts with Binspector keeping track of the atoms in the binary that were evaluated to continue analysis. Since Binspector knows the types of these atoms (that is, how they will be interpreted) it can tweak them to try and throw off file reading code. Let's take a look at what Binspector produces with our sample file and grammar:

```
~/Desktop/binsample$ binspector -i file.bin -t format.bfft -s user_name_t -m fuzz
Scanned 21 nodes
Fuzzing 2 weak points . done.
Generated 4 files
```

Binspector has generated four files, each one a corruption of `file.bin` in a known way. It also produces an attack summary file which details what it has done:

```
main.first.length
    ? attack_type : usage
    ? use_count   : 1
    ? offset      : 0
    ? bits        : 8
    ? type        : unsigned
    ? big_endian  : true
    > file_0_z.bin
    > file_0_o.bin
main.last.length
    ? attack_type : usage
    ? use_count   : 1
    ? offset      : 7
    ? bits        : 8
    ? type        : unsigned
    ? big_endian  : true
    > file_7_z.bin
    > file_7_o.bin
```

The file is delineated per-attack. You will notice there are two atoms Binspector decided to attack: the `length`s of the two `pascal_t` structures. The lines prefixed by `?` reveal what Binspector knows about those particular atoms that are relevant to fuzzing them. The lines prefixed by `>` reveal files derived from the known good file but that have been attacked. For the case of integer atoms it changes the values to all-zeroes (`_z`) and all-ones (`_o`).

You now have four files against which you can harden the file code of your `pascal_t`-reading application, each of which may throw it completely off the rails.

## Shuffle Attacks

Many file formats or substructures within them are block-based. For example, a PNG contains a series of chunks that flesh out its contents. A PNG always starts with an IHDR chunk, and always finishes with an IEND chunk, and in the middle can be a varying number of others. For example:

```
IHDR | gAMA | sBIT | bKGD | oFFs | pCAL | pHYs | tIME | tEXt | IDAT | zTXt | IEND
```

A shuffle attack is based on the observation that contiguous chunks of data may affect input code differently if they are rearranged. We know that these chunks together occupy _N_ bytes in the file, and this is true regardless of the order they are in. Therefore we are free to shuffle them in-place, and we know the rest of the file should still hold up. For example what if we tried to open the above PNG that had been altered thusly:

```
IHDR | bKGD | gAMA | IDAT | oFFs | pCAL | pHYs | sBIT | tEXt | tIME | zTXt| IEND
```

Whether or not the above reordering still constitutes a valid PNG is irrelevant. What matters is _how input code will handle it_. It may look enough like a PNG to begin the file input process, only to be thrown into the weeds when it is faced with an unexpected chunk.

Since we may not want to shuffle _every_ array found in a binary, Binspector has a special keyword to enable this kind of attack. Lets modify the format grammar slightly and see what kind of fuzzing result comes out:

```
struct pascal_t
{
  unsigned 8 big length;
  unsigned 8 big string[length];

  summary str(@string);
}

struct user_name_t
{
  pascal_t name[2] shuffle;

  summary summaryof(name[0]), " ", summaryof(name[1]);
}
```

In this format grammar we have consolidated `first` and `last` into a two-element array, and have suffixed the statement with the `shuffle` keyword. This lets Binspector know that we are interested in producing a shuffle attack on what it finds. The resulting fuzz produces the following attack summary:

```
main.name
    ? attack_type : shuffle
    ? array_size  : 2
    > file_0_s_1.bin
main.name[0].length
    ? attack_type : usage
    ? use_count   : 1
    ? offset      : 0
    ? bits        : 8
    ? type        : unsigned
    ? big_endian  : true
    > file_0_0_z.bin
    > file_0_0_o.bin
main.name[1].length
    ? attack_type : usage
    ? use_count   : 1
    ? offset      : 7
    ? bits        : 8
    ? type        : unsigned
    ? big_endian  : true
    > file_0_7_z.bin
    > file_0_7_o.bin
```

In a hex editor, it is easy to see the shuffle file's contents have been reordered:

{% img left /images/binfile_fs.png %}

In this particular case the shuffle attack is unlikely to reveal any problems with an application's read code. However (as is the case with PNG) it does not take much to turn an innocuous file on its head, and perhaps cause input code to nosedive in turn.

Fuzzing as a means of security testing is as much art as it is science. Many enhancements can go into Binspector's current engine to make it more useful than it is (for example, producing files that have been attacked in several ways, not just one). Also, intelligent fuzzing itself should be augmented with more broad-spectrum tests, including more traditional fuzzing. Nevertheless, Binspector does provide a valuable subset of attacks, and makes them easily available to users.
