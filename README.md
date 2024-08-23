# SimpleStrokeFontFormat
Version 0.1 technical specification for the SimpleStrokeFontFormat (.ssff)

## Table of Contents
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Terminology and Conventions](#terminology-and-conventions)
  - [Basic terms](#basic-terms)
  - [Number notation](#number-notation)
  - [Array types](#array-types)
  - [Structural types](#structural-types)
- [File Format](#file-format)
  - [SimpleStrokeFont](#simplestrokefont)
    - [TableOffset](#tableoffset)
    - [TableTag](#tabletag)
  - [CharMapTable]


## Motivation
While attempting to re-write [stb_truetype](https://github.com/nothings/stb/blob/master/stb_truetype.h) in the Zig programming language, I came to realize how unnecessarily over-engineered TrueType and OpenType fonts were. Those formats aimed to be a 'universal' solution to rendering any possible shape for any possible writing system.

The downsides are that over the years of their specifications they have tacked on more and more data tables to handle edge cases, different operating systems, and complex writing systems. Many of the data tables have multiple sub-formats as well, and provide alternate interpretations of the same data redundantly.

This means that code supporting these formats needs to gracefully handle any/all possible configurations of the internal data, or otherwise communicate that they only support a subset of those files.

In addition, the file sizes of these fonts can become quite large when supporting a large number of unicode characters and font styles. It's understandable given the goal of TrueType and OpenType to provide a data format that will always render correctly and accurately to the font desinger's intention.

However for a significant portion of real-world use cases this level of 'perfection' is unnecessary, and oftentimes an application just needs a font format that is 'good-enough' to get the job done, and which would benefit from faster processing and smaller memory footprint.

#### Goals
- Store character shapes in a compact format with a limited range of expresion, but still plenty enough to allow for moderate stylistic creativity.
- Store character shapes in a state that is easy for the CPU/GPU to process and rasterize with minimal additional calculations.
- Support horizontal and vertical text layouts in both forward and reverse directions.
- Support simple A/B kerning given two adjacent unicode characters
- Support fonts with both small and large character sets efficiently
- Support bold and italic font styles in the same file with minimal additional data
- Store data at correct byte alignments (relative to file start) in Little Endian format
- Have only ONE major version of the file format: 1.x, which provides the  specifification for all *required* aspects of the font
  - Minor versions add only additional specifications for optional data in a backwards-compatible way for existing implementations of older format versions
  -  (While specification is in development version 0.x is the standin for 1.x, and changes to 0.x may not be backwards compatible)

#### Non-Goals
- Replace TrueType/OpenType as the industry standard
  - Those formats provide far more nuance than this format does and are already deeply embedded in font rendering pipelines.
- Support especially complex writing systems. Efforts *may* be made to support some of these cases, as long as the solution fits within the above goals
- Allow *complete* creative freedom when designing font glyphs/chars
- Support alternate glyph shapes for very small font sizes
- Support full-color emojis/icons

## Terminology and Conventions
#### Basic terms
| Term          | Explaination |
|---------------| ------------ |
| SSFF, .ssff   | Shorthand for 'SimpleStrokeFontFormat' and its coresponding file extension|
| TTF, .ttf     | Shorthand for 'TrueType Font' and its coresponding file extension |
| OTF, .otf     | Shorthand for 'OpenType Font' and its coresponding file extension |
| Little Endian | Individual bytes in a numeric value are stored smallest-byte-first when reading from a buffer. For example, the hexidecimal number `0xAABBCCDD` would be  stored as `[0xDD, 0xCC, 0xBB, 0xAA]` |
| "Guaranteed"  | The term "guaranteed" is used throughout this specification to mean that <ins>**IF THE FONT FILE IS CREATED CORRECTLY AND IN FULL COMPLIANCE WITH THIS SPECIFICIATION**</ins> then the described condition is guaranteed to be true |

#### Number notation
| Type | Value | Value as Decimal |
| ---- | -----------: | ---: |
| Decimal | 42 | 42 |
| Hexidecimal| 0x2A | 42 |
| Binary | 0b00101010 | 42 |

#### Primitive types
| Type | Size (Bytes) | Min Value | Max Value | Byte Order |
| :--- | :----------: | --------: | --------: | ---------- |
| `bool` | 1 | 0 (false) | 1 (true) | N/A |
| `u8` | 1 | 0 | 255 | N/A |
| `i8` | 1 | -128  | 127 | N/A |
| `u16` | 2 | 0 | 65535 | Little Endian |
| `i16` | 2 | -32768 | 32767 | Little Endian |
| `u32` | 4 | 0 | 4294967295 | Little Endian |
| `i32` | 4 | -2147483648 | 2147483647| Little Endian |

#### Array types
| Type | Description | Example |
| ---- | ---- | ---- |
| `[]T` | An array of variable length containing items of type `T`. The length of this array must be provided separately | `[]u8`: an array of bytes in a UTF-8 string |
| `[N]T` |  An array with a *defined* static length `N` containing items of type `T` | `[4]u8`: the bytes in a single UTF-8 encoded codepoint

#### Structural types
These types wrap a set of related data fields, usually so they can be reused in multiple places. The layout of a structural type will be described using the following method with the fictional stucture as an example:

### `EmployeeRecordTable`
| Total Size | File Alignment |
| :-------- | -------- |
| 8 + ( [employee count] * [total size of `EmpRecord`] ) | 4 |

| | Offset | Type | Allowed Values | Description |
| :--: | ----------: | -------- | :--------: | ---------- |
| company tag | +0 | [`CompanyTag(u8)`](#companytag) | (see type) | A numeric tag indicating what company these employees work for |
| employee count | +4 | [`u32`](#primitive-types)  | all | how many employees are stored in the employee records list in this table  |
| record list | +8 | `[]EmpRecord` | (see type) | An array of employee records stored in sorted order according to their employee id |

##### Notes:
- The 'Offest' column describes the byte offset *from the start of the structure*, NOT from the start of the file (With the exception of the [`SimpleStrokeFont`](#simplestrokefont) structure, which IS the font file itself).
- 'File Alignment' refers to the byte alignment the structure is guaranteed to be stored at *relative to file start*. This means that as long as you load the file data into a buffer that is aligned to this alignment or greater, a pointer to the structure's byte offset can be directly interpreted as a pointer to a matching data type in the programing language of your choice (dependant on programming language capabilities)

#### Enumeration types
These types are primitive numeric types with a finite list of static specification-defined allowed values. Only the *numeric value* is stored in the actual font file, but the specification provides human-readable meaning for the values to assist in code implementation (the 'Tag' column). Following from the fictional 'Employee' example above, enumeration types will be described in the following format:

### `CompanyTag`
| Base Type | Size |
| :-------: | ---- |
|[`u8`](#primitive-types) | 1 |

| Tag | Value | Info |
| :--: | :---------: | ---- |
| Goggle | 1 | search engine |
| Amazing | 2 | web store |
| WalkMart | 3 | department store | 
| NewFlix | 4 | media streaming |

[Table of Contents](#table-of-contents)
***
***

# File Format
The font file in its entirety (or the portion of a data buffer the file is located at) is represented with the following structure:
### `SimpleStrokeFont`
| Total Size  | File Alignment             |
|:-----------:|:--------------------------:|
| (see below) | (recommended 4, see notes) |

|               | Offset               | Type                                 | Allowed Value(s) | Description |
|:-------------:|---------------------:|:------------------------------------:|:----------------:|:------------|
| SSFF Tag      | +0                   | [`u32`](#primitive-types)            | 0x46465353       | The UTF-8 string "SSFF" interpreted as a Little Endian u32 |
| Total Size    | +4                   | [`u32`](#primitive-types)            | any(u32)         | The *total* byte length of the entire font file |
| Minor Verson  | +8                   | [`u16`](#primitive-types)            | 0                | The minor version of SSFF this file adheres to  |
| Major Version | +10                  | [`u8`](#primitive-types)             | 1                | The major version of SSFF this file adheres to  |
| Table Count   | +11                  | [`u8`](#primitive-types)             | any(u8)          | How many data tables the font contains. |
| Table Offsets | +12                  | [`[]TableOffset`](#tableoffset)  | (see type)       | An array of Tag/Offest pairs used to locate the file location of a specific data table |
| Table Data    | +(12+(TableCount*8)) | (various)                            | (n/a)            | The remainder of the font file data, organized into individual data tables |

This is the top-level structure that encompasses the entirety of the data for a distinct font.

Each entry in the [`[]TableOffset`](#tableoffset) array MUST be in sorted order according to the numeric value of its [`TableTag`](#tabletag), with all required tables coming before all optional tables. And in fact, since required tables are *required* to exist and always in sorted order, consumer code has the option to hard-code file offsets to more quickly locate data offsets of required tables, as long as the font file is known to be properly formatted according to this specificiation (see [`TableTag`](#tabletag) for details).

The "Table Data" section of the file should be treated as a byte array (`[]u8`) with an unknown/undefined data layout until an individual data table is located within it using the [`[]TableOffset`](#tableoffset) array

It is recommended to load the font file into a buffer (or a location within a buffer) that is aligned to a byte alignment of 4. The SSFF specification requires that primitive types are always located at a correct byte alignment relative to the begining of the font file, and the largest alignment of any type used in the specification is 4. This allows the consuming code to take advantage of an optimization where a numeric value can be directly interperted from the data buffer by simply casting its memory pointer/offset as a pointer/reference to the matching type in the language (dependant on consuming language capabilities, languages that do not support pointer casts can still assemble the numeric value by shifting each byte into the type manually, and most languages include standard functions to do exactly that)

[Table of Contents](#table-of-contents)
***

### `TableOffset`
| Total Size | File Alignment |
|:----------:|:--------------:|
| 8          | 4              |

|                | Offset | Type                        | Allowed Value(s) | Description |
|:--------------:|-------:|:---------------------------:|:----------------:| ----------- |
| Table Tag      | +0     | [`TableTag(u8)`](#tabletag) | (see type)       | A tag indicating which table is located at the following file location |
| Table Location | +4     | [`u32`](#primitive-types)   | any(u32)         | Table location (byte offset) relative to the *start of the font file* |

Each TableOffset is a tag/offset pair that tells consumer code where to find a specific data table.

Required table offsets are guaranteed[*](#terminology-and-conventions) to exist and be in sorted order in a properly formed font file, allowing consumer code to 'hard-code' the location of the `TableOffset` relative to the file start for all required tables as long as the font file is known to be correctly formed (see [`TableTag`](#tabletag) for details).

[Table of Contents](#table-of-contents)
***

### `TableTag`
| Base Type               | Size | File Alignment |
|:-----------------------:|------|----------------|
|[`u8`](#primitive-types) | 1    | 4              |

| Tag      | Value | Required | Table Indicated                  | Guaranteed[*](#terminology-and-conventions) [`TableOffset`](#tableoffset) location from File Start|
|:--------:|:-----:|:--------:|:--------------------------------:|:-----:|
| CharMap  | 0     | Yes      | [`CharMapTable`](#charmaptable)  | +12   |
| Vertex   | 1     | Yes      | [`VertexTable`](#vertextable)    | +20   |
| Stroke   | 2     | Yes      | [`StrokeTable`](#stroketable)    | +28   |
| Glyph    | 3     | Yes      | [`GlyphTable`](#glyphtable)      | +36   |
| Info     | 4     | No       | [`InfoTable`](#infotable)        | ---   |
| Kerning  | 5     | No       | [`KerningTable`](#kerningtable)  | ---   |
| Ligatures| 6     | No       | [`LigatureTable`](#ligaturetable)| ---   |
| Lang     | 7     | No       | [`LangTable`](#langtable)        | ---   |

An enumeration type that defines which numeric value of the "Table Tag" field in [`TableOffset`](#tableoffset) referst to which font data table. Also included are the guaranteed[*](#terminology-and-conventions) offsets relative to font file start where the [`TableOffset`](#tableoffset) of required tables can be found (in a properly formatted font file)

As a note, TTF and OTF use table tags based on human-readable ascii strings. The SSFF has been designed to offer maximal opportunities for code optimizations, in this case optimizing for the common use of a [switch-statement](https://en.wikipedia.org/wiki/Switch_statement#Compilation) to identify table tags. Although human-readable strings may be an ergonomic shortcut for developers, it imposes a slight disadvantage for compilers. 

[Table of Contents](#table-of-contents)
***

### `CharMapTable`
| Total Size | File Alignment |
|:----------:|:--------------:|
| 8          | 4              |

|                 | Offset | Type                        | Allowed Value(s) | Description |
|:---------------:|-------:|:---------------------------:|:----------------:| ----------- |
| Char Count      | +0     | [`TableTag(u8)`](#tabletag) | (see type)       | How many character codes this font contains |
| Smallest Char   | +4     | [`u32`](#primitive-types)   | any(u32)         | The smallest character code this font contains |
| Largest Char    | +8     | [`u32`](#primitive-types)   | any(u32)         | The largest character code this font contains |
| Contiguous Count| +12    | [`u32`](#primitive-types)   | any(u32)         | The number of contiguous segments of supported character codes in this |

Each TableOffset is a tag/offset pair that tells consumer code where to find a specific data table.

Required table offsets are guaranteed[*](#terminology-and-conventions) to exist and be in sorted order in a properly formed font file, allowing consumer code to 'hard-code' the location of the `TableOffset` relative to the file start for all required tables as long as the font file is known to be correctly formed (see [`TableTag`](#tabletag) for details).

[Table of Contents](#table-of-contents)
***