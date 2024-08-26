# SimpleShapeFontFormat
Version 0.1 technical specification for the SimpleShapeFontFormat (.ssff)

## Table of Contents
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Terminology and Conventions](#terminology-and-conventions)
  - [Basic terms](#basic-terms)
  - [Number notation](#number-notation)
  - [Array types](#array-types)
  - [Structural types](#structural-types)
  - [Enumeration types](#enumeration-types)
  - [Code samples](#code-samples)
- [File Format](#file-format)
  - [SimpleShapeFont](#simpleshapefont)
    - [TableOffset](#tableoffset)
    - [TableTag](#tabletag)
  - [CharMapTable](#charmaptable)
    - [CharMapSegment](#charmapsegment)


## Motivation
While attempting to re-write [stb_truetype](https://github.com/nothings/stb/blob/master/stb_truetype.h) in the Zig programming language, I came to realize how unnecessarily over-engineered TrueType and OpenType fonts were. Those formats aimed to be a 'universal' solution to rendering any possible shape for any possible writing system.

The downsides are that over the years their specifications have tacked on more and more data tables to handle edge cases, different operating systems, and complex writing systems. Many of the data tables have multiple sub-formats as well, and provide alternate interpretations of the same data redundantly.

This means that code supporting these formats needs to gracefully handle any/all possible configurations of the internal data, or otherwise communicate that they only support a subset of those files.

In addition, the file sizes of these fonts can become quite large when supporting a large number of unicode characters and font styles. It's understandable given the goal of TrueType and OpenType to provide a data format that will always render correctly and accurately to the font desinger's intention.

However for a significant portion of real-world use cases this level of 'perfection' is unnecessary, and oftentimes an application just needs a font format that is 'good-enough' to get the job done, and which would benefit from faster processing and smaller memory footprint.

#### Goals
- Store character shapes in a compact format with a limited range of expresion, but still plenty enough to allow for moderate stylistic creativity.
- Store character shapes in a state that is easy for the CPU/GPU to process and rasterize with minimal additional calculations.
- Support horizontal and vertical text layouts in both forward and reverse directions.
- Support class-grouped kerning given two adjacent unicode characters
- Support fonts with both small and large character sets efficiently
- Support bold and italic font styles in the same file with minimal additional data
- Store data at correct byte alignments (relative to file start) in Little Endian format
- Have only ONE major version of the file format: 1.x, which provides the  specifification for all *required* aspects of the font
  - Minor versions add only additional specifications for optional data in a backwards-compatible way for existing implementations of older format versions
  - (While specification is in development version 0.x is the standin for 1.x, and changes to 0.x may not be backwards compatible)

#### Non-Goals
- Replace TrueType/OpenType as the industry standard
  - Those formats provide far more nuance than this format does and are already deeply embedded in font rendering pipelines.
- Support especially complex writing systems. Efforts *may* be made to support some of these cases, as long as the solution fits within the above goals
- Allow *complete* creative freedom when designing font glyphs/chars
- Support alternate glyph shapes for very small font sizes
- Support full-color emojis/icons
- Provide the 'best' compression possible for a given font
  - Instead, the goal is to reduce the data required to describe the font, rather than finding clever ways to compress/encode it.
  - A simple layout allowing simple program code to access the data should be a first-class feature, not an afterthought

## Terminology and Conventions
#### Basic terms
| Term          | Explaination |
|---------------| ------------ |
| SSFF, .ssff   | Shorthand for 'SimpleShapeFontFormat' and its coresponding file extension|
| TTF, .ttf     | Shorthand for 'TrueType Font' and its coresponding file extension |
| OTF, .otf     | Shorthand for 'OpenType Font' and its coresponding file extension |
| Little Endian | Individual bytes in a numeric value are stored smallest-byte-first when reading from a buffer. For example, the hexidecimal number `0xAABBCCDD` would be  stored as `[0xDD, 0xCC, 0xBB, 0xAA]` |
| "Guaranteed"  | The term "guaranteed" is used throughout this specification to mean that <ins>**IF THE FONT FILE IS CREATED CORRECTLY AND IN FULL COMPLIANCE WITH THIS SPECIFICIATION**</ins> then the described condition is guaranteed to be true |

#### Number notation
| Type       | Value      | Value as Decimal |
|------------|-----------:|---:|
| Decimal    | 42         | 42 |
| Hexidecimal| 0x2A       | 42 |
| Binary     | 0b00101010 | 42 |

#### Primitive types
| Type   | Size (Bytes) | Min Value   | Max Value | Byte Order |
|:-------|:------------:|------------:|----------:|------------|
| `bool` | 1            | 0 (false)   | 1 (true)  | N/A |
| `u8`   | 1            | 0           | 255       | N/A |
| `i8`   | 1            | -128        | 127       | N/A |
| `u16`  | 2            | 0           | 65535     | Little Endian |
| `i16`  | 2            | -32768      | 32767     | Little Endian |
| `u32`  | 4            | 0           | 4294967295| Little Endian |
| `i32`  | 4            | -2147483648 | 2147483647| Little Endian |

#### Array types
| Type   | Description                                                                                                    | Example |
|--------|----------------------------------------------------------------------------------------------------------------|---------|
| `[]T`  | An array of variable length containing items of type `T`. The length of this array must be provided separately | `[]u8`: an array of bytes in a UTF-8 string |
| `[N]T` |  An array with a *defined* static length `N` containing items of type `T`                                      | `[4]u8`: the bytes in a single UTF-8 encoded codepoint

#### Structural types
These types wrap a set of related data fields, usually so they can be reused in multiple places. The layout of a structural type will be described using the following method with the fictional stucture as an example:

### `EmployeeRecordTable`
| Total Size                                             | File Alignment |
|:-------------------------------------------------------|----------------|
| 8 + ( [employee count] * [total size of `EmpRecord`] ) | 4              |

|                | Offset | Type                            | Allowed Values | Description |
|:--------------:|-------:|---------------------------------|:--------------:|-------------|
| Company Tag    | +0     | [`CompanyTag(u8)`](#companytag) | (see type)     | A numeric tag indicating what company these employees work for |
| Employee Count | +4     | [`u32`](#primitive-types)       | all            | how many employees are stored in the employee records list in this table  |
| Record List    | +8     | `[]EmpRecord`                   | (see type)     | An array of employee records stored in sorted order according to their employee id |

##### Notes:
- The 'Offest' column describes the byte offset *from the start of the structure*, NOT from the start of the file (With the exception of the [`SimpleShapeFont`](#simpleshapefont) structure, which IS the font file itself).
- 'File Alignment' refers to the byte alignment the structure is guaranteed to be stored at *relative to file start*. This means that as long as you load the file data into a buffer that is aligned to this alignment or greater, a pointer to the structure's byte offset can be directly interpreted as a pointer to a matching data type in the programing language of your choice (dependant on programming language capabilities)

#### Enumeration types
These types are primitive numeric types with a finite list of static specification-defined allowed values. Only the *numeric value* is stored in the actual font file, but the specification provides human-readable meaning for the values to assist in code implementation (the 'Tag' column). Following from the fictional 'Employee' example above, enumeration types will be described in the following format:

### `CompanyTag`
| Base Type               | Size |
|:-----------------------:|------|
|[`u8`](#primitive-types) | 1    |

| Tag      | Value | Info |
|:--------:|:-----:|------|
| Goggle   | 1     | search engine |
| Amazing  | 2     | web store |
| WalkMart | 3     | department store | 
| NewFlix  | 4     | media streaming |

#### Code Samples
Throughout this document will be provided code samples that show how this file format can be used. They will usually be written in Zig, as that is the language I (the author) am more comfortable with, has generally all the same features as C, and I find it easier to read and translate into other languages for people that don't know either C or Zig. C examples *may* be provied as well on a case-by-case basis to provide clarity where a special consideration must be made for C.

Below is an identical code sample in both Zig and C for context.
##### Zig
```zig
// import the standard library using the name 'std'
const std = @include('std');

// A structural type named 'Employee'
const Employee = struct {
  // A field called 'first_name' that holds 20 bytes(u8)
  first_name: [20]u8,
  // A field called 'last_name' that holds 20 bytes(u8)
  last_name: [20]u8,
  // A field called 'salary' that holds a floating-point decimal
  salary: f32,
};

// A function that takes an employee, prints its data, and returns
// whether or not the employee makes at least $100000 a year (true/false)
fn print_employee(Employee employee) bool {
  // print employee data to standard out
  std.debug.print("Name = {s} {s}\nSalary = {f}", .{employee.first_name, employee.last_name, employee.salary});
  // return whther employee makes at least $100000 a year
  return employee.salary >= 100000.0;
}
```
##### C

```c
// Use the C Standard Input/Output library
#include <stdio.h>

// A structural type named 'Employee'
struct Employee {
  // A field called 'first_name' that holds 20 bytes(chars)
  char first_name[20];
  // A field called 'last_name' that holds 20 bytes(chars)
  char last_name[20];
  // A field called 'salary' that holds a floating-point decimal
  float salary;
}

// A function that takes an employee, prints its data, and returns
// whether or not the employee makes at least $100000 a year (true/false)
bool print_employee(Employee employee) {
  // print employee data to standard out
  printf("Name = %s %s\nSalary = %f", employee.first_name, employee.last_name, employee.salary);
  // return whther employee makes at least $100000 a year
  return employee.salary >= 100000.0;
}
```

[Table of Contents](#table-of-contents)
***
***

# File Format
The font file in its entirety (or the portion of a data buffer the file is located at) is represented with the following structure:
### `SimpleShapeFont`
| Total Size  | File Alignment             |
|:-----------:|:--------------------------:|
| (see below) | (recommended 4, see notes) |

|               | Offset               | Type                             | Allowed Value(s) | Description |
|:-------------:|---------------------:|:--------------------------------:|:----------------:|:------------|
| SSFF Tag      | +0                   | [`u32`](#primitive-types)        | 0x46465353       | The UTF-8 string "SSFF" interpreted as a Little Endian u32 |
| Total Size    | +4                   | [`u32`](#primitive-types)        | any(u32)         | The *total* byte length of the entire font file |
| Minor Verson  | +8                   | [`u16`](#primitive-types)        | 0                | The minor version of SSFF this file adheres to  |
| Major Version | +10                  | [`u8`](#primitive-types)         | 1                | The major version of SSFF this file adheres to  |
| Table Count   | +11                  | [`u8`](#primitive-types)         | any(u8)          | How many data tables the font contains. |
| Table Offsets | +12                  | [`[]TableOffset`](#tableoffset)  | (see type)       | An array of Tag/Offest pairs used to locate the file location of a specific data table |
| Table Data    | +(12+(TableCount*8)) | (various)                        | (n/a)            | The remainder of the font file data, organized into individual data tables |

This is the top-level structure that encompasses the entirety of the data for a distinct font.

Each entry in the [`[]TableOffset`](#tableoffset) array MUST be in sorted order according to the numeric value of its [`TableTag`](#tabletag), with all required tables coming before all optional tables. And in fact, since required tables are *required* to exist and always in sorted order, consumer code has the option to hard-code file offsets to more quickly locate data offsets of required tables, as long as the font file is known to be properly formatted according to this specificiation (see [`TableTag`](#tabletag) for details).

The "Table Data" section of the file should be treated as a byte array (`[]u8`) with an unknown/undefined data layout until an individual data table is located within it using the [`[]TableOffset`](#tableoffset) array

It is recommended to load the font file into a buffer (or a location within a buffer) that is aligned to a byte alignment of 4. The SSFF specification requires that primitive types are always located at a correct byte alignment relative to the begining of the font file, and the largest alignment of any type used in the specification is 4. This allows the consuming code to take advantage of an optimization where a numeric value can be directly interperted from the data buffer by simply casting its memory pointer/offset as a pointer/reference to the matching type in the language (dependant on consuming language capabilities, languages that do not support pointer casts can still assemble the numeric value by shifting each byte into the type manually, and most languages include standard functions to do exactly that)

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

### `TableTag`
| Base Type               | Size | File Alignment |
|:-----------------------:|------|----------------|
|[`u8`](#primitive-types) | 1    | 4              |

| Tag      | Value | Required | Table Indicated                  | Guaranteed[*](#terminology-and-conventions) [`TableOffset`](#tableoffset) location from File Start|
|:--------:|:-----:|:--------:|:--------------------------------:|:-----:|
| CharMap  | 0     | Yes      | [`CharMapTable`](#charmaptable)  | +12   |
| Shape    | 1     | Yes      | [`ShapeTable`](#shapetable)      | +20   |
| Glyph    | 2     | Yes      | [`GlyphTable`](#glyphtable)      | +28   |
| Metrics  | 3     | Yes      | [`MetricsTable`](#metricstable)  | +36   |
| Kerning  | 4     | No       | [`KerningTable`](#kerningtable)  | ---   |
| Ligatures| 5     | No       | [`LigatureTable`](#ligaturetable)| ---   |
| Lang     | 6     | No       | [`LangTable`](#langtable)        | ---   |

An enumeration type that defines which numeric value of the "Table Tag" field in [`TableOffset`](#tableoffset) refers to which font data table. Also included are the guaranteed[*](#terminology-and-conventions) offsets relative to font file start where the [`TableOffset`](#tableoffset) of required tables can be found (in a properly formatted font file)

As a note, TTF and OTF use table tags based on human-readable ascii strings. The SSFF has been designed to offer maximal opportunities for code optimizations, in this case optimizing for the common use of a [switch-statement](https://en.wikipedia.org/wiki/Switch_statement#Compilation) to identify table tags. Although human-readable strings may be an ergonomic shortcut for developers, it imposes a slight disadvantage for compilers. 

[Table of Contents](#table-of-contents)
***

### `CharMapTable`
| Total Size                | File Alignment |
|:-------------------------:|:--------------:|
| 16 + (Segment Count * 12) | 4              |

|               | Offset | Type                                 | Allowed Value(s) | Description |
|:-------------:|-------:|:------------------------------------:|:----------------:| ----------- |
| Char Count    | +0     | [`u32`](#primitive-types)            | any(u32)         | How many total unicode codepoints this font contains |
| Smallest Char | +4     | [`u32`](#primitive-types)            | any(u32)         | The smallest unicode codepoint this font contains |
| Largest Char  | +8     | [`u32`](#primitive-types)            | any(u32)         | The largest unicode codepoint this font contains |
| Segment Count | +12    | [`u32`](#primitive-types)            | any(u32)         | The number of contiguous segments of supported codepoints in the Character Map |
| Segment List  | +16    | [`[]CharMapSegment`](#charmapsegment)| (see type)       | A list of structs that each describe a segment of contiguous unicode codepoints supported by this font, and an index that shows where the segment of codepoints begins in the [`GlyphTable`](#glyphtable) glyph list |

This table is essentially the 'key' to accessing the rest of the data in the font. It maps a set of supported [unicode codepoints](https://en.wikipedia.org/wiki/Unicode#Codespace_and_code_points) to the glyph that needs to be drawn to represent them. Because the set of supported codepoints may not be entirely contiguous (meaning there are 'holes' in the codepoint covereage), each *contiguous* range of supported codepoints is given its own [`CharMapSegment`](#charmapsegment) that describes the range of codepoints the segment covers, and what index in the [`GlyphTable`](#glyphtable) refers to the first char code in that segment.

All contiguous codepoints within a segment are guaranteed[*](#terminology-and-conventions) to be contiguous glyph indexes starting from the first. For example, if CharSegment [1000 -> 1500] starts at Glyph Index 400, then glyphs [400, 401, 402, 403, ... 900] represent codepoints [1000, 1001, 1002, 1003, ... 1500].

The SSFF format adheres to the convention that all codepoints are **unicode** codepoints, and font authors should make sure the codepoints they support have glyphs that match the semantic expectations of those codepoints. Strictly speaking, however, this specification makes no checks for codepoint correctness, and font authors may add extra codepoints outside the range of what unicode specifies for things unicode doesn't cover, like custom icons.

Glyph Index 0 is reserved for a glyph representing a codepoint that is not supported by the font, and generally speaking the correct action to take if the user requests such a glyph is to return glyph index 0 instead, but that is up to the specific code implementation in question.

### `CharMapSegment`
| Total Size | File Alignment |
|:----------:|:--------------:|
| 12          | 4             |

|                      | Offset | Type                        | Allowed Value(s) | Description |
|:--------------------:|-------:|:---------------------------:|:----------------:| ----------- |
| First Char in Segment| +0     | [`u32`](#primitive-types)   | any(u32)         | The first codepoint in this contiguous segment of supported codepoints |
| Last Char in Segment | +4     | [`u32`](#primitive-types)   | any(u32)         | The last codepoint in this contiguous segment of supported codepoints  |
| Glyph Table Location | +8     | [`u32`](#primitive-types)   | any(u32)         | What index in the glyph table corresponds to the first char code in this segment |

[Table of Contents](#table-of-contents)
***

### `ShapeTable`
| Total Size                                  | File Alignment |
|:-------------------------------------------:|:--------------:|
| +8 + (Shape Count * 6) + (Vertex Count * 2) | 4              |

|                   | Offset                 | Type                       | Allowed Value(s) | Description |
|:-----------------:|-----------------------:|:--------------------------:|:----------------:| ----------- |
| Shape Count       | +0                     | [`u32`](#primitive-types)  | any(u32)         | How many shapes are used in this font (length of Shape Start List and Shape Type List) |
| Vertex Count      | +4                     | [`u32`](#primitive-types)  | any(u32)         | How many vertices are used in this font (length of Shape Vertex List) |
| Shape Start List  | +8                     | [`[]u32`](#primitive-types)| any(u32)         | A list that ties a shape index to its start position in the Stroke Vertex List |
| Shape Type List   | +8 + (Stroke Count * 4)| [`[]ShapeType`](#shapetype)| (see type)       | A list that describes the shape type for a given |
| Shape Vertex List | +8 + (Stroke Count * 6)| [`[]Vertex`](#vertex)      | (see type)       | A list that holds all shape vertices |

This table describes a list of individual shapes used in complete character glyphs. The reasoning behind separating the two is that many glyphs can (and should) re-use the same shape, reducing the overall memory footprint by letting glyphs simply list what shapes they need and where.

Every shape has a 

This table holds all *unique* vertices of the font. The SSFF specification limits vertex dimensions to the [`u8`](#primitive-types) type, meaning a dimension only has 256 unique values it can possible have, for a total of a *maximum* of 65536 possible unique vertices. However, since a font will usually want visual uniformity between characters, many of the unique vertex values will be re-used throughout many of the font's glyphs. For this reason, the SSFF simply keeps track of all unique values of vertices used in the font, and the strokes that define a glyph's shape merely index into this array to find the actual vertex value.

It is entirely possible that a font author could define glyph shapes that fail to re-use common vertices, even where they ***could*** do so without affecting the actual visual result. For example, starting the bottom-left corner of the character 'A' at [0, 80] but starting the bottom-left corner of 'Ä' at [0, 78] and then using the glyph's baseline offsets to counteract the [0, -2] difference. With the exception of the 'umlauts', both characters could share all the same vertices for their main glyph shape if they were aligned in the [256, 256] grid similairly.

It is the responsibility of the font author (or the software the font author is using to create their font) to align their glyph shapes properly to take advantage of this optimisation where possible. Even in the worst-case, the additional space taken up by vertices should not push an SSFF font into a files size comparable to an eqivalent TTF/OTF font.

### `EdgeType`
| Base Type               | Size | File Alignment |
|:-----------------------:|------|----------------|
|[`u8`](#primitive-types) | 1    | 1              |

| Tag                | Value | Vertices Used | Description |
|:-------------------|:-----:|:-------------:|:------------|
| Line               | 0     | 2             | Line using 2 new vertices as the [start, end] |
| LineContinue       | 1     | 1             | Line using 1 previous vertex as [start] and 1 new vertex as [end] |
| LineCloseLoop      | 2     | 0             | Line using 1 previous vertex as [start] and the first vertex in set as [end] |
| QuadBeizer         | 3     | 3             | Quadratic beizer using 3 new vertices as [start, control, end] |
| QuadBeizerContinue | 4     | 2             | Quadratic beizer using 1 previous vertex as [start], and 2 new vertices as [control, end] |
| QuadBeizerCloseLoop| 5     | 1             | Quadratic beizer using 1 previous vertex as [start], 1 new vertex as [control], and the first vertex in set as [end] |
| CubeBeizer         | 6     | 4             | Cubic beizer using 4 new vertices as [start, control 1, control 2, end] |
| CubeBeizerContinue | 7     | 3             | Cubic beizer using 1 previous vertex as [start], and 3 new vertices as [control 1, control 2, end] |
| CubeBeizerCloseLoop| 8     | 2             | Cubic beizer using 1 previous vertex as [start], 2 new vertices as [control 1, control 2], and the first vertex in set as [end] |
| CircleRadius       | 9     | 1             | The radius of a circle, using 1 vertex as [[Radius, (unused)]]. This edge MUST be a 'Left' edge, and MUST be paired with a Point 'Right' edge |
| ElipseRadius       | 10    | 1             | The radii of an axis-aligned elipse, using 1 vertex as [[Radius X, Radius Y]]. This edge MUST be a 'Left' edge, and MUST be paired with a Point 'Right' edge |
| CircleRingRadius   | 11    | 1             | The radii of a circular ring, using 1 vertex [[Radius Outer, Radius Inner]]. This edge MUST be a 'Left' edge, and MUST be paired with a Point 'Right' edge |
| ElipseRingRadius   | 12    | 2             | The radii of an axis-aligned eliptic ring, using 2 vertices as [[Radius Outer X, Radius Inner X], [Radius Outer Y, Radius Inner Y]]. This edge MUST be a 'Left' edge, and MUST be paired with a Point 'Right' edge |
| Point              | 13    | 1             | A singular point that requires no linear interpolation (all interpolated points would just equal the original point) |

### `Vertex`
| Total Size | File Alignment |
|:----------:|:--------------:|
| 2          | 2              |

|            | Offset | Type                     | Allowed Value(s) | Description |
|:----------:|-------:|:------------------------:|:----------------:| ----------- |
| X Position | +0     | [`u8`](#primitive-types) | any(u8)          | Distance from the bottom-left corner of a square in the 'right' direction |
| Y Position | +1     | [`u8`](#primitive-types) | any(u8)          | Distance from the bottom-left corner of a square in the 'up' direction  |

[Table of Contents](#table-of-contents)
***