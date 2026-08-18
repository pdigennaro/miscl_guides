# Java NIO `ByteBuffer` — Complete Guide

`ByteBuffer` is one of the most important classes in Java NIO. If you're working with `SocketChannel`, `DatagramChannel`, files, binary protocols, or low-level networking, you'll use it constantly.

The key idea is:

> A `ByteBuffer` is a fixed-size block of bytes **plus state that tracks which part of the block you're currently reading or writing**.

The most important concepts to understand are:

**capacity → limit → position**, plus `flip()`, `clear()`, `compact()`, `rewind()`, and the distinction between **relative** and **absolute** operations.

---

# 1. Creating a `ByteBuffer`

There are three common ways.

## Heap buffer

```java
ByteBuffer buffer = ByteBuffer.allocate(1024);
```

This allocates approximately 1024 bytes in the Java heap.

Conceptually:

```text
JVM Heap

ByteBuffer
   |
   v
byte[1024]
```

You can access the underlying array:

```java
byte[] array = buffer.array();
```

---

## Direct buffer

```java
ByteBuffer buffer = ByteBuffer.allocateDirect(1024);
```

The actual storage is allocated outside the normal Java heap.

```text
Java Heap                  Native Memory

ByteBuffer  -------------> [ 1024 bytes ]
```

Direct buffers are particularly useful for I/O because the operating system can often transfer data directly between native memory and a device/socket/file.

For example:

```java
SocketChannel channel = ...;

ByteBuffer buffer = ByteBuffer.allocateDirect(4096);

channel.read(buffer);
```

Direct buffers are typically more expensive to allocate, so applications often **allocate them once and reuse them**.

---

## Wrapping an existing `byte[]`

```java
byte[] data = {1, 2, 3, 4};

ByteBuffer buffer = ByteBuffer.wrap(data);
```

No copy is required.

The buffer uses the array as its backing storage.

---

# 2. The three fundamental properties

Every `ByteBuffer` has three extremely important values:

```text
capacity
limit
position
```

Understanding these is basically the key to understanding `ByteBuffer`.

Suppose:

```java
ByteBuffer buffer = ByteBuffer.allocate(10);
```

Initially:

```text
capacity = 10
limit    = 10
position = 0
```

Visually:

```text
position
   |
   v
+---+---+---+---+---+---+---+---+---+---+
|   |   |   |   |   |   |   |   |   |   |
+---+---+---+---+---+---+---+---+---+---+
  0   1   2   3   4   5   6   7   8   9
                                          ^
                                          |
                                        limit
                                        capacity
```

---

# 3. `capacity`

`capacity` is the total number of bytes the buffer can hold.

```java
ByteBuffer buffer = ByteBuffer.allocate(10);

System.out.println(buffer.capacity());
```

Output:

```text
10
```

The capacity never changes.

You can't resize a `ByteBuffer`.

If you need more space, you normally create a new buffer.

---

# 4. `position`

`position` indicates **where the next read or write will occur**.

Initially:

```java
position = 0
```

If you write:

```java
buffer.put((byte) 10);
```

then:

```text
position = 1
```

Write another:

```java
buffer.put((byte) 20);
```

Now:

```text
position = 2
```

Conceptually:

```text
        position
           |
           v
+----+----+----+----+----+----+----+----+----+----+
| 10 | 20 |    |    |    |    |    |    |    |    |
+----+----+----+----+----+----+----+----+----+----+
  0    1    2
```

The position always indicates the **next element**.

---

# 5. `limit`

`limit` indicates the boundary beyond which the buffer cannot currently read or write.

Initially:

```text
limit = capacity
```

For:

```java
ByteBuffer.allocate(10);
```

you get:

```text
position = 0
limit    = 10
capacity = 10
```

But `limit` becomes particularly important when switching from writing to reading.

---

# 6. Writing data

Consider:

```java
ByteBuffer buffer = ByteBuffer.allocate(10);

buffer.put((byte) 10);
buffer.put((byte) 20);
buffer.put((byte) 30);
```

State:

```text
position = 3
limit    = 10
capacity = 10
```

Memory:

```text
             position
                |
                v
+----+----+----+----+----+----+----+----+----+----+
| 10 | 20 | 30 |    |    |    |    |    |    |    |
+----+----+----+----+----+----+----+----+----+----+
  0    1    2    3

                                                  ^
                                                  |
                                                limit
```

---

# 7. The critical operation: `flip()`

Suppose you have written three bytes:

```java
ByteBuffer buffer = ByteBuffer.allocate(10);

buffer.put((byte) 10);
buffer.put((byte) 20);
buffer.put((byte) 30);
```

Now:

```text
position = 3
limit = 10
```

