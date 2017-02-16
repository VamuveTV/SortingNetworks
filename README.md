*I present novel and state-of-the-art sorting of 4 int32_t
and sorting of 6 int8_t, as examples, using SSE4, and some thoughts
and notes on fast sorting of small fixed-size arrays.*

These are the fastest known approaches on modern CPUs.
They are completely branchless and extremely fast.
For example, 4 int32_t can be sorted in ~18 cycles on Skylake.

These examples can be modified to sort signed or unsigned data as long
as the total size of the data is <= 128 bits (16 bytes). Note that
trying to use AVX2 to push that to 256 bits will NOT HELP
because of the 3 cycle interlane latency that makes shuffling
across the 128-bit lane boundary too expensive to be useful.

In that case, or in the case that you can't support SSE4,
efficient scalar code can also be generated that isn't too much
slower, but it's not immediately evident what the optimal approach
is in assembly, nor how to coerce C/C++ compilers to produce that
assembly from C/C++ code. (I discuss the approach at the end of this
documentation.)

Not all compilers correctly generate optimal assembly for either the
SSE or scalar code, so I provide assembly versions too. In fact, this
is not a strong enough statement; many compilers FAIL MISERABLY
at both the SSE and scalar cases for n >= 3. CL in particular
(Microsoft Visual C++) just dies horribly in a fire, so the assembly
snippets may not be a bad idea. Profile and/or check your disassembly.

Note that these assembly snippets use the Windows x64 calling convention,
but then proceed to clobber a bunch of registers they shouldn't. Normally
they'd be inlined. Feel free to callee-save registers that your
application cannot safely clobber.

These code snippets work on the principle of sorting networks.
Conventional sorting algorithms like quicksort and mergesort
are not great at small array sizes. One notices in profiling that
simpler sorts like insertion and selection tend to win. However,
even these are not NEARLY as fast as they could be for
fixed-size, small arrays.

Available sorts and their function names:

>>> SSE Assembly (fast as hell. FASTEST option on modern CPUs.
					 these are written in MASM for Win64;
					 but it's Intel syntax and you can make the small
					 modifications required for other assemblers.)
Sorting 4 int32_t  |  simdsort4a()
Sorting 4 int32_t  |  simdsort4a_noconstants()
Sorting 4 int32_t  |  simdsort4a_nofloat()

>>> SSE C++ (make sure generated assembly matches):
Sorting 4 int32_t  |  simdsort4()
Sorting 6 int8_t   |  simdsort6()

>>> Scalar Assembly (these are written in MASM for Win64;
						but it's Intel syntax and you can make the small
						modifications required for other assemblers.)
Sorting 2 int32_t  |  sort2a()
Sorting 3 int32_t  |  sort3a()
Sorting 4 int32_t  |  sort4a()
Sorting 5 int32_t  |  sort5a()
Sorting 6 int32_t  |  sort6a()

>>> Scalar C++ (make sure generated assembly matches)
Sorting 2 int32_t  |  sort2()
Sorting 6 int32_t  |  sort6()


Okay, if you've made it this far, let's discuss
fast CAS operations, which are the heart of sorting
networks. CAS == Compare And Swap. Given two values,
'a', and 'b', leave them as they are if 'a' is less
than 'b', i.e. if they are in sorted order. However,
swap them if 'a' is greater than or equal to 'b'.
Thus after a CAS operation 'a' and 'b' are in sorted
order no matter what order they came in as.

A series of CAS operations can deterministically sort
a fixed-size array. Typically one can optimize for depth
(minimum number of operations given infinite parallel
processing) or for size (minimum number of operations given
no parallel processing). For n == 4 these two optimal solutions
are actually given by the same network, with depth 3 and
size 5.

Scalar first: how do you efficiently CAS? Again, note that
lots of compilers don't produce optimal assembly no matter
what C++ you give them. But what is the optimal assembly?
Well, on modern processors, the answer is conditional moves:

	; inputs: eax, r9d
	; scratch register: edx
	cmp	eax, r9d
	mov	edx, eax
	cmovg	eax, r9d
	cmovg	r9d, edx
	; eax and r9d have been swapped if necessary such that eax is now <= r9d

See the function 'sort6' in 'sorts.cpp' for an attempt at some C++ code
that has a decent chance of compiling into CAS operations that look like that.
Again, they OFTEN DON'T, especially the CL compiler and g++. Use the assembly
snippets instead, or at least profile and inspect your disassembly to be sure.

The SSE sorts are rather more sophisticated, but the basic principle
is to branchlessly generate shuffle index masks at runtime and then
use them to permute the order of the data in parallel, performing
one complete level of the sorting network at a time.

I provide 3 versions of the assembly for sorting 4 int32_t. The fastest
should be the plain 'simdsort4a'. It performs float reinterpretation
and relies on some constant loads, but both of these are USUALLY
better than the alternatives. However:

Older CPUs may incur too much latency from the reinterpretation to be
worth it. For such CPUs, try 'simdsort4a_nofloat.asm'.

Applications that cannot afford to have these constants occupying L1
may wish to try 'simdsort4a_noconstants.asm', which does not eat
up any cache space with constants - everything is done with immediates
and some fairly nasty bit twiddling.
