----------------    -------------------------------------------------
title:              Directions for New Fundamental C++ I/O Facilities

document number:    PxxxxR0

authors:            Dalton M. Woodard <dalton@daltonmwoodard.com>

date:               2019-01-17

audience:           LEWG, Direction Group

---------------------------------------------------------------------

# Abstract
This paper outlines the problems with current I/O facilities in C++, directions
and goals of future fundamental I/O facilities, and an outline of a suite of
customization points and concepts.

# Change Log

## Revision 0
Initial directions document.

# Introduction
The C++ language and standard library serve many communities with diverse needs.
However, regardless of other requirements, every program that is to be of any
use must communicate with the world outside of itself. We submit that C++ as it
exists today is severely hindered by its inability to directly serve all, or
indeed most, of its users' needs regarding program I/O. This paper seeks to
chart a course for the future of the C++ standard library's support of those
needs.

## A Brief Tangent into Terminology
Since I/O is an overloaded term, we must be careful to state what we mean in
the scope of this document. For the remainder of this discussion, when we refer
to (fundamental) I/O facilities we mean exactly the abstractions, vocabulary,
and control structures required for transfer of bytes -- typically this means
transfer across a program's boundaries in a way that interacts with the outside
world.

Hence, by I/O we _do not_ mean any of the following:

    - Object serialization and deserialization
    - Text formatting and encoding
    - Program localization and internationalization
    - Graphical output and human/computer interaction
    - Filesystem access and control

These are, of course, necessary and useful facilities for C++ to support, either
directly in the standard or by way of enabling the usual zero-cost abstractions
required to develop useful libraries. We anticipate that whatever fundamental
I/O facilities are developed and standardized as a result of this initiative
must either enable or be sufficiently orthogonal to the above to allow for
seamless and efficient integration with existing and future libraries in this
space.

# Motivation
## Problems with the Status Quo
Recent papers submitted to WG21 ([P1031], [P1026], [N4412]) along with numerous
blog posts and articles written by members of the C++ users community have
discussed the shortcomings of the existing C++ I/O facilities. These concerns
are primarily with iostreams, and secondarily with the lack of first-class
support in C++ for efficiently and correctly accessing certain features of
operating systems in hosted environments. For context we will summarize those
we find to be most relevant to the current discussion.

### Conflated Concerns of `std::basic_streambuf`
At a minimum, `std::basic_streambuf` concerns itself with underlying I/O device
creation & control, maintenance of and seeking within streaming read and/or
write state, actual transfer of bytes from and to the read and/or write state,
respectively, and character encoding conversion.

Each of these concerns is, in principle, orthogonal. Moreover, it is in general
not possible for a consumer of a `std::basic_streambuf` to query which of these
behaviors is actually implemented by a derived class, making certain code
depending on said behaviors unreliable at best.

This also makes it needlessly difficult to extend iostreams. Writing a custom
`std::basic_streambuf` requires consideration of all of these concerns, where
typically one wants nothing more than a data source or sink.

### Limited Efficiency of `std::basic_streambuf`
When buffered -- the typical case for both implementation and usage --
`std::basic_streambuf` imposes additional copies of data for each I/O
operation. In [P1031] and [P1026] N. Douglas discusses the effects these
extra copies have on the best possible latency and throughput that can be
achieved on modern storage and network interface hardware using iostreams.

Even when explicitly unbuffered, the interface of `std::basic_streambuf` is
unsuitable for accessing efficient I/O primitives provided by common operating
systems. This is because `std::basic_streambuf::sgetn()` and
`std::basic_streambuf::sputn()` operate on single character arrays at a time,
whereas, for example, the POSIX `read()` and `write()` primitives have vectored
alternatives `readv()` and `writev()` capable of transacting reads and writes of
multiple byte arrays in a single invocation.

The unavoidable conclusion is that C++ programs are fundamentally limited in
the performance of their I/O subsystems when using iostreams.

### Obfuscation of I/O Errors
The iostreams interface provides no portable way to determine the root cause of
failure when reading from or writing to a stream's underlying device encounters
an error. This is a simple consequence of the employed type erasure mechanism.
Errors from arbitrary sources would have to be marshaled into (or simply
constrained to) a single type, and the legacy design choices make it infeasible
to do this with exceptions.

This makes iostreams an inappropriate choice for systems that are particularly
sensitive to the correctness of error handling at I/O boundaries where exact
failure modes must be known.

### Inability for Standard C++ to Access Shared Memory and Memory Mapped Files
In [P1026] N. Douglas discusses the need for language support in standard C++
to correctly access shared memory and memory mapped files. The general problem
consists in the memory and object models' lack of support for interacting with
memory that is changed by other programs. This presents a major problem for
ensuring the correctness and long-term support of large scale C++ systems that
will continue to rely, and rely more heavily on, shared memory technologies.