If you want to **read the bytes you just wrote**, you call:

```java
buffer.flip();
```

`flip()` essentially does:

```java
limit = position;
position = 0;
```

So:

```text
BEFORE flip()

position = 3
limit    = 10

AFTER flip()

position = 0
limit    = 3
```

Visually:

```text
      readable data
<---------------------->

position            limit
   |                  |
   v                  v
+----+----+----+----+----+----+----+----+----+----+
| 10 | 20 | 30 |    |    |    |    |    |    |    |
+----+----+----+----+----+----+----+----+----+----+
```

Now:

```java
buffer.get();
```

returns:

```text
10
```

and position becomes:

```text
1
```

---

# 8. The typical ByteBuffer lifecycle

This pattern is extremely common:

```java
ByteBuffer buffer = ByteBuffer.allocate(1024);

// WRITE MODE
buffer.put(...);
buffer.put(...);
buffer.put(...);

// switch to READ MODE
buffer.flip();

// READ
while (buffer.hasRemaining()) {
    byte b = buffer.get();
}

// prepare for writing again
buffer.clear();
```

Think:

```text
WRITE
  ↓
flip()
  ↓
READ
  ↓
clear()
  ↓
WRITE
```

This cycle appears everywhere in Java NIO.

---

# 9. `remaining()`

`remaining()` tells you how many bytes exist between:

```text
position
```

and:

```text
limit
```

Formula:

```java
remaining = limit - position;
```

Example:

```text
position = 2
limit = 8
```

Then:

```java
buffer.remaining()
```

returns:

```text
6
```

---

# 10. `hasRemaining()`

This:

```java
buffer.hasRemaining()
```

is basically:

```java
buffer.position() < buffer.limit()
```

A common reading loop:

```java
while (buffer.hasRemaining()) {
    byte b = buffer.get();
}
```

---

# 11. Why this pattern is common

You previously encountered code like:

```java
byte[] data = new byte[buffer.remaining()];
buffer.get(data);
```

Now it should make more sense.

Suppose:

```text
position = 5
limit = 12
```

Then:

```java
buffer.remaining()
```

is:

```text
7
```

So:

```java
byte[] data = new byte[7];
```

Then:

```java
buffer.get(data);
```

copies the remaining 7 bytes from the `ByteBuffer` into the array.

Position then advances:

```text
position = 12
limit = 12
```

So:

```java
buffer.hasRemaining()
```

becomes:

```text
false
```

---

# 12. `put()`

There are many ways to put data into a buffer.

Single byte:

```java
buffer.put((byte) 10);
```

Array:

```java
byte[] data = {10, 20, 30};

buffer.put(data);
```

Part of an array:

```java
buffer.put(data, 0, 2);
```

which inserts:

```text
10
20
```

---

# 13. `get()`

Single byte:

```java
byte value = buffer.get();
```

Multiple bytes:

```java
byte[] data = new byte[5];

buffer.get(data);
```

This reads 5 bytes.

---

# 14. Relative vs absolute operations

This distinction is important.

## Relative operation

```java
buffer.get();
```

uses the current `position`.

For example:

```text
position = 3
```

then:

```java
buffer.get();
```

reads:

```text
buffer[3]
```

and changes position:

```text
position = 4
```

Similarly:

```java
buffer.put((byte) 10);
```

writes at `position` and advances it.

---

## Absolute operation

You can provide an index:

```java
byte b = buffer.get(3);
```

This reads index 3 directly.

It **does not change `position`**.

Likewise:

```java
buffer.put(3, (byte) 100);
```

modifies byte 3 without moving the position.

Example:

```java
ByteBuffer buffer = ByteBuffer.allocate(10);

buffer.put((byte) 10);
buffer.put((byte) 20);

System.out.println(buffer.position());
```

prints:

```text
2
```

Now:

```java
buffer.get(0);
```

returns:

```text
10
```

but:

```java
System.out.println(buffer.position());
```

still prints:

```text
2
```

---

# 15. `clear()`

A common misunderstanding:

> `clear()` does **not actually erase the bytes**.

Instead:

```java
buffer.clear();
```

does essentially:

```text
position = 0
limit = capacity
```

Suppose:

```text
Before clear:

position = 5
limit = 5
capacity = 10
```

After:

```text
position = 0
limit = 10
capacity = 10
```

The old bytes may still physically exist.

But as far as `ByteBuffer` is concerned, they can now be overwritten.

---

# 16. `rewind()`

`rewind()` sets:

```text
position = 0
```

but leaves the limit unchanged.

Example:

```text
Before:

position = 7
limit = 10

After rewind():

position = 0
limit = 10
```

This is useful if you want to read the same data again.

Example:

