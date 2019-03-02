# BitpackingTools
Bitpacking/serialization libraries used interally for Unity Store <a href="https://assetstore.unity.com/packages/tools/network/network-sync-transform-nst-98453">NetworkSyncTransform</a> and <a href="https://assetstore.unity.com/packages/tools/network/transform-crusher-free-version-117313">TransformCrusher</a> Assets 

To make these extensions accessible add:
```using emotitron.Compression;```

## ArraySerializeExt class

The Array Serializer extension lets you bitpack directly to and from byte[], uint[] and ulong[] buffers. Because there is no wrapper class such as Bitstream, you need to maintain read/write pointers. The methods automatically increment this pointers for you however.

### Basic Usage:
```cs
 byte[] myBuffer = new byte[64];

int writepos = 0;
myBuffer.WriteBool(true, ref writepos);
myBuffer.WriteSigned(-666, ref writepos, 11);
myBuffer.Write(999, ref writepos, 10);

int readpos = 0;
bool restoredbool = myBuffer.ReadBool(ref readpos);
int restoredval1 = myBuffer.ReadSigned(ref readpos, 11);
uint restoredval2 = (uint)myBuffer.Read(ref readpos, 10);
```
### Advanced Usage (Unsafe)
For sequential writes and reads of a byte[] or uint[] arrays, there are unsafe methods that internally treat these arrays as a ulong[], resulting in up to 4x faster reads and writes. These are all contained in ArraySerializerUnsafe.cs, which can be deleted for projects where you don't want to enable Allow Unsafe Code.
```cs
byte[] myBuffer = byte[100];
uint val1 = 666;
int val2 = -999;

fixed (byte* bPtr = myBuffer)
{
  // Cast the byte* to ulong*
  ulong* uPtr = (ulong*)bPtr;
  
  int writepos = 0;
  // Two different write methods. Inject() and Write(). 
  // Both are about the same speed, so it's up to personal preference.
  val1.InjectUnsafe(uPtr, ref writepos, 10);
  ArraySerializeUnsafe.WriteSigned(uPtr, val2, ref writepos, 11);

  readpos = 0;
  // Unsafe pointers can't be the first argument of extensions, so there is no pretty way to do this.
  uint restored1 = (uint)ArraySerializeUnsafe.ReadUnsafe(uPtr, ref readpos, 10);
  int restored2 = ArraySerializeUnsafe.ReadSigned(uPtr, ref readpos, 11);
}
```

## PrimitiveSerializeExt class

The Primitive Serializer extension lets you bitpack directly to and from ulong, uint, ushort and byte primitives. Because there is no wrapper class such as Bitstream, you need to maintain read/write pointers. The methods automatically increment the pointer for you, as it is passed to the methods as a reference. NOTE: Extension methods cannot pass the first argument reference so the Write() return value must be applied to the buffer being written to.

### Basic Usage:
```cs
  ulong myBuffer = 0;
  
  int writepos = 0;
  // Note that primitives are reference types, so the return value needs to be applied.
  myBuffer = myBuffer.WritetBool(true, ref writepos);
  myBuffer = myBuffer.WriteSigned(-666, ref writepos, 11);
  myBuffer = myBuffer.Write(999, ref writepos, 10);
  
  int readpos = 0;
  bool restoredbool = myBuffer.ReadBool(ref readpos);
  int restoredval1 = myBuffer.ReadSigned(ref readpos, 11);
  uint restoredval2 = (uint)myBuffer.Read(ref readpos, 10);
```

### Alternative Usage
An alternative to Write() is Inject(), which has the value being written as the first argument, allowing us to pass the buffer as a reference. NOTE: There is no return value on this method, as the buffer is passed by reference and is altered by the method.
```cs
  int writepos = 0;
  // Note that the buffer is passed by reference, and is altered by the method.
  true.Inject(ref myBuffer, ref writepos);
  (-666).InjectSigned(ref myBuffer, ref writepos, 11);
  999.InjectUnsigned(ref myBuffer, ref writepos, 10);
 ```

## PackedBits and PackedBytes
For fields that have large potential ranges, but have values that hover at or near zero there are PackedBits and PackedBytes serialization options.

```cs
int holdpos, writepos = 0, readpos = 0;

buffer.WriteSignedPackedBits(0, ref writepos, 32);
buffer.WriteSignedPackedBits(-100, ref writepos, 32);
buffer.WriteSignedPackedBits(int.MinValue, ref writepos, 32);

holdpos = readpos;
int restored1 = buffer.ReadSignedPackedBits(ref readpos, 32);
Debug.Log("ZERO = " + restored1 + " with " + (readpos - holdpos) + " written bits");

holdpos = readpos;
int restored2 = buffer.ReadSignedPackedBits(ref readpos, 32);
Debug.Log("-100 = " + restored2 + " with " + (readpos - holdpos) + " written bits");

holdpos = readpos;
int restored3 = buffer.ReadSignedPackedBits(ref readpos, 32);
Debug.Log("MIN = " + restored3 + " with " + (readpos - holdpos) + " written bits");
```

This returns the results of 
```
ZERO = 0 with 6 written bits
-100 = -100 with 14 written bits
MIN = -2147483648 with 38 written bits
```
For 32 bits of variable size, there is a 6 bit sizer added, making this less than ideal if the values will be large. However if the values often stay closer to zero, this can save quite a bit of space. Note that a value of zero only took 6 bits, and a value of -100 only took 14 bits.

### PackedBits
Values serialized using ``WritePackedBits()`` are checked for the position of the highest used signifigant bit. All zero bits on the left of the value are not serialized, and the value is preceeded by a write of several bits for size info.

| sizer | bits | break even  |
|-------|------|-------------|
| 4     | 8    | 15          |
| 5     | 16   | 2047        |
| 6     | 32   | 67108863    |
| 7     | 64   | 1.44115E+17 |

### PackedBytes
PackedBytes work in a similar way to PackedBits, except rather than counting bits, it counts used bytes. Values serialized using ``WritePackedBytes()`` are checked for the position of the highest used signifigant bit, and that is rounded up the the nearest byte. All empty bytes on the left of the value are not serialized, and the value is preceeded by a write of several bits for size info. The resulting compression is similar in size to Varints. The disadvantage of PackedBytes vs PackedBits is the rounding up to the nearest whole number of bytes, which may or may not be worth the reduced sizer size. It also has a lower threshold of where there is a savings.

| sizer | bits | break even  |
|-------|------|-------------|
| 1     | 8    | 0           |
| 2     | 16   | 255         |
| 3     | 32   | 16777215    |
| 4     | 64   | 7.20576E+16 |

### SignedPackedBits / SignedPackedBytes
Signed types (int, short and sbyte) are automatically zigzagged to move the sign bit from the msb position to the lsb position, keeping the pattern of "closer to zero, the smaller the write" true for negative numbers.
