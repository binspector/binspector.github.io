---
layout: post
title: "Atoms and Structures"
date: 2014-10-24 16:48:59 -0700
comments: true
categories: format-grammar atoms structures
---

Binspector's format grammar centers around two fundamental data types: the _structure_ and the _atom_. While the introductory post provided an overview of these two types, in this post I will explain each in more fine-grained detail, along with a couple examples.

<!-- more -->

# Structures

Structures are the containers used to describe zero or more _fields_, which themselves are either structures or atoms. Each grammar is required to have at least one structure, and the default structure Binspector will start analysis with is `main` [^1]. Structures can be defined in any order in the format grammar, but no two may have the same name.

Here is an example of a complete, well-formed and utterly boring format grammar:

    struct main
    {
    }

In the above example we have a single structure, `main`, which is empty. Structures can refer to other structures that have already been declared, and it is through this method that relationships can be defined within a format:

    struct c_string
    {
        // empty
    }
    
    struct main
    {
        c_string first_name;
        c_string last_name;
    }

Which will result in the following structure hierarchy, where `first_name` and `last_name` are both children of `main` and siblings of one another. Binspector processes fields in the order they are defined:

{% img center /images/structures.png %}

Although we are using multiple structures to define a more complex grammar, Binspector will still process no binary data. What is missing are atoms, the second essential building block.

# Atoms

Atoms are indivisible units of binary data- hence the name. Atoms provide a means of describing the type for a collection of bits: information on how the bits are to be interpreted. The atom is described with several attributes: the _type_, _identifier_, _size_ and _offset_, each of which are detailed below.

## Type

The type for an atom is made up of three distinct parts: sign, width, and endianness. Each atom declaration requires all three.

### Sign

The sign of the atom can be one of three values: `signed`, `unsigned` or `float`, and have meaning similar to languages like C and C++. The use of `float` is restricted to atoms of 32 and 64 bits in size, and its use with other sizes will result in an error.

### Width

The width of the atom is defined in the number of raw bits that make it up. Typically the values are byte-aligned (8, 16, etc.) but do not need to be. The maximum width of a single atom is 64 bits.

### Endianness

The endianness of an atom can be one of two values: `big` or `little`. It is also possible to use an expression to represent the endianness of an atom. This is useful for some binary formats (e.g., tiff) where the reader of the binary file must discern the endianness of the data. In those situations the expression must return a boolean value. When the value is `true` the atom is `big` endian, otherwise it is `little` endian.

Though unnecessary, atoms of widths 8 or less still must specify an edian interpretation. This is a limitation of the language grammar.

#### Example

    struct main
    {
        float 32 big scale;
    }

Here we have a single atom in `main` named `scale` to be interpreted as a 32-bit floating-point value. We have just given substance to a template file, as we have instructed Binspector to consume and interpret the first 32 bits of some binary format. Atoms inserted into a structure will be analyzed based on the current read position, which advances atom by atom.

## Size

The identifier for the atom is its name. These must be unique within a structure. Additionally no field may be named `main` or `this`.

In addition to being singletons, atoms can also describe static or dynamic arrays. A field's size is an optional parameter that instruct Binspector to instantiate multiple atoms under a single field. There are several field size declarations, and they all result in contiguous arrays. The declaration types are the _integer_, _while_, _terminator_, and _delimiter_. Integer arrays will be detailed below, while the others are described in a [previous post][1]. Size descriptions are appended to the field idenitifier with brackets `[ ]`. If an expression is used to describe the size of a field, it is evaluated at the time the field is to be read.

### Integer Size

The integer field size is the most basic and is used to define a discrete array of entries. For example:

    struct main
    {
        unsigned 16 big magic_word[2];
    }

Above we have described a single field `magic_word` that interprets two 16-bit values. An array of zero elements is valid. That being said it is possible to specify the format grammar to avoid them, making output easier to read.

It is also possible to derive the length of a field based on values previously found in the binary format. An example of this is `pascal_t`, covered in the [introductory post][2].

## Offset

Binary files are not always formatted a contiguous way. Sometimes they specify offsets relative to locations in the data where other data can be found. An example of this is the tiff IFD metadata format, where values beyond a certain size are appended to the end of the metadata block and offsets are specified within the metadata itself as to where the extended information resides. Binspector allows for offsets as a means for fetching remote data and analyzing it as part of a structure.

Offsets are always specified with absolute values. For binary formats that use relative offsets, the language has primitives to assist in converting between relative and absolute offsets.

### Example

In the following example let us extend our `pascal_t` to include an extra field that specifies the remote location for the string data as an absolute offset:

    struct pascal_remote_absolute_t
    {
        unsigned 8  big length;
        unsigned 32 big absolute_offset;
        unsigned 8  big string[length] @ absolute_offset;
    }

In the example our first atom is the length of the string and the second is the remote offset. Since the value is absolute we can use it without modification in the third atom definition. Binspector will seek to that absolute offset within the file and proceed to read `length` 8-bit values to constitute `string`.

In the case we are dealing with remote offsets we need to convert them to absolute offsets before Binspector can find the data correctly. In such case we need to know which field is the basis for the relative offset, and construct an expression to use its offset to compute the final, absolute offset. In a relative-offset-based remote pascal_t structure, given that the remote offset is relative to the length byte of the string, we might have:

    struct pascal_remote_relative_t
    {
        unsigned 8  big length;
        unsigned 32 big relative_offset;
        unsigned 8  big string[length] @ startof(@length) + relative_offset;
    }

In this example we use the `startof()` routine to fetch the absolute offset of the length atom in the file and use it in conjunction with the relative offset of the string to compute the absolute offset of the string data.

Within routine calls it is necessary to refer to fields with a prefixed `@` symbol, otherwise the value of the field will be used instead of the field itself. This is loosely similar to pass-by-reference v. pass-by-value. However when the field is part of an expression the `@` should be omitted (e.g., `startof(main.foo.bar)`).

For readability's sake it would be fine to use a `const` or `invis` field to construct the absolute offset before using it:

    struct pascal_remote_relative_t
    {
        unsigned 8  big length;
        unsigned 32 big relative_offset;
        const absolute_offset = startof(@length) + relative_offset;
        unsigned 8  big string[length] @ absolute_offset;
    }

Reading remote data does not affect the read position once the reading is complete. For example given the following fields:

     unsigned 8 big offset;
     unsigned 8 big remote_data @ offset;
     unsigned 8 big some_value;

The bytes for `offset` and `some_value` are adjacent to one another in the binary file. However in the analysis `remote_data` will be between them as the remote data is brought in to its interpreted location in the file at that time. In addition an analyzed structure’s starting and ending offsets (`startof()` and `endof()` values) are only affected by its nonremote (local) data, though this may change in a later version.

[^1]: It is possible to specify a different starting structure from the command line.

[1]: /blog/2014/10/15/predicated-arrays/
[2]: /blog/2014/10/13/binspector-a-binary-format-analysis-tool/