```java
buffer.flip();

process(buffer);

buffer.rewind();

processAgain(buffer);
```

---

# 17. `compact()`

This one is particularly important for networking.

Imagine you've received:

```text
A B C D E F
```

and consumed:

```text
A B C
```

but haven't consumed:

```text
D E F
```

The state might be:

```text
          position
             |
             v
+---+---+---+---+---+---+---+---+---+---+
| A | B | C | D | E | F |   |   |   |   |
+---+---+---+---+---+---+---+---+---+---+
              3           6
                          ^
                          |
                        limit
```

Calling:

```java
buffer.clear();
```

would effectively tell the buffer:

> Everything can be overwritten.

That's not what we want because `D E F` haven't been processed yet.

Instead:

```java
buffer.compact();
```

moves unread data to the beginning:

```text
+---+---+---+---+---+---+---+---+---+---+
| D | E | F |   |   |   |   |   |   |   |
+---+---+---+---+---+---+---+---+---+---+
              ^
              |
           position
```

Now more data can be appended.

This pattern is very common:

```java
channel.read(buffer);

buffer.flip();

parseMessages(buffer);

buffer.compact();
```

---

# 18. `flip()` vs `clear()` vs `rewind()` vs `compact()`

This is worth memorizing.

| Method      |             Position |        Limit | Main purpose                              |
| ----------- | -------------------: | -----------: | ----------------------------------------- |
| `flip()`    |                    0 | old position | write → read                              |
| `clear()`   |                    0 |     capacity | start writing fresh                       |
| `rewind()`  |                    0 |    unchanged | reread data                               |
| `compact()` | after remaining data |     capacity | preserve unread data and continue writing |

The conceptual cycle is:

```text
              flip()
    WRITE -------------> READ
      ^                    |
      |                    |
      |      clear()       |
      +--------------------+
```

Or when some unread data remains:

```text
              flip()
    WRITE -------------> READ
      ^                    |
      |                    |
      |      compact()     |
      +--------------------+
```

---

# 19. Primitive types

`ByteBuffer` doesn't only work with individual bytes.

It can encode primitive values.

For example:

```java
ByteBuffer buffer = ByteBuffer.allocate(100);

buffer.putInt(123);
buffer.putLong(5000L);
buffer.putFloat(3.14f);
buffer.putDouble(123.456);
buffer.putChar('A');
buffer.putShort((short) 10);
```

Corresponding reads:

```java
buffer.flip();

int a = buffer.getInt();
long b = buffer.getLong();
float c = buffer.getFloat();
double d = buffer.getDouble();
char e = buffer.getChar();
short f = buffer.getShort();
```

---

# 20. How many bytes do primitive types consume?

Typically:

```text
byte     = 1
short    = 2
char     = 2
int      = 4
float    = 4
long     = 8
double   = 8
```

For example:

```java
ByteBuffer buffer = ByteBuffer.allocate(20);

buffer.putInt(100);
```

Position becomes:

```text
4
```

Then:

```java
buffer.putLong(500);
```

position becomes:

```text
12
```

---

# 21. Byte order / endianness

This becomes very important when implementing network protocols.

Suppose:

```java
buffer.putInt(0x12345678);
```

An integer occupies four bytes.

It could be stored:

```text
12 34 56 78
```

or:

```text
78 56 34 12
```

These are different byte orders.

---

## Big endian

```text
0x12345678

12 34 56 78
```

Most significant byte first.

Java `ByteBuffer` defaults to:

```java
ByteOrder.BIG_ENDIAN
```

---

## Little endian

```text
78 56 34 12
```

Set it using:

```java
buffer.order(ByteOrder.LITTLE_ENDIAN);
```

Check it:

```java
System.out.println(buffer.order());
```

For network protocols, the specification usually defines the byte order.

Traditionally, "network byte order" means **big-endian**.

---

# 22. Example binary packet

Imagine you're designing a protocol:

```text
+---------+---------+----------------+
| version | type    | payload length |
| 1 byte  | 1 byte  | 4 bytes        |
+---------+---------+----------------+
```

You could build it as:

```java
ByteBuffer packet = ByteBuffer.allocate(6);

packet.put((byte) 1);
packet.put((byte) 3);
packet.putInt(1024);
```

Then send:

```java
packet.flip();

channel.write(packet);
```

Parsing:

```java
byte version = packet.get();
byte type = packet.get();
int length = packet.getInt();
```

This is one of the major reasons `ByteBuffer` is so useful for networking.

---

# 23. Strings and `ByteBuffer`

There is no:

```java
buffer.putString(...)
```

because strings need an encoding.

Use `Charset`.

For example:

```java
String message = "Hello World";

ByteBuffer buffer =
        StandardCharsets.UTF_8.encode(message);
```

