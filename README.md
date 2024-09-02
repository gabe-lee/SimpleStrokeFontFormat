# SimpleStrokeFontFormat
Version 0.1 technical specification for the SimpleStrokeFontFormat (.ssff)

## Table of Contents
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Terminology and Conventions](#terminology-and-conventions)
  - [Basic terms](#basic-terms)
  - [Number notation](#number-notation)
  - [Primitive types](#primitive-types)
  - [Semantic types](#semantic-types)
  - [Array types](#array-types)
  - [Structural types](#structural-types)
  - [Enumeration types](#enumeration-types)
  - [Code samples](#code-samples)
- [The unorm7/inorm7 types](#the-unorm7inorm7-types)
  - [unorm7](#unorm7)
  - [inorm7](#inorm7)
- [File Format](#file-format)
  - [SimpleStrokeFont](#simplestrokefont)
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
| SSFF, .ssff   | Shorthand for 'SimpleStrokeFontFormat' and its coresponding file extension|
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
| Type     | Size (Bytes) | Min Value   | Max Value | Byte Order    | Description |
|:---------|:------------:|------------:|----------:|---------------|:------------|
| `bool`   | 1            | 0 (false)   | 1 (true)  | N/A           | A 1-byte value representing either 'true' (1) or false (0) |
| `uint8`  | 1            | 0           | 255       | N/A           | A 1-byte unsigned integer |
| `int8`   | 1            | -128        | 127       | N/A           | A 1-byte signed integer |
| `uint16` | 2            | 0           | 65535     | Little Endian | A 2-byte unsigned integer |
| `int16`  | 2            | -32768      | 32767     | Little Endian | A 2-byte signed integer |
| `uint32` | 4            | 0           | 4294967295| Little Endian | A 4-byte unsigned integer |
| `int32`  | 4            | -2147483648 | 2147483647| Little Endian | A 4-byte signed integer |

#### Array types
| Type   | Description                                                                                                    | Example |
|--------|----------------------------------------------------------------------------------------------------------------|---------|
| `[]T`  | An array of variable length containing items of type `T`. The length of this array must be provided separately | `[]uint8`: an array of bytes in a UTF-8 string |
| `[N]T` |  An array with a *defined* static length `N` containing items of type `T`                                      | `[4]uint8`: the bytes in a single UTF-8 encoded codepoint

#### Structural types
These types wrap a set of related data fields, usually so they can be reused in multiple places. The layout of a structural type will be described using the following method with the fictional stucture as an example:

### `EmployeeRecordTable`
| Total Size                                             | File Alignment |
|:-------------------------------------------------------|----------------|
| 8 + ( [employee count] * [total size of `EmpRecord`] ) | 4              |

|                | Offset | Type                            | Allowed Values | Description |
|:--------------:|-------:|---------------------------------|:--------------:|-------------|
| Company Tag    | +0     | [`CompanyTag(uint8)`](#companytag) | (see type)     | A numeric tag indicating what company these employees work for |
| Employee Count | +4     | [`uint32`](#primitive-types)       | all            | how many employees are stored in the employee records list in this table  |
| Record List    | +8     | `[]EmpRecord`                   | (see type)     | An array of employee records stored in sorted order according to their employee id |

##### Notes:
- The 'Offest' column describes the byte offset *from the start of the structure*, NOT from the start of the file (With the exception of the [`SimpleStrokeFont`](#simplestrokefont) structure, which IS the font file itself).
- 'File Alignment' refers to the byte alignment the structure is guaranteed to be stored at *relative to file start*. This means that as long as you load the file data into a buffer that is aligned to this alignment or greater, a pointer to the structure's byte offset can be directly interpreted as a pointer to a matching data type in the programing language of your choice (dependant on programming language capabilities)

#### Enumeration types
These types are primitive numeric types with a finite list of static specification-defined allowed values. Only the *numeric value* is stored in the actual font file, but the specification provides human-readable meaning for the values to assist in code implementation (the 'Tag' column). Following from the fictional 'Employee' example above, enumeration types will be described in the following format:

### `CompanyTag`
| Base Type               | Size |
|:-----------------------:|------|
|[`uint8`](#primitive-types) | 1    |

| Tag      | Value | Info |
|:--------:|:-----:|------|
| Goggle   | 1     | search engine |
| Amazing  | 2     | web store |
| WalkMart | 3     | department store | 
| NewFlix  | 4     | media streaming |

#### Code Samples
Throughout this document will be provided code samples that show how this file format can be used. The tradition is to provide code in ANSI C, however C does not have many of the modern sensibilities that modern languages tend to follow for readability. In order to provide maximum understandability for the widest range of readers, code will be given in [Golang](https://github.com/a8m/golang-cheat-sheet), which has a highly readable syntax. Code samples will avoid using syntax shortcuts and Go-specific types, using onle the bare-bones syntax for maximum transferability to other languages.

```go
// Import the standard 'fmt' library
import "fmt"

// A constant value that holds a 4-byte integer and always equals 2024
const year uint32 = 2024

// A changeable value that holds a 2-byte integer with an initial value of 102
var day uint16 = 102

// A structural type named 'Employee'
type Employee struct {
  first_name: [20]uint8 // A field called 'first_name' that holds 20 bytes(uint8)
  last_name: [20]uint8  // A field called 'last_name' that holds 20 bytes(uint8)
  salary: float32       // A field called 'salary' that holds a 4-byte floating-point decimal
};

// A function that takes an employee, prints its data, and returns
// whether or not the employee makes at least $100000 a year (true/false)
func print_employee(employee Employee) bool {
  // print employee data to standard out
  fmt.Println("Name = ", employee.first_name, " ", employee.last_name, "\nSalary = ", employee.salary);
  // return whther employee makes at least $100000 a year
  return employee.salary >= 100000.0;
}

// Defining an instance of an Employee type
const bob = Employee{
  first_name: "Robert",
  last_name: "Robertson",
  salary: 68500.00,
}

// Application start point
func main() {
  // Call the 'print_employee' function with the predefined 'bob' employee, saving the result into a variable
  var bob_makes_six_figures = print_employee(bob)
  // Print a message to the user depending on if 'bob' makes >= 100000.00 salary
  if bob_makes_six_figures {
    fmt.Println("Bob makes a ton of money!")
  } else {
    fmt.Println("Bob makes a some money...")
  }
}
```

[Table of Contents](#table-of-contents)
***
***

# The unorm7/inorm7 types

### `unorm7`
| Type     | Base Type                   | Size (Bytes) |
|:--------:|:---------------------------:|:-------------|
| `unorm7` | [`uint8`](#primitive-types) | 1            |

SSFF uses this special semantic integer type to encode a floating-point value using 1 byte (with reduced precision). A large percentage of data in (most) font formats is usually taken up by the actual vertices of glyph geometry and other suplementary values that operate on them. Instead of storing data directly as 16 or 32 bit floating/fixed point values, this encoding represents a compromise between memory footprint and additional processing requirement before rasterization. However, as seen below, the decoding process is trivial compared to many other compression techniques

In the `unorm7` format, the integer value range `[0, 128]` is directly mapped to the floating point range `[0.0, 1.0]`, and although the semantic intent is to support this range primarily, the same transformation algorithm will map the integer range `[128, 255]` to the floating point range `[1.0, 1.9921875]`. Examples values are in the table below:

| Integer Value | Float Value | | Integer Value | Float Value | 
|--------------:|:------------|-|--------------:|:------------|
|             0 |   0.0       | |           128 |   1.0       |
|             1 |   0.0078125 | |           129 |   1.0078125 |
|             2 |   0.015625  | |           130 |   1.015625  |
|           ... |   ...       | |           ... |   ...       |
|             4 |   0.03125   | |           132 |   1.03125   |
|           ... |   ...       | |           ... |   ...       |
|             8 |   0.0625    | |           136 |   1.0625    |
|           ... |   ...       | |           ... |   ...       |
|            16 |   0.125     | |           144 |   1.125     |
|           ... |   ...       | |           ... |   ...       |
|            32 |   0.25      | |           160 |   1.25      |
|           ... |   ...       | |           ... |   ...       |
|            64 |   0.5       | |           192 |   1.5       |
|           ... |   ...       | |           ... |   ...       |
|           127 |   0.9921875 | |           255 |   1.9921875 |
|           128 |   1.0       | |               |             |
 
There are several reasons for setting the integer to float ratio at `128 == 1.0`. Primarily, it means the integer value can be directly transformed to any [IEEE 754 floating point format](https://en.wikipedia.org/wiki/IEEE_754) with trivial bitwise operations. For example, observe the bitwise layout of the numbers in integer and floating point below:
```
INTEGER BINARY RANGE (0, 128]    ==       x_xxxxxxx
FLOAT 16 BINARY RANGE (0.0, 1.0] == 0_0111x_xxxxxxx000

INTEGER BINARY EXACTLY ZERO      ==       0_0000000
FLOAT 16 BINARY EXACTLY ZERO     == 0_00000_0000000000

INTEGER BINARY RANGE (0, 128]    ==          x_xxxxxxx                
FLOAT 32 BINARY RANGE (0.0, 1.0] == 0_0111111x_xxxxxxx0000000000000000

INTEGER BINARY EXACTLY ZERO      ==          0_0000000
FLOAT 32 BINARY EXACTLY ZERO     == 0_00000000_00000000000000000000000

INTEGER BINARY RANGE (0, 128]    ==             x_xxxxxxx
FLOAT 64 BINARY RANGE (0.0, 1.0] == 0_0111111111x_xxxxxxx000000000000000000000000000000000000000000000

INTEGER BINARY EXACTLY ZERO      ==             0_0000000
FLOAT 64 BINARY EXACTLY ZERO     == 0_00000000000_0000000000000000000000000000000000000000000000000000
```

It can be observed that using the `128 == 1.0` ratio, any `unorm7` integer value ***GREATER THAN ZERO*** can be directly transformed into the desired float format by performing a bitwise left-shift (`<<` operator in most languages) and then a bitwise 'OR' (`|` operator in most languages) with a static bit mask for the floating point exponent bits.

The zero value of floating point numbers is problematic with this observation, as the zero value has a different value for its exponent bits. This can be solved either by using an 'IF' statement to return the float value `0.0` when the integer value equals `0` *OR* by using a bitwise smear of the integer value and performing a bitwise 'AND' using the smeared value on the exponent mask before applying it to the final result with the 'OR' operation.

Below are sample functions (in Golang for readability) showing how this can be done

```go
// Needed for directly bit-casting ints to floats in Golang
import "math"

const MASK_16  uint16 = 0b_0_01110_0000000000
const SHIFT_16 uint   = 3

const MASK_32  uint32 = 0b_0_01111110_00000000000000000000000
const SHIFT_32 uint   = 16

const MASK_64  uint32 = 0b_0_01111111110_0000000000000000000000000000000000000000000000000000
const SHIFT_64 uint   = 45

func unorm7_to_float32_if(val uint8) float32 {
  if val == 0 {
    return 0.0
  }
  var float_bits uint32 = MASK_32 | (uint32(val) << SHIFT_32)
  // This function directly casts the exact bits of an integer as a floating point number with the same bit-width
  return math.Float32frombits(float_bits) 
}

func unorm7_to_float32_smear(val uint8) float32 {
  var shifted_val uint32 = (uint32(val) << SHIFT_32)
  var smear_val uint32 = shifted_val
  smear_val = smear_val | (smear_val << 1)
  smear_val = smear_val | (smear_val << 2)
  smear_val = smear_val | (smear_val << 4)
  smear_val = smear_val | (smear_val << 8)
  // All bits in MASK_32 are overlapped by the bit smear now
  // If original value >= 1, all overlapping bits will be `1`
  // If original value == 0, all overlapping bits will be `0`
  // Performing an 'AND' between `MASK_32` and `smear_val` will toggle the mask
  // on or off depending on whether the original value was zero or not
  var float_bits uint32 = (MASK_32 & smear_val) | shifted_val
  // This function directly casts the exact bits of an integer as a floating point number with the same bit-width
  return math.Float32frombits(float_bits) 
}

// Functions for float16 and float64 follow the exact same pattern using their respective constants,
// with the following caveats:
//   - Golang does not have a float16 type so the algorithm is not applicable in this language, but may be valid in other languages
//   - The `unorm7_to_float64_smear` function requires one additional smear at the end of the chain with a bit-shift of `1`: `smear_val = smear_val | (smear_val << 1)`
```

Another reason for this format is that it reduces the range of expression a font author has to work with. It may sound negative, and it does limit the amount of nuance a glyph shape can be described in, but by that same token it helps reduce mistakes in font consitency and allows more opportunities to optimize a font file's layout and compression.

Finally, the ability to consider values greater than 128 as floating points greater than 1.0 can be considered a 'feature' for supporting extra-wide or extra-tall glyphs at no extra processing or storage costs.

### `inorm7`
| Type     | Base Type                   | Size (Bytes) |
|:--------:|:---------------------------:|:-------------|
| `inorm7` | [`uint8`](#primitive-types) | 1            |

This is the *signed* counterpart to `unorm7`. The integer range `[0, 127]` maps to the floating point range `[+0.0, +0.9921875]` and the integer range `[128, 255]` maps to the floating point range `[-0.0, -0.9921875]`

**NOTE** that this is *not* a a "two's complement" encoding. The uppermost bit (bit 7) is represents the sign bit, and the lower 7 bits (bits 0-6) are the magnitude. This is to match the way [IEEE 754 floating point formats](https://en.wikipedia.org/wiki/IEEE_754) operate and simplifies the decoding operation. Example values are listed below:

| Integer Value | Float Value  | | Integer Value | Float Value  | 
|--------------:|:-------------|-|--------------:|:-------------|
|             0 |   +0.0       | |           128 |   -0.0       |
|             1 |   +0.0078125 | |           129 |   -0.0078125 |
|             2 |   +0.015625  | |           130 |   -0.015625  |
|           ... |   ...        | |           ... |   ...        |
|             4 |   +0.03125   | |           132 |   -0.03125   |
|           ... |   ...        | |           ... |   ...        |
|             8 |   +0.0625    | |           136 |   -0.0625    |
|           ... |   ...        | |           ... |   ...        |
|            16 |   +0.125     | |           144 |   -0.125     |
|           ... |   ...        | |           ... |   ...        |
|            32 |   +0.25      | |           160 |   -0.25      |
|           ... |   ...        | |           ... |   ...        |
|            64 |   +0.5       | |           192 |   -0.5       |
|           ... |   ...        | |           ... |   ...        |
|           127 |   +0.9921875 | |           255 |   -0.9921875 |
 
The bitwise mapping is shown below
```
INTEGER BINARY RANGE (0, 127]            == 0_______xxxxxxx
FLOAT 16 BINARY RANGE (+0.0, +0.9921875] == 0_01110_xxxxxxx000

INTEGER BINARY EXACTLY ZERO              == 0_______0000000
FLOAT 16 BINARY EXACTLY ZERO             == 0_00000_0000000000

INTEGER BINARY RANGE (128, 255]          == 1_______xxxxxxx
FLOAT 16 BINARY RANGE (-0.0, -0.9921875] == 1_01110_xxxxxxx000

INTEGER BINARY RANGE (0, 127]            == 0__________xxxxxxx                
FLOAT 32 BINARY RANGE (+0.0, +0.9921875] == 0_01111110_xxxxxxx0000000000000000

INTEGER BINARY EXACTLY ZERO              == 0__________0000000
FLOAT 32 BINARY EXACTLY ZERO             == 0_00000000_00000000000000000000000

INTEGER BINARY RANGE (128, 255]          == 1__________xxxxxxx                
FLOAT 32 BINARY RANGE (-0.0, -0.9921875] == 1_01111110_xxxxxxx0000000000000000

INTEGER BINARY RANGE (0, 127]            == 0_____________xxxxxxx
FLOAT 64 BINARY RANGE (+0.0, +0.9921875] == 0_01111111110_xxxxxxx000000000000000000000000000000000000000000000

INTEGER BINARY EXACTLY ZERO              == 0_____________0000000
FLOAT 64 BINARY EXACTLY ZERO             == 0_00000000000_0000000000000000000000000000000000000000000000000000

INTEGER BINARY RANGE (128, 255]          == 1_____________xxxxxxx
FLOAT 64 BINARY RANGE (-0.0, -0.9921875] == 1_01111111110_xxxxxxx000000000000000000000000000000000000000000000
```
The only change to the algorithms compared to the `unorm7` versions is that the top bit of the integer is separated from the bottom 7 and placed in the topmost bit position of the floating point bit layout:

```go
// Needed for directly bit-casting ints to floats in Golang
import "math"

const SIGN_BIT_8 uint8 = 0b_10000000
const VAL_MASK_8 uint8 = 0b_01111111

const MASK_16  uint16    = 0b_0_01110_0000000000
const VAL_SHIFT_16 uint  = 3
const SIGN_SHIFT_16 uint = 5

const MASK_32  uint32    = 0b_0_01111110_00000000000000000000000
const VAL_SHIFT_32 uint  = 16
const SIGN_SHIFT_32 uint = 8

const MASK_64  uint32    = 0b_0_01111111110_0000000000000000000000000000000000000000000000000000
const VAL_SHIFT_64 uint  = 45
const SIGN_SHIFT_32 uint = 8

func inorm7_to_float32_if(val uint8) float32 {
  if val == 0 {
    return 0.0
  }
  var float_bits uint32 = uint32(val & SIGN_BIT_8) << SIGN_SHIFT_32
  float_bits = float_bits | MASK_32 | (uint32(val & VAL_MASK_8) << VAL_SHIFT_32)
  // This function directly casts the exact bits of an integer as a floating point number with the same bit-width
  return math.Float32frombits(float_bits) 
}

func inorm7_to_float32_smear(val uint8) float32 {
  var shifted_val uint32 = (uint32(val & VAL_MASK_8) << VAL_SHIFT_32)
  var smear_val uint32 = shifted_val
  smear_val = smear_val | (smear_val << 1)
  smear_val = smear_val | (smear_val << 2)
  smear_val = smear_val | (smear_val << 4)
  smear_val = smear_val | (smear_val << 8)
  // All bits in MASK_32 are overlapped by the bit smear now
  // If original value >= 1, all overlapping bits will be `1`
  // If original value == 0, all overlapping bits will be `0`
  // Performing an 'AND' between `MASK_32` and `smear_val` will toggle the mask
  // on or off depending on whether the original value was zero or not
  var float_bits uint32 = uint32(val & SIGN_BIT_8) << SIGN_SHIFT_32
  float_bits = float_bits | (MASK_32 & smear_val) | shifted_val
  // This function directly casts the exact bits of an integer as a floating point number with the same bit-width
  return math.Float32frombits(float_bits) 
}

// Functions for float16 and float64 follow the exact same pattern using their respective constants,
// with the following caveats:
//   - Golang does not have a float16 type so the algorithm is not applicable in this language, but may be valid in other languages
//   - The `unorm7_to_float64_smear` function requires one additional smear at the end of the chain with a bit-shift of `1`: `smear_val = smear_val | (smear_val << 1)`
```

The main use of this type is for describing both positive and negative offsets from an existing point, for example in kerning

# File Format
The font file in its entirety (or the portion of a data buffer the file is located at) is represented with the following structure:
### `SimpleStrokeFont`
| Total Size  | File Alignment             |
|:-----------:|:--------------------------:|
| (see below) | (recommended 4, see notes) |

|               | Offset               | Type                             | Allowed Value(s) | Description |
|:-------------:|---------------------:|:--------------------------------:|:----------------:|:------------|
| SSFF Tag      | +0                   | [`uint32`](#primitive-types)        | 0x46465353       | The UTF-8 string "SSFF" interpreted as a Little Endian uint32 |
| Total Size    | +4                   | [`uint32`](#primitive-types)        | any(uint32)         | The *total* byte length of the entire font file |
| Minor Verson  | +8                   | [`uint16`](#primitive-types)        | 0                | The minor version of SSFF this file adheres to  |
| Major Version | +10                  | [`uint8`](#primitive-types)         | 1                | The major version of SSFF this file adheres to  |
| Table Count   | +11                  | [`uint8`](#primitive-types)         | any(uint8)          | How many data tables the font contains. |
| Table Offsets | +12                  | [`[]TableOffset`](#tableoffset)  | (see type)       | An array of Tag/Offest pairs used to locate the file location of a specific data table |
| Table Data    | +(12+(TableCount*8)) | (various)                        | (n/a)            | The remainder of the font file data, organized into individual data tables |

This is the top-level structure that encompasses the entirety of the data for a distinct font.

Each entry in the [`[]TableOffset`](#tableoffset) array MUST be in sorted order according to the numeric value of its [`TableTag`](#tabletag), with all required tables coming before all optional tables. And in fact, since required tables are *required* to exist and always in sorted order, consumer code has the option to hard-code file offsets to more quickly locate data offsets of required tables, as long as the font file is known to be properly formatted according to this specificiation (see [`TableTag`](#tabletag) for details).

The "Table Data" section of the file should be treated as a byte array (`[]uint8`) with an unknown/undefined data layout until an individual data table is located within it using the [`[]TableOffset`](#tableoffset) array

It is recommended to load the font file into a buffer (or a location within a buffer) that is aligned to a byte alignment of 4. The SSFF specification requires that primitive types are always located at a correct byte alignment relative to the begining of the font file, and the largest alignment of any type used in the specification is 4. This allows the consuming code to take advantage of an optimization where a numeric value can be directly interperted from the data buffer by simply casting its memory pointer/offset as a pointer/reference to the matching type in the language (dependant on consuming language capabilities, languages that do not support pointer casts can still assemble the numeric value by shifting each byte into the type manually, and most languages include standard functions to do exactly that)

### `TableOffset`
| Total Size | File Alignment |
|:----------:|:--------------:|
| 8          | 4              |

|                | Offset | Type                        | Allowed Value(s) | Description |
|:--------------:|-------:|:---------------------------:|:----------------:| ----------- |
| Table Tag      | +0     | [`TableTag(uint8)`](#tabletag) | (see type)       | A tag indicating which table is located at the following file location |
| Table Location | +4     | [`uint32`](#primitive-types)   | any(uint32)         | Table location (byte offset) relative to the *start of the font file* |

Each TableOffset is a tag/offset pair that tells consumer code where to find a specific data table.

Required table offsets are guaranteed[*](#terminology-and-conventions) to exist and be in sorted order in a properly formed font file, allowing consumer code to 'hard-code' the location of the `TableOffset` relative to the file start for all required tables as long as the font file is known to be correctly formed (see [`TableTag`](#tabletag) for details).

### `TableTag`
| Base Type               | Size | File Alignment |
|:-----------------------:|------|----------------|
|[`uint8`](#primitive-types) | 1    | 4              |

| Tag      | Value | Required | Table Indicated                  | Guaranteed[*](#terminology-and-conventions) [`TableOffset`](#tableoffset) location from File Start|
|:--------:|:-----:|:--------:|:--------------------------------:|:-----:|
| CharMap  | 0     | Yes      | [`CharMapTable`](#charmaptable)  | +12   |
| Stroke   | 1     | Yes      | [`StrokeTable`](#stroketable)    | +20   |
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
| Char Count    | +0     | [`uint32`](#primitive-types)            | any(uint32)         | How many total unicode codepoints this font contains |
| Smallest Char | +4     | [`uint32`](#primitive-types)            | any(uint32)         | The smallest unicode codepoint this font contains |
| Largest Char  | +8     | [`uint32`](#primitive-types)            | any(uint32)         | The largest unicode codepoint this font contains |
| Segment Count | +12    | [`uint32`](#primitive-types)            | any(uint32)         | The number of contiguous segments of supported codepoints in the Character Map |
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
| First Char in Segment| +0     | [`uint32`](#primitive-types)   | any(uint32)         | The first codepoint in this contiguous segment of supported codepoints |
| Last Char in Segment | +4     | [`uint32`](#primitive-types)   | any(uint32)         | The last codepoint in this contiguous segment of supported codepoints  |
| Glyph Table Location | +8     | [`uint32`](#primitive-types)   | any(uint32)         | What index in the glyph table corresponds to the first char code in this segment |

[Table of Contents](#table-of-contents)
***

### `StrokeTable`
| Total Size                                  | File Alignment |
|:-------------------------------------------:|:--------------:|
| +8 + (Stroke Count * 4) + (Stroke Data Len) | 4              |

|                        | Offset                 | Type                          | Allowed Value(s) | Description |
|:----------------------:|-----------------------:|:-----------------------------:|:----------------:| ----------- |
| Stroke Count           | +0                     | [`uint32`](#primitive-types)  | any(uint32)      | How many total strokes are used in this font |
| Short Stroke Count     | +4                     | [`uint32`](#primitive-types)  | any(uint32)      | How many strokes are indexed in the 'Short Stroke Start List'|
| Long Stroke Count      | +8                     | [`uint32`](#primitive-types)  | any(uint32)      | How many strokes are indexed in the 'Long Stroke Start List' |
| Stroke Data Len        | +12                    | [`uint32`](#primitive-types)  | any(uint32)      | Byte length of 'Stroke Data List' |
| Short Stroke Start List| +16                    | [`[]uint16`](#primitive-types)| any(uint16)      | A list that ties a 'Stroke Index' <= 'Short Stroke Count' to its start position in the Stroke Data List. The value is the byte offset within 'Stroke Data Buffer' divided by 8, finding the real byte offset can be done |
| Stroke Data Buffer| +8 + (Stroke Count * 4)| [`[]u8`](#primitive-types)       | (see type)       | A data buffer containing the data describing each stroke. **NOTE** that the [`[]Stroke`](#stroke) structural type does not have a static size, and the Stroke Data Buffer can only be indexed using the corresponding byte start in Stroke Start List |

This table describes a list of individual strokes used in complete character glyphs. The reasoning behind separating the two is that many glyphs can (and should) re-use the same strokes (where possible and sensible), reducing the overall memory footprint by letting glyphs simply list what strokes they need and where.

SSFF does not support complex shapes for glyphs like most other font formats. Instead the SimpleStrokeFontFormat adheres to the idea that written language predating computers has always been 'dragging a drawing implement along a path' and/or 'filling in a collection of simple geometric shapes', and there is no reason a digital font needs any additional decorative nuances to be perfectly legible.

This table holds all *unique* vertices of the font. The SSFF specification limits vertex dimensions to the [`uint8`](#primitive-types) type, meaning a dimension only has 256 unique values it can possible have, for a total of a *maximum* of 65536 possible unique vertices. However, since a font will usually want visual uniformity between characters, many of the unique vertex values will be re-used throughout many of the font's glyphs. For this reason, the SSFF simply keeps track of all unique values of vertices used in the font, and the strokes that define a glyph's shape merely index into this array to find the actual vertex value.

It is entirely possible that a font author could define glyph shapes that fail to re-use common vertices, even where they ***could*** do so without affecting the actual visual result. For example, starting the bottom-left corner of the character 'A' at [0, 80] but starting the bottom-left corner of 'Ä' at [0, 78] and then using the glyph's baseline offsets to counteract the [0, -2] difference. With the exception of the 'umlauts', both characters could share all the same vertices for their main glyph shape if they were aligned in the [256, 256] grid similairly.

It is the responsibility of the font author (or the software the font author is using to create their font) to align their glyph shapes properly to take advantage of this optimisation where possible. Even in the worst-case, the additional space taken up by vertices should not push an SSFF font into a files size comparable to an eqivalent TTF/OTF font.

### `EdgeType`
| Base Type               | Size | File Alignment |
|:-----------------------:|------|----------------|
|[`uint8`](#primitive-types) | 1    | 1              |

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
| X Position | +0     | [`uint8`](#primitive-types) | any(uint8)          | Distance from the bottom-left corner of a square in the 'right' direction |
| Y Position | +1     | [`uint8`](#primitive-types) | any(uint8)          | Distance from the bottom-left corner of a square in the 'up' direction  |

[Table of Contents](#table-of-contents)
***