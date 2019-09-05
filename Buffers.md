# System.Buffers

- **Writing**
    - [IBufferWriter\<T\>](#ibufferwritert)
- **Reading**
    - [ReadOnlySequence\<T\>](#readonlysequencet)
       - [Accessing data](#)
       - [Gotchas](#)
    - [SequenceReader\<T\>](#sequencereadert)
       - [Accessing data](#)
       - [Gotchas](#)
   
## [IBufferWriter\<T\>](https://docs.microsoft.com/en-us/dotnet/api/system.buffers.ibufferwriter-1?view=netstandard-2.1)

`IBufferWriter<T>` is a new type for synchronous memory managed buffered writing. At the lowest level the interface is extremely simple and allows you to get access to a `Memory<T>` or `Span<T>`, write to it and say how many of those bytes were written.

```C#
void WriteHello(IBufferWriter<byte> writer)
{
    // Request at least 5 bytes
    Span<byte> span = writer.GetSpan(5);
    Span<char> helloSpan = "Hello".AsSpan();
    int written = Encoding.ASCII.GetBytes(helloSpan, span);
    
    // Tell the writer how many bytes we wrote
    writer.Advance(written);
}
```

The above method requests at least 5 bytes from the `IBufferWriter<byte>` using `GetSpan(5)` then writes the ASCII string "Hello" the `Span<byte>` returned. It then calls `Advance(written)` to indicate how many bytes were written.

### Gotchas

## [ReadOnlySequence\<T\>](https://docs.microsoft.com/en-us/dotnet/api/system.buffers.readonlysequence-1)
- `GetSpan`/`GetMemory` returns a buffer with at least the requested amount of memory. Don't assume exact buffer sizes.

![image](https://user-images.githubusercontent.com/95136/64049773-cbd04600-cb2a-11e9-9d37-404488a2d6d5.png)

`ReadOnlySequence<T>` is a struct that can represent a contiguous or discontiguous sequence of T. It can be constructed from:
1. A `T[]`
1. A `ReadOnlyMemory<T>`
1. A pair of linked list node [`ReadOnlySequenceSegment<T>`](https://docs.microsoft.com/en-us/dotnet/api/system.buffers.readonlysequencesegment-1?view=netstandard-2.1) and index to represent the start and end position of the sequence

The 3rd representation is the most interesting one as it has performance implications on various operations on the `ReadOnlySequence<T>`:

|Representation|Operation|Complexity|
---|---|---
|`T[]`/`ReadOnlyMemory<T>`|`Length`|`O(1)`
|`T[]`/`ReadOnlyMemory<T>`|`GetPosition(long)`|`O(1)`
|`T[]`/`ReadOnlyMemory<T>`|`Slice(int, int)`|`O(1)`
|`T[]`/`ReadOnlyMemory<T>`|`Slice(SequencePostion,  SequencePostion)`|`O(1)`
|`ReadOnlySequenceSegment<T>`|`Length`|`O(1)`
|`ReadOnlySequenceSegment<T>`|`GetPosition(long)`|`O(number of segments)`
|`ReadOnlySequenceSegment<T>`|`Slice(int, int)`|`O(number of segments)`
|`ReadOnlySequenceSegment<T>`|`Slice(SequencePostion, SequencePostion)`|`O(1)`

Because of this mixed representation, the `ReadOnlySequence<T>` exposes indexes as `SequencePosition` instead of an integer. A `SequencePosition` is an opaque value that represents an index into the `ReadOnlySequence<T>` where it originated. It consists of 2 parts, a integer and an object, what these 2 values represent are tied to the implementation of `ReadOnlySequence<T>`.

### Accessing data

The `ReadOnlySequence<T>` exposes data as a enumerable of `ReadOnlyMemory<T>`. Enumerating each of the segments can be done using a simple foreach:

```C#
long FindIndexOf(in ReadOnlySequence<byte> buffer, byte data)
{
    long position = 0;
    
    foreach (ReadOnlyMemory<byte> segment in buffer)
    {
        ReadOnlySpan<byte> span = segment.Span;
        var index = span.IndexOf(data);
        if (index != -1)
        {
            return position + index;
        }
        
        position += span.Length;
    }
    
    return -1;
}
```

The above method searches each segment for a specific byte. If we need to keep track of each segment's `SequencePosition` then [`ReadOnlySequence.TryGet`](https://docs.microsoft.com/en-us/dotnet/api/system.buffers.readonlysequence-1.tryget?view=netcore-3.0) would be more appropriate. Lets change the above code to return a `SequencePosition` instead of an integer. This has the added benefit of making the caller avoid a second scan to get the data a specific index.

```C#
SequencePosition? FindIndexOf(in ReadOnlySequence<byte> buffer, byte data)
{
    SequencePosition position = buffer.Start;
    
    while (buffer.TryGet(ref position, out ReadOnlyMemory<byte> segment))
    {
        ReadOnlySpan<byte> span = segment.Span;
        var index = span.IndexOf(data);
        if (index != -1)
        {
            return buffer.GetPosition(position, index);
        }
    }
    return null;
}
```

The combination of `SequencePosition` and `TryGet` act like an enumerator. The position field is modified at the start of each iteration to be start of each segment within the `ReadOnlySequence<T>`. 

The above method exists as a extension method on `ReadOnlySequence<T>`. We can use [PositionOf](https://docs.microsoft.com/en-us/dotnet/api/system.buffers.buffersextensions.positionof?view=netstandard-2.1) to simplify the above code:

```C#
SequencePosition? FindIndexOf(in ReadOnlySequence<byte> buffer, byte data) => buffer.PositionOf(data);
```

#### Processing a ReadOnlySequence\<T\> 

Processing a `ReadOnlySequence<T>` can be challenging because data can be split across multiple segments within the sequence. For the best performance split code in to a fast path that deals with the single segment case and a slow path deals with the data split across segments. There are a couple of ways of parsing data in multi-segmented sequences:

- Use the [`SequenceReader<T>`](#sequencereadert)
- Parse data segment by segment, keeping track of the `SequencePosition` and index within the segment parsed. this will avoid unnecessary allocations but may be inefficient (especially for small buffers).
- Copy the `ReadOnlySequence<T>` to a contiguous array and treat it like a single buffer:
  - If the `ReadOnlySequence<T>` has a length less then 256 it may be reasonable to copy the data into a stack allocated buffers (using the `stackalloc` keyword).
  - Copy the `ReadOnlySequence<T>` into a pooled array using `ArrayPool<T>.Shared`.
  - Use `ReadOnlySequence<T>.ToArray()`. This will allocate a new `T[]` on the heap so it's not recommended in hot paths.

The below examples show a couple of commons cases for parsing `ReadOnlySequence<byte>`:

##### Processing Binary Data

This example parses a 4 byte big-endian integer length from the start of the `ReadOnlySequence<byte>`.

```C#
bool TryParseHeaderLength(ref ReadOnlySequence<byte> buffer, out int length)
{
    // If we don't have enough space, we can't get the length
    if (buffer.Length < 4)
    {
        length = 0;
        return false;
    }
    
    // Grab the first 4 bytes of the buffer
    var lengthSlice = buffer.Slice(buffer.Start, 4);
    if (lengthSlice.IsSingleSegment)
    {
        // Fast path since it's a single segment
        length = BinaryPrimitives.ReadInt32BigEndian(lengthSlice.First.Span);
    }
    else
    {
        // We have 4 bytes split across multiple segments, since it's so small we can copy it to a
        // stack allocated buffer, this avoids a heap allocation.
        Span<byte> stackBuffer = stackalloc byte[4];
        lengthSlice.CopyTo(stackBuffer);
        length = BinaryPrimitives.ReadInt32BigEndian(stackBuffer);
    }
    
    // Move the buffer 4 bytes ahead
    buffer = buffer.Slice(lengthSlice.End);
    
    return true;
}
```

##### Processing Text Data

This example finds the first newline (`\r\n`) in the `ReadOnlySequence<byte>` and returns it via the out line parameter. It also trims that line (excluding the `\r\n` from the input buffer).

```C#
static bool TryParseLine(ref ReadOnlySequence<byte> buffer, out ReadOnlySequence<byte> line)
{
    SequencePosition position = buffer.Start;
    SequencePosition previous = position;
    var index = -1;
    line = default;

    while (buffer.TryGet(ref position, out ReadOnlyMemory<byte> segment))
    {
        ReadOnlySpan<byte> span = segment.Span;

        // Look for \r in the current segment
        index = span.IndexOf((byte)'\r');

        if (index != -1)
        {
            // Check next segment for \n
            if (index + 1 >= span.Length)
            {
                var next = position;
                if (!buffer.TryGet(ref next, out ReadOnlyMemory<byte> nextSegment))
                {
                    // We're at the end of the sequence
                    return false;
                }
                else if (nextSegment.Span[0] == (byte)'\n')
                {
                    //  We found a match
                    break;
                }
            }
            // Check the current segment of \n
            else if (span[index + 1] == (byte)'\n')
            {
                // Found it
                break;
            }
        }

        previous = position;
    }

    if (index != -1)
    {
        // Get the position just before the \r\n
        var delimeter = buffer.GetPosition(index, previous);
        
        // Slice the line (excluding \r\n)
        line = buffer.Slice(buffer.Start, delimeter);
        
        // Slice the buffer to get the renamining data after the line
        buffer = buffer.Slice(buffer.GetPosition(2, delimeter));
        return true;
    }

    return false;
}
```

### Gotchas

There are a couple of quirks when dealing with a `ReadOnlySequence<T>/SequencePosition` vs a normal `ReadOnlySpan<T>/ReadOnlyMemory<T>/T[]/int`:
- `SequencePosition` is a position marker for a specific `ReadOnlySequence<T>`, not an absolute position. As it is relative to a specfic `ReadOnlySequence<T>`, it doesn't have meaning if used outside of the `ReadOnlySequence<T>` where it originated.
- Arithmetic cannot be performed on `SequencePosition` without the `ReadOnlySequence<T>`. This means doing simple things like `position++` looks like `ReadOnlySequence<T>.GetPosition(position, 1)`.
- `GetPosition(long)` does **not** support negative indexes. This means it's impossible to get the second last character without walking all segments.
- `SequencePosition`(s) cannot be compared. This makes it hard to know if one position is greater than or less than another position and makes it hard to write some parsing algorithms.
- `ReadOnlySequence<T>` is bigger than an object reference and should be passed by in or ref where possible. This reduces copies of the struct.

## [SequenceReader\<T\>](https://docs.microsoft.com/en-us/dotnet/api/system.buffers.sequencereader-1?view=netcore-3.0)

`SequenceReader<T>` is a new type that was introduced in .NET Core 3.0 to simplify the processing of `ReadOnlySequence<T>`. It tries unify the differences between single segment and multi segment `ReadOnlySequence<T>`. It also provides helpers for reading binary and text data (byte and char) that may or may not be split across segments. 

There are a built in methods for dealing with parsing both binary and delimeted data. Lets take a look at what those same methods look like with the `SequenceReader<T>`:

#### Accessing data

`SequenceReader<T>` has methods for enumerating data inside of the `ReadOnlySequence<T>` directly. Below is an example of parsing a `ReadOnlySequence<byte>` a `byte` at a time:

```C#
while (!reader.End)
{
    var b = reader.CurrentSpan[reader.CurrentSpanIndex];
    Process(b);
    reader.Advance(1);
}
```

The `CurrentSpan` exposes the current segment's `Span` (similar to what we do in the method manually).

#### PositionOf

Here's an example of the `PositionOf` implementation using the `SequenceReader<T>`:

```C#
SequencePosition? FindIndexOf(in ReadOnlySequence<byte> buffer, byte data)
{
    var reader = new SequenceReader<byte>(buffer);

    while (!reader.End)
    {
        // Search for the byte in the current span
        var index = reader.CurrentSpan.IndexOf(data);
        if (index != -1)
        {
            // We found it so advance to the position
            reader.Advance(index);
            
            // Return the position
            return reader.Position;
        }
        // Skip the current segment since there's nothing in it
        reader.Advance(reader.CurrentSpan.Length);
    }

    return null;
}
```

#### Processing Binary Data

This example parses a 4 byte big-endian integer length from the start of the `ReadOnlySequence<byte>`.

```C#
bool TryParseHeaderLength(ref ReadOnlySequence<byte> buffer, out int length)
{
    var reader = new SequenceReader<byte>(buffer);
    return reader.TryReadBigEndian(out length);
}
```

#### Processing Text Data

```C#
static ReadOnlySpan<byte> NewLine => new byte[] { (byte)'\r', (byte)'\n' };

static bool TryParseLine(ref ReadOnlySequence<byte> buffer, out ReadOnlySequence<byte> line)
{
    var reader = new SequenceReader<byte>(buffer);

    if (reader.TryReadTo(out line, NewLine))
    {
        buffer = buffer.Slice(reader.Position);

        return true;
    }

    line = default;
    return false;
}
```

### Gotchas

- `SequenceReader<T>` is a mutable struct, it should always be passed by reference (in/ref).
- `SequenceReader<T>` is a ref-struct and it can only be used in synchronous methods.
- `SequenceReader<T>` is a forward only reader and `Advance` does not support negative numbers.