Reading:

```java
String message =
        StandardCharsets.UTF_8.decode(buffer).toString();
```

Another approach:

```java
byte[] bytes = message.getBytes(StandardCharsets.UTF_8);

buffer.put(bytes);
```

---

# 24. Strings in binary network protocols

A common format is:

```text
[length][UTF-8 data]
```

For example:

```java
String text = "Hello";

byte[] bytes = text.getBytes(StandardCharsets.UTF_8);

ByteBuffer buffer = ByteBuffer.allocate(4 + bytes.length);

buffer.putInt(bytes.length);
buffer.put(bytes);
```

Packet:

```text
+----------------+-----------------------+
| length = 5     | H | e | l | l | o   |
| 4 bytes        | 5 bytes               |
+----------------+-----------------------+
```

To decode:

```java
buffer.flip();

int length = buffer.getInt();

byte[] textBytes = new byte[length];
buffer.get(textBytes);

String text =
        new String(textBytes, StandardCharsets.UTF_8);
```

---

# 25. Using `ByteBuffer` with `SocketChannel`

This is one of its most important applications.

Suppose:

```java
SocketChannel channel = ...;

ByteBuffer buffer = ByteBuffer.allocate(4096);

int bytesRead = channel.read(buffer);
```

Notice something important:

`SocketChannel.read()` **writes into the buffer**.

Therefore the buffer must be in **write mode**.

After:

```java
channel.read(buffer);
```

you might have:

```text
position = 500
limit = 4096
```

To process those bytes:

```java
buffer.flip();
```

Now:

```text
position = 0
limit = 500
```

Then:

```java
while (buffer.hasRemaining()) {
    byte b = buffer.get();
}
```

---

# 26. Writing to a `SocketChannel`

The reverse occurs when sending.

You first put bytes into the buffer:

```java
ByteBuffer buffer = ByteBuffer.allocate(1024);

buffer.put((byte) 10);
buffer.put((byte) 20);
buffer.put((byte) 30);
```

State:

```text
position = 3
limit = 1024
```

Then:

```java
buffer.flip();
```

State:

```text
position = 0
limit = 3
```

Then:

```java
channel.write(buffer);
```

`SocketChannel` **reads bytes from the ByteBuffer**.

---

# 27. Important: `channel.write()` may not write everything

This is extremely important in non-blocking networking.

You may have:

```java
buffer.remaining() == 1000
```

and call:

```java
channel.write(buffer);
```

But only 400 bytes might be written.

The buffer automatically updates:

```text
Before:

position = 0
limit = 1000


After channel.write():

position = 400
limit = 1000
```

Therefore:

```java
buffer.hasRemaining()
```

will still return:

```text
true
```

You cannot assume:

```java
channel.write(buffer);
```

sends the entire buffer.

In blocking mode, a common loop is:

```java
while (buffer.hasRemaining()) {
    channel.write(buffer);
}
```

In a non-blocking event loop, you generally store the partially written buffer and resume when the channel becomes writable again.

---

# 28. Why `ByteBuffer`'s position system is useful

Consider:

```java
channel.write(buffer);
```

Suppose the first call writes 300 bytes.

Java changes:

```text
position = 300
```

Next time:

```java
channel.write(buffer);
```

automatically starts at byte 300.

You don't need to manually track:

```text
bytesAlreadySent
```

The `ByteBuffer` itself contains that state.

---

# 29. Example SocketChannel read lifecycle

A realistic pattern looks like:

```java
ByteBuffer buffer = ByteBuffer.allocate(8192);

int bytesRead = channel.read(buffer);

if (bytesRead == -1) {
    // remote side closed connection
}

buffer.flip();

while (buffer.hasRemaining()) {
    byte b = buffer.get();

    // process byte
}

buffer.clear();
```

---

# 30. Partial network messages

Networking becomes more interesting because one `read()` does not necessarily correspond to one message.

Suppose the network protocol sends:

```text
[length][payload]
```

A single call:

```java
channel.read(buffer);
```

might return:

```text
[length][half of payload]
```

The rest may arrive later.

This is where:

```java
compact()
```

becomes extremely useful.

Example:

```java
channel.read(buffer);

buffer.flip();

parseCompleteMessages(buffer);

buffer.compact();
```

`parseCompleteMessages()` consumes only complete packets.

Any incomplete bytes remain.

`compact()` moves them to the beginning so the next:

```java
channel.read(buffer);
```

can append more data.

---

# 31. `BufferUnderflowException`

Suppose:

```java
buffer.remaining() == 2;
```

Then you try:

```java
int value = buffer.getInt();
```

But `getInt()` requires 4 bytes.

Java throws:

