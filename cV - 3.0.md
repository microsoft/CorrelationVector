# Correlation Vector v3.0 [Draft]

**Correlation Vector** (a.k.a. **cV**) is a tracing standard that evolved internally at Microsoft, based on a light weight vector clock.

This document covers:

- the additional design goals of the cV that are addressed w.r.t the v2.1 specification
- the format used on the wire when transmitting the vector across components involved in a trace
- the protocols used for managing the vector and the specific operators involved

## Design Goals

Here are the key **additional** design goals:

1. Support interoperability with W3C Distributed Tracing [Link](https://github.com/w3c/distributed-tracing)
2. Support for vector reset semantics to support traces of arbitrary depth
3. Support for versioning

## Format

Represented by the following simply grammar ABNF specification

1. vector = ver delim base suffix
2. ver = "A"
3. base = 21char lastChar
4. char = %x30-39 / %x41-5A / %61-7A / %x2F / %2B ; base64 characters
5. lastChar = "A" / "Q" / "g" / "w" ; base64 characters with four least significant bits of zero
6. suffix = init / init vector
7. init = delim tick / sharp id delim tick / dash id delim tick
8. vector = term / term vector
9. term = delim tick / under id delim tick
10. tick = 1*8(%x30-39 / %x41-46) ; hex encoded counter
11. id = 16(%x30-39 / %x41-46) ; hex encoded id
12. delim = %x2E ; '.'
13. sharp = %x23 ; '\#' (reset operation indicative)
14. dash = %x2d ; '\-' (parent span id indicative)
15. under = %x5f ; '\_' (spin operation indicative)

**Maximum length**: 128 bytes (assuming a UTF-8 encoding)

**Note**\: The following characters are reserved for the specification \{ ‘.’, ‘\-’, ‘\_’, ‘\#’, ‘\!’ \}

**Note**: The encoded **_{Base}_** consists of 22 base64 characters and, if left unrestricted, technically allows for 132-bit values. In order to support interoperability with tracing standards that use UUIDs, the **_{Base}_** value is restricted to 128-bits assuring reliable conversion to/from the UUID format. This is the reason the last element of the **_{Base}_** is restricted to a subset of allowable base64 characters.

**Note**: Base64 is case sensitive. Since the cV base is base64 encoded, 'a' is distinct from 'A' in the base. Counters and IDs are encoded in hexadecimal where there is no such distinction. However, this indifference does not allow us to apply a lexical sort. So by convention and for a more accessible sort, the hexadecimal encoding is restricted to only **upper case** characters, when used for IDs or counters.

### Examples

1. A.PmvzQKgYek6Sdk/T5sWaqw.0 // Base: PmvzQKgYek6Sdk/T5sWaqw; Version: A
2. A.PmvzQKgYek6Sdk/T5sWaqw.B
3. A.e8iECJiOvUGPvOVtchxG9g.F.A.23 // All elements in the vector are logical ticks that resulted from an Increment operation
4. A.e8iECJiOvUGPvOVtchxG9g-304773F68A307E98.1.F.A.234 // 304773F68A307E98 is a Span ID from the incoming W3C TraceParent header
5. A.e8iECJiOvUGPvOVtchxG9g.1.F.A.23_93816B91E430A7BB.1 // 93816B91E430A7BB was generated during a Spin operation
6. A.e8iECJiOvUGPvOVtchxG9g#B6A5FFD77977E2AE.0 // B6A5FFD77977E2AE was generated during a Reset operation

## Operators

To help define the vector operators are the following definitions and notations:

- X is a valid base, i.e. 22 base64 encoded characters
- S is the correlation vector suffix, i.e. the suffix section excluding the base. A.**XS** would denote a valid cV 3.0.
- V is a valid correlation vector.
- N is a valid 4-byte unsigned integer
- M is an 8-byte unsigned integer designed to offer a combination of sort and entropy and constructed as follows:
  - M is composed of two 4-byte sections referred to as AB where A is the most significant section that is derived from UTC time as follows:
    - If x is the UTC time in Ticks,
      - Drop the least significant N bits of x to create a tick that increments, where N is the amount of bits dropped based on the Spin Parameter **Spin Counter Interval**.
    - Select the least significant N bits from the output of the previous step that yields a N-bit coarse tick counter where N is the amount of bits indicated in the Spin Parameters in **Spin Counter Periodicity**.

    Using a coarse spin interval, A therefore reflects a coarse tick counter that increments approximately every 6.5 milliseconds and overflows once in a little over 328 days.
  - B is a randomly generated unsigned integer whose size is indicated in the Spin Parameter **Spin Entropy**. 32 bits of entropy yields a probability of collision of 1.15% for 10,000 trials. 

The use case for M is described in the Spin operator section. It is also used when resetting the correlation vector.

### Seed

**Definition**: {} => A.**X**.0

Invoked when initializing the vector.

### Increment

**Definition**: **V**.N => **V**.(N+1)

The Increment operator may be invoked arbitrarily to denote a logical tick. But it must be invoked prior to the creation of a new outgoing control flow to generate a unique vector value suitable for transmission such as, on an outgoing HTTP/RPC call.

Failure to increment on an outgoing span may result in child spans with identical vector values which breaks trace reconstruction.

If an increment is not possible, the workaround is to have the component that receives the vector value implement a Spin operation instead of an Extend (see below).

### Increment Examples

1. A.PmvzQKgYek6Sdk/T5sWaqw.9 => A.PmvzQKgYek6Sdk/T5sWaqw.A
2. A.PmvzQKgYek6Sdk/T5sWaqw.1.F.A.23 => A.PmvzQKgYek6Sdk/T5sWaqw.1.F.A.24
3. A.PmvzQKgYek6Sdk/T5sWaqw-304773F68A307E98.4 => A.PmvzQKgYek6Sdk/T5sWaqw-304773F68A307E98.5
4. A.PmvzQKgYek6Sdk/T5sWaqw.1.F.A.23_B6A5E62FC38E9974.1 => A.PmvzQKgYek6Sdk/T5sWaqw.1.F.A.23_B6A5E62FC38E9974.2
5. A.PmvzQKgYek6Sdk/T5sWaqw#B6A5FFD77977E2AE.0 => A.PmvzQKgYek6Sdk/T5sWaqw#B6A5FFD77977E2AE.1

### Extend

The Extend operator is to be used once on an incoming control flow, to extend the vector to create a vector element for the receiving span, such as on an incoming HTTP/RPC call.

**Definition**: **V** => **V**.0

### Extend Examples

1. A.PmvzQKgYek6Sdk/T5sWaqw.9 => A.PmvzQKgYek6Sdk/T5sWaqw.9.0
2. A.PmvzQKgYek6Sdk/T5sWaqw.1.F.A.23 => A.PmvzQKgYek6Sdk/T5sWaqw.1.F.A.23.0
3. A.PmvzQKgYek6Sdk/T5sWaqw-304773F68A307E98.4 => A.PmvzQKgYek6Sdk/T5sWaqw-304773F68A307E98.4.0
4. A.PmvzQKgYek6Sdk/T5sWaqw.1.F.A.23_B6A5E62FC38E9974.1 => A.PmvzQKgYek6Sdk/T5sWaqw.1.F.A.23_B6A5E62FC38E9974.1.0
5. A.PmvzQKgYek6Sdk/T5sWaqw#B6A5FFD77977E2AE.1 => A.PmvzQKgYek6Sdk/T5sWaqw#B6A5FFD77977E2AE.1.0

### Spin

The Spin operator is used instead of the Extend operator, in situations where the parent span cannot atomically increment the vector for each child span.

Here are some common examples:

1. Publisher-consumer control flow patterns where the parent span in the publisher may not aware of the number of consumers, which can be greater than 1
2. Components that consume queues denoting incoming control flows and which support multiple dequeue retry attempts and without the capability for maintaining local state for tracking previous dequeue attempts in the current span.
3. Bad actor components which repeatedly pass along the same vector value on outgoing control flows without increments.
4. Very high throughput components where the interlocked increment operation is a significant overhead.

In such instances, without a uniquely incremented value on each incoming flow, the uniqueness constraint that promulgates that each span has its own unique cV is violated. This impacts downstream diagnostics as identical vector values are repeated on multiple spans.

The Spin operator mitigates this, by inserting an element with entropy so that repeated invocations result in unique values. However purely using entropy erodes the utility of the vector when it is used to provide a sort during data processing. For this reason, the construction precedes this with an element derived from UTC time. This supports the sort property of the vector while reducing the chances of collision.

**Definition**: **V** => **V**\_**M**.0

#### Spin Parameters

The output of a cV can be changed to output different values.

**Spin Counter Interval**: There are two different options for this:
- **Coarse** indicates that the 24 least significant bits of the UTC time counter are dropped.
- **Fine** indicates that the 16 least significant bits of the UTC time counter are dropped.

If the computer increments UTC time based on ticks, where 10 million ticks is equal to 1 second, **Coarse** will increment the spin counter every 1.67 seconds \(2\^24\*10\^\-7 seconds\) and **Fine** will increment the spin counter every 6.5 milliseconds \(2\^16\*10\^\-7 seconds\).

**Spin Counter Periodicity**: This indicates how many bits of the counter value described in the first part of **M** will be stored.
- **None** stores no \(0\) bits of the counter.
- **Short** stores 16 bits (2 bytes).
- **Medium** stores 24 bits (3 bytes).
- **Long** stores 32 bits (4 bytes).

**Spin Entropy**: This indicates how many bits of entropy described in the second part of **M** will be stored.
- **None** generates no \(0\) bits of entropy.
- **One** generates 8 bits (1 byte) of entropy.
- **Two** generates 16 bits (2 bytes) of entropy.
- **Three** generates 24 bits (3 bytes) of entropy.
- **Four** generates 32 bits (4 bytes) of entropy.

#### Spin Examples

1. A.PmvzQKgYek6Sdk/T5sWaqw.9 => A.PmvzQKgYek6Sdk/T5sWaqw.9_B6A6A13E588CF82F.0
2. A.PmvzQKgYek6Sdk/T5sWaqw.1.F.A.23 => A.PmvzQKgYek6Sdk/T5sWaqw.1.F.A.23_B6A6A13E588CF82F.0
3. A.PmvzQKgYek6Sdk/T5sWaqw-304773F68A307E98.4 => A.PmvzQKgYek6Sdk/T5sWaqw-304773F68A307E98.4_B6A6A13E588CF82F.0
4. A.PmvzQKgYek6Sdk/T5sWaqw.1.F.A.23_B6A5E62FC38E9974.1 => A.PmvzQKgYek6Sdk/T5sWaqw.1.F.A.23_B6A5E62FC38E9974.1_B6A6A13E588CF82F.0
5. A.PmvzQKgYek6Sdk/T5sWaqw#B6A5FFD77977E2AE.1 => A.PmvzQKgYek6Sdk/T5sWaqw#B6A5FFD77977E2AE.1_B6A6A13E588CF82F.0

### Reset

The Reset operator is meant to be invoked implicitly, whenever an operation can potentially result in the total vector length exceeding its maximum length. The mutation is specific to the operation that is being attempted, with the basic goal of truncating the vector.

- Increment: A.**XS**.N => A.**X**#**M**.(N+1)
- Extend: A.**XS** => A.**X**#**M**.0
- Spin: A.**XS** => A.**X**#**M**.0

Note: When Reset is invoked in the context of the Spin operator, the operations are identical to those of Extend. The reason for this is that the Reset operator already introduces a sort-entropy element (by introducing M) that makes the vector value unique to the span, thereby making the addition of another such element moot.

In all cases, the Reset Operation records S, i.e. the value associated with the previous value of the vector. This value needs to be recorded for reconstructing the trace and is associated with its replacement M. In other words, the association tuple (S, M) needs to be captured in the trace in conjunction with invoking the Reset operator. S does not include the extension; the extension is preserved for the resultant reset vector.

While Reset also uses **M**, there are no reset parameters; it always assumes a Long spin counter periodicity and a Long spin entropy (64 bits or 8 bytes). However, the spin counter interval is up to the implementation.

### Reset Examples

Let:

- **X** = PmvzQKgYek6Sdk/T5sWaqw
- **S** = .1.FA.A1.23_B6A5E62FC38E9974.1_B6A6A13E588CF82F.2A.AB.213_B6A92D24A00C0F9B.47.8B.12.34.A123.2B.23.41A

**XS** is 123 bytes (assuming ASCII encoding and 1 byte per character). A.**XS** is 125 bytes, and A.**XS**.F is 127 bytes, where F is the hexadecimal value 0xF. Incrementing, extending, or spinning will reset the correlation vector.

- Increment: A.**XS**.F => A.**X**#B6B3AB078D8000FA.10
  - Recorded mapping: **S** <=> B6B3AB078D8000FA
- Extend: A.**XS**.F => A.**X**#B6B3AB078D8000FA.0
  - Recorded mapping: **S**.F <=> B6B3AB078D8000FA
- Spin: A.**XS**.F => A.**X**#B6B3AB078D8000FA.0
  - Recorded mapping: **S**\_B6B3AB07F13A1230 <=> B6B3AB078D8000FA

Note that the properties of M as defined above allows for sorting events across the trace reset boundary, even in the absence of intermediate spans that contain the previous value of the vector. This is based on the time component that is the most significant 4 bytes in M.

## Key differences with cV 2.1

Summarizing the key differences from v2.1:

1. The base is preceded with a single base64 character for version and reserved for future use. For 3.0, this is simply 'A' (base64 0).
2. The vector element counters are encoded in Hex unlike the decimal encoding in v2.1. They continue to be 4 byte unsigned counters.
3. Support for two **optional** vector prefix segments that use a reserved character followed by an 8 byte hex encoded identifier to support

   - Reset (prefixed by '#')
   - W3C interoperability (prefixed by '-')

4. Support for **optional** prefix sections on each vector element to support Spin (prefixed by '_') followed by a 8 byte hex encoded identifier.
5. With the Reset operator, the cV 3.0 format can support causality chains of indefinite depth. It is automatically invoked when other operations can exceed the maximum length. This arrangement means that the updated format does not rely on the '!' character in v2.1 to indicate further immutability.

## Interoperability with cV 2.1

The typical use case is when a system using cV v3.0 receives a call from a component that uses v2.1.

For the incoming v2.1 cV of the form X.V, where:

- **X** is a compliant 22 base64 encoded base
- **V** is the '.' delimited vector

The equivalent v3.0 cV has the form: A.**X**.**V**

**Note**: While it is possible to convert a cV 3.0 to cV 2.1, it is **not** recommended for a system using cV 3.0 to propagate to a system with cV 2.1. This is primarily on account of the more compact hex encoding on the v3.0 format which can generate equivalent cV 2.1 (that uses decimal) values that exceed maximum length.

### cV 2.1 to cV 3.0 conversion examples

1. PmvzQKgYek6Sdk/T5sWaqw.0 => A.PmvzQKgYek6Sdk/T5sWaqw.0
2. e8iECJiOvUGPvOVtchxG9g.1.23 => A.e8iECJiOvUGPvOVtchxG9g.1.23

### cV 2.1 with trailing '!'

The presence of the trailing '!' character indicates that the incoming vector is immutable as per the 2.1 specification. The recommended approach when dealing with such a cV in the 2.1 specification would be to create a new cV (with a new base using the Seed operator) and emit the association between the new and old cVs for trace reconstruction.

To convert such a v2.1 cV to v3.0, invoke the Reset operator after adding the version prefix.

### Immutable cV 2.1 to cV 3.0 conversion example

**cV 2.1**: CgOLQOn9Gkmd4pM720ciZA.1.15.3226329855.4111101367.10.23.8.3226332926.1671828776.2345.12.3.243.544.3226336576.3422508575.23.1.34!

=>

**cV 3.0**: A.CgOLQOn9Gkmd4pM720ciZA#B6B3AB078D8000FA.0

- Recorded mapping: .1.15.3226329855.4111101367.10.23.8.3226332926.1671828776.2345.12.3.243.544.3226336576.3422508575.23.1.34! <=> B6B3AB078D8000FA

## Interoperability with W3C

The primary [W3C](https://github.com/w3c/distributed-tracing) header that communicates the position of the incoming request in a trace is **traceparent**, which has the following format:

**version-format** "-" **trace-id** "-" **parent-id** "-" **trace-flags**

For the purposes of interoperability, we are primarily concerned with mapping **trace_id** and **parent-id** to their respective analogs in the cV format.

Semantically,

- **trace_id** maps to the cV base. They have identical entropy (at least in the current versions).
- **parent-id** maps to the incoming parent vector, i.e. the vector without its most recent extension.

### TraceParent to cV 3.0

The typical use case is when a cV instrumented system receives an incoming call with traceparent header.

Generating a cV from an incoming traceparent value, has the following construction steps:

- Append with the cV 3.0 version 'A.'.
- Encode the trace_id into base64 and append it.
- Append the '-' character denoting the W3C interoperability.
- Convert the parent_id to upper case, and append it.
- Append a new vector counter initialized to zero, with the '.' delimiter.

The generated cV will have the following form: A.\[**encode_base64(trace_id)**\]\-\[**uppercase(parent_id)**\].0

### W3C to cV 3.0 conversion example

**traceparent**: 00-0af7651916cd43dd8448eb211c80319c-b9c7c989f97918e1-01

=> 

**cV 3.0**: A.CvdlGRbNQ92ESOshHIAxnA-B9C7C989F97918E1.0

### cV 3.0 to TraceParent

The typical use case is when a cV instrumented system is making a call out to component that only understands the W3C headers.

The cV encodes more information in the vector than that encoded in the parent_id in traceparent. For this reason, when converting to cV 3.0, we generate a new span ID and associate this with the span.

Generating a traceparent from an existing cV 3.0 value, has the following construction steps:

- Start with a prefix for the appropriate version format. This is currently "00-".
- Encode the cV base (without the version character) into lower case hexadecimal.
- Append a "-" character.
- Generate a new span ID (random generation, 8 byte, lower case hexadecimal encoded)
  - Emit the association between the vector segment and the newly created span ID for trace reconstruction.
- Append the span ID.
- Append a "-" character
- Append suitable trace-flags as defined in the W3C specification. Default would be "00".

The generated traceparent will have the following form: 00-[**lowercase(encode_hex(cVBase))**]-[**lowercase(random_span_id)**]-[*trace-flags*]

Note that the cV should have been incremented prior to the traceparent generation for outgoing spans.

### cV 3.0 to W3C conversion example

**cV 3.0**: A.PmvzQKgYek6Sdk/T5sWaqw.1.F.A.23_B6A5E62FC38E9974.2

=>

**traceparent**: 00-3e6bf340a8187a4e92764fd3e6C59aab-10f076ab0ba9d1c9-00

- Recorded mapping: .1.F.A.23_B6A5E62FC38E9974.2 <=> 10f076ab0ba9d1c9
