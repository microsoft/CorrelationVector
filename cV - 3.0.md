# Correlation Vector v3.0 [Draft]

**Correlation Vector** (a.k.a. **cV**) is a tracing standard that evolved internally at Microsoft, based on a light weight vector clock.

This document covers:

- the additional design goals of the cV that are addressed w.r.t the v2.1 specification
- the format used on the wire when transmitting the vector across components involved in a trace
- the protocols used for managing the vector and the specific operators involved

## Design Goals

Here are the key **additional** design goals:

1. Support interop with W3C Distributed Tracing [Link](https://github.com/w3c/distributed-tracing)
2. Support for vector reset semantics to support traces of arbitrary depth
3. Support for versioning

## Format

Represented by the following simply grammar ABNF specification

1. vector = base ver delim suffix
2. base = 21char lastChar
3. ver = "A"
4. char = %x30-39 / %x41-5A / %61-7A / %x2F / %2B ; base64 characters
5. lastChar = "A" / "Q" / "g" / "w" ; base64 characters with four least significant bits of zero
6. suffix = init / init vector
7. init = tick / sharp id delim tick / dash id delim tick
8. vector = term / term vector
9. term = delim tick / under id delim tick
10. tick = 1*8(%x30-39 / %x41-46) ; hex encoded counter
11. id = 16(%x30-39 / %x41-46) ; hex encoded id
12. delim = %x2E ; '.'
13. sharp = %x23 ; '#' (reset operation indicative)
14. dash = %x2d ; '-' (parent span id indicative)
15. under = %x5f ; '_' (spin operation indicative)

**Maximum length**: 128 bytes (assuming a UTF-8 encoding)

**Note**: The following characters are reserved for the specification { ‘.’, ‘-’, ‘_’, ‘#’, ‘!’ }

**Note**: The encoded **_{Base}_** consists of 22 base64 characters and, if left unrestriced, technically allows for 132-bit values. In order to support interop with tracing standards that use UUID's, the **_{Base}_** value is restricted to 128-bits assuring reliable conversion to/from the UUID format. This is the reason the last element of the **_{Base}_** is restricted to a subset of allowable base64 characters.

### Examples

1. PmvzQKgYek6Sdk/T5sWaqwA.0 // Base: PmvzQKgYek6Sdk/T5sWaqwA
2. PmvzQKgYek6Sdk/T5sWaqwA.B
3. e8iECJiOvUGPvOVtchxG9g1A.F.A.23 // All elements in the vector are logical ticks that resulted from an Increment operation
4. e8iECJiOvUGPvOVtchxG9gA-304773F68A307E98.1.F.A.234 // 304773F68A307E98 is a Span Id from the incoming W3C TraceParent header
5. e8iECJiOvUGPvOVtchxG9gA.1.F.A.23_93816B91E430A7BB.1 // 93816B91E430A7BB was generated during a Spin operation
6. e8iECJiOvUGPvOVtchxG9g1A#B6A5FFD77977E2AE.0 // B6A5FFD77977E2AE was generated during a Reset operation

## Operators

To help define the vector operators are the following definitions and notations:

- X is a valid base, i.e. 23 base64 encoded characters (terminating with 'A' indicative of version)
- S is the correlation vector suffix, i.e. the suffix section excluding the base. XS would denote a valid correlation vector.
- V is a valid correlation vector
- N is a valid 4-byte unsigned integer
- M is an 8-byte unsigned integer designed to offer a combination of sort and entropy and constructed as follows:
  - M is composed of two 4-byte sections referred to as AB where A is the most significant section that is derived from UTC time as follows:
    - If x is the UTC time in Ticks,
      - Drop the least significant 16 bits of x to create a coarse tick that increments approximately every 6.5 milliseconds
    - Select least significant 32 bits from the output of the previous step that yields a 32-bit coarse tick counter where each tick is approximately 6.5 milliseconds

    A therefore reflects a coarse tick counter that increments approximately every 6.5 milliseconds and overflows once in a little over 328 days.
  - B is the least significant 4-byte section and is randomly generated. 32 bits of entropy yields a probability of collision of 1.15% for 10,000 trials.

The use case for M is described in the Spin operator section.

### Seed

**Definition**: {} => X.0

Invoked when initializing the vector.

### Increment

**Definition**: V.N => V.(N+1)

The Increment operator may be invoked arbitrarily to denote a logical tick. But it must be invoked prior to the creation of a new outgoing control flow to generate a unique vector value suitable for transmission such as, on an outgoing HTTP/RPC call.

Failure to increment on an outgoing span may result in child spans with identical vector values which breaks trace reconstruction.

If an increment is not possible, the workaround is to have the component that receives the vector value implement a Spin operation instead of an Extend (see below).

### Increment Examples

1. PmvzQKgYek6Sdk/T5sWaqwA.9 => PmvzQKgYek6Sdk/T5sWaqwA.A
2. PmvzQKgYek6Sdk/T5sWaqwA.1.F.A.23 => PmvzQKgYek6Sdk/T5sWaqwA.1.F.A.24
3. PmvzQKgYek6Sdk/T5sWaqwA-304773F68A307E98.4 => PmvzQKgYek6Sdk/T5sWaqwA-304773F68A307E98.5
4. PmvzQKgYek6Sdk/T5sWaqwA.1.F.A.23_B6A5E62FC38E9974.1 => PmvzQKgYek6Sdk/T5sWaqwA.1.F.A.23_B6A5E62FC38E9974.2
5. PmvzQKgYek6Sdk/T5sWaqwA#B6A5FFD77977E2AE.0 => PmvzQKgYek6Sdk/T5sWaqwA#B6A5FFD77977E2AE.1

### Extend

The Extend operator is to be used once on an incoming control flow, to extend the vector to create a vector element for the receiving span, such as on an incoming HTTP/RPC call.

**Definition**: V => V.0

### Extend Examples

1. PmvzQKgYek6Sdk/T5sWaqwA.9 => PmvzQKgYek6Sdk/T5sWaqwA.9.0
2. PmvzQKgYek6Sdk/T5sWaqwA.1.F.A.23 => PmvzQKgYek6Sdk/T5sWaqwA.1.F.A.23.0
3. PmvzQKgYek6Sdk/T5sWaqwA-304773F68A307E98.4 => PmvzQKgYek6Sdk/T5sWaqwA-304773F68A307E98.4.0
4. PmvzQKgYek6Sdk/T5sWaqwA.1.F.A.23_B6A5E62FC38E9974.1 => PmvzQKgYek6Sdk/T5sWaqwA.1.F.A.23_B6A5E62FC38E9974.1.0
5. PmvzQKgYek6Sdk/T5sWaqwA#B6A5FFD77977E2AE.1 => PmvzQKgYek6Sdk/T5sWaqwA#B6A5FFD77977E2AE.1.0

### Spin

The Spin operator is used instead of the Extend operator, in situations where the parent span cannot atomically increment the vector for each child span.

Here are some common examples:

1. Publisher-consumer control flow patterns where the parent span in the publisher may not aware of the number of consumers, which can be greater than 1
2. Components that consume queues denoting incoming control flows and which support multiple dequeue retry attempts and without the capability for maintaining local state for tracking previous dequeue attempts in the current span.
3. Bad actor components which repeatedly pass along the same vector value on outgoing control flows without increments.
4. Very high throughput components where the interlocked increment operation is a significant overhead.

In such instances, without a uniquely incremented value on each incoming flow, the uniqueness constraint that promulgates that each span has its own unique cV is violated. This impacts downstream diagnostics as identical vector values are repeated on multiple spans.

The Spin operator mitigates this, by inserting an element with entropy so that repeated invocations result in unique values. However purely using entropy erodes the utility of the vector when it is used to provide a sort during data processing. For this reason, the construction precedes this with an element derived from UTC time. This supports the sort property of the vector while reducing the chances of collision.

**Definition**: V => V_M.0

### Spin Examples

1. PmvzQKgYek6Sdk/T5sWaqwA.9 => PmvzQKgYek6Sdk/T5sWaqwA.9_B6A6A13E588CF82F.0
2. PmvzQKgYek6Sdk/T5sWaqwA.1.F.A.23 => PmvzQKgYek6Sdk/T5sWaqwA.1.F.A.23_B6A6A13E588CF82F.0
3. PmvzQKgYek6Sdk/T5sWaqwA-304773F68A307E98.4 => PmvzQKgYek6Sdk/T5sWaqwA-304773F68A307E98.4_B6A6A13E588CF82F.0
4. PmvzQKgYek6Sdk/T5sWaqwA.1.F.A.23_B6A5E62FC38E9974.1 => PmvzQKgYek6Sdk/T5sWaqwA.1.F.A.23_B6A5E62FC38E9974.1_B6A6A13E588CF82F.0
5. PmvzQKgYek6Sdk/T5sWaqwA#B6A5FFD77977E2AE.1 => PmvzQKgYek6Sdk/T5sWaqwA#B6A5FFD77977E2AE.1_B6A6A13E588CF82F.0

### Reset

The Reset operator is meant to be invoked implicitly, whenever an operation can potentially result in the total vector length exceeding its maximum length. The mutation is specific to the operation that is being attempted, with the basic goal of truncating the vector.

- Increment: XS.N => X#M.(N+1)
- Extend: XS => X#M.0
- Spin: XS => X#M.0

Note: When Reset is invoked in the context of the Spin operator, the operations are identical to those of Extend. The reason for this is that the Reset operator already introduces a sort-entropy element (by introducing M) that makes the vector value unique to the span, thereby making the addition of another such element moot.

In all cases, the Reset Operation records S, i.e. the value associated with the previous value of the vector. This value needs to be recorded for reconstructing the trace and is associated with its replacement M. In other words, the association tuple (S, M) needs to be captured in the trace in conjunction with invoking the Reset operator.

### Reset Examples

Let:

- X = PmvzQKgYek6Sdk/T5sWaqwA
- S = .1.FA.A1.23_B6A5E62FC38E9974.1_B6A6A13E588CF82F.2A.AB.213_B6A92D24A00C0F9B.47.8B.12.34.A123.2B.23.41.AB

XS is 126 bytes (assuming ASCII encoding and 1 byte per character).

- Increment: XS.F => X#B6B3AB078D8000FA.10
  - Recorded mapping: S <=> B6B3AB078D8000FA
- Extend: XS => X#B6B3AB078D8000FA.0
  - Recorded mapping: S <=> B6B3AB078D8000FA
- Spin: XS => X#B6B3AB078D8000FA.0
  - Recorded mapping: S <=> B6B3AB078D8000FA

Note that the properties of M as defined above allows for sorting events across the trace reset boundary, even in the absence of intermediate spans that contain the previous value of the vector. This is based on the time component that is the most significant 4 bytes in M.

## Key differences with cV 2.1


## Interop with cV 2.1

## Interop with W3C

### TraceParent to cV 3.0

### cV 3.0 to TraceParent