```text
BufferUnderflowException
```

So binary parsers should often check:

```java
if (buffer.remaining() >= Integer.BYTES) {
    int value = buffer.getInt();
}
```

---

# 32. `BufferOverflowException`

Suppose:

```text
remaining = 2
```

but you do:

```java
buffer.putInt(100);
```

`putInt()` requires four bytes.

You'll get:

```text
BufferOverflowException
```

---

# 33. Example parser with partial messages

Imagine packets are:

```text
4-byte length
payload
```

A parser can begin:

```java
if (buffer.remaining() < Integer.BYTES) {
    return;
}

buffer.mark();

int length = buffer.getInt();

if (buffer.remaining() < length) {
    buffer.reset();
    return;
}

byte[] payload = new byte[length];

buffer.get(payload);
```

This introduces another useful feature:

```java
mark()
reset()
```

---

# 34. `mark()` and `reset()`

You can remember a position:

```java
buffer.mark();
```

Then move forward:

```java
buffer.getInt();
buffer.get();
buffer.get();
```

And return:

```java
buffer.reset();
```

For example:

```java
buffer.mark();

int length = buffer.getInt();

if (buffer.remaining() < length) {
    buffer.reset();
    return;
}
```

This is particularly useful when you start parsing a packet but discover that the complete packet hasn't arrived yet.

---

# 35. `duplicate()`

You can create another buffer object that shares the same bytes:

```java
ByteBuffer duplicate = buffer.duplicate();
```

Important:

```text
original position/limit
```

and:

```text
duplicate position/limit
```

are independent.

But the underlying memory is shared.

Conceptually:

```text
ByteBuffer A ----+
                 |
                 +----> same memory
                 |
ByteBuffer B ----+
```

So:

```java
duplicate.put(0, (byte) 100);
```

also changes what `buffer.get(0)` sees.

This can be extremely useful when different components need their own view of the same data.

---

# 36. `slice()`

`slice()` creates a buffer representing the **remaining part of another buffer**.

Suppose:

```text
position = 3
limit = 8
```

Then:

```java
ByteBuffer slice = buffer.slice();
```

The slice represents:

```text
original[3..7]
```

Its own state starts as:

```text
position = 0
limit = 5
capacity = 5
```

But it shares the underlying memory.

For protocol parsing, this can be useful.

Example:

```java
int payloadLength = 100;

ByteBuffer payload = buffer.slice();

payload.limit(payloadLength);
```

Now `payload` represents the payload without copying the bytes.

---

# 37. Zero-copy-style parsing

Suppose your packet is:

```text
HEADER | PAYLOAD
```

Instead of doing:

```java
byte[] payload = new byte[length];

buffer.get(payload);
```

you can sometimes use:

```java
ByteBuffer payload = buffer.slice();

payload.limit(length);
```

Then manually advance the original:

```java
buffer.position(buffer.position() + length);
```

This avoids copying the payload.

For high-performance networking code this can matter.

---

# 38. Read-only buffers

You can make a buffer read-only:

```java
ByteBuffer readOnly = buffer.asReadOnlyBuffer();
```

Reads work:

```java
byte b = readOnly.get();
```

Writes don't:

```java
readOnly.put((byte) 10);
```

will throw:

```text
ReadOnlyBufferException
```

This is useful when exposing data to code that should not modify it.

---

# 39. `array()` and `arrayOffset()`

Heap buffers normally have an accessible backing array:

```java
ByteBuffer buffer = ByteBuffer.allocate(1024);

byte[] array = buffer.array();
```

But direct buffers generally don't:

```java
ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

buffer.array();
```

throws:

```text
UnsupportedOperationException
```

You can test:

```java
buffer.hasArray();
```

---

# 40. `isDirect()`

Check whether the buffer is direct:

```java
if (buffer.isDirect()) {
    System.out.println("Direct buffer");
}
```

---

# 41. Comparing heap and direct buffers

### Heap

```java
ByteBuffer.allocate(4096);
```

Advantages:

* cheap allocation
* managed naturally by JVM
* accessible through `byte[]`
* convenient for ordinary application code

Disadvantages:

* native I/O may require copying

### Direct

```java
ByteBuffer.allocateDirect(4096);
```

Advantages:

* potentially better for repeated native I/O
* useful for high-performance NIO

Disadvantages:

* more expensive allocation
* memory management is less straightforward
* no normal backing `byte[]`
* usually should be pooled/reused

A rough rule is:

> Use heap buffers unless you have an I/O/performance reason to prefer direct buffers.

For something such as a high-performance networking/torrent implementation, reusable direct buffers can make sense, but benchmarking should guide the decision rather than assuming they're always faster.

---

# 42. Buffer views for primitive types

