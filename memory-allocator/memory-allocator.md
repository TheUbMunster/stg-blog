# Memory Allocation
<img align="center" src="memory-allocator-icon.png" style="max-width: 50%; max-height: 50%; margin: 0px auto 0px auto; display: block;" alt="A computer render of a heap represented as a fragmented block"/>

Have you ever thought about *how you think*? It might be fair to say that computers have no choice! Computers are very exact,
and do exactly what they're told to do. This includes how computers "remember things". There are various methods for how to
accomplish memory management in a computer, in a former post I talked about how ***the stack*** works. In this post I'll be
talking about how memory allocators work.

If you read my previous blog post about [***the stack***](./blog.html?post=stack), or you know how
it works, then you understand how much of the memory that exists in the stack isn't "data" (although some of it is), but
*metadata* instead. This metadata consists of things like return addresses, unnamed intermediate values for calculations,
debug information, and many other things irrelevant to the *actual data* for your program. Even worse, it makes sense to
organize its structure around the control flow of your program, rather than *[literally any other method]* that would make
sense for "data storage". With that in mind, lets talk about...

## Dynamic memory allocation

Golly, **what a headache**. It seems to me that separating the portion of memory that seems to be good for keeping
track of control (execution) from the *data* data is an essential idea. Before getting into how we might do that,
lets first take a simplified look at how memory is laid out for a typical process.

<picture>
  <!-- User prefers light mode: -->
  <source srcset="ProcessLayoutWinUnixLight.png" media="(prefers-color-scheme: light)"/>

  <!-- User prefers dark mode: -->
  <source srcset="ProcessLayoutWinUnixDark.png"  media="(prefers-color-scheme: dark)"/>

  <!-- User has no color preference: -->
  <img align="center" src="ProcessLayoutWinUnixLight.png" style="max-width: 50%; max-height: 50%; margin: 0px auto 0px auto; display: block;" alt="Diagram of the memory layout of processes on unix and windows"/>
</picture>



There's a few notable things about the layouts of unix/windows processes:

- In windows, the stack grows towards low memory addresses. Note that this means that the (virtual) memory address 0 is
the absolute limit for the stack size.
- In unix, the stack grows towards the heap. This means the "absolulte largest size the stack can be" is a bit less
obvious. It depends on the size of the heap.
- In both, similar implicit limits exist for the "Heap".
- In windows, "Program Image" roughly equates to ".Text" + ".Data" + ".BSS" in unix.
- "Kernel Memory" in both refers to the memory that the operating system occupies. The kernel memory is in a processes
virtual memory map for the sake of syscalls[^1].

And guess what? There *is* (essentially) a "second stack" that we can use for our *data* data, it's the heap!

To interact with the heap, on unix there's `s/brk()`, which increases/decreases the size of the heap. `s/brk()` usage
is often discouraged because it isn't thread safe, and because the "default" C memory allocator `malloc()` uses
`s/brk()`, so any additional usage of it will mess up `malloc()`[^2]. If we intend to use `s/brk()` for our memory
allocator, we either need to not include the standard C library (which may be an issue if any libraries we use require
the library), or we need to make sure our dynamic memory allocator has the *exact* same API as the standard
implementation, and link against our implementation instead.

On windows, `HeapAlloc()`, `LocalAlloc()` and `GlobalAlloc()`[^3] all interact with the heap, but do so in different
ways. `malloc()` usually wraps `HeapAlloc()` on windows, and although the problems with `s/brk()` above don't apply due
to the presense of actual management with these three functions, I would rather that a cross platform implementation
of *my* memory allocator use logically similar sources of memory. Although these functions internally manipulate the heap
directly, there isn't an API method to "`s/brk()`" on windows directly.

`malloc()` is smart in that it makes large increments in `s/brk()` on unix & simply wraps `HeapAlloc()` on windows,
this prevents heap thrashing[^4] and reduces the number of necessary syscalls. `malloc()` is implementation defined
though, so you can't always rely on it working this way. Fortunately, the implementers are smart and attempt to back
`malloc()` with the best solution for the platform.

POSIX[^5] specifies `mmap()`, which allocates more memory pages that exist outside of the Process Memory layout
diagram above[^6], fetched straight from the kernel's free pages. `mmap()` has some additional features, namely the
ability to share the memory mapped in this fashion with other processes, allowing these processes the ability to
communicate with each other as they run.

