---
layout: post
categories: [Parrot, Embedding]
title: Parrot in Parrot, A New Frontend
---

Several days ago I wrote a post about writing a new
[JavaScript in JavaScript][jsinjs] compiler for Parrot. It seems reasonable to
me that JavaScript hackers would rather write their compiler for JavaScript in
JavaScript itself instead of having to write it in C or PIR or NQP or
whatever. With that idea in the back of my head and with some of the recent [IMCC][imcc_cleanups] and [Packfile][packfile_tasklist] changes that I've
been working on recently, I started formulating an interesting idea about what
to do with Parrot's frontend in the coming months.

[jsinjs]: http://whiteknight.github.com/2010/12/07/javascript_on_parrot_plan.html
[imcc_cleanups]: /2011/01/15/packfile_changes_and_compilers.html
[packfile_tasklist]: /2011/01/15/packfile_changes_and_compilers.html

For the basic PIR compiler frontend (better known as the current Parrot
executable) with the task of compiling a program written in PIR down to PBC
and executing it, we have a few basic tasks that we need to do:

1. Create the interpreter. This includes some basic command-line parsing for a
   handful of parameters that must be set during interpreter initialization.
   Specifically `--hash-seed`, `--gc`, and `--gc-threshold`.
2. Parse the rest of the flags, separating out the arguments into three
   sets: The settings for the interpreter itself (warning and debug flags to
   activate, a few other settings) the settings for IMCC (the format of the
   input and output files, if any), and the arguments to pass to the PIR
   program.
3. Invoke IMCC to parse the input file into a proper Packfile, passing it
   any relevant commandline arguments.
4. Wrap the arguments for the user program up into an array PMC
5. Execute the program with the wrapped argument PMC.
6. Destroy the interpreter and exit the program.

This is a pretty simple list of things. My idea starts off with an equally
simple premise: What if we did less startup work in C, and jumped into a
quick PIR "prefix" to do the rest? We would need to make a few modifications
to IMCC, maybe add a function or two to the embedding API, and change a few
things in the makefile, but it shouldn't be too hard. The sequence would look
like this:

In C:

1. Create the interpreter, parsing the necessary subset of commandline
   arguments.
2. Wrap up the commandline arguments into an array PMC (or even a Hash PMC)
3. Get a reference to the prefix bytecode (compiled in to the binary, like a
   pbc_to_exe program), load it into the interpreter and jump directly to it,
   passing all remaining arguments.

In PBC:

4. Create a new [PIR compiler PMC][pirpmc]. Register it with `compreg`.
5. Parse out the remaining commandline arguments, separating out the ones that
   go to the user program, and using the rest to set parameters on the interp
   and the PIR compiler PMC.
6. Use the new PIR compiler PMC to compile the executable into a PackFile PMC.
7. Get a reference to the `:main` function from the packfile and execute it,
   passing in user arguments.
8. Do any necessary cleanup and exit the prefix program.

In C Again:

9. Destory the interpreter and exit the program.

[pirpmc]: /2011/01/18/imcc_interface_functions.html

This doesn't necessarily look any simpler, but the reality is that we end up
with much cleaner code. All the code in PIR for instance can be wrapped in
a single exception handler, instead of having to check the output of every
single C API call for success. We also gain the ability to perform many tasks
in PBC through the runloop which really want to be done that way: argument
processing (PCC is much more powerful than `longopt`) and the ability to write fundamental PMC types needed for bootstrapping in PIR (the new IMCC compiler
PMC being the perfect example). PIR is also the most natural place to be
creating PMCs like the PIR compiler PMC, registering it, and calling methods
on it. Plus, we really get a jump on the idea of rewriting portions of Parrot
in Lorito.

Believe it or not, there could actually be a performance *boost* from doing
this, although it would probably be negligible in size.