You can expose a `ByteBuffer` as another type of buffer.

For example:

```java
IntBuffer ints = buffer.asIntBuffer();
```

Then:

```java
ints.put(10);
ints.put(20);
ints.put(30);
```

There are:

```java
asCharBuffer()
asShortBuffer()
asIntBuffer()
asLongBuffer()
asFloatBuffer()
asDoubleBuffer()
```

These are useful in certain numerical/native-memory scenarios.

---

# 43. Inspecting the state

While learning NIO, I strongly recommend logging:

```java
System.out.println(
    "position=" + buffer.position()
    + ", limit=" + buffer.limit()
    + ", capacity=" + buffer.capacity()
    + ", remaining=" + buffer.remaining()
);
```

For example:

```java
ByteBuffer buffer = ByteBuffer.allocate(10);

System.out.println(buffer);

buffer.putInt(123);

System.out.println(buffer);

buffer.flip();

System.out.println(buffer);

buffer.getInt();

System.out.println(buffer);
```

`ByteBuffer.toString()` itself typically gives useful state information like:

```text
java.nio.HeapByteBuffer[pos=0 lim=10 cap=10]
```

---

# 44. A complete example

Let's walk through every state.

```java
ByteBuffer buffer = ByteBuffer.allocate(10);
```

State:

```text
pos = 0
lim = 10
cap = 10
```

Then:

```java
buffer.put((byte) 10);
buffer.put((byte) 20);
buffer.put((byte) 30);
```

State:

```text
pos = 3
lim = 10
cap = 10
```

Contents:

```text
[10][20][30][?][?][?][?][?][?][?]
             ^
             position
```

Then:

```java
buffer.flip();
```

State:

```text
pos = 0
lim = 3
cap = 10
```

Now:

```java
byte x = buffer.get();
```

returns:

```text
10
```

State:

```text
pos = 1
lim = 3
cap = 10
```

Then:

```java
byte[] remaining = new byte[buffer.remaining()];
```

`remaining()` is:

```text
3 - 1 = 2
```

so:

```text
byte[2]
```

Then:

```java
buffer.get(remaining);
```

produces:

```text
[20, 30]
```

State:

```text
pos = 3
lim = 3
cap = 10
```

Then:

```java
buffer.clear();
```

State:

```text
pos = 0
lim = 10
cap = 10
```

Ready to receive new data.

---

# 45. One mental model that makes `ByteBuffer` much easier

Think of a buffer as having three markers:

```text
0              position              limit            capacity
|-----------------|--------------------|------------------|
     consumed          available            inaccessible
```

During reading:

```text
0          position               limit
|-------------|---------------------|
 already read       remaining
```

and:

```java
remaining() == limit - position
```

During writing:

```text
0               position                         limit
|------------------|-------------------------------|
 already written           writable
```

The exact same `position` concept therefore works in both modes.

---

# 46. The confusing thing: ByteBuffer doesn't really have a "mode"

People commonly talk about:

```text
write mode
read mode
```

but there isn't actually something like:

```java
buffer.setMode(READ);
```

The buffer doesn't store a mode flag.

What changes are simply:

```text
position
limit
```

When we say:

> "flip switches the buffer to reading mode"

what we really mean is:

```java
limit(position);
position(0);
```

and those values make subsequent reads operate over the data that was just written.

---

# 47. Common mistake #1 — forgetting `flip()`

Wrong:

```java
ByteBuffer buffer = ByteBuffer.allocate(100);

buffer.put("hello".getBytes());

channel.write(buffer);
```

At this point:

```text
position = 5
limit = 100
```

So the channel sees bytes starting from position 5 rather than your message starting at position 0.

Correct:

```java
buffer.put("hello".getBytes());

buffer.flip();

channel.write(buffer);
```

---

# 48. Common mistake #2 — calling `flip()` twice

Suppose:

```java
position = 100
limit = 1024
```

Call:

```java
buffer.flip();
```

Now:

```text
position = 0
limit = 100
```

If you accidentally call:

```java
buffer.flip();
```

again:

```text
position = 0
limit = 0
```

Now there is nothing to read.

---

# 49. Common mistake #3 — using `clear()` when `compact()` is needed

Suppose you've only processed part of a network message.

If you do:

```java
buffer.clear();
```

you lose track of the unread bytes.

Instead:

```java
buffer.compact();
```

preserves them.

This distinction becomes crucial when implementing TCP protocol parsers.

---

# 50. Common mistake #4 — assuming one read equals one packet

With TCP:

```java
channel.read(buffer);
```

might give you:

```text
half a message
```

or:

```text
three messages
```

or:

```text
one message + half of the next
```

`ByteBuffer` is just holding a **byte stream**.

