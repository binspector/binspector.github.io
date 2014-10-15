---
layout: post
title: "Big Trouble in Little Endian"
date: 2014-10-13 21:00:07 -0700
comments: true
categories: format-grammar
---

Exif is a body of specifications related to the embedding of metadata inside images. (Interestingly enough, when embedded inside a JPEG it uses a structure based on TIFF, making for a well-formed TIFF within a JPEG.) One of the features, for better or for worse, is that Exif can be stored either big- or little-endian. It is the responsibility of the input code to detect which mode the incoming Exif is in, and to interpret ensuing values correctly.

<!-- more -->

The start of the Exif data block will always be a 16-bit value. How it is interpreted will determine if the ensuing data is big- or little-endian:

```
struct tiff_t
{
    unsigned 16 big header; // 0x4949 (little) or 0x4d4d (big)
    ...
```

While Binspector handles endianness in atoms, the last thing I wanted to do was double-specify the format, one for each endian option. What I wanted was to be able to predicate the endianness of an atom in an _expression_, not just a keyword (`big` v. `little`), and to have that expression take file data into consideration in the process (`header == 0x4d4d`). So let it be done:

```
struct tiff_t
{
    unsigned 16 big header; // 0x4949 (little) or 0x4d4d (big)

    enumerate (header)
    {
        0x4D4D : const BE_k = true;
        0x4949 : const BE_k = false;
    }

    unsigned 16 BE_k tag_mark;

    invariant ok_tag_mark = tag_mark == 42;
}
```

The advantage of the `enumerate` construct above is that if `header` is neither of the two expected values, the analysis will fail. With the `typedef` operation, we can add a little syntactic sugar to make subsequent atoms even more readable:

```
struct tiff_t
{
    unsigned 16 big header; // 0x4949 (little) or 0x4d4d (big)

    enumerate (header)
    {
        0x4D4D : const BE_k = true;
        0x4949 : const BE_k = false;
    }

    typedef unsigned 32 BE_k long_t;
    typedef unsigned 16 BE_k word_t;

    word_t tag_mark;

    invariant ok_tag_mark = tag_mark == 42;

    long_t ifd_offset; // usually 8 for IFD 0

    // ...and on to teasing apart the IFD structure
}
```

`ifd_t` will inherit typedefs declared in its ancestry, which means it (and its substructures) can take advantage of `word_t`, `long_t`, etc. The endian handling is limited to the top-level `tiff_t` structure, and everything it contains is as readable as if endianness was never an issue in the first place.