On windows, `VirtualAlloc()` is very similar to `mmap()` in that it fetches more pages from the kernel. It can even
lazily allocate memory[^7], and share it with other processes. Note that page*s* requested in either of these fasions
are guaranteed to be consecutive within single calls, but subsequent calls may not be adjacent to pages from a previous
call. This is more likely to be the case if you have other things in your process calling `VirtualAlloc()`/`mmap()`.
Additionally, both of these functions have additional features, and I encourage you to do your own reading about these
systems.

### Implementation

Aside: although these concepts are almost universally applicable, I will be discussing this post from a C/++-esq
perspective.

With either of these functions, for every dynamic value you wish to store, nothing's really stopping you from using the
page(s) for your single variable, but since mnay values are far less than 4KiB or (page size), this would be very
wasteful. With this in mind, we know that we'll need to have some internal management mechanism.

My goal was to make a memory allocator that was compatible on both windows and linux and wouldn't interfere with
`malloc()`, so first I wrote cross-platform methods to allocate and free page(s).

```c
void* p_alloc(size_t pageCount)
{
	void* result;
	size_t byteCount = pageCount * p_size();
#if _WIN32
	result = VirtualAlloc(NULL, byteCount, MEM_COMMIT, PAGE_READWRITE);
	if (result == NULL)
	{
		fprintf(stderr, "Windows Virtual Alloc Failed: %d", GetLastError());
		exit(-1);
	}
#else
	result = mmap(0, byteCount, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANON, -1, 0);
	if (result == MAP_FAILED)
	{
		fprintf(stderr, "POSIX mmap failed: %d", errno);
		exit(-1);
	}
#endif
	return result;
}
```

This page allocation function allocates for you `pageCount` pages in a platform agnostic way. We make sure to set
appropriate settings (e.g., we do not want to enable the ability for this memory to be executable). I leave `p_free()`
as an excersize to the reader.

You'll notice that we allocate memory in terms of `pageCount`, which isn't that useful alone. Since we're ultimately
working with requests in terms of `bytes`, we need a way to know how large a page is on the given platform. Here's how
I did that.

```c
size_t p_size()
{
	static size_t ps = 0;
#if _WIN32
	if (ps == 0)
	{
		SYSTEM_INFO si;
		GetSystemInfo(&si);
		ps = si.dwPageSize;
	}
#else
	if (ps == 0)
		ps = getpagesize();
#endif
	return ps;
}
```

Since batching calls (e.g., the way `malloc()` does it) can reduce syscalls, we want to ensure that the several places
throughout our code where we have to check the size of the pages (so we can calculate the number of bytes we have), is
fast, and doesn't have too many system calls. Even with the fanciest of operating systems (to my knowledge), changing
the size of pages on your system would require a system reboot. Thus, we only need to check the page size once (via a
syscall), and we cache it in a static local and return that cached value for all subsequent calls.

I'd like to get onto the algorithm itself now, but there's an important thing to note. Since we're *writing* a memory
allocator, that means we can't *use* a/the memory allocator (since it *isn't written yet*). That means whatever data
structures we decide to use need to be simple, in-place, and need to function with "raw memory". Given that we're
allocating chunk(s) of pages.

Since one of our goals is to divvy up these large page(s) into small pieces of memory, lets define a "node". Let's say
this node's responsibility is denoting a "small memory piece", and it has the necessary information to observe adjacent
nodes too.

```c
struct HeapNodeHeader
{
	HeapNodeHeader();
	HeapNodeHeader(unsigned long long size, bool isCurrentFree);
	unsigned long long header_data; //stores info like the "size" of this node
	//Do not derefence, instead "return &mem".
	void* mem; //this member sits at the location of the user's memory.

	bool isImmediatePreviousFree();
	bool isCurrentFree();
	bool isImmediateNextFree();

	unsigned long long size();
	void setCurrentFree(bool isFree);
	void setSize(unsigned long long size);

	HeapNodeHeader* immediatePrevNeighbor();
	HeapNodeHeader* immediateNextNeighbor();
};
```

Good. However, lets say that we have a ptr to a `HeapNodeHeader` from one of our pages. How do we find its left/right
neighbors? We know that these nodes "sit next to each other", but given that these nodes are different sizes (since the
user requests memory of different sizes), this might cause a problem. We imagine that the *next* neighbor sits at 
`this + sizeof(HeapNodeHeader) + size()`, but what about the previous neighbor? To know where it sits, we would need
to know how large it is in order to calculate how far back it is (classic chicken or the egg type problem).

To fix this, let's introduce another data structure.

```c
struct HeapNodeFooter
{
	HeapNodeHeader* header;
};
```

