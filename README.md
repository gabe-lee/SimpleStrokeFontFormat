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
  - Those formats provide far more nuance than this format does and are already deepy embedded in font rendering pipelines.
- Support especially complex writing systems. Efforts *may* be made to support some of these cases, as long as the solution fits within the above goals
- Allow *complete* creative freedom when designing font glyphs/chars
- Support alternate glyph shapes for very small font sizes
- Support full-color emojis/icons

## Terminology and Conventions
#### Basic terms
| Term | Explaination |
| ---- | ------------ |
| SSFF, .ssff | Shorthand for 'SimpleStrokeFontFormat' and its coresponding file extension|
| TTF, .ttf | Shorthand for 'TrueType Font' and its coresponding file extension |
| OTF, .otf | Shorthand for 'OpenType Font' and its coresponding file extension |
| Little Endian | Individual bytes in a numeric value are stored smallest-byte-first when reading from a buffer. For example, the hexidecimal number `0xAABBCCDD` would be  stored as `[0xDD, 0xCC, 0xBB, 0xAA]` |

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
|8 + ( [employee count] * [total size of `EmpRecord`] ) | 4 |

| | Offset | Type | Allowed Values | Description |
| :--: | ----------: | -------- | :--------: | ---------- |
| company tag | +0 | `u32` | 1, 2, 3, 4 | A numeric tag indicating what company these employees work for |
| employee count | +4 | `u32`  | all | how many employees are stored in the employee records list in this table  |
| record list | +8 | `[]EmpRecord` | (see `EmpRecord`) | An array of employee records stored in sorted order according to their employee id |

##### Notes:
- The 'Offest' column describes the byte offset *from the start of the structure*, NOT from the start of the file.
- 'File Alignment' refers to the byte alignment the structure is guaranteed to be stored at *relative to file start*. This means that as long as you load the file data into a buffer that is aligned to this alignment or greater, a pointer to the structure's byte offset can be directly interpreted as a pointer to a matching data type in the programing language of your choice (dependant on programming language capabilities)


## File Format
The font file in its entirety (or the portion of a data buffer the file is located at) is represented with the following structure:
### `SimpleStrokeFont`
| Total Size | File Alignment |
| :-------- | -------- |
|8 + ( [employee count] * [total size of `EmpRecord`] ) | 4 |

| | Offset | Type | Allowed Values | Description |
| :--: | ----------: | -------- | :--------: | ---------- |
| SSFF tag | +0 | [`u32`](#primitive-types) | 1179013971 | The UTF-8 string "SSFF" interpreted as a Little Endian u32 |
| Minor Verson | +4 | [`u16`](#primitive-types) | highest minor version |The minor version of SSFF this file adheres to (signals what additional features *may* be present) |
| Major Version | +6 | [`u8`](#primitive-types)  | 1 | The major version of SSFF this file adheres to  |