We believe these limitations, among others, are what lead C++ software systems
to frequently reinvent fundamental I/O abstractions, or to limit themselves to
consuming only the lowest-level, inexpressive, and error-prone I/O facilities
provided by the operating system, C standard library, or hardware drivers.

## Goals, Expectations, and Requirements
Our principal goal is to develop a common vocabulary of concepts, customization
points, and algorithms for connecting I/O devices (sources and sinks) to
consumers and producers of binary data. This vocabulary should be amenable to
as diverse a set of implementations on both ends as possible, and available in
both hosted and freestanding environments.

Hence, semantics should not be imposed, or imposed with minimal restriction,
upon I/O devices to allow for as many back-ends as possible. These may be files
or network sockets accessed by application code on hosted systems, storage
hardware or network interface drivers accessed by systems software, or I/O
interfaces to peripheral devices in embedded systems, among many other
possibilities.

Likewise, consumption and supply of binary data from and to I/O devices should
be as minimally invasive as possible. Concepts for data types that can be
trivially transferred across I/O boundaries and concepts for ranges thereof
should be specified alongside the required customization points and range
machinery required to interact with these in expressive and composable ways.

This diversity of possible use-cases and our discussion above imply that we
must find an adaptable mechanism for communicating failure modes when errors at
program I/O boundaries occur. We have found the general consensus in the
community to be that dual `std::error_code` APIs are unsatisfactory for a
number of reasons, not least of which is a frequent underspecification of when
exceptions can be thrown. Likewise, since we desire for the suggested
vocabulary to be usable in freestanding environments, exceptions, at least
those which could be thrown in principle by the library, are an immediate
non-starter. Judicious specification of concepts constraining on the `noexcept`
property of certain operations may help in this respect, as well as provide
assurance to callers that those operations _cannot_ fail by propagating
exceptions.

We are hopeful that recent proposals such as [P0709], [P1028], and [P0323] may
provide a starting point for specifying such a mechanism, although we are aware
of at least one requirement not wholly addressed by those proposals: composed
I/O algorithms, those that perform multiple calls to more primitive I/O
transfer routines, may fail at intermediate steps. In such a case, it is
paramount that both the number of bytes successfully transferred and the exact
failure mode are communicated back to the caller.

The diversity of use-cases also implies that the vocabulary must be agnostic to
allocations and whence memory regions are supplied to write from and read into.
We expect that customization points should be supplied for I/O devices to
specify some preferred location, size, and alignment of buffers to be used, when
applicable.

As a secondary goal, analogous vocabulary for asynchronous I/O should be
considered. Care must be taken though to ensure interoperability with the
future of Executors, Coroutines, and Ranges. This leads us to our third goal.

This common vocabulary should support the design and standardization of the
Networking TS. We expect that much of the non-networking specific material of
the TS in its current form can be subsumed by the work proposed herein. This
goal should inform the design of new fundamental I/O facilities.

As with the Networking TS, our final goal should be to support standardization
of certain concrete types for file access on hosted systems. Parts of [P1031]
should be considered for this purpose, modified if necessary to interoperate
with the common vocabulary we are suggesting.

TODO(consult others on potential requirements not listed here)
...

## Non-Goals
As mentioned in the introduction, there are a number of things we do not mean by
I/O. Each of these comprises, therefore, a non-goal.

We do not propose to extend the current iostreams design, nor invent an
iostreams v2. Our expectation is that many of iostreams' capabilities --
localization and character encoding conversion, in particular -- can be
achieved through a judicious combination of new text formatting library support
(likely supplied by [P0645]), future language and library additions championed
by the SG16 Unicode Study Group, and Ranges.

Neither do we propose to standardize an object serialization and
deserialization library at this time. Further implementation and usage
experience should be gained with eventual language support for Reflection as
well as any new I/O facilities standardized as a result of our current efforts
before that bridge is crossed, if at all.

Lastly, aside from those parts of [P1031] mentioned above, our immediate
efforts should avoid in-depth specification of filesystem interaction.

# Design
What follows is a high-level description of a library of customization points
and concepts. It is intended as a prototype rather than a specification, and as
material to provoke further discussion. We use the identifier `IOError` as a
currently unspecified type or concept to be determined later (as described
above, an adaptable mechanism for communicating errors will be needed).

## Transferable Types
We need a set of concepts to distinguish types that can be transferred across
I/O boundaries, written to or read from buffers, and so on without additional
modification in such a way that their representation determines their value.
Hence, these should pick out scalar types without padding bits and
user-defined, trivially copyable types without padding types composed thereof.
Hence, we have

```
template <class T>
concept UniquelyRepresented = std::has_unique_object_representations_v <T>;

template <class T>
concept TriviallyBufferable = TriviallyCopyable <T> && UniquelyRepresented <T>;
```

Certain I/O algorithms also have a need to inspect buffers at intermediate
steps to, for instance, search within a range of bytes transferred thus far.
For generality and to sidestep language restrictions around inspecting object
representations at compile time (e.g. one cannot in general perform a
`static_cast<const std::byte*>()` in a `constexpr` function), we propose a
concept to distinguish types that behave sufficiently like bytes for the
purposes of I/O.

```
template <class T>
concept Byte = TriviallyBufferable <T> && Regular <T> && sizeof (T) == 1;
```

We also need concepts to distinguish range types whose contents can be
transferred in the same way.

```
template <class Rng>
concept TriviallyBufferableRange = ranges::ContiguousRange <Rng> &&
    ranges::SizedRange <Rng> &&
    TriviallyBufferable <ranges::iter_value_t <ranges::iterator_t <Rng>>>;

template <class Rng>
concept ByteRange = TriviallyBufferableRange <Rng> &&
    Byte <ranges::iter_value_t <ranges::iterator_t <Rng>>>;
```

## Buffer Customization Points and Concepts
From ranges of transferable types we can produce views over existing memory for
I/O algorithms to consume:

```
template <class T>
    requires Byte <std::remove_cv_t <T>>
class basic_buffer
{
public:
    // ...
    constexpr T * data () const noexcept;
    constexpr std::size_t size () const noexcept;
};
```

Customization points can bridge the gap between contiguous ranges for which
the library can automatically synthesize a `basic_buffer` and user-defined
types that wish to opt-in to providing a view over, say, some internal memory
region to be written from or read into. We propose two customization point
objects.

```
/* implementation defined */ buffer;
/* implementation defined */ const_buffer;
```

The former, `buffer`, produces views of type `basic_buffer <T>`, where `T` may
be a qualified type `const U` depending on the argument to `buffer`, whereas
the latter always produces views of type `basic_buffer <const T>`.

A concept `Buffer` should distinguish the types of expressions we can provide
to `buffer` and `const_buffer` to compute those views over memory. Two
additional concepts can distinguish when we can only read from a memory region
versus read from write into a memory region computed by `buffer`:

```
template <class T>
concept ConstBuffer = Buffer <T>;
template <class T>
concept MutableBuffer = Buffer <T> && _IsMutableBuffer <T>;
```

Where `_IsMutableBuffer` is an exposition only concept testing whether the
computed `basic_buffer` views a read only memory region (i.e., whether `T` in
`basic_buffer <T>` is a qualified type `const U`).

I/O algorithms that operate on sequences of buffers should be constrained with
analogous concepts:

```
template <class Rng>
concept BufferRange = ranges::ForwardRange <Rng> &&
    Buffer <ranges::iter_reference_t <ranges::iterator_t <Rng>>>;
template <class Rng>
concept ConstBufferRange = BufferRange <Rng>;
template <class T>
concept MutableBufferRange = BufferRange <Rng> &&
    MutableBuffer <ranges::iter_reference_t <ranges::iterator_t <Rng>>>;
```

The Networking TS currently makes use of a named type requirement
`DynamicBuffer`. We should also provide a concept matching a similar
specification.

Notice that by using ranges and the customization points `buffer` and
`const_buffer` in our concept specifications we make available the full breadth
and depth of the C++ Ranges library for use in implementing and using the
library components proposed herein. Experimental work in implementing this
vocabulary has proved this approach to be tremendously expressive.

## Device Customization Points and Concepts
TODO

## Future Work
TODO

# Open Questions
1. [P1026] proposed creation of a _Elsewhere Memory Study Group_ (formerly
   suggested as a _Data Persistence Study Group_). The work suggested in the
   current paper and future work in this space could benefit from a dedicated
   venue for discussion. The ambitious goals listed above also suggest the need
   to seek out guidance and input from domain experts in, at the very least,
   systems and kernel development, high performance computing, and embedded
   systems development as stakeholders in the future of C++ library support for
   I/O. Should a new study group be formed for this purpose?

2. In what time frame do we want to ship a library implementing the vocabulary
   suggested above? We expect that with enough effort a minimally viable
   product sufficient to support the Networking TS, as suggested above, can be
   ready to ship with C++2b. We believe there has been enough implementation
   and usage experience with similar vocabulary to achieve this, especially if
   care is taken to make use of some existing idioms established by the
   Networking TS (and its predecessor ASIO), and the Ranges library.
   Additionally, we expect the surface area of the library to be reasonably
   compact.

