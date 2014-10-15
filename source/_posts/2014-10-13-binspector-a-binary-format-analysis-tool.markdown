---
layout: post
title: "Binspector: A Binary Format Analysis Tool"
date: 2014-10-13 11:17:31 -0700
comments: true
categories: general
---

Binary formats and files are inescapable. Although optimal for computers to read, sussing them manually requires exacting patience. Every developer has a moment in their career with a hex editor open, staring blankly at screenfuls of `0xDEADBEEF` or UTF-8 encoded multibyte unicode. Binspector[^1] was born when I found myself scouring JPEGs to make sure their Exif and IPTC/IIM metadata blocks were telling consistent stories. The tool has evolved into something genuinely useful, and I am excited to share it in the hopes others will benefit from it as well.

<!-- more -->

The goal of Binspector is to help bridge the gap between binary formats and the developers who wrestle with them. It does this in many ways, but this post seeks to cover the two most significant. First, Binspector leverages a domain-specific language to formally describe a binary format. Second, the tool provides interactive analysis into the contents of a binary file.

## Formal Description

Binspector seeks to formalize both the _interpretation_ and the _context_ of binary data. The interpretation of data is simply the value of its bits- its size, endianness, and sign. In Binspector these declarations are called ***atoms***. Context is the larger scope in which the data is found: now that we know what the value _is_, what does the value _mean_? The basic contextual building block in Binspector is called a ***structure***. Combined, these two attributes form the type of binary data we are dealing with.

### Example

The following is a small grammar describing two structures and the relationship between them:

```
struct pascal_t
{
  unsigned 8 big length;
  unsigned 8 big string[length];
}

struct user_name_t
{
  pascal_t first;
  pascal_t last;
}
```

It is easy to see that a `pascal_t` is a length-prefixed 8-bit string of some kind. The interpretation of the bits are described in `pascal_t`: everything here is a byte[^2]. Also, there is a relationship that is defined when `string`'s value is derived in part from `length`. `user_name_t` is comprised of two `pascal_t`s, one for the first name and one for the last. This grammar format produces an abstract syntax tree:

{% img center /images/user_name_t.png %}

It is this structured formalization (both the interpretation of binary data and the relationships between the data) that makes possible the rest of what Binspector can do. Once a format grammar has been applied to an actual binary file, this tree becomes concrete and can be analyzed.

## Analysis

Binspector attempts to interpret a binary file against a format grammar. During this process the tool builds out a tree that reports to the user what it has found. This tree can then be analyzed by a user in one of several ways, the most common of which is a command-line interface.

### Example

Let's take the above description and apply it to the following binary data, seen here in the excellent [Hex Fiend](http://ridiculousfish.com/hexfiend/):

{% img left /images/binfile.png %}

We would invoke Binspector with something like:

```
~$ binspector -i file.bin -t format.bfft -s user_name_t
```

The tool will generate a concrete syntax tree based on the abstract tree defined in the format grammar:

{% img center /images/name_processed.png %}

Binspector will drop into a command-line interface so we can navigate the parse tree. To facilitate navigation, think of structures as folders and atoms as files:

```
$main$ ls
(user_name_t) main
{
    (pascal_t) first
    (pascal_t) last
}
$main$ cd first
$main.first$ ls
(pascal_t) first
{
    (u8) length: 6
    (u8) string[6]
}
$main.first$ cd string
$main.first.string$ ls
(u8) string[6]
{
    (u8) [0]: 70
    (u8) [1]: 111
    (u8) [2]: 115
    (u8) [3]: 116
    (u8) [4]: 101
    (u8) [5]: 114
}
```

The `$`s bookend the current working branch in the analysis tree, and you can navigate into and out of structures with the `cd` command. Likewise, the `ls` command lists the contents of the current structure. As we navigate, we can get specific information about otherwise raw data. For example, to see a breakdown of a specific field we use the `detail_field` command (or just `df`). These details include the location in the file the field represents, the raw bits used for interpretation, and its value:

```
$main.first.string$ detail_field this[2]
     path: main.first.string[2]
   format: 8-bit unsigned
   offset: 3
      raw: 0x73
    value: 115 (0x73)
$main.first.string$
```

These details are confirmed by looking at the same byte in a hex editor:

{% img left /images/binfile_s.png %}

Over a dozen analytic operations are available to users from the command-line interface.

(It is important to note here that the Binspector CLI is in no way POSIX compliant. It just borrows a handful of terms from that interface to make navigating with a command line more approachable.)

All this is well and good, but it still requires the user go pretty far into the tree to see valuable data. What's needed is more meaning in a structure, the bubbling up of information to better understand what is going on. Binspector can handle that with a couple additions to the format grammar:

```
struct pascal_t
{
  unsigned 8 big length;
  unsigned 8 big string[length];

  summary str(@string);
}

struct user_name_t
{
  pascal_t first;
  pascal_t last;

  summary summaryof(first), " ", summaryof(last);
}
```

Lines 6 and 14 now contain `summary` statements which are expressions intended to summarize the contents of a structure. `user_name_t` leverages the summaries of the structures it contains with the `summaryof` command. The resulting output in the command-line interface is now far more helpful:

```
$main$ ls
(user_name_t) main (Foster Brereton)
{
    (pascal_t) first (Foster)
    (pascal_t) last (Brereton)
}
```

Though the use case may be fictional, it is easy to see the general usefulness of these capabilities for real-world formats.

# Open Source

I am excited to make Binspector available as open source software. The repository is on GitHub and includes build scripts, sources, documentation, and some format grammars (currently known as `bfft`s). I have been compiling a list of features and changes that I would like to see happen in the tool and hope the community gets involved. More importantly I would love to see a community-built corpus of format grammars developed and shared.

So please, grab a copy of the sources, kick the tires on the tool, and let me know what you think!

[^1]: Though a portmanteau of "binary inspector", the _i_ in Binspector is short (rhymes with "tin").
[^2]: The endianness specifier `big` is required even for 8-bit values; this is a limitation of the language.
