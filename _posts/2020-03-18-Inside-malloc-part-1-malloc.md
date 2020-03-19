---
layout: post
title: Inside of memory allocation - part 1 - malloc().
---

Heap overflow at it's finest. 

_Note: I'm also learning. If you find any wrong or misleading information, please write to me._  
_Note: All the code you may find at my github_  
_Note: In this blog posts I'm using sources from glibc-2.27. Structures, code and macros may differ between versions._
_Note: I will focus on 32-bit systems as in my opinion those are easier to understand._

Hello everyone after all this time! Lots of things were going on but I will not bore you with that. Today I want to present you insides of malloc function and memory allocation in general which I hope will help you to understand how heap overflow works. 

To start, we need a vulnerable program, like the one below. Our goal would be to print the secret to standard output.
```cpp
#include<stdlib.h>
#include<stdio.h>

void secret(){
    printf("%s", "53CR3T");
}

int main(){
    char *first = (char*)malloc(100);
    char *second = (char*)malloc(100);
    gets(first);
    free(first);
    free(second);
    puts("The end.");
    return 0;
}
```
The program is fairly easy:
1. Allocate memory for two buffers,
2. Write data to _first_ buffer.
3. Free alocated data.

For someone who's not into security this program looks normal - we allocated memory and then freed it. So what's the problem? I want you to focus on this next sentence, because it is very, very important. Never ever use _gets_! If you are a man who reads documentation and warnings that compiler prints out, you already know that "it is impossible to tell without knowing the data  in  advance  how  many  characters  gets() will read, and because gets() will continue to store characters past the end of the buffer, it is  extremely  dangerous  to  use"[1] and "the `gets' function is dangerous and should not be used".[2]
This particular vulnerability will let us to overwrite the buffer from _second_.

_Okey, it will let us to override the second buffer and maybe segfault, but how can we redirect code execution to secret function?_

And here we will go deep into malloc (prepare yourself for lots of theory)! Firstly let's focus on what malloc allocates. Defienietly some data structure and in this case it's called _memory chunk_. There are two types of memory chunks - an allocated chunk and free chunk:  
An allocated chunk looks like this[3]:
```
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ <--- chunk
|             Size of previous chunk, if unallocated (P clear)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Size of chunk, in bytes                     |A|M|P|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ <--- mem
|             User data starts here...                          .
.                                                               .
.             (malloc_usable_size() bytes)                      .
.                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ <--- nexchunk
|             (size of chunk, but used for application data)    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Size of next chunk, in bytes                |A|0|1|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
Where "chunk" is the front of the chunk for the purpose of most of the malloc code, but "mem" is the pointer that is returned to the user.  "Nextchunk" is the beginning of the next contiguous chunk.  
Chunks always begin on even word boundaries, so the mem portion (which is returned to the user) is also on an even word boundary, and thus at least double-word aligned.  
Free chunks are stored in circular doubly-linked lists, and look like this:  
```
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ <--- chunk
|             Size of previous chunk, if unallocated (P clear)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Size of chunk, in bytes                     |A|0|P|  'head:'
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ <--- mem
|             Forward pointer to next chunk in list             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Back pointer to previous chunk in list            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Unused space (may be 0 bytes long)                .
.                                                               .
.                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ <--- nextchunk
|             Size of chunk, in bytes                           |  'foot:'
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Size of next chunk, in bytes                |A|0|0|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
In code above structures are presenting like this:  
```cpp
struct malloc_chunk {

  INTERNAL_SIZE_T      mchunk_prev_size;  /* Size of previous chunk (if free).  */
  INTERNAL_SIZE_T      mchunk_size;       /* Size in bytes, including overhead. */

  struct malloc_chunk* fd;         /* double links -- used only if free. */
  struct malloc_chunk* bk;

  /* Only used for large blocks: pointer to next larger size.  */
  struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */
  struct malloc_chunk* bk_nextsize;
};
```
Now lets imagine we called lots of mallocs and freed every second of them. It caused fragmentation of our memory. It theory we have got half of it free, but malloc needs to know which one were freed and what are their sizes in order to use them again. There should be some way to tell malloc which adresses have been freed...  
And indeed there is! This way are _bins_. Malloc source code present bins as follows[3]:
```
Bins

    An array of bin headers for free chunks. Each bin is doubly
    linked.  The bins are approximately proportionally (log) spaced.
    There are a lot of these bins (128). This may look excessive, but
    works very well in practice.  Most bins hold sizes that are
    unusual as malloc request sizes, but are more usual for fragments
    and consolidated sets of chunks, which is what these bins hold, so
    they can be found quickly.  All procedures maintain the invariant
    that no consolidated chunk physically borders another one, so each
    chunk in a list is known to be preceeded and followed by either
    inuse chunks or the ends of memory.

    Chunks in bins are kept in size order, with ties going to the
    approximately least recently used chunk. Ordering isn't needed
    for the small bins, which all contain the same-sized chunks, but
    facilitates best-fit allocation for larger chunks. These lists
    are just sequential. Keeping them in order almost never requires
    enough traversal to warrant using fancier ordered data
    structures.

    Chunks of the same size are linked with the most
    recently freed at the front, and allocations are taken from the
    back.  This results in LRU (FIFO) allocation order, which tends
    to give each chunk an equal opportunity to be consolidated with
    adjacent freed chunks, resulting in larger free chunks and less
    fragmentation.

    To simplify use in double-linked lists, each bin header acts
    as a malloc_chunk. This avoids special-casing for headers.
    But to conserve space and improve locality, we allocate
    only the fd/bk pointers of bins, and then use repositioning tricks
    to treat these as the fields of a malloc_chunk*.
```

