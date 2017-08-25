---
title: "The Limits of Copy-on-write: How Ruby Allocates Memory"
#title: How Ruby Allocates Memory, and the Limits of Copy-on-write
#title: "The Curious Case of the Bloated Unicorn: How Ruby Manages Memory"
published_at: 2017-08-24T13:39:04Z
hook: TODO
---

Anyone who's run Unicorn (or Puma, or Einhorn) will have
noticed a curious phenomena. Worker processes that have
been forked from a master start with low memory usage, but
before too long will bloat to a similar size as their
parent. In a big production installation, workers can be
100s of MBs or more, and before long memory is far and away
the constrained resource on your boxes. CPUs sit mostly
idle.

Modern operating systems have virtual memory management
systems that provide ***copy-on-write*** facilities
designed to prevent this exact situation. A process's
virtual memory is segmented into 4k pages. When it forks,
its child initially shares all those pages with its parent.
Only when the child starts to modify one of them does the
kernel intercept the call, copy the page, change the
duplicate, and reassign its use for the child.

!fig src="/assets/ruby-memory/child-processes.svg" caption="Child processes transitioning from mostly shared memory to mostly copied as they mature."

So what's going on in the case of Unicorn? Most software
has a sizeable collection of static objects that are
initialized once, and sit in memory largely unmodified
throughout its entire lifetime. If nothing else, child
processes should have no problem sharing that collection
with their parent, but apparently they're not able to (or
to only a very minimal extent). To solve this mystery,
we'll have to understand how Ruby allocates memory.

## Slabs and slots (#slabs-and-slots)

Lets start with a very brief overview of object allocation
in Ruby and then walk through some the relevant code. Ruby
requests memory from the operating system in chunks that it
refers to internally as ***heap pages***. The naming is a
little unfortunate because these aren't the same thing as
the 4k page that your OS will hand out, although Ruby does
specifically size its heap pages so that they'll use OS
pages efficiency by maximizing the use of a multiple of
them (usually four 4k OS pages = one 16k heap page).

TODO: Diagram of slabs and slots

You might also hear a heap page referred to as a "heap"
(plural "heaps"), "slab", or "arena". I'd prefer to work
with one of the last two for less ambiguity, but I'm going
to stick with ***heap page*** for a single chunk and
***heap*** for the collection of all heap pages because
that's what they're called everywhere in Ruby's source.

A heap page consists of a header and a number of
***slots***. Each slot can hold an `RVALUE`, which is an
in-memory Ruby object (more on this in a moment).

### Heap initialization (#heap)

Ruby's heap is initialized by `Init_heap` ([in
`gc.c`][initheap]), called from `ruby_setup` ([in
`eval.c`][rubysetup]), which is the core entry point for a
Ruby process which also initializes the stack and VM.

``` c
void
Init_heap(void)
{
    heap_add_pages(objspace, heap_eden,
        gc_params.heap_init_slots / HEAP_PAGE_OBJ_LIMIT);

    ...
}
```

It decides on an initial number of pages based on a number
of target slots. This gets a default of 10,000, but can be
tweaked through configuration or environmental variable:

``` c
#define GC_HEAP_INIT_SLOTS 10000
```

The number of slots in a page is calculated roughly the way
that you'd expect ([in `gc.c`][heappagealignlog]). We start
with a target size of 16k (also 2^14 or `1 << 14`), shave a
few bytes off that we expect `malloc` to need for
bookkeeping [1], subtract a few more bytes for a header,
and then divide by the known size of `RVALUE`:

``` c
/* default tiny heap size: 16KB */
#define HEAP_PAGE_ALIGN_LOG 14
enum {
    HEAP_PAGE_ALIGN = (1UL << HEAP_PAGE_ALIGN_LOG),
    REQUIRED_SIZE_BY_MALLOC = (sizeof(size_t) * 5),
    HEAP_PAGE_SIZE = (HEAP_PAGE_ALIGN - REQUIRED_SIZE_BY_MALLOC),
    HEAP_PAGE_OBJ_LIMIT = (unsigned int)(
        (HEAP_PAGE_SIZE - sizeof(struct heap_page_header))/sizeof(struct RVALUE)
    ),
}
```

Ruby uses some pragma magic to ensure that an `RVALUE`
occupies 40 bytes. I'll save you some calculations, and
just tell you directly that in the 64-bit normal case all
this means that Ruby initially allocates 24 pages at 408
slots each [2]. That heap is subsequently grown if more
memory is needed.

### RVALUE: An object in a memory slot (#rvalue)

A single slot in a heap page holds an `RVALUE`, which is an
in-memory representation of a Ruby object. Here's its
definition ([from `gc.c`][rvalue]):

``` c
typedef struct RVALUE {
    union {
        struct RBasic  basic;
        struct RObject object;
        struct RClass  klass;
        struct RFloat  flonum;
        struct RString string;
        struct RArray  array;
        struct RRegexp regexp;
        struct RHash   hash;
        struct RData   data;
        struct RTypedData   typeddata;
        struct RStruct rstruct;
        struct RBignum bignum;
        struct RFile   file;
        struct RNode   node;
        struct RMatch  match;
        struct RRational rational;
        struct RComplex complex;
    } as;

    ...
} RVALUE;
```

For me this is where the mystique around how Ruby can
generically assign any type to any variable finally starts
to make sense; we immediately see that an `RVALUE` is
essentially a big list of any type that Ruby might hold in
memory. These types are all compacted with a C `union`. The
union's total size is only as big as the largest individual
type in the list.

To get a slightly more concrete understanding of a slot,
lets dig into the common Ruby string a little more (from
[ruby.h][rstring]):

