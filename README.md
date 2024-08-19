# SimpleStrokeFontFormat
##### Version 0.1 technical specification for the SimpleStrokeFontFormat (.ssff)

## Table of Contents

## Motivation
While attempting to re-write [stb_truetype](https://github.com/nothings/stb/blob/master/stb_truetype.h) in the Zig programming language, I came to realize how unnecessarily over-engineered TrueType and OpenType fonts were. Those formats aimed to be a 'universal' solution to rendering any possible shape for any possible writing system.

The downsides are that over the years of their specifications they have tacked on more and more data tables to handle edge cases, different operating systems, and complex writing systems. Many of the data tables have multiple sub-formats as well, and provide alternate interpretations of the same data redundantly.

This means that code supporting these formats needs to gracefully handle any/all possible configurations of the internal data, or otherwise communicate that they only support a subset of those files.

In addition, the files sizes of these fonts can become quite large when supporting a large number of unicode characters and font styles. It's understandable given the goal of TrueType and OpenType to provide a data format that will always render correctly and accurately to the font desinger's intention.

However for a significant portion of real-world use cases this level of 'perfection' is unnecessary, and oftentimes an application just needs a font format that is 'good-enough' to get the job done, and which would benefit from faster processing and smaller memory footprint.

#### Goals
- Store character shapes in a compact format with a limited range of expresion, but still plenty enough to allow for moderate stylistic creativity.
- Store character shapes in a state that is easy for the CPU/GPU to process and rasterize with minimal additional calculations.
- Support horizontal and vertical text layouts in both forward and reverse directions.
- Support simple A/B kerning given two adjacent unicode characters
- Support fonts with both small and large character sets efficiently
- Have only ONE major version of the file format: 1.x, which provides the  specifification for all *required* aspects of the font
  - Minor versions add only additional specifications for optional data in a backwards-compatible way for existing implementations of older format versions
  -  (While specification is in development version 0.x is the standin for 1.x, and changes to 0.x may not be backwards compatible)

#### Non-Goals
- Replace TrueType/OpenType as the industry standard
  - Those formats provide far more nuance than this format does and are already deepy embedded in font rendering pipelines.
- Support especially complex writing systems. Efforts *may* be made to support some of these cases, as long as the solution fits within the above goals
- Allow *complete* creative freedom when designing font glyphs/chars
- Support alternate glyph shapes for very small font sizes

## Terminology and Conventions
#### Basic terms
| Term | Explaination |
| ---- | ------------ |
| SSFF | Shorthand for 'SimpleStrokeFontFormat'|
| .ssff | File extension used by the SimpleStrokeFontFormat |
| TTF | Shorthand for 'TrueType Font' |
| .ttf | File extension useb by TrueType fonts |
| OTF | Shorthand for 'OpenType Font' |
| .otf | File extension used by OpenType fonts |
| Little Endian | Individual bytes in a numeric value are stored smallest-byte-first when reading from a buffer. For example, the hexidecimal number `0xAABBCCDD` would be  stored as `[0xDD, 0xCC, 0xBB, 0xAA]` |

#### Primitive types used in file format
| Type | Size (Bytes) | Min Value | Max Value | Byte Order |
| :--- | :----------: | --------: | --------: | ---------- |
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

#### Example description of structure layout
In many places throughout this document, the exact layout of a data structure or table will be described using the following method with the fictional stucture as an example:

| `EmployeeRecordTable` | offset | type | allowed values | description |
| :--: | ----------: | -------- | :--------: | ---------- |
| `company_tag` | +0 | `u32` | 1, 2, 3, 4 | A numeric tag indicating what company these employees work for |
| `employee_count` | +4 | `u32`  | all | how many employees are stored in the employee records list in this table  |
| `record_list` | +8 | `[]EmpRecord` | (see `EmpRecord`) | An array of employee records stored in sorted order according to their employee id |

