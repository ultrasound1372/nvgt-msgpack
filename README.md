# nvgt-msgpack (a fully spec-compliant MessagePack serializer and deserializer in NVGT)
This repository contains a fully-featured MessagePack serializer and deserializer in pure NVGT, based on [version 2 of the MessagePack specification](https://github.com/msgpack/msgpack/blob/9aa092d6ca81f12005bd7dcbeb6488ad319e5133/spec.md). It is designed to be simply included into your code and used as a library.

## What is MessagePack?
MessagePack (henceforth referred to as msgpack) is a fast, efficient streamed binary format for data interchange, built on similar principles as JSON. That is, it does not have real need of a schema, and allows the communication of arbitrary structured data in a way that makes sense for programs, while boasting a compact representation for such a format. As it is binary it is obviously not designed to be human-readable or edited by humans, so it can focus on that compactness, while at its core being almost directly translatable from and to JSON. See [its website](https://msgpack.org) for more details.  
Like many binary formats, msgpack is a streaming format which allows you to decode data as it arrives, without having to receive the entire stream beforehand. A msgpack stream is made up of a chain of msgpack values, encoded as a particular format as defined by the specification, concatenated end-to-end. There is no real start of stream or end of stream indicator, any number of contiguous valid msgpack values is a valid msgpack stream. Receiving partial data is absolutely possible, but there will be no ambiguity as to whether the data is complete or not.

# Usage
This library defines the following.

- A value (**mp_value**) type, which stores all the kinds of data that msgpack is capable of handling and allows simple retrieval and construction by the program. This type is necessary as msgpack is dynamically typed and NVGT is not.
- An **mp_ext** object, used to represent msgpack's extension type mechanism, and an **mp_timestamp** object to represent msgpack time stamps packed in extension types.
- An **mp_map** object, which serves as the direct analogue for the msgpack mapping, itself a close mapping to JSON's object type or the dictionary type in many languages. It is needed, rather than the direct use of NVGT dictionaries, due to type erasure present in NVGT dictionaries and the fact that msgpack does not require the keys of its maps to be only of the string type.
- An **mp_decoder** object (referred to as an unpacker in other implementations) which deserializes msgpack data fed to it into **mp_value** objects.
- An **mp_encoder** object (referred to as a packer in other implementations) which serializes **mp_value** objects into msgpack streams.
- Various enumerations useful when interacting with msgpack streams, of which the most important for users are **mp_type** and **mp_decoder_state**, listed below.
- Some convenience functions to avoid the need to work with encoder and decoder objects in the simplest of cases, that being when none of the streaming features are needed and the data is almost certainly valid and complete.
- Some other useful constants, including common exception strings and known extension type codes.

For deserialization, the typical usage pattern would be to instantiate an **mp_decoder**, passing the msgpack data to be deserialized to its constructor and possibly one or more later calls to `mp_decoder.push`. Then a loop that repeatedly calls `mp_decoder.try_read_value()`, checks its state, and when ready calls `mp_decoder.get_value()` would iterate through all the values stored in the msgpack stream, until you ran out of data to read.

For serialization, the typical usage pattern would be to instantiate an **mp_encoder** and call the appropriate write methods, whether `write_value`, its type-specific versions or overloads that accept bare primitives, and then call `mp_encoder.read()` and `mp_encoder.flush()` as warranted.

Note that many functions in this library have the possibility to throw exceptions. Many of them simply arise from improper calls, such as trying to get a value of the wrong type, but importantly two that can arise from the incoming data itself is an improper key type or exceeded recursion limit. It is recommended that you wrap your msgpack usage inside a try/catch block and check the exception info, all msgpack exceptions begin with "msgpack", and important ones are defined in constant strings.

# Enumerations
## mp_type
This enumeration stores all possible types that msgpack is capable of representing, what the specification refers to as format families. It is required to be checked against for retrieval code to function properly. Failure to do so will result in exceptions being thrown when getting to the wrong type.

- **MPT_INT**: An integer. Msgpack supports all the integer types NVGT does, that being (u)int8/16/32/64. It also has 7-bit positive and 5-bit negative formats, which internally map to uint8 and int8 respectively. Encoders will shrink to the smallest type that can represent a given quantity. When retrieving integers, the issues of width and sign are left completely up to the user, NVGT facilitates the automatic casting and sign extension of any of these types. Importantly, it allows getting a signed int to an unsigned one and vice versa and copies the bit pattern directly, which can produce completely unexpected values to code that isn't careful to match the signs!
- **MPT_NIL**: The nil type, in various places called nil, null, None and undefined. This type has no value to retrieve.
- **MPT_BOOLEAN**: A boolean, which can only store the values true or false. Maps to NVGT's bool type.
- **MPT_FLOAT**: A floating point value, stored as either single-precision (NVGT's float) or double-precision (NVGT's double). Getting to the other type is possible, but note that getting from a represented double to an internal float is liable to lose precision, whereas the reverse is not true. Serializers may choose to convert a double to a float to save 4 bytes, and this one does, but only if it will incur no loss of precision.
- **MPT_STRING**: A text string. Importantly, according to the version 2 specification, all values of this type should be encoded in valid UTF-8, I.E. this is not for storing arbitrary bytes. Maps to NVGT's string type, where this constraint is not enforced, NVGT strings may contain arbitrary bytes.
- **MPT_BIN**: A binary string (bytestring). Designed for serializing binary data rather than text. Correct usage of this and the above string type for their respective purposes is crucial to interoperability with other msgpack libraries and any defined schema applications may use. Also maps to NVGT's string type.
- **MPT_ARRAY**: An array, also called a list in some languages, functionally equivalent to JSON's array type. Stores a series of values accessed by an index in a well-defined order. Like JSON, arrays need not be homogeneous, I.E. an array can contain any combination of values within it. Maps to an array of value handles (`mp_value@[]`) in NVGT.
- **MPT_MAP**: A mapping, also called a dictionary or dict, similar to JSON's object type. Associates keys with values. Maps to `mp_map` objects in NVGT. NVGT dictionaries allow only strings as keys, and for API compatibility and ease of use this is preserved in this library's map object. Msgpack itself places no constraints on keys, including the existence of duplicate keys, but many libraries require these types be easily hashable and that there are no duplicates. This library will silently coerce integer, float, boolean, and nil values to string representations to allow loading such data if not in strict mode, and will always coerce bin types, but note that this adds the potential of coerced values ending up as duplicate keys, at which point the original value may be lost. If you attempt to deserialize a dictionary which stores array, map, or extension types as keys, an exception will be thrown and this map cannot be deserialized. The same is true in other implementations.
- **MPT_EXT**: A msgpack-specific type, representing a tuple of an int8 indicating a specific extension type, and a bytestring representing the payload. Per the specification, extension types with a value less than 0 are strictly reserved for the msgpack specification itself to define, leaving the user with the available values 0-127. Each application may register their own extension types and deal with them as they desire, any of these are defined to be application-specific. Maps to `mp_ext` objects in NVGT.
- **MPT_UNDETERMINED**: Type is not known yet. Should never actually occur, is used merely as a placeholder in internal bookkeeping. If it does occur anywhere, expect at best a thrown exception, at worst undefined behavior.

## mp_decoder_state
This enumeration stores all possible states of an **mp_decoder**, as returned by the various read calls and the `get_state()` method. This is used to communicate where along the decoder is back to the caller so that it can be used properly.

- **MPDS_READY**: The decoder is ready to read a new value from the stream, no prior value is in the middle of being read. The next immediate thing you should do is call either the `try_read_value()` method or the `read_format()` method directly. A single byte will be read from the stream to determine the format of the next value, or if this is the end of the stream.
- **MPDS_COUNT**: The format byte has been read, and now a number indicating the length of some container or the value of a numeric type needs to be read. This state is transitioned to from the `read_format()` method, and indicates you should call the `read_count()` method next. This method will read 1, 2, 4, or 8 bytes from the stream depending on the format.
- **MPDS_EXT_TYPE**: The format specifies an extension type is to be read, and the size has already been determined. Next the int8 which determines the type of the extension is to be read by calling `read_ext_type()`, which reads one byte from the stream.
- **MPDS_CONTENT**: The content of a msgpack value needs to be read, after its length has already been determined from the count state. This state indicates you should call `read_content()`. This method will read any number of bytes from the stream to satisfy the content of the value, which may include recursively reading other values inside arrays and/or maps.
- **MPDS_READ**: A value has been fully read from the stream and is ready for retrieval. In order to continue, you must call the `get_value()` method to retrieve it, or the `discard_value` method to drop it. Calling either one of these at this state will clear the internal value state and transition the decoder back to the **MPDS_READY** state, at which point you may continue to read values from the stream.
- **MPDS_MORE_DATA**: During the task of reading a value from the stream via any of the read methods, the decoder has determined it needs more bytes fed to it to continue to read the whole value. When more bytes are available, you should feed them to the decoder with the `push(string)` method, and if it returns true, proceed from the point you left off. If you are calling read methods directly, you should call `get_state()` here to see what state the decoder has transitioned back to in order to know which read method to resume.
- **MPDS_END_DATA**: The stream ran out of bytes directly upon attempting to read a format byte, potentially indicating that the stream is finished. This is not necessarily the case, however, chunked input could simply have terminated on a value boundary by chance. If you know there is more data, push and read format again. If you know that is all the data, you are finished decoding this stream. It is advised you call `mp_decoder.reset()` to clear its internal buffers.
- **MPDS_INVALID**: An invalid condition has occurred, and/or the stream is malformed. The decoder is now jammed and no further operations on it will succeed. Resetting it is your only option.
- **MPDS_INVALID_OPERATION**: This is returned if you attempt to call the wrong method for the current state, but causes no actual state transition.

# Constants
The following public constants are a defined part of this API.

## Exceptions
These string constants are used in throw statements for common cases, making the job of exception checking slightly easier on the caller. Note, however, that not all exception scenarios are covered by these constants, some exceptions are only thrown in one possible case and thus are not defined here.
All exceptions thrown by this library start with the string "msgpack " to help a little with checking such things, E.G. startswith on the exception string.

- `MP_TYPE_MISMATCH_EXCEPTION`: Thrown when an attempted type conversion is impossible.
- `MP_INVALID_KEY_TYPE_EXCEPTION`: Thrown when an unsupported key type is encountered when encoding or decoding a map. Whether a given type is supported depends on the setting of strict key mode.
- `MP_RECURSION_LIMIT_EXCEPTION`: Thrown when the maximum recursion depth is exceeded when encoding or decoding.
- `MP_LARGE_VALUE_EXCEPTION`: Thrown when attempting to encode a value that exceeds the representable bounds in msgpack. This happens when the length of a string or ext data, the number of values in an array, or the number of pairs in a map exceeds (2^32)-1.

## Known Extension Type Codes
These int8 constants are used as the type code for known extension types, those types for which the library itself defines a conversion beyond ext objects.

- `MP_EXT_TIMESTAMP`: The timestamp type as defined in the msgpack specification, storing seconds and nanoseconds since the unix epoch in 4, 8 or 12 bytes. See `mp_timestamp`.
- `MP_EXT_VECTOR`: Msgpack serialization for the NVGT vector type defined by this library. The type code is 86, corresponding to the ASCII letter 'V'. The payload is the three floats concatenated in the order x y z in network byte order.

# Functions
The functions provided here are convenience methods that allow you to avoid having to manage encoder and decoder instances, should you know that you have the complete stream of data for decoding or the complete set of values for encoding beforehand.

## mp_dumps
Serialize the given value object to a string and return it in one step.
```
string mp_dumps(mp_value@ v, bool sort_map_keys = false, bool strict_map_keys=true, uint max_recursion = 100);
```

### Arguments

- `mp_value@ v`: The value to serialize.
- `bool sort_map_keys`: Whether the serialized representation of a map should be sorted, with its pairs in the order given by the sorting of its keys. Defaults to false.
- `bool strict_map_keys`: Whether the keys of maps should be constrained to the str and bin types (strict key mode). Defaults to true.
- `uint max_recursion`: The maximum recursion depth, the highest allowed level of nesting of arrays and maps. Defaults to 100.

### Returns
`string`: The serialized representation of the given value.

### Remarks
This function is a shortcut for creating an encoder instance to handle a single value. If you wish, the output of successive calls of this function may be concatenated to form a valid msgpack stream, but see `dumps_chain` for a more efficient way of doing that. The null handle will be serialized as the nil value thanks to the encoder's `write_value` method.

See the remarks of the encoder's constructor and `write_value` methods for the other arguments.

## mp_dumps_chain
Serialize the given list of value objects to a msgpack stream and return the complete contents as a string.
```
string mp_dumps_chain(mp_value@[]& v, bool sort_map_keys = false, bool strict_map_keys = true, uint max_recursion = 100);
```

### Arguments

- `mp_value@[]& v`: An array of value handles representing the values to be serialized.
- `bool sort_map_keys`: Whether the serialized representation of a map should be sorted, with its pairs in the order given by the sorting of its keys. Defaults to false.
- `bool strict_map_keys`: Whether the keys of maps should be constrained to the str and bin types (strict key mode). Defaults to true.
- `uint max_recursion`: The maximum recursion depth, the highest allowed level of nesting of arrays and maps. Defaults to 100.

### Returns
`string`: The msgpack stream formed by writing all of these values in the order given by the passed array.

### Remarks
This function is a shortcut for creating an encoder instance to handle multiple values all known ahead of time. Unlike passing a value of type array to `dumps`, the array here is not serialized as a msgpack array. Rather it operates by calling the temporary encoder's `write_value` method on each array item successively, producing a stream of values. Any null handles in the array will be serialized as the nil value thanks to the encoder's `write_value` method.

See the remarks of the encoder's constructor and `write_value` methods for the other arguments.

## mp_loads
Deserialize a value from a string containing a complete representation of a single value.
```
mp_value@ mp_loads(string data, bool strict_map_keys = true, uint max_recursion = 100);
```

### Arguments

- `string data`: The msgpack representation of the value you wish to deserialize.
- `bool strict_map_keys`: whether all map keys should be further constrained to be of the str or bin types (strict key mode). Defaults to true.
- `uint max_recursion`: The maximum recursion depth, the highest allowed level of nesting of arrays and maps. Defaults to 100.

### Returns
`mp_value@`: A loaded value object if deserialization was successful, or null if an error occurred.

### Remarks
This function is a shortcut for creating a decoder to deserialize a single value. The returned value will be null if any state other than **MPDS_READ** occurs in the temporary decoder after calling `try_read_value`, including if any exception is thrown. Any thrown exception will be swallowed by this function and does not propagate to the caller. If the passed string contains more than one serialized value, only the first will be returned, and the rest will be silently discarded. See `loads_chain` for a function for such a case.

See the remarks for the decoder's constructor for the other arguments.

## mp_loads_chain
Deserialize a stream of values from a string containing any number of complete serialized values.
```
mp_value@[]@ mp_loads_chain(string data, bool strict_map_keys = true, uint max_recursion = 100);
```

### Arguments

- `string data`: The msgpack representation of the stream of values you wish to deserialize.
- `bool strict_map_keys`: whether all map keys should be further constrained to be of the str or bin types (strict key mode). Defaults to true.
- `uint max_recursion`: The maximum recursion depth, the highest allowed level of nesting of arrays and maps. Defaults to 100.

### Returns
`mp_value@[]@`: A handle to an array containing all values that were able to be deserialized successfully.

### Remarks
This function is a shortcut for creating a decoder to deserialize a stream of values, if you know for a fact that you have the complete stream available at once. It will repeatedly call `try_read_value` and `get_value` on the temporary decoder as long as the states will let it, and will return upon the first error or other state transition it encounters. Any incomplete values (ones that result in **MPDS_MORE_DATA**) will cause the function to stop immediately, as will the invalid state and any exceptions thrown. When it returns, the list of values obtained up to that point will be returned, or an empty array if not even one value could be deserialized successfully. Any exceptions are swallowed by this function and do not propagate to the caller, and the returned array will never be null.

See the remarks for the decoder's constructor for the other arguments.

# Classes
## mp_value
Value objects are the basic unit of data in this library, allowing the storage and communication of the complete set of msgpack supported types. All deserialized data, and all data to be serialized, should ultimately be wrapped inside mp_value instances.  

Value objects support automatic conversion to the specified type, but note that attempting to coerce to the wrong type will throw `MP_TYPE_MISMATCH_EXCEPTION`. This conversion may be facilitated by the various `get_*` methods, by calling the type and passing the value, or by leveraging implicit conversion via assignment. This is done to support ease of use, but proper treatment is paramount!

If you wish to support serialization and deserialization of your own custom objects into msgpack, you should implement the `mp_value@ opImplCast()` and `mp_value@ opCast()` methods to yield serialized values from your objects, and a constructor of the form `obj(mp_value@ v)`, which will allow turning a deserialized value handle back into your object, via whatever semantics you deem necessary. In this form each serialized object should be only a single value, likely an array, map or ext type. It is not advised to serialize your objects as completely independent chains of values, as this makes conversion much harder and runs the potential of not knowing when you have enough data when the chain is broken. Implementing `opConv()` and `opImplConv()` is optional, but may allow slightly less verbose code if you wish to construct values from these yourself, before passing them on.

### Constructors

| Signature | Purpose | Notes |
| --- | --- | --- |
| `mp_value()` | Constructs the nil value, a value that corresponds to the msgpack nil type. |
| `mp_value(bool)` | Wraps the bool primitive. |
| `mp_value(float)` | Wraps the float primitive. |
| `mp_value(double)` | Wraps the double primitive. | (1) |
| `mp_value(string, bool is_bin = false)` | Wraps the string primitive. | (2) |
| `mp_value(mp_value@[]&)` | Wraps an array of value types into the msgpack array type. | (3) |
| `mp_value(mp_map&)` | Wraps a map object into the msgpack map type. | (3) |
| `mp_value(mp_ext&)` | Wraps an ext object into the msgpack extension type. | (4) |
| `mp_value(integer)` | Wraps any of the primitive integer types (u)int8/16/32/64. | (5) |

#### Notes

1. This may be wrapped as either the single-precision or double-precision format depending on the value passed in. If the double can safely be cast to a float with no loss of precision, it will be stored as one.
2. Throws an exception if the string is longer than (2^32)-1 bytes, as that is the highest length that can be represented in msgpack. The function of the **is_bin** boolean is to decide whether it should be packed as msgpack's str or bin types. By default, all strings are packed as text, and should be valid UTF-8. If it should be treated as an opaque bytestring, pass true as the second argument.
3. Throws an exception if the map or array has more than (2^32)-1 items. The passed in value should be a reference rather than a handle to avoid any passing of null. Note that it is possible to modify these objects after a value object wrapping them has been constructed, and this has the potential to change the format if the size has changed. Doing so up until the value has actually been passed to an encoder should be safe, as the value's format property will change to reflect any mutation it picks up. Do not rely on this behavior, however, as mutations may not actually reach the value. Instead, construct values as late as possible to avoid synchronization issues.
4. Throws an exception if the ext object's data is longer than (2^32)-1 bytes.
5. Any numeric values passed in will be coerced to the smallest type capable of representing that number before encoding, to take up the smallest number of bytes. In addition, all positive numbers will be coerced to unsigned types before this takes place, leaving negative numbers to use the signed encodings alone. As such, the format of the resulting value objects depends on the actual number rather than the type that was passed in.

### Methods
#### is_nil
Returns true if this value represents nil, false otherwise.
```
const bool is_nil();
```
##### Remarks
This method exists because nil has no proper value it could get to, and we do not allow null value handles in place of it.

#### get_*
Type-specific get methods, the explicit way to get that type from a value.

1. `bool get_bool();`
2. `string get_string();` (1)
3. `float get_float(bool allow_int_source = false);` (2)
4. `double get_double(bool allow_int_source = false);` (2)
5. `uint64 get_uint64();` (3)
6. `int64 get_int64();` (3)
7. `uint32 get_uint32()`, or equivalently `uint get_uint();` (3)
8. `int32 get_int32()`, or equivalently `int get_int();` (3)
9. `uint16 get_uint16();` (3)
10. `int16 get_int16();` (3)
11. `uint8 get_uint8();` (3)
12. `int8 get_int8();` (3)
13. `mp_value@[]@ get_array();` (4)
14. `mp_map@ get_map();` (4)
15. `mp_ext@ get_ext();`

##### Returns
The specified type if possible. If not possible, `MP_TYPE_MISMATCH_EXCEPTION` will be thrown.

##### Notes
1. This get method is used for both the str and bin types, as the function of these methods is based on the NVGT types they return. The value's type property will determine whether this should be treated as a text or binary string.
2. The **allow_int_source** parameter determines whether integer types will be coerced to floating point types (true) or whether attempting to do so will throw the type mismatch exception (false). Especially in the case of getting to **float**, loss of precision concerns apply!
3. All of these methods will function so long as the value is of type **MPT_INT**, automatic coercion is internally performed. This means that bits are copied directly and truncated or sign-extended as needed. In particular, this means that you are allowed to get an unsigned type to a signed type and vice versa, which can have undesirable behavior if done wrong! It is safe to get a signed or unsigned type to a larger signed type, and it is always safe to get a smaller type to a larger type if they are both the same signedness. Doing anything else runs the risk of producing unexpected values as a result of two's complement. If you are uncertain exactly what type the value was stored as and thus what conversions are safe, check the value's format property, which tells you exactly what binary representation the value uses, or check the signed property to determine its signedness alone if you intend to use a 64-bit return type.
4. Any mutation performed on the resulting handle may propagate back to the value object it came from, including if this object is stored inside some container. As mentioned above, this may cause the format property in particular to change.

### Properties
#### type
The type (format family) of this value expressed as a value from the **mp_type** enumeration.
```
const int type;
```

#### format
The format (specific representation) of this value expressed as a value from the **mp_format** enumeration.
```
const int format;
```

#### signed
Whether the format is a signed numeric value. Useful only for integers.
```
const int signed;
```
This property returns one of three possible values.

- 0: The format is an unsigned type, which is any of the **MPF_UINT** formats or **MPF_POS_FIXINT**.
- 1: The format is a signed type, which is any of the **MPF_INT** formats or **MPF_NEG_FIXINT**.
- -1: The format does not have sign bit semantics. True for every format family besides **MPT_INT** and **MPT_FLOAT**, where **MPT_FLOAT** always returns signed.

### Supported Operations
Value objects support equality comparison (`==`), which compares recursively by type and value. Importantly, this means that due to the type not matching, an integer value will never compare equal to a float value even if they are numerically equal, and likewise a string value will never compare equal to a bin value even if they have the same contents.
Different formats of the same type, however, such as **MPF_FLOAT32** and **MPF_FLOAT64**, will compare equal if the highest upcasts of them both compare equal. This applies to integers as well, which upcast to the 64-bit type of the same sign, except if one format is signed and the other is unsigned. They will always compare unequal in that case, even if they might be equal in their binary representations.

Value objects support implicit and explicit conversion to `bool`, `float`, `double`, `string`, (u)int8/16/32/64, `mp_value@[]`, `mp_map`, and `mp_ext`. They also support implicit and explicit casting to `mp_value@[]@`, `mp_map@`, and `mp_ext@`. In the case of casting to the wrong type, the cast will simply return null, standard for an invalid cast, rather than the type mismatch exception being thrown.

In the case of implicit or explicit conversion to string, all but array, map, and ext are supported. The others will yield a string representation suitable for their format. String and bin will be represented as-is, Floating point types will be stringified from double, integers will be stringified from uint64 or int64 depending on their sign, booleans will be represented by the strings "true" or "false", and nil will be represented by the string "null". Attempting to convert an array, map or ext to string will throw the type mismatch exception.

## mp_ext
An mp_ext object is used to represent msgpack extension types, which are a tuple of an 8-bit signed integer (the type code) and a byte string (the data payload).

### Constructor
Construct an ext object with the given type code and data. Note that the msgpack spec itself reserves type codes -128..-1, so only 0..127 should be used for application-specific types.
```
mp_ext(int8 type, string data);
```

### Properties
#### type
The type code, as an 8-bit signed integer.
```
const int8 type;
```

#### data
The payload of the extension type.
```
const string data;
```

### Supported Operations
Ext objects support equality comparison (`==`) which compares by type and data.

An ext object with the type code **MP_EXT_TIMESTAMP** can be cast to an **mp_timestamp** object.

An ext object with the type code **MP_EXT_VECTOR** can be converted to a **vector** object defined by NVGT, using the explicit conversion of the form `vector(ext)`.
A **vector** object can likewise be implicitly or explicitly converted to an ext object.  
The serialization of vectors, which is a custom implementation by this library, is achieved by using the type code 86 (ASCII V), and a payload consisting of the three floats (in network byte order) written in the order x y z.

## mp_timestamp
A timestamp representing seconds and nanoseconds since the unix epoch (1970-01-01T0:00+0:00) stored as the standard msgpack extension type -1, the constant **EXT_TIMESTAMP**.

Warning: Due to the fact that NVGT's timestamp precision is microseconds, round-tripping through NVGT timestamp objects is not lossless! The nanoseconds will be truncated to 0 and you will be left with microsecond precision.

### Constructors

| Signature | Purpose |
| --- | --- |
| `mp_timestamp(mp_ext &in);` | Construct an **mp_timestamp** object from an ext object of the correct type and format. If the passed in ext object is not the correct type and format, the type mismatch exception will be thrown. This constructor is used for explicit conversions. |
| `mp_timestamp(int64 seconds, uint nanoseconds);` | Construct an **mp_timestamp** object holding the specified timestamp, expressed in seconds since the epoch and nanoseconds since that second. Throws an exception beginning with "msgpack invalid timestamp" if nanoseconds is greater than 999999999. |
| `mp_timestamp(int64 microseconds);` | Construct an **mp_timestamp** object from the provided number of microseconds, which like the NVGT timestamp expresses the number of microseconds since the Unix epoch. |
| `mp_timestamp(timestamp&);` | Construct an **mp_timestamp** object based on the provided NVGT timestamp object. |

### Methods
#### as_microseconds
Expresses the time of this timestamp in microseconds since the epoch, compatible with NVGT's timestamp convention.
```
const int64 as_microseconds();
```

### Properties
#### seconds
The number of seconds since the epoch this timestamp represents, excluding leap seconds.
```
const int64 seconds;
```

#### nanoseconds
The number of nanoseconds remaining after the above number of seconds.
```
const uint nanoseconds;
```

### Supported Operations
Mp_timestamp objects support equality comparison (`==`) with **mp_ext** objects, and will return not equal if either the time stamps do not match or if the other ext object is not a timestamp. They support relational comparison (`==, !=, <, <=, >, >=`) with **mp_timestamp** and NVGT **timestamp** objects. In the case of comparing with NVGT **timestamp** objects, comparison is made with full nanosecond precision, rather than throwing away nanoseconds to compare at the microsecond level. Thus these objects will not compare equal unless the nanoseconds portion of this timestamp object only stores down to microsecond precision, and has the actual nanoseconds part at 0.

Mp_timestamp objects support explicit conversion to timestamp in the form `timestamp(obj)`, and both implicit and explicit casts and conversion to ext objects.

## mp_map
A map object is similar to a dictionary in function, and is required in order to serialize a key/value store into msgpack, as you cannot encode bare dictionaries.

Note that, unlike NVGT dictionaries, a msgpack map can have multiple types of objects for its keys, mentioned above. All non-string keys are coerced to strings using the string conversion provided by the value object, to allow you to still use strings when getting. If you want to add values with text(str) keys to a map you can use the set methods that take a string key. If you wish to use any other key type, including bin, you must use the `set_pair` method explicitly.

The map object attempts as best it can to mirror the dictionary API (including method aliases provided by NVGT) as well as having automatic type-choosing set and get methods for different types of objects (represented by '?' for argument types). But in order to grant full functionality it also provides extra methods. For simplicity, methods taken directly from the dictionary API will not have their own documentation written out fully, instead they will be listed below under dictionary equivalents. Consult the NVGT dictionary documentation for behaviors of such methods.

### Methods
#### Dictionary Equivalents

- `bool exists(string key);`
- `bool delete(string key);` and `void erase(string key);`
- `void delete_all();` and `void clear();`
- `uint get_size();` and `uint size();`
- `bool is_empty();` and `bool empty();`
- `string[]@ get_keys();`
- `bool get(string key, ? &out val);`
- `void set(string key, ? val);`

##### Remarks
The '?' argument type used in **get** and **set** is not truly capable of supporting any type of object, as dictionary is. Instead, it is constrained to the types of objects that value objects can hold. That is `bool`, `float`, `double`, `string`, `(u)int8/16/32/64`, `mp_value@[]`, `mp_map`, and `mp_ext`.  
Furthermore, when getting to the array, map and ext types, the type to store the result in MUST be a handle, and should be passed to the get function including the `@`. That is something like the following.
```
// assume we have some map called m, containing another map under the key submap. Getting it must be done like this.
map@ submap;
bool s = m.get("submap", @submap);
```

In contrast, as with values, the set method requires a reference to the object in question rather than a handle, so no `@` decoration should be used on its argument. NVGT is capable of converting a handle into a reference, so if you are storing a variable containing one of these with the handle type, passing that variable to the set function should work with no issues, assuming this handle is not null.

Finally, the automatic get methods here will not perform the same string coercion as is done for keys, but require matching types, with the exception that (as with value conversion) you are allowed to get integers to floats, and any integer type can be gotten to any other. If the type does not match no exception will be thrown, the get method will simply return false, just as happens with dictionary.

#### Direct Value Get
Get the value type directly from a map, allowing you to do type checking and the like on it.
```
mp_value@ get(string key);
```

##### Arguments

- `string key`: The key to look up

##### Returns
A handle to a value object if the value exists under that key, or null if it does not.

##### Remarks
If the value is nil, a nil value will be returned. The only case where null will be returned is if the key does not exist.

#### Dictionary-style Direct Value Get
Functions like the above method, returning a bare value type, but doing so in a way compatible with the dictionary's get convention.
```
bool get(string key, mp_value@ &out outval);
```

##### Arguments

- `string key`: The key to look up
- `mp_value@ &out outval`: A value handle where the gotten value should be stored.

##### Returns
True if the get operation was successful, false if it was not.

##### Remarks
This call maps directly to the get method on the underlying dictionary. If it succeeds, the variable you passed will now point to a value object stored within the map. If it does not, the content of that variable is undefined. As with the other handle types, the variable and call must be formed as follows.
```
// assume map m containing key data
mp_value@ data = null;
bool s = m.get("data", @data);
```

#### set_pair
Set a key/value pair directly to a map.
```
void set_pair(mp_value@ key, mp_value@ v);
```

##### Arguments

- `mp_value@ key`: The key to use, expressed as a bare value.
- `mp_value@ v`: The value to map this key to.

##### Remarks
This method is needed if you wish to use key types other than msgpack's str type, potentially for interoperability. The key is coerced to a string for lookups, but the value you pass will be used directly when serializing. Importantly, if the key cannot be coerced to a string due to it being an array, map or ext type, the type mismatch exception from that string conversion will be thrown out to you. Furthermore, if the string coercion of this key matches that of any key already in the map, that key and value will be silently overwritten.

Null handles are automatically converted to nil values for storage, and this is done before the coercion mentioned above. This means that if you store a null key, to get it back you must call get with "null" as the key.

#### get_pairs
Expresses the map as a 2d array of key/value pairs, with optional sorting.
```
mp_value@[][]@ get_pairs(bool sort_keys = false);
```

##### Arguments

- `bool sort_keys`: Whether the keys should have their coerced strings sorted lexicographically. Defaults to false, which returns them in whatever order the underlying dictionary chooses.

##### Returns
`mp_value@[][]@`: A handle to a 2d array, with each subarray containing two items, the key and the value.

##### Remarks
The key sorting provided here is useful for equality checking and checksums, as dictionaries, and thus maps, are inherently unordered. As it sorts the string representations of keys, there is no guarantee that it will sort the same as some other implementation or language's sorting, which may handle non-string types differently. Modifying this array will not result in any modifications to the map it came from, the handle type is merely intended to avoid extra copying.  
This method is primarily used by equality checking and the encoder, but is documented here as it may be useful to users as well.

#### set (with only bare value)
The opposite of the above direct value get functions, uses a string key but lets you set a bare value object as the value.
```
void set(string key, mp_value@ v);
```

##### Arguments

- `string key`: The key to use
- `mp_value@ v`: The bare value to map this key to.

##### Remarks
As with **set_pair**, a null value handle is automatically converted to the nil value before storage. The key will be stored and serialized as the text (str) type.

#### Explicit Setters (set_*)
Set methods with named types, rather than mere deduction based on their second argument.

1. `void set_nil(string key);` (1)
2. `void set_array(string key, mp_value@[] & v);` (2)
3. `void set_map(string key, mp_map& v);` (2)
4. `void set_ext(string key, mp_ext& v);` (2)
5. `void set_string(string key, string v);` (3)
6. `void set_bin(string key, string v);` (4)
7. `void set_bool(string key, bool v);`
8. `void set_float(string key, float v);`
9. `void set_double(string key, double v);`
10. `void set_uint8(string key, uint8 v);` (5)
11. `void set_int8(string key, int8 v);` (5)
12. `void set_uint16(string key, uint16 v);` (5)
13. `void set_int16(string key, int16 v);` (5)
14. `void set_uint32(string key, uint32 v);` or `void set_uint(string key, uint v)` (5)
15. `void set_int32(string key, int32 v);` or `void set_int(string key, int v);` (5)
16. `void set_uint64(string key, uint64 v);` (5)
17. `void set_int64(string key, int64 v);` (5)

##### Notes

1. Only set method that takes no value argument, as it simply maps that key to a nil value. Equivalent to `mp_map.set(key, null);` which itself is equivalent to `mp_map.set(key, @mp_value());`. Provided to be more explicit and easier to read.
2. As with value constructors, required to be a reference so you cannot pass null. Tracking mutations is not guaranteed.
3. Stores value as the text (str) type.
4. Stores value as the binary (bin) type. Equivalent to `mp_map.set(key, @mp_value(v, true));`. Provided to be more explicit and easier to read.
5. Unlike the overloaded set method, which will take your variable type as-is and convert based on that, these methods will coerce your variable to the type of the argument and work based on that. The same size and sign conversions apply.

##### Remarks
These methods are for setting types explicitly rather than using the automatic conversion presented by the regular overloaded set method, and without requiring you to use the bare value methods. That conversion is done internally. They are provided for convenience, and internally map to calling the above set method with a string key and bare value created from the v argument.

#### has_key
Checks if the map contains the given key, with a stronger equality guarantee on the key.
```
bool has_key(mp_value@ key);
```

##### Arguments

- `mp_value@ key`: The key to look up

##### Returns
`bool`: True if this key is in the map, false otherwise.

##### Remarks
This method functions similarly to `exists`, but whereas `exists` will return true if the given string key is found in the map, `has_key` will only return true if the key found under that string coersion is equal to the passed key. Thus it may return false where `exists` would return true, because the raw values do not agree. This method is useful for validation and to detect string collisions.

### Supported Operations
Map objects support equality comparison (`==`), which compares recursively by contents. Importantly this applies to the bare types of keys as well, not their string coersion. Two maps will only compare equal if every detail of their keys and values agree.

## mp_decoder
An mp_decoder object reads data from a msgpack stream and yields deserialized values obtained from that data in the form of value objects.  
Note that decoding is a multi-step process due to the fact that msgpack streams can be formed of concatenated msgpack streams, as well as the nature of a streaming format where not all data may be available upon initialization of the decoder.

### Constructor
Create a decoder ready for use, with optional initial data to pass right away.
```
mp_decoder(string initial = "", bool fixed_length = false, bool strict_map_keys = true, uint max_recursion = 100);
```

#### Arguments

- `string initial`: The initial data to pass to the decoder, the beginning of the stream you wish to decode. This will be fed to its internal buffer just as if you had initialized it empty and then pushed this block. Defaults to empty, no initial data available.
- `bool fixed_length`: Whether the decoder should operate in fixed-length mode. Defaults to false.
- `bool strict_map_keys`: Whether the decoder should further constrain all map keys to be of the str or bin types (strict key mode). Defaults to true.
- `uint max_recursion`: The maximum recursion depth, the highest allowed level of nesting of arrays and maps. Defaults to 100.

#### Remarks
In fixed-length mode, the decoder will assume that the provided initial data is the totality of the stream that should be decoded, and further pushing is disallowed. It will also automatically transition to the **MPDS_END_DATA** state upon reading the final value in the stream, even before `read_format()` is called.

If strict key mode is enabled (the default), the decoder will throw the invalid key exception if it encounters any key types other than str and bin. If it is disabled, the decoder will accept all types except those of array, map and ext for map keys.
These key types are preserved in the map object and can be re-serialized just as they are, but internal coercion to strings for lookups and storage runs the risk of a collision with some other key represented the same way, and exactly which value is preserved at that point is entirely dependent on the undefined ordering of a map.  
Note that this collision can still occur in strict key mode if a key of type str and another of type bin have the exact same contents, but it is hoped that this is much less likely to occur in any sane usage.

The maximum recursion depth limits the number of recursive decoders created to handle reading arrays and maps, which may be arbitrarily nested according to msgpack. As these are handled by making use of recursion on the NVGT side, too much nesting could lead to dramatically increased memory usage, performance degradation, or the worst case, a stack overflow error in NVGT when recursive function calls go too deep. This limit allows you to bail out at a sane level of nesting before that happens, and the default of 100 should be more than anyone needs. You may set it even lower or higher for your own use cases and constraints.

The maximum recursion level and strict key mode can only be set during object creation, meaning they are not valid arguments to the reset method. They are used for the lifetime of any decoder object. If you wish to operate under a different set of constraints, you will have to create another decoder object.

### Methods
#### reset
Reset the decoder state and clear all internal buffers, preparing it as new.
```
void reset(string initial = "", bool fixed_length = false);
```

##### Arguments

- `string initial`: The initial data to pass to the decoder, the beginning of the stream you wish to decode. This will be fed to its internal buffer just as if you had initialized it empty and then pushed this block. Defaults to empty, no initial data available.
- `bool fixed_length`: Whether the decoder should operate in fixed-length mode. Defaults to false.

##### Remarks
See constructor for the meanings of the arguments.

Msgpack decoders internally make use of a datastream to handle the translation between bytes and representations the code can work with. All push calls simply append data to this internal buffer, but reading data from it does not cause it to be cycled out. Thus it is advised that you always call reset after you are finished working with a particular burst of streamed msgpack, which re-initializes the internal stream and thus empties this buffer. Failure to do so can result in a memory leak as the stream accumulates more and more data.  
As part of re-initializing the internal stream and states, any values in the process of being decoded are thrown away and irrecoverable, including any recursive decoding for maps and arrays. After a call to reset, the decoder will always be in the state **MPDS_READY**, ready to begin anew.

The maximum recursion depth and strict key mode are not altered by this method, and continue to hold the values passed during object creation.

#### push
Feed more data to the decoder.
```
bool push(string data);
```

##### Arguments

- `string data`: A non-empty string of bytes to be appended to the internal buffer, representing the continuation of the stream since the last push.

##### Returns
`bool`: Whether the decoder believes it can continue decoding now that this data is available.

##### Remarks
If the decoder is in fixed-length mode, push returns false immediately. Otherwise, it will write to its internal buffer and return true if it believes that decoding may be continued.

If the decoder could already have continued to decode data before you pushed this, then the return value is still true but has no relation to the pushed data. Else it decides whether it can do so based on how many bytes it can guess it needs.
If it aborted with **MPDS_MORE_DATA** in the middle of reading things it knows the size of, such as numeric values and the str, bin, and ext types, then true will only be returned if it has enough bytes to read the full value it was expecting. Else if it is in the middle of reading a map or array, where counts are given in terms of items rather than bytes, providing even a single byte may return true, as that may indeed allow it to continue.  
As such, a return value of true does not guarantee that the next read will be able to read the full value, it only suggests it. What it does guarantee, though, is that the decoder has transitioned out of the **MPDS_MORE_DATA** state, and thus further reads may be attempted.
If you are in the **MPDS_MORE_DATA** state and attempt a read, any of those methods will return **MPDS_INVALID_OPERATION** immediately, without changing the decoder's actual state. Once push returns true, reads may be continued as normal.

Multiple calls to push *MUST* pass contiguous, non-overlapping segments of the same encoded msgpack stream or a concatenation of valid, self-contained msgpack streams. Failure to do so may result in a malformed stream and the decoder behaving improperly as a result.

#### get_state
Returns the current state of the decoder.
```
const mp_decoder_state get_state();
```

##### Returns
`mp_decoder_state`: The current state the decoder is in.

##### Remarks
All potential states and their meanings are listed in the *mp_decoder_state* enumeration above, except note that **MPDS_INVALID_OPERATION** is only for the sake of return values and is not an actual state the decoder may transition to, so it will never be returned by this method.

#### get_value
Retrieves the last completely read value from the decoder and prepares it to read another.
```
mp_value@ get_value();
```

##### Returns
A handle to a populated value object if available, else null.

##### Remarks
This method should only be called when the decoder's state is **MPDS_READ**, at which point a loaded value handle will be returned. If it is called at any other time, null will be returned. As in other cases, should the value read be nil, a value object loaded with nil will be returned rather than null.

In addition to yielding the current value, this method resets the decoder's state to **MPDS_READY**, prepared for a further read, with one exception. If the decoder is in fixed-length mode and no further bytes are available, the decoder's state is instead reset to **MPDS_END_DATA**, forbidding all further reads.

#### discard_value
Throws away the last completely read value from the decoder and prepares it to read another.
```
bool discard_value();
```

##### Returns
`bool`: True if there was indeed a value to discard, false otherwise.

##### Remarks
This method functions very similarly to `get_value`, including its behavior in regards to the current state. It will return false if the current state is not **MPDS_READ**. It is basically equivalent to calling `get_value` with nowhere to actually store the value, thus throwing the handle away, but is more explicit about it.

#### try_read_value
Attempt to read a value from a msgpack stream, working through all steps possible.
```
mp_decoder_state try_read_value();
```

##### Returns
`mp_decoder_state`: The state of the decoder after the attempted read.

##### Remarks
This convenience method goes as far as it can through each possible state and corresponding read function below, in order to reduce the amount of boilerplate required to read a value from a msgpack stream. Once it has gone as far as it can, the decoder's current state will be returned. If it is **MPDS_READ**, then you are ready to call `get_value` and then potentially call this method again. Other potential states it may return are **MPDS_INVALID** to signify an invalid stream and jammed decoder, **MPDS_MORE_DATA** to signify that more data should be pushed before reading again, or **MPDS_END_DATA** to signify that the stream ran out of bytes immediately upon trying to read format. If any exceptions are thrown by the read methods it calls, they will be passed on to you as the caller, with the decoder's state having been set to **MPDS_INVALID**.

#### read_format
Read the format byte, serving as a header for each value, from the msgpack stream, and prepare logic based on it.
```
mp_decoder_state read_format(int &out outformat = void);
```

##### Arguments

- `int &out outformat`: An output parameter to fill with the format the byte represents. This argument is optional and will default to not communicating anything if not specified.

##### Returns
`mp_decoder_state`: The state of the decoder after attempting to read a format byte. The resulting state indicates what methods should be called next.

##### Remarks
Depending on the specific format encountered, the decoder will most likely need to read more data to build a whole value, but the specifics vary for each format. Sometimes it must read a count (which is also used for reading integers and floats), sometimes it must read an extension type code, sometimes it must read the contents of a string, bin, map or array, and sometimes it can transition to **MPDS_READ** immediately, which is the case for small integers, booleans and nil. It can also return **MPDS_INVALID** indicating an invalid format byte. attempting to call this method in any state other than **MPDS_READY** will return **MPDS_INVALID_OPERATION** immediately.

#### read_count
Read the count (size) or numeric value of the item indicated by the previously read format.
```
mp_decoder_state read_count(uint &out outcount = void);
```

##### Arguments

- `uint &out outcount`: An output parameter to fill with the length that was read, if the current format calls for a length in bytes or items. This parameter is optional, and will default to not communicating anything if not specified.

##### Returns
`mp_decoder_state`: The state of the decoder after attempting to read the count. The resulting state indicates what methods should be called next.

##### Remarks
This method reads either a length of a string or size of a container stored in 1, 2, or 4 bytes, or a numeric value stored in 1, 2, 4, or 8 bytes, whether integer or float. The output parameter only has meaning if a length was read (it has no meaning for numeric values) and simply tells the number that was read from the stream. Calling this method in any state other than **MPDS_COUNT** will return **MPDS_INVALID_OPERATION** immediately.

#### read_ext_type
Read an extension type code from the stream.
```
mp_decoder_state read_ext_type(int8 &out out_ext_type = void);
```

##### Arguments

- `int8 &out out_ext_type`: The type code of the extension type for this value. This parameter is optional, and will default to not communicating anything if not specified.

##### Returns
`mp_decoder_state`: The state of the decoder after attempting to read the extension type code. The resulting state indicates what methods should be called next.

##### Remarks
This method simply reads a signed 8-bit integer from the stream, which represents the type code for this extension type. After this is read, the bytes content of the extension will be read. This method is only used for extension types, and calling in any state other than **MPDS_EXT_TYPE** will return **MPDS_INVALID_OPERATION** immediately.

#### read_content
Reads the contents of a msgpack value from the stream.
```
mp_decoder_state read_content();
```

##### Returns
`mp_decoder_state`: The state of the decoder after attempting to read the content. The resulting state indicates what methods should be called next.

##### Remarks
This method is used to read the general content of a msgpack value once its size is known.  
For the str, bin, and ext types, it reads a fixed number of bytes specified by the count and format, and will bail out in the **MPDS_MORE_DATA** state immediately if not enough bytes are available to meet that expected size.  
For the map and array types, which specify their sizes in terms of the number of items they contain, it will go into a recursive reading mode and attempt to read as many values as it can, until it either reads all of the container it was told to or runs out of data. This repeats each time more data is pushed and it is called again.
This may result in the decoder seemingly being stuck in the **MPDS_CONTENT** state for many calls, alternating with **MPDS_MORE_DATA**. This is because any of the other states from the recursive decoder do not make it up to the root decoder to be visible to the caller, as that would break obvious logic of how the root decoder should be operated. Maps and arrays can also be nested arbitrarily.

If the decoder encounters a map with a key of type map, array or ext, **MP_INVALID_KEY_TYPE_EXCEPTION** will be thrown, and it will transition to **MPDS_INVALID**.
If the decoder is in strict key mode, the same will happen for keys of the integer, float, boolean and nil types, as the only valid types in strict key mode are str and bin.  
If the maximum recursion depth is exceeded, the exception **MP_RECURSION_LIMIT_EXCEPTION** will be thrown, and the decoder will transition to **MPDS_INVALID**.  
Should the recursive decoder ever end up in an invalid state, this will reach the root decoder and all of the container will be lost, no partial retrieval is possible.

If the content has been read completely, the decoder will transition to **MPDS_READ** and the value may then be obtained, as there are no steps after this for any kind of value.
Calling this method in any state other than **MPDS_CONTENT** will return **MPDS_INVALID_OPERATION** immediately.

## mp_encoder
An mp_encoder takes input values, in the form of mp_value objects, and serializes them into a msgpack stream.  
Note that unlike the decoder, the operation of an encoder is much simpler, due to the multiple steps of encoding and recursion being handled internally by the encoder itself, and the absence of the possibility of incomplete data.

### Constructor
Create an encoder ready for use, with optionally specified constraints.
```
mp_encoder(bool strict_map_keys = true, uint max_recursion = 100);
```

#### Arguments

- `bool strict_map_keys`: Whether the keys of maps should be constrained to the str and bin types (strict key mode). Defaults to true.
- `uint max_recursion`: The maximum recursion depth, the highest allowed level of nesting of arrays and maps. Defaults to 100.

#### Remarks
If strict key mode is enabled (the default), the encoder will throw the invalid key exception if you attempt to encode a map with any key types other than str and bin. If it is disabled, the encoder will accept all types except those of array, map and ext for map keys.

The maximum recursion depth limits the number of recursive encode calls for arrays and maps, which may be arbitrarily nested according to msgpack. As these are handled by making use of recursion on the NVGT side, too much nesting could lead to dramatically increased memory usage, performance degradation, or the worst case, a stack overflow error in NVGT when recursive function calls go too deep. This limit allows you to bail out at a sane level of nesting before that happens, and the default of 100 should be more than anyone needs. You may set it even lower or higher for your own use cases and constraints.  
For the case of encoding, the maximum recursion level is also a way to prevent an infinite loop, runaway memory usage, and stack overflow if a deliberate cycle is created in the data to be encoded, as the encoder does not keep track of all parent values it has encoded in the child calls. Msgpack data should always be acyclic, as the msgpack format defines no mechanism by which cycles could even be introduced much less represented.

The maximum recursion level and strict key mode can only be set during object creation and are used for the lifetime of any encoder object. If you wish to operate under a different set of constraints, you will have to create another encoder object.

### Methods
#### write_value
Serializes a bare value.
```
void write_value(mp_value@ v, bool sort_map_keys = false);
```

##### Arguments

- `mp_value@ v`: The bare value to be serialized.
- `bool sort_map_keys`: Whether the serialized representation of a map should be sorted, with its pairs in the order given by the sorting of its keys. Defaults to false.

##### Remarks
As with `mp_map.set`, a null handle will be automatically converted to the nil value. See the remarks for `mp_map.get_pairs()` on the sorting used.

If the maximum recursion depth is exceeded while encoding, **MP_RECURSION_LIMIT_EXCEPTION** will be thrown.  
If strict key mode is enabled and a map is encountered with a key type other than str or bin, the exception **MP_INVALID_KEY_TYPE_EXCEPTION** will be thrown.

This method handles any recursion present in values that hold maps or arrays, but has no facilities for allevatinging cycles. Any cycles will result in either a stack overflow or the recursion limit exceeded exception being thrown.

If any exception is thrown from this or any of the other writing functions, the content of the internal stream is undefined, and is likely to contain partial data. Any fully written values from prior calls can be recovered, but especially in the case of recursion limit exceeded, recovery is likely unfeasible for the partial value written with this call.

#### write
Serialize a value to the stream, automatically guessing what type to serialize.
```
void write(? v);
```

##### Arguments

- `? v`: An argument of type `bool`, `float`, `double`, `string`, `(u)int8/16/32/64`, `mp_value@[]`, `mp_map`, or `mp_ext`. This is the value to be serialized.

##### Remarks
This method guesses the type via a full set of overloads, just as `mp_map.set` does, and internally calls `write_value(@value(v))`.

#### Explicit Writers (write_*)
Write methods with named types, rather than mere deduction based on their argument.

1. `void write_nil();` (1)
2. `void write_array(mp_value@[]& v);` (2)
3. `void write_map(mp_map& v);` (2)
4. `void write_ext(mp_ext& v);` (2)
5. `void write_string(string v);` (3)
6. `void write_bin(string v);` (4)
7. `void write_bool(bool v);`
8. `void write_float(float v);`
9. `void write_double(double v);`
10. `void write_uint8(uint8 v);` (5)
11. `void write_int8(int8 v);` (5)
12. `void write_uint16(uint16 v);` (5)
13. `void write_int16(int16 v);` (5)
14. `void write_uint32(uint32 v);` or `void write_uint(uint v)` (5)
15. `void write_int32(int32 v);` or `void write_int(int v);` (5)
16. `void write_uint64(uint64 v);` (5)
17. `void write_int64(int64 v);` (5)

##### Notes

1. Only write method that takes no value argument, as it simply writes a nil value. Equivalent to `mp_encoder.write_value(null);` which itself is equivalent to `mp_encoder.write_value(value());`. Provided to be more explicit and easier to read.
2. As with value constructors, required to be a reference so you cannot pass null. Mutation tracking is not valid beyond this point, once the data is encoded it will not change. Any further mutations to the object will result in the object and the data serialized from it being out of sync. Exceptions thrown by these calls are likely to have left partial data in the buffer, constituting an invalid msgpack stream.
3. Stores value as the text (str) type.
4. Stores value as the binary (bin) type. Equivalent to `write_value(mp_value(v, true));`. Provided to be more explicit and easier to read.
5. Unlike the overloaded write method, which will take your variable type as-is and convert based on that, these methods will coerce your variable to the type of the argument and work based on that. The same size and sign conversions apply.

##### Remarks
These methods are for writing types explicitly rather than using the automatic conversion presented by the regular overloaded write method, and without requiring you to use the bare value method. As with the overloaded write method, they internally map to calling `write_value(@value(v))`.

#### read
Reads some number of bytes from the internal stream that data is written to.
```
string read(uint count = 0);
```

##### Arguments

- `uint count`: The number of bytes to read from the stream, or 0 for all bytes available, just as in datastream.

##### Returns
`string`: The data written to the internal stream of the requested size or lower, or the empty string if no more data is available.

##### Remarks
This method maps directly to the read method on the internal datastream, and thus will behave exactly as that one does, with newly written data appearing at the end. It is provided, with the count included, for easy chunking, by merely reading in the requested chunk size.

#### flush
Flushes all remaining data from the encoder's internal buffer and empties it.
```
string flush();
```

##### Returns
`string`: Any remaining data in the internal buffer

##### Remarks
This is not exactly the same as calling read(0), though it does that too and returns that. Instead, after doing that, it closes and re-initializes the internal stream, so that its buffer is empty again. Just like with `mp_decoder.reset`, it is advised you do this after working with each stream burst, or else you may leak memory.

#### Other Methods On The Underlying Stream
In addition to read, these methods are defined which map directly to identical calls on the underlying stream, so see the datastream documentation for details on their usage.

- `bool rseek(uint64);`
- `bool rseek_end(uint64 = 0);`
- `bool rseek_relative(int64);`

### Properties
The following properties are defined which map directly to the underlying stream of an encoder.

- `uint64 available;`
- `bool good;`
- `bool bad;`
- `bool fail;`
- `bool eof;`
- `int64 rpos;`
- `int64 wpos;`

# Debug Mode
The file debug.patch is provided to enable debug statements in the library when applied. Use `git apply debug.patch` to enable it and `git apply -R debug.patch` to disable it.
This debug mode is designed for testing and development of the msgpack library itself, in order to test correctness, and should not be used for production code. It should not even be used for code in testing that makes use of the msgpack library, unless you believe you've found a bug in the library's behavior.

# Contributing
If you spot any bugs in the library and/or documentation, or places where things could be improved, issues and pull requests are welcome.

The formatting used here is an attempt to conform to NVGT's code style, which itself is sort of enforced by Artistic Style using a config that comes with the NVGT repository. The changes I make to that style are removal of excessive blank lines in the middle of methods, removal of padding of the angle brackets used to denote the templated array type, and removal of any spaces between the name and opening parenthesis of the throw function. This formatting is subject to change slightly if better methods are discovered, but tabs are still to be used for indentation.

Any contribution must keep the file debug.patch able to be applied with it in order to be merged. If the pull request does not do so itself, I will attempt to do so.
This means that the best way of working on the library is to put it into debug mode first and then make your changes on top of that, adding new debug statements as necessary, and finally before merging strip those debug statements and update the patch.

The script debugstrip.py has been provided to aid with this once debug.patch no longer cleanly reverts, which will strip all lines containing dbgout and everything after the marker `/// BEGIN DEBUG ///`. The result of this should be diffed against the prior version to provide a new debug.patch, using a command similar to the following, assuming both are committed at least temporarily.
```
git diff --binary --histogram --output=debug.patch HEAD HEAD~
```
The resulting debug.patch should then be committed. Due to this noise in the history, if temporary branches are not used pull requests may be squash merged.