The new PIR compiler PMC that I've been talking about will probably be written
in C initially, but with a PIR frontend we can write the compiler wrapper
itself in PIR, and register it there. Also, we can start to do really cool
things, like:

    .sub main :main :multi(...)
        .param pmc help :named("--help")
        # Print usage info here
    .end
    
    .sub main :main :multi(...)
        .param pmc args
        # Run the program here
    .end
    
So a packfile can define a multisub as it's main entry point, and the frontend
can process the necessary arguments and dispatch to it. This is just a short
example and obviously it raises more questions than it answers, but it is
interesting to think about. The Rakudo guys have been doing something like
this in their compiler, but Perl6 has defined dispatch semantics for the
program entrypoint routine, and Parrot needs to be flexible enough to support
multiple schemes.

As a short aside, I do believe it would be far better if we processed Parrot's
command-line arguments into a Hash PMC instead of an Array PMC because we
suddenly gain `O(1)` access to our arguments instead of needing to `O(N)`
search for each one we care about. 

Anway, I've gotten off topic. I may expand on this more later.

Here is a short example of what a basic entry-point routine in Parrot would
look like, after some of the proposed changes to the PIR compiler and
[Exception PMC backtrace improvements][backtraces]:

    .sub __parrot_entry_point :anon :main
        .param pmc args
        
        .local pmc interp
        interp = get_interp
        push_eh __global_ex_handler

        .local pmc pir_compiler
        pir_compiler = new ['PIRCompiler']
        compreg "PIR", pir_compiler

        .local pmc compiler_args
        .local pmc program_args
        .local string program_file
        .local pmc interp_args
        (program_file, compiler_args, program_args, interp_args) = '__parse_args'(args)
        interp.'set_options'(interp_args)
        pir_compiler.'set_options'(compiler_args)

        .local pmc packfile
        .local pmc program_main
        packfile = pir_compiler.'compile_file'(program_file)
        packfile.'run_init_functions'()
        program_main = packfile.'get_main'()
        program_main(program_args)
        exit 0

      __global_ex_handler:
        .local pmc exception
        .local int exit_code
        .get_results(exception)
        finalize exception
        pop_eh
        __print_exception_backtrace(exception)
        exit_code = __get_exception_exit_code(exception)
        exit exit_code
    .end

[backtraces]: /2011/01/14/exception_backtraces.html
    
This code is pretty straight-foward, though it would get a little bit more
complicated if we wanted to support additional options such as the `-o`
commandline argument, which compiles the program to a .pbc file and writes it
out but does not execute it, or the `-r` option which compiles to an output
.pbc file like `-o`, but then immediately reads in from that .pbc file and
executes it. However, even with these options added I suspect a prefix program
like this would be less than 500 lines of well-written PIR. Depending on how
much or how little we want to do here, it could be much less.

By writing a frontend routine in PIR itself (or, any Parrot language for that
matter), I think we gain a couple of things. First, we get Parrot processing
power immediately, instead of having to do a bunch of operations in C,
carefully converting types back and forth between C types and Parrot types,
and having check every single operation for a thrown exception.

Second, by registering the PIR compiler PMC from running PBC code and not
directly from inside libparrot during interpreter initialization, we decouple
the two systems and gain the ability to remove IMCC from libparrot and move
it into its own library. In such a system, it's easily conceivable that we
never register a compiler for PIR if one isn't needed by the user.

Third, we gain a lot more flexibility with command-line arguments, and can
start talking about using multidispatch semantics for `:main`, among other
things

Fourth, we can start defining things, like HLL type mappings and other
semantic modifications before we ever enter the compiler. This will allow the
compiler to automatically setup constant values using user-defined subtypes,
or to register in necessary utility pieces like an NCI call frame generator,
meta object protocol types, multidispatch handlers, concurrency scheduler, and
other pieces of a fully-pluggable Parrot before we ever enter into the
compiler. Instead of having to set up `:anon :init :load` routines to register
these types during compilation, we have them ready for us beforehand.

Really it's a lot to take in and a lot to think about, but I predict we could
have a PBC front-end routine for Parrot like what I've described above by the
3.3 release in April if we really want it. I think I do want it. What do other
people think?