There are 4 types of bins:
  Fastbins - memory chunks up to 80 bytes, arranged in LIFO order, freed in `malloc_consolidate`.
  ```
    malloc manages fastbins very conservatively yet still
    efficiently, so fragmentation is rarely a problem for values less
    than or equal to the default.  The maximum supported value of MXFAST
    is 80. You wouldn't want it any higher than this anyway.  Fastbins
    are designed especially for use with many small structs, objects or
    strings -- the default handles structs/objects/arrays with sizes up
    to 8 4byte fields, or small strings representing words, tokens,
    etc. Using fastbins for larger objects normally worsens
    fragmentation without improving speed.
    (...)
    Fastbins are not doubly linked.  It is faster to single-link them, and
    since chunks are never removed from the middles of these lists,
    double linking is not necessary. Also, unlike regular bins, they
    are not even processed in FIFO order (they use faster LIFO) since
    ordering doesn't much matter in the transient contexts in which
    fastbins are normally used.

    Chunks in fastbins keep their inuse bit set, so they cannot
    be consolidated with other free chunks. malloc_consolidate
    releases all chunks in fastbins and consolidates them with
    other free chunks.
  ```

  Unsorted bin - when chunk of any size (small or large) gets freed it firstly is placed in unsorted bin, when malloc gives it another one chance to be used (FIFO order).
  ```
    All remainders from chunk splits, as well as all returned chunks,
    are first placed in the "unsorted" bin. They are then placed
    in regular bins after malloc gives them ONE chance to be used before
    binning. So, basically, the unsorted_chunks list acts as a queue,
    with chunks being placed on it in free (and malloc_consolidate),
    and taken off (to be either used or placed in bins) in malloc.
  ```

  Small bins - following code presents a way to count maximum small chunk size - after doing some math you will get 512 bytes (for 32-bit system and 1024 for 64-bit) (FIFO order). Chunks inside bins are the same size.
  ```
    #define in_smallbin_range(sz)  \
        ((unsigned long) (sz) < (unsigned long) MIN_LARGE_SIZE)
    (...)
    #define MIN_LARGE_SIZE    ((NSMALLBINS - SMALLBIN_CORRECTION) * SMALLBIN_WIDTH)
    (...)
    #define NSMALLBINS         64
    #define SMALLBIN_WIDTH    MALLOC_ALIGNMENT
    #define SMALLBIN_CORRECTION (MALLOC_ALIGNMENT > 2 * SIZE_SZ)
    (...)
    /* MALLOC_ALIGNMENT is the minimum alignment for malloc'ed chunks.  It
    must be a power of two at least 2 * SIZE_SZ, even on machines for
    which smaller alignments would suffice. It may be defined as larger
    than this though. Note however that code and data structures are
    optimized for the case of 8-byte alignment.  */
    #define MALLOC_ALIGNMENT (2 * SIZE_SZ < __alignof__ (long double) \
			  ? __alignof__ (long double) : 2 * SIZE_SZ)
    (...)
    /* The corresponding word size.  */
    #define SIZE_SZ (sizeof (INTERNAL_SIZE_T))
    (...)
    - INTERNAL_SIZE_T might be signed or unsigned, might be 32 or 64 bits
  ```

  Large bins - chunks of size more than 512 bytes (FIFO order).

You have come this far? Nice! Now we will talk about implemetation of `malloc` and `free`.  
So what happens when you call `malloc`?   
Signature of malloc looks like follows:
```cpp
static void *
_int_malloc (mstate av, size_t bytes)
```
After definitions of some necessary variables (if any is needed I will show it to you, for now we may omit them), size check is performed to block allocation of too big chunks. If chunk is to big `ENOMEM = 0x4000000c,	/* Cannot allocate memory */` error will be returned:
```cpp
  /*
     Convert request size to internal form by adding SIZE_SZ bytes
     overhead plus possibly more to obtain necessary alignment and/or
     to obtain a size of at least MINSIZE, the smallest allocatable
     size. Also, checked_request2size traps (returning 0) request sizes
     that are so large that they wrap around zero when padded and
     aligned.
   */

  checked_request2size (bytes, nb);   
   ~
   |
   |  This is defined as follows
   |
  \./
  /* Same, except also perform an argument and result check.  First, we check
   that the padding done by request2size didn't result in an integer
   overflow.  Then we check (using REQUEST_OUT_OF_RANGE) that the resulting
   size isn't so large that a later alignment would lead to another integer
   overflow.  */
  #define checked_request2size(req, sz) \
  ({				    \
    (sz) = request2size (req);	    \
    if (((sz) < (req))		    \
        || REQUEST_OUT_OF_RANGE (sz)) \
      {				    \
        __set_errno (ENOMEM);	    \
        return 0;			    \
      }				    \
  })
  (...)
  /* pad request bytes into a usable size -- internal version */
  #define request2size(req)                                         \
  (((req) + SIZE_SZ + MALLOC_ALIGN_MASK < MINSIZE)  ?             \
   MINSIZE :                                                      \
   ((req) + SIZE_SZ + MALLOC_ALIGN_MASK) & ~MALLOC_ALIGN_MASK)
  (...)
  /*
   Check if a request is so large that it would wrap around zero when
   padded and aligned. To simplify some other code, the bound is made
   low enough so that adding MINSIZE will also not wrap around zero.
 */
#define REQUEST_OUT_OF_RANGE(req)                                 \
  ((unsigned long) (req) >=						      \
   (unsigned long) (INTERNAL_SIZE_T) (-2 * MINSIZE))
```
Later we check if the there are any usable areas. What is an `area`?
```
arena:     current total non-mmapped bytes allocated from system
```
I think this is pretty self-explainatory. As it is very unlikley that there is no arena, I will skip this part.  
Next we are checking if requested bytes fits into fastbin - take a look at `set_max_fast` as by default every chunk up to 64 bytes is by default fastbin and not up to 80:
```cpp
 if ((unsigned long) (nb) <= (unsigned long) (get_max_fast ()))
  ~
  |
  |     This is defined as follows
 \./
 static inline INTERNAL_SIZE_T
get_max_fast (void)
{
  if (global_max_fast > MAX_FAST_SIZE)
    __builtin_unreachable ();
  return global_max_fast;
}
(...)
#define set_max_fast(s) \
  global_max_fast = (((s) == 0)						      \
                     ? SMALLBIN_WIDTH : ((s + SIZE_SZ) & ~MALLOC_ALIGN_MASK))
(...)
/* The maximum fastbin request size we support */
#define MAX_FAST_SIZE     (80 * SIZE_SZ / 4)
(...)
#if MORECORE_CONTIGUOUS
  if (av != &main_arena)
#endif
  set_noncontiguous (av);
  if (av == &main_arena)
    set_max_fast (DEFAULT_MXFAST);
  atomic_store_relaxed (&av->have_fastchunks, false);
(...)
#define DEFAULT_MXFAST     (64 * SIZE_SZ / 4)
```
If chunk fits in fastbin we need an index of size bin. There are 10 fastbins, each separated by 8 bytes in size - so first one is 16 bytes, second 24 and so on, and so on.
```cpp
    idx = fastbin_index (nb);
    mfastbinptr *fb = &fastbin (av, idx);
    mchunkptr pp;
    victim = *fb;
    ~
    |
    |   This is defined as follows
   \./
   #define fastbin(ar_ptr, idx) ((ar_ptr)->fastbinsY[idx])

    /* offset 2 to use otherwise unindexable first 2 bins */
    #define fastbin_index(sz) \
      ((((unsigned int) (sz)) >> (SIZE_SZ == 8 ? 4 : 3)) - 2)


    /* The maximum fastbin request size we support */
    #define MAX_FAST_SIZE     (80 * SIZE_SZ / 4)

    #define NFASTBINS  (fastbin_index (request2size (MAX_FAST_SIZE)) + 1)
```
You need to remeber that if this is the first malloc, the fastbins are empty, so even if the size id smaller than 64, small chunk will be returned. If fasbins are in the list, take the first element out of it and returns it to the user.
```cpp
    if (victim != NULL)
	{
	  if (SINGLE_THREAD_P)
	    *fb = victim->fd;
	  else
	    REMOVE_FB (fb, pp, victim);
    (...)
    void *p = chunk2mem (victim);
	alloc_perturb (p, bytes);
	return p;
```
Ok, so we are now out of the fastbins. The next check that is performed checks if declared size fits into small bins range.
```cpp
 /*
     If a small request, check regular bin.  Since these "smallbins"
     hold one size each, no searching within bins is necessary.
     (For a large request, we need to wait until unsorted chunks are
     processed to find best fit. But for small ones, fits are exact
     anyway, so we can check now, which is faster.)
   */

    if (in_smallbin_range (nb))
      {
        idx = smallbin_index (nb);
        bin = bin_at (av, idx);
    ~
    | 
    |   This is defined as follows
   \./
    #define in_smallbin_range(sz)  \
      ((unsigned long) (sz) < (unsigned long) MIN_LARGE_SIZE)
    (...)
    #define smallbin_index(sz) \
    ((SMALLBIN_WIDTH == 16 ? (((unsigned) (sz)) >> 4) : (((unsigned) (sz)) >> 3))\
     + SMALLBIN_CORRECTION)
    (...)
    /* addressing -- note that bin_at(0) does not exist */
    #define bin_at(m, i) \
    (mbinptr) (((char *) &((m)->bins[((i) - 1) * 2]))			      \
             - offsetof (struct malloc_chunk, fd)
```
After this we check if bins are pointing to themselves and if not we are returning the last element from the bin.
```cpp
    if ((victim = last (bin)) != bin)
        {
          bck = victim->bk;
	  if (__glibc_unlikely (bck->fd != victim))
	    malloc_printerr ("malloc(): smallbin double linked list corrupted");
          set_inuse_bit_at_offset (victim, nb);
          bin->bk = bck;
          bck->fd = bin;
```
For large fragments, this procedure consists of 3 lines of code, which I leave understanding as an exercise:
```cpp
 /*
     If this is a large request, consolidate fastbins before continuing.
     While it might look excessive to kill all fastbins before
     even seeing if there is space available, this avoids
     fragmentation problems normally associated with fastbins.
     Also, in practice, programs tend to have runs of either small or
     large requests, but less often mixtures, so consolidation is not
     invoked all that often in most programs. And the programs that
     it is called frequently in otherwise tend to fragment.
   */

  else
    {
      idx = largebin_index (nb);
      if (atomic_load_relaxed (&av->have_fastchunks))
        malloc_consolidate (av);
    }
```
The next part of malloc function is taking care of placing chunks from unsorted bin to other bins or it picks chunks to use those again. I do not want to go on this one into details, the most usable thing you need to know is that every freed chunked goes to unsorted bin to be given another one chance to be allocated and what's more important it will be alocated again only if it is a exact fit.
```cpp
/*
     Process recently freed or remaindered chunks, taking one only if
     it is exact fit, or, if this a small request, the chunk is remainder from
     the most recent non-exact fit.  Place other traversed chunks in
     bins.  Note that this step is the only place in any routine where
     chunks are placed in bins.

     The outer loop here is needed because we might not realize until
     near the end of malloc that we should have consolidated, so must
     do so and retry. This happens at most once, and only when we would
     otherwise need to expand memory to service a "small" request.
   */
```

And for today this is it! I decided to divide it into parts. The next one will be about free() and the third one about exploiting the heap.
Hope you enjoyed it and take care!

----
****
[1]. Linux manual page of _gets_, section BUGS.  
[2]. Warning GCC gaves you whenever compiling sources with _gets_ inside.  
[3]. Source code of malloc.c; https://elixir.bootlin.com/glibc/glibc-2.27/source/malloc/malloc.c#L4138