``` c
struct RString {
    struct RBasic basic;
    union {
        struct {
            long len;
            char *ptr;
            union {
                long capa;
                VALUE shared;
            } aux;
        } heap;
        char ary[RSTRING_EMBED_LEN_MAX + 1];
    } as;
};
```

We won't dig into `RString` too deeply, but looking at this
struct yields a few points of interest:

* It internalizes `RBasic`, which is another struct that's
  common to all in-memory Ruby types that helps us easily
  distinguish between them.

* A union with `char ary[RSTRING_EMBED_LEN_MAX + 1]` shows
  us that while the contents of a string might be stored in
  the OS heap, a sufficiently short string will be inlined
  right into an `RString` value, which means that the whole
  thing can fit right into a pre-allocated slot.

* A string can reference another string (`VALUE shared`)
  and share the memory occupied by its contents.

### VALUE: A pointer or scalar (#value)

Looking at the definition of `RVALUE` shows us that while
Ruby holds many types in a slot, it doesn't hold all of
them. Anyone who's written a Ruby C extension before will
be familiar with a similarly named type called a `VALUE`.
Its implementation is quite a bit simpler; it's a pointer
[from `ruby.h`][value]:

``` c
typedef uintptr_t VALUE;
```

This is where Ruby's implementation gets a little nasty.
While `VALUE` is often a pointer to an `RVALUE`, it may
also hold some types that will fit right into a pointer's
size by comparing values to constants or using bit-shifting
techniques. `true`, `false`, and `nil` are the easiest to
reason about because they're all predefined (from
[ruby.h][rubyconsts]):

``` c
enum ruby_special_consts {
    RUBY_Qfalse = 0x00,		/* ...0000 0000 */
    RUBY_Qtrue  = 0x14,		/* ...0001 0100 */
    RUBY_Qnil   = 0x08,		/* ...0000 1000 */

    ...
}
```

A fixnum (i.e. very roughly a number that fits in 64 bits)
is a little more complicated. One is stored by
left-shifting a value by one bit, and then setting a flag
in the rightmost position:

``` c
enum ruby_special_consts {
    RUBY_FIXNUM_FLAG    = 0x01,	/* ...xxxx xxx1 */

    ...
}

#define RB_INT2FIX(i) (((VALUE)(i))<<1 | RUBY_FIXNUM_FLAG)
```

Similar techniques are used to store "flonums" (i.e.
floating point numbers) and symbols. When the time comes to
identify what's occupying a `VALUE`, Ruby compares pointer
values to a list of flags that it knows about for these
stack-bound types; if none match, it goes to heap ([from
`ruby.h`][rbclassof]):

``` c
static inline VALUE
rb_class_of(VALUE obj)
{
    if (RB_IMMEDIATE_P(obj)) {
        if (RB_FIXNUM_P(obj)) return rb_cInteger;
        if (RB_FLONUM_P(obj)) return rb_cFloat;
        if (obj == RUBY_Qtrue)  return rb_cTrueClass;
        if (RB_STATIC_SYM_P(obj)) return rb_cSymbol;
    }
    else if (!RB_TEST(obj)) {
        if (obj == RUBY_Qnil)   return rb_cNilClass;
        if (obj == RUBY_Qfalse) return rb_cFalseClass;
    }
    return RBASIC(obj)->klass;
}
```

Keeping certain types of values on the stack has the
advantage that they don't need to occupy a slot in the
heap. It also allows Ruby to perform faster computations
with them. "Flonum" was a relatively recent addition to the
language, and its author [estimated that it sped up simple
floating point calculations by ~2x][flonum].

## Allocating an object (#allocating)

Now that we're armed with a basic understanding of the
heap, we're getting closer to understanding why our mature
Unicorn processes can't share anything with the parent
(some readers may have guessed already). To fully solidify
our understanding, lets walk through how Ruby initializes a
slot for a basic string.

The entry point is `str_new0` (from [`string.c`][strnew0]):

``` c
static VALUE
str_new0(VALUE klass, const char *ptr, long len, int termlen)
{
    VALUE str;

    ...

    str = str_alloc(klass);
    if (!STR_EMBEDDABLE_P(len, termlen)) {
        RSTRING(str)->as.heap.aux.capa = len;
        RSTRING(str)->as.heap.ptr = ALLOC_N(char, (size_t)len + termlen);
        STR_SET_NOEMBED(str);
    }

    if (ptr) {
        memcpy(RSTRING_PTR(str), ptr, len);
    }

    ...

    return str;
}
```

Just like we speculated when examining the `RString` struct
earlier, we can see that Ruby embeds the new value right
into the slot if it's short enough. Otherwise is uses
`ALLOC_N` to allocate new space for the string and sets a
pointer (`as.heap.ptr`) internal to the slot to reference
it.

### Initializing a slot (#slot-initialization)

## Closing the case on bloated workers (#bloated-workers)

## Towards compaction (#compaction)

[1] `malloc`'s bookkeeping is compensated for so that we
can keep a heap page fitting nicely into a multiple of OS
pages without overflowing onto another OS page. Because
pages are the smallest unit that an OS will allocate to a
process, this would make for an inefficient use of memory.

[2] Astute readers may notice that we start with only 9,792
(24 * 408) total slots, despite requesting 10,000.

[flonum]: https://bugs.ruby-lang.org/issues/6763
[heappagealignlog]: gc.c#L660
[initheap]: gc.c#L2377
[rbclassof]: ruby.h#L1970
[rstring]: ruby.h#L954
[rubyconsts]: ruby.h#L405
[rubysetup]: eval.c#L46
[rvalue]: gc.c#L410
[strnew0]: string.c#L702
[value]: ruby.h#L79