What we'll do, is whenever we allocate memory, (via our `malloc()` equivalent), we'll make sure to account for the size
of both `HeapNodeHeader` and `HeapNodeFooter`. We'll put the `HeapNodeFooter` at the very end of a node. This means, all
we need to do to get our *previous* neighbor is `((HeapNodeFooter*)(this - sizeof(HeapNodeFooter)))->header`. Easy!
Modify your `HeapNodeHeader` definitions/implementations appropriately.

At this point, with several nodes, memory would probably look something like this:

```text
=======================================================================================================================
| HNH | (user memory) | HNF | HNH | (more user memory) | HNF | HNH | (more user memory) | HNF | HNH | (more user mem...
=======================================================================================================================
```

At the beginning of our program (perhaps in our heap constructor), lets simply allocate a single page, put a HNH/HNF in
it and mark it as free.

On a side note, many operating systems enjoy or even require "aligned access" for performance and other reasons. Namely
we should make sure that every "(user memory)" address sits on a 16-byte aligned address. All this means is that as we
create HNH/HNF, we should do it in a way where the location of the user memory looks something like `0x6789ABC0` (i.e.,
the address ends in 0b0000, indicating it's evenly divisible by 16). Since the memory page itself is 16-byte
aligned, but we have a leading HNH before the actual user memory, we need to do things like:

- Artificially introduce padding before the first HNH in a set of page(s) to ensure that
`sizeof(padding) + sizeof(HeapNodeHeader)` is 16-aligned so that "(user memory)" is *also* 16-aligned
- Artifically increase the requested size of our `malloc()` call so that the
`sizeof(user memory) + sizeof(HeapNodeFooter)` will allow the following HNH to also be properly placed, so that *its*
"(user memory)" is also 16-byte aligned.

In the current implementation, `sizeof(HeapNodeHeader) == 16` (but that includes the size of `mem`, and for intents and
purposes, `mem` is the *body* of the node, which we don't want to include). So lets say the size of the header is 8,
and `sizeof(HeapNodeFooter) == 8`. Assuming 4KiB, this means the page at the beginning will be something like:

```text
=======================================================================================================================
| 8 bytes of padding | HNH |              (giant body comprising almost the whole page)               | HNF | padding |
=======================================================================================================================
```

To keep our algorithms simple, the padding at the end of the page exists because we placed the HNF in a way where a HNH
could be validly placed after it (with correct alignment etc), even though it's the end of a page and there won't ever
be a HNH there.

Since we artifically inflate the size of nodes to ensure their alignment, this also means that the size of any node
will be (a multiple of 16) + 8. This means that the size (in binary) will always have three zeroes in the bottom bits.
Awesome! We can use this extra space for three bitflags. We could use all three bits to store flags, but let's just
store a bitflag indicating if *this* node is free. If we ever need to check if a neighbor is free, we just access it
through the footer or size calculation like formerly outlined.

Let's discuss our first allocation. When our `malloc()` is called for the first time, we do have this giant free node.
We can set the free flag to false, and return `&mem`, but chances are the requested memory size isn't nearly as large
as the node is in its current state. If we did this, well, we wouldn't be any less better off than just returning the
entire page for the `malloc()` instead. To make this better, let's cut up the node into two pieces, and give the 
properly sized node to the caller.

```c
//make sure it's larger than the mandatory size
size = (size < sizeof(HeapNodeHeader::HeapNodeMandatoryBody)) ? sizeof(HeapNodeHeader::HeapNodeMandatoryBody) : size;
size = (size % 16 != 0) ? size + (16 - (size % 16)) : size; //make sure it's a multiple of 16

//if node is large enough to be split into two pieces:

HeapNodeFooter *ff = candidate->myFooter();
size_t oldBigSize = candidate->size();
candidate->setSize(size);
candidate->setCurrentFree(false);
candidate->myFooter()->header = candidate; //by setting size first, myFooter() points to the location of the new footer
ff->header = candidate->immediateNextNeighbor(); //by setting the footer, this points to the location of the new HNH
HeapNodeHeader* nf = ff->header;
nf->setCurrentFree(true);
nf->setSize(
	oldBigSize - 
	(sizeof(HeapNodeHeader) + 
	sizeof(HeapNodeFooter) + 
	size - 
	sizeof(HeapNodeHeader::HeapNodeMandatoryBody));
```

The above code splits it up into two pieces, and returns the left half for `malloc()`, the right half is the remaining
free bit. Note that in the circumstance that the requested size is "pretty large" we need to make sure that there's
room for a HNH/HNF combo to be interjected in the "body" of the node in this fashion. If we were to do this, and the
"free bit"'s size was smaller than the minimum size for a node (i.e., 16 bytes in our implementation), then there's
no point splitting it up, and we may as well just use the whole node for the `malloc()`, even if it's a tiny bit
oversized. This is fine for our big-O, because this size overhead for these nodes is O(1).

Now that we've split up this node, we could imagine the page looks something like:

```text
=======================================================================================================================
| 8 bytes of padding | HNH | (user memory) | HNF | HNH |                (free memory)                 | HNF | padding |
=======================================================================================================================
```

Now lets say that the user calls our `free()`, the memory looks something like:

```text
=======================================================================================================================
| 8 bytes of padding | HNH | (free memory) | HNF | HNH |                (free memory)                 | HNF | padding |
=======================================================================================================================
```

Now I ask a question: what if the user calls `malloc(approximately sizeof(HNH+HNF+both free memory))`? Conceptually,
we *do* have enough room in this page for their request, however, what we *have* is two separate nodes, neither of
which can fit the requested size. What we *could* do, is undo the "splitting" of the free memory into two nodes, and
combine them back into a single node. This of course only works if the nodes (2+) that we want to merge together are
all *free* nodes (we don't want to mess with the user's active memory). It would be annoying to start at the beginning
of the page and walk through all the nodes via `immediateNextNeighbor()` to check for nodes that we could coalesce
into single nodes, so instead, lets be smart and check for the possibility of coalescing nodes together *whenever a
node gets freed*.

Whenever a node gets freed, there are four circumstances to consider:
- it's previous neighbor is free, but it's next neighbor isn't
- it's previous neighbor isn't free, but it's next is
- neither neighbor is free
- both neighbors are free

It's easy to imagine what to do in each of these four circumstances. I'll show what I did for the first case, but I'll
leave the next as an excersize to the reader.

```c
//if prev is free, next is not free:

//no need to mark prev as free (already free)
currentNode->myFooter()->header = prevNode; //make current footer point to new head (prev)
prevNode->setSize(
	prevNode->size() + 
	currentNode->size() + 
	(sizeof(HeapNodeHeader) - 
	sizeof(HeapNodeHeader::HeapNodeMandatoryBody) + 
	sizeof(HeapNodeFooter))); //update size
//prevNode's old footer, and this node's current footer have both been abandoned by this point, no *need* for cleanup
```

By ensuring this occurs every time we free a node, we've created an invariant in our data structure[^8]. By making
sure that our avaliable free nodes are always maximally sized, this help us waste less memory by not making unecessary
requests for more pages.

The one thing that could make this better, would be if we could occasionally copy/paste all the free nodes to one end
of the page (and coalesce them together), and all the used nodes to the other end of the page. Unfortunately, this
doesn't work because the pointers we passed via our `malloc()` would no longer point to correct locations. We could
accomplish this by doubly-indirecting access to the backing user memory. I.e., our `malloc()` returns a `void**`. The
user would dereference twice to access their memory, and they could safely copy this pointer to share around their
code. However, we could maintain access to the middle pointer, defragment the heap (i.e., the aforementioned
copy/paste and movement of the data, and then update the middle pointer to properly point to the new location).
We would only need to make sure that the inital pointer that points to the middle pointer never changes. In fact,
this feature of "heap defragmentation" is something that garbage collected memory management[^9] uses all the time.

If you wish to implement a heap in this fashion, go for it! However, you'll be suprised how effective node coalescing
alone can improve performance, which is what I went with for simplicity.

We've done well so far, but we still have a looming issue. At the very beginning, we allocated a single page for our
purposes. What if that isn't enough? Well, we could allocate more pages, but we still have our original pages with
memory that's being used by the user still, and we're about to add a new one that's going to be utilized too. Since
we have two blocks of pages, we need to manage them both. Sounds to me like we're going to need a dynamic memory
structure...

Don't think too hard! Remember the padding we had at the beginning of the pages? What if we use that space to form a
linked list between our pages? While we're at it, so long as we expand this padding by a multiple of 16, we can
maintain our important alignment scheme. Here's the data structure I put at the beginning of my page blocks:

```c
struct PageHeader
{
	PageHeader *prev, *next; //linked list of pages.
	HeapNodeHeader *efl_head, *efl_tail; //*ignore me for now*
	unsigned long long sizeBytes; //the size of these page(s) in bytes, always a multiple of the OS's p_size()
	HeapNodeHeader* immediateFirstNode(); //very first element in this page(s) block
	HeapNodeHeader* immediateLastNode(); //very last element in this page(s) block
	bool isEntirePageFree(); //do these page(s) consist of a single free HeapNode?
};
```

Front and center, we store a doubly linked list connecting all of our pages together. Now we can manage multiple pages.
Keeping track of them is important, because if a page becomes fully free (and if we have several of them), it would be
good to free some of them back to the kernel, so it can be used for other processes if necessary. This wasn't obviously
necessary since all of our `free()` operations would simply take the passed ptr and subtract `sizeof(HeapNodeHeader)`
to access its HNH.

Our dynamic memory allocator now works! Is there anything else we can do to make it more efficient?

There are several options, but one of the most popular ones is an Explicit Free List implementation. Currently, the
list of all free nodes that we'd like to consider for `malloc()` is implicit, i.e., we start at the head of the page
and walk forward until we find a node that can fit our user's request (split it up into two pieces if we can). We can
do better. In our page header, we have two pointers, `efl_head` & `efl_tail` that store pointers to HNH's of both the
largest free, and smallest free node in this page block. Initially, with a single free node, both of these pointers
point to the single giant free node in the page. As the memory allocator gets used, and the page block gets peppered
with both utilized and free nodes, we can use a convenient trick.

Remember how we enforced a minimum node size? Besides the fact that it would be silly to have a free node with `0`
size due to the overhead of HNH/HNF, our minimum size due to padding of `16` bytes was coincedentally convenient.
Pointers 64 bit platforms are 8 bytes, and for free nodes (i.e., the nodes whose body *aren't being used by the
user*) we can use their body to store a doubly linked list, that connect all of the free nodes together. This means we
can look through *just* the free nodes when we're looking for a node for a `malloc()` request. Here's an updated form
of HNH:

```c
struct HeapNodeHeader
{
	HeapNodeHeader();
	HeapNodeHeader(unsigned long long size, bool isCurrentFree);
	unsigned long long header_data; //stores info like the "size" and "is this node free"
	union HeapNodeMandatoryBody
	{
		void* mem;
		struct
		{
			//"prev" goes larger, "next" goes smaller
			HeapNodeHeader *prev, *next;
		} efl; //explicit free list
	} content;
	bool isImmediatePreviousFree();
	bool isCurrentFree();
	bool isImmediateNextFree();

	unsigned long long size();
	void setCurrentFree(bool isFree);
	void setSize(unsigned long long size);

	HeapNodeHeader* immediatePrevNeighbor();
	HeapNodeFooter* myFooter();
	HeapNodeHeader* immediateNextNeighbor();
};
```

The user memory pointer, and the explicit free list are in a union together, which saves us space. (After all, we only
use the EFL if the node is free, so the user isn't using the memory). At this point, whenever we `malloc()` something,
it's trivial to remove this node from the EFL & stitch the broken ends back together (make sure the update the PH if
it turns out that this change would change the head or tail of the EFL). However, if we're freeing a node, all we have
access to (via `ptr - sizeof(HeapNodeHeader)`) is the node's header. If we want to add it to the EFL, we need to find
another free node in order to walk through the EFL and stitch it into place. However, our invariant guarantees nothing
about how often a free node occurs. If we have an *extremely large* page block, and as we walk through our
previous/next neighbors, it may be a long time before we finally find a free node to stitch it into the EFL. By keeping
track of a doubly linked list of all the pages, we can check if our node lies within the memory range of each of the
pages (i.e., `ptr > page_header && ptr < page_header + page_header.sizeBytes`), and if so, we can stitch it in via
accessing the EFL via the pointers stored in the page header.

There's lots of other optimizations we can do (some are mutually exclusive)!
- ensure that the page headers in the doubly linked list are stored in memory address order, perhaps even implement a
skip list[^10] to accelerate the search for the correct page header when freeing memory.
- ensure that the page headers in the doubly linked list are stored in order of their largest or smallest EFL member,
allowing for a system where searching for a page with a large enough free node is only *just* large enough to fit it,
saving the larger nodes for other allocations.
- ensure that all the elements in the EFL are sorted smallest to biggest, so that every time you need to `malloc()`,
you can check if the largest EFL element in a page block is large enough to fit the request. If not, you know there's
nothing bigger in the EFL of that page, so you can skip checking it all! Additionally, if your requested size is larger
than the smallest node in the EFL (but not by very much), you can start at the small end of the EFL and walk in the
largerward direction until you find the first EFL element that's *just barely* big enough to hold your request, thus
saving on memory. Similar logic can be applied if the requested size is only slightly smaller than the largest element
in the EFL.
- If nodes in the EFL are large enough to support the size (i.e., increase the mandatory size of a heap node to store
this information on a per-node basis), you can optionally implement a skip list throughout nodes in the EFL.
- Come up with a heuristic to decide if a search through an EFL can be terminated early, and instead choose an EFL
element that's "good enough" rather than continuing to waste too much time for little benefit (time/space tradeoff).
- Check if upon creation of new page blocks, if it's actually adjacent to a previous `mmap()`/`VirtualAlloc()` call,
and if so, instead adopt one page into the other instead of making a new entry in the page's doubly linked list.
- Have multiple EFL's per page block, where each EFL has been binned for a certain size of EFL entries, to accelerate
the EFL search, I would expect this to work well for an *extremely large* page block with a *really long* EFL, by
splitting the EFL into (e.g.) four separate EFL's, you can expect to speed up you EFL search by 4x!

There are several "typical" designs for dynamic memory allocators, and it's possible that if you go for one design, you
can still learn something from other designs. Give it a google.

Note that I'm using the term "optimization" a bit loosely here. It's certainly possible that depending on your
program's memory usage pattern, some of these behaviors would cause worse performance. If you're going out of your way
to implement a memory allocator for production, I hope you make sure to do some benchmarking!

## Debugging tips

Implement a `heap_check` function, that you can perform on a heap at any time to ensure that the layout of the heap,
the implications of flags etc. are all consistent. A classic example of what you could do is walk through the page
block and calculate the sum of all the nodee sizes and ensure it matches the size stored in the page header.

Throughout your code, put assertions (that get stripped in release builds if you want) to make sure that implicit
assumptions are actually true every step of the way.

## Conclusion

You can find a full implementation of my heap at https://github.com/TheUbMunster/stg-heap. If this is something you
choose to undertake, there are several tutorials on the internet about it that talk about the decision making processes
and many examples to learn from. Make sure to put your own spin on it. Try out an optimization, or a different
management scheme, perhaps you'll discover something new! A peer of mine not too long ago discovered a management
scheme that decreases the cost of overall allocations, and people out there are already implementing this scheme[^11]!

Good luck!

### Contact
If I've said something incorrect, or have anything else to say to me, contact me @ 
[tangleboom@gmail.com](mailto:tangleboom@gmail.com), I would love to update this post with the most accurate
information!

<!--## Footnotes-->

[^1]: I.e., wrappers for "secure methods" of interacting with the operating system that requires permission elevation.

[^2]: `malloc()` assumes it has full control of `s/brk()`, so any intermixed calls will lead to invalid program state.

[^3]: [Microsoft docs](https://learn.microsoft.com/en-us/windows/win32/memory/comparing-memory-allocation-methods).

[^4]: Thrashing refers to a scenario where the underlying implementation of a collection that gets repeatedly added
    to and removed from causes repeated expensive operations relating to memory (re)allocation/deallocation (in this case,
    repeated calls to `s/brk()` or internal heap manipulation caused by `HeapAlloc()`).

[^5]: POSIX is a collection of features/API that different operating systems can implement for ease of cross-platform
    development and intercompatibility. Many versions of unix are fully/partially POSIX compliant, including linux and
    MacOS.

[^6]: To keep the ledgerging simple, your operating system dispenses memory in entire *pages*. Pages are often 4KiB in
    size, but there are several sizes that you can set.

[^7]: Lazy allocation/initialization/computation refers to the idea of beginning a task, not certain if you're going
    to utilize the resource (but it is avaliable if you need it).

[^8]: An invariant in a data structure means that the data structure has a certain articulable property that *is
    always true*. In this case, our invariant is that there are never two adjacent free nodes (because if there were,
    our code responsible for merging free nodes would have coalesced them into a single node).

[^9]: [This article](https://blog.gceasy.io/what-is-java-heap-fragmentation/) explains heap fragmentation very simply

[^10]: [A skip list](https://en.wikipedia.org/wiki/Skip_list) is a data structure similar to a linked list, except it
    has additional pointers pointing *several spaces* left and right in the linked list, Allowing for a binary search to be
    performed on it. Downside is, your linked list nodes take `O(log(n))` space where `n` is the number of elements in the
    linked list.

[^11]: [Link to paper](https://arxiv.org/abs/2204.10455).