Your application protocol determines where messages begin and end.

---

# 51. Common mistake #5 — assuming `write()` sends everything

Wrong:

```java
channel.write(buffer);

buffer.clear();
```

because some data could remain.

Better:

```java
while (buffer.hasRemaining()) {
    channel.write(buffer);
}
```

for blocking channels.

For non-blocking channels, you generally preserve the buffer and resume the write later.

---

# 52. Common mistake #6 — `buffer.capacity()` vs `remaining()`

Suppose:

```text
capacity = 4096
position = 200
limit = 700
```

Then:

```java
capacity()
```

returns:

```text
4096
```

while:

```java
remaining()
```

returns:

```text
500
```

If you want to extract the unread bytes:

```java
byte[] data = new byte[buffer.remaining()];
buffer.get(data);
```

not:

```java
new byte[buffer.capacity()];
```

---

# 53. Common mistake #7 — forgetting signed Java bytes

Java's:

```java
byte
```

is signed:

```text
-128 ... 127
```

Protocols frequently specify unsigned bytes:

```text
0 ... 255
```

Suppose:

```java
byte b = buffer.get();
```

To interpret it as unsigned:

```java
int value = Byte.toUnsignedInt(b);
```

or:

```java
int value = b & 0xFF;
```

Example:

```java
byte b = (byte) 255;

System.out.println(b);
```

prints:

```text
-1
```

But:

```java
System.out.println(Byte.toUnsignedInt(b));
```

prints:

```text
255
```

This comes up constantly in network protocols.

---

# 54. Unsigned short

Likewise:

```java
short value = buffer.getShort();
```

is signed.

To convert:

```java
int unsigned = Short.toUnsignedInt(value);
```

---

# 55. Example packet parser

Imagine a packet:

```text
1 byte version
1 byte type
2 bytes sequence number
4 bytes payload length
N bytes payload
```

You could implement:

```java
static Packet readPacket(ByteBuffer buffer) {
    byte version = buffer.get();
    byte type = buffer.get();

    int sequence =
            Short.toUnsignedInt(buffer.getShort());

    int payloadLength = buffer.getInt();

    byte[] payload = new byte[payloadLength];

    buffer.get(payload);

    return new Packet(
            version,
            type,
            sequence,
            payload
    );
}
```

That's much cleaner than manually manipulating individual bytes.

---

# 56. But real network parsers must handle partial packets

A more realistic parser does something like:

```java
static Packet tryReadPacket(ByteBuffer buffer) {

    int headerSize = 8;

    if (buffer.remaining() < headerSize) {
        return null;
    }

    buffer.mark();

    byte version = buffer.get();
    byte type = buffer.get();

    int sequence =
            Short.toUnsignedInt(buffer.getShort());

    int payloadLength = buffer.getInt();

    if (buffer.remaining() < payloadLength) {
        buffer.reset();
        return null;
    }

    byte[] payload =
            new byte[payloadLength];

    buffer.get(payload);

    return new Packet(
            version,
            type,
            sequence,
            payload
    );
}
```

Then:

```java
buffer.flip();

Packet packet;

while ((packet = tryReadPacket(buffer)) != null) {
    process(packet);
}

buffer.compact();
```

This is a very typical NIO architecture.

---

# 57. The full networking lifecycle

For a reusable receive buffer:

```java
ByteBuffer receiveBuffer =
        ByteBuffer.allocateDirect(64 * 1024);
```

Your event loop might conceptually do:

```java
int bytesRead =
        socketChannel.read(receiveBuffer);

if (bytesRead == -1) {
    closeConnection();
    return;
}

receiveBuffer.flip();

while (true) {
    Packet packet =
            tryReadPacket(receiveBuffer);

    if (packet == null) {
        break;
    }

    process(packet);
}

receiveBuffer.compact();
```

Visually:

```text
Network
   |
   v
SocketChannel.read()
   |
   v
+--------------------+
| ByteBuffer WRITE   |
+--------------------+
          |
        flip()
          |
          v
+--------------------+
| ByteBuffer READ    |
+--------------------+
          |
      parse packets
          |
        compact()
          |
          v
+--------------------+
| preserve leftovers |
| + receive more     |
+--------------------+
```

That is one of the most important patterns to understand when learning Java NIO.

---

# 58. `ByteBuffer` with `DatagramChannel`

UDP also uses `ByteBuffer`.

Receiving:

```java
ByteBuffer buffer =
        ByteBuffer.allocate(2048);

SocketAddress sender =
        datagramChannel.receive(buffer);

buffer.flip();
```

Then:

```java
byte[] data =
        new byte[buffer.remaining()];

buffer.get(data);
```

Sending:

```java
ByteBuffer buffer =
        ByteBuffer.wrap(data);

datagramChannel.send(
        buffer,
        destinationAddress
);
```

This is directly relevant to implementations of protocols built over UDP.

---

# 59. Why `flip()` is such a weird name

The name comes from the idea of flipping the buffer from:

```text
FILLING
```

to:

```text
DRAINING
```

Before:

```text
0 ---------------- position ------------ capacity
        DATA              FREE
```

After `flip()`:

```text
position=0 ------- limit
        DATA
```

So `flip()` essentially says:

> "The data ends where my current position is. Now start reading from the beginning."

---

# 60. A useful vocabulary: filling and draining

Sometimes thinking in terms of:

```text
write/read
```

gets confusing because a channel and a buffer are doing opposite operations.

For example:

```java
channel.read(buffer)
```

means:

* READ from **channel**
* WRITE into **buffer**

So it's often easier to think:

```text
fill the buffer
drain the buffer
```

### Filling

```java
channel.read(buffer);
```

### Prepare to drain

```java
buffer.flip();
```

### Drain

```java
process(buffer);
```

### Prepare to refill

```java
buffer.clear();
```

or:

```java
buffer.compact();
```

---

# 61. The four concepts I would memorize first

If you only remember one section from this guide, make it this one.

### `position`

```text
Where the next operation happens.
```

### `limit`

```text
How far the current operation is allowed to go.
```

### `capacity`

```text
Physical size of the buffer.
```

And always:

```text
0 <= position <= limit <= capacity
```

Then remember:

```java
remaining()
```

means:

```text
limit - position
```

---

# 62. And memorize these four methods

### `flip()`

```text
I finished writing.
I want to read what I wrote.
```

```java
buffer.flip();
```

### `clear()`

```text
I'm done reading everything.
I want to overwrite the buffer.
```

```java
buffer.clear();
```

### `compact()`

```text
I haven't read everything.
Keep the remaining bytes and let me receive more.
```

```java
buffer.compact();
```

### `rewind()`

```text
I want to read the same data again from the start.
```

```java
buffer.rewind();
```

---

# 63. Cheat sheet

```java
// CREATE
ByteBuffer buffer = ByteBuffer.allocate(1024);
ByteBuffer direct = ByteBuffer.allocateDirect(1024);


// WRITE
buffer.put((byte) 1);
buffer.putInt(123);
buffer.putLong(456L);
buffer.put(byteArray);


// WRITE → READ
buffer.flip();


// READ
byte b = buffer.get();
int i = buffer.getInt();
long l = buffer.getLong();


// AVAILABLE DATA
int n = buffer.remaining();


// ANYTHING LEFT?
boolean remaining = buffer.hasRemaining();


// READ EVERYTHING
byte[] data = new byte[buffer.remaining()];
buffer.get(data);


// READ AGAIN
buffer.rewind();


// START FRESH
buffer.clear();


// PRESERVE UNREAD DATA
buffer.compact();


// STATE
buffer.position();
buffer.limit();
buffer.capacity();


// ENDIANNESS
buffer.order(ByteOrder.BIG_ENDIAN);
buffer.order(ByteOrder.LITTLE_ENDIAN);


// ABSOLUTE ACCESS
buffer.get(5);
buffer.put(5, (byte) 10);


// SHARED VIEWS
buffer.duplicate();
buffer.slice();


// READ ONLY
buffer.asReadOnlyBuffer();
```

---

# 64. The mental picture I recommend keeping

Whenever you see a `ByteBuffer`, imagine:

```text
                 position
                    |
                    v
0                                       capacity
|-------------------------------------------|
|xxxxxxxxxxxxxxxxxxxx|......................|
                     ^
                     |
                   position

                                  ^
                                  |
                                limit
```

For reading:

```text
0          position                 limit
|-------------|-----------------------|
 already used        remaining
```

For writing:

```text
0            position                limit
|---------------|----------------------|
  already filled       free space
```

And remember:

```java
remaining() == limit - position
```

Everything else builds on that.

---

## The most important NIO pattern

For network programming, this pattern is worth becoming completely comfortable with:

```java
ByteBuffer buffer =
        ByteBuffer.allocateDirect(8192);

// receive bytes
channel.read(buffer);

// WRITE MODE → READ MODE
buffer.flip();

// process as many complete messages as possible
while (canParseMessage(buffer)) {
    parseMessage(buffer);
}

// preserve incomplete message bytes
buffer.compact();
```

Then the next network read appends new bytes after those leftovers:

```java
channel.read(buffer);
```

Once **`position`, `limit`, `flip()`, and `compact()`** become intuitive, `ByteBuffer` stops looking strange and Java NIO networking becomes substantially easier to understand.
