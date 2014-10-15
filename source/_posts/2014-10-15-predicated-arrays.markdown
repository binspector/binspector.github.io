---
layout: post
title: "Predicated Arrays"
date: 2014-10-15 10:20:35 -0700
comments: true
categories: format-grammar arrays
---

It is normal for formats to specify an array of data that is not length-prefixed. Instead, the format usually mandates some kind of marker to signify the end of the array. The canonical example of this is a null-terminated C string. We call these arrays _predicated_ because there is a check- a predicate- that when met denotes the end of the array. In this post we'll take a look at some of the tools available within Binspector to handle these kinds of arrays.

<!-- more -->

## Terminators

If an atom specifies a terminator for its array size, Binspector will continue to grow the array until the terminator is found. The terminator is then included in the array. Using the above example, a null-terminated string would take this form:

```
unsigned 8 big string[terminator: 0];
```

One might visualize Binspector's process like so:

{% img center /images/terminator.png %}

There are a handful of restrictions to the use of terminators. First, terminators are always integer values, and cannot be used for structure arrays. Also, because the terminator is appended at the end of the resulting array, it must be the same type as the atom it terminates.

## Delimiters

An atom with a size delimiter is very similar to one with a terminator, however there are two distinctions between them. The first is that the delimited value is _not_ included in the atom's resulting array. As a consequence of the this the second difference is that the delimiter's type need not be the same as the array it delimits.

The most common use case for a delimited atom declaration is when Binspector should skip over some uninteresting portion of a binary file until a sought-after piece is found. For example in a JPEG grammar one might want to skip over the image data stream, requiring a delimiter field until the end of image marker is found:

```
struct main
{
    //... prior JPEG template file declaration
    unsigned 8  big image_stream[delimiter: 0xFFD9];

    unsigned 16 big eoi_marker; // will be 0xFFD9
}
```

Here we have an 8-bit array filled with image stream data that the format grammar is uninterested in. The delimiter is a 16-bit value, which is fine because the value will not be included in `image_stream`. As noted in the example the following 16-bit value will be `0xFFD9`.

The bit length of the delimiter is deduced from its value rounded to a byte. Like terminators, delimiters cannot be applied to structures (_note: why not?_). Finally, delimiters are far more efficient than `peek` when trying to do look-ahead processing.

## While

The final predicate that can be applied to arrays is the `while` statement, which includes an expression that is evaluated after every element of the array is read. `while` statements _can_ be used for the size of a structure array, so there can be many bytes read between subsequent evaluations of the `while` predicate. The most common use case for `while` predicates is via slots and signals:

```
struct main
{
  slot done = false;

  other_structure_t my_array[while: !done];
}
```

Binspector is depending on something inside `other_structure_t` to modify the `done` slot via a signal. If the signal never happens, the array will expand until the end of the binary is found, and should be considered a bug in the grammar.
