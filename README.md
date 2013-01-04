********

Ignore this repository.  AlephOne is the full source code for all of this.
Refer to that.  This is especially important if you are trying to get MIDI right
from the few hints that only exist here in DSPCompiler.  AlephOne is the full source
for the iOS app "Cantor".

********



The part of this project that is mature is the Fretless and DeMIDI code.  These are the literal code that is used for AlephOne's MIDI generation.  It works very well for fretless MIDI.  It includes polyphony and fretless handling, while intonation and pitch rounding are not included here.  Contact me if you are trying to use this for iOS and need more detail than what is included here.  (No, I will not post a fully buildable iOS project on github again, especially before I release my own version into the store.)

* Fretless provides a way to render MIDI as fully polyphonic fretless
* It includes notions of special polyphony rules (full poly, string mode, solo mode), via the abstraction of polyphonic groups, which are a notion like channels within an instrument.
* It has custom note ties for MIDI, which are required to allow bending to work correctly in spite of how MIDI specifies it should work.
* The note ties are used not only to get arbitrary bends regardless of the bend setting, but for legato as well (legato defined as restarting audio phase on new note versus continuing the phase for note in same polyphony group)






Experimental
Experimental
Experimental Unfinished



I am working on AlephOne, and though I am not quite ready to open source the whole project, there are bits of it that are definitely appropriate for being open sourced because getting compatibility among developers is a major issue right now when dealing with MIDI.  Now that I am working on a custom sound engine that deals very directly with my special MIDI variant, I am hoping to get some help from other developers that use vDSP routinely.   So here's a compiler to generate vDSP kernels that is written in Python so that it is easy to change.

Apple's vDSP is a library from the Accelerate.Framework that lets you write data parallel code (ie: vector computing).  It is essentially an assembly language called from a C library where userspace arrays are treated as giant registers in which every element in the array is synchronously computed at the same time.  Projects like OpenCL and CUDA exist specifically because of this problem, though OpenCL isn't quite yet available on iOS yet (and therefore on older devices).  The principles are similar whether the parallelism comes from running on a GPU or from using vector instructions on the main processor.

This is the beginnings of a project to handle vDSP code in a reasonable way.  It is a Python compiler that generates the C snippets that would be very hard to maintain if vDSP code is written directly.  It takes as input a variant of LISP, so that it is trivial to write the compiler and write additions and optimizers to it as the need arises.  

The main problem I have with vDSP is when I am translating a function from my inner loop that was written by iterating per sample, the code turns inside out.  You end up writing something that looks like what the compiler would have generated (like OpenCL and CUDA kernels).  For example, the original code might look like:

	//Compute something over a loop
	//There could be a lot of statements in here at some point
	for(int i=0;i<length;i++)
	{
		float a = b[i] * c[i];
		d[i] = 1 - a[i];
	}

So when you start refactoring the code, it begins to look like this:

	#define INPARALLEL(statement,length) { for(int i=0; i<length; i++) { statement } }

	INPARALLEL(a[i]  = b[i],length)
	INPARALLEL(a[i] *= c[i],length)
	INPARALLEL(d[i]  = 1,length)
	INPARALLEL(d[i] -= a[i],length)

Which roughly translates to how it gets done with calls to vDSP:

	xDSP_vcp(b,a,length);
	vDSP_vmul(c,1, a,1, a,1, length);
	vDSP_vsfill(d,1, &one, length);
	vDSP_vsub(d,1, a,1, d,1, length);

So, the lack of clarity and the task of allocating an optimal number of temporary "registers" as code changes can make this kind of code hard to maintain.  A LISP syntax custom language can make it trivial to make a compiler for, and Python is a reasonable language for tasks which require a lot of symbolic manipulation, and it is very widely available and well known.  Here is a source file to be turned into a vDSP kernel:

	( do
		(vset output  (vadd (vmul w0 w1) (vsmul w2 w3)))
		(vset output1 (vadd a (vmul x (vadd y0 y1))))
		(vset output2 (vadd c (vmul output1 (vsadd y2 sy3))))
	)

LISP is more of a family of languages based around this trivial syntax.  Its main benefit is that it is a direct representation of the syntax tree, and it is easy to treat it as a data structure.  With a small Python based compiler for it, adding in support for some of the more specialized vDSP functions and writing optimizers to avoid redundant work is really quite easy.
 
When run something through the compiler, it generates this (notice that the number of registers used is minimized):


	/*( do
	    (in x y z one b c)
	    (vset w (vadd (vmul x y)(vsmul z one)))
	    (vset a (vadd (vmul b c)(vmul z w)))
	    (out w)
	)*/
	vDSP_vmul(x,1,y,1,accumulator1,1,index);
	vDSP_vsmul(z,1,&one,accumulator2,1,index);
	vDSP_vadd(accumulator1,1,accumulator2,1,w,1,index);
	vDSP_vmul(b,1,c,1,accumulator2,1,index);
	vDSP_vmul(z,1,w,1,accumulator1,1,index);
	vDSP_vadd(accumulator2,1,accumulator1,1,a,1,index);
