I've started work on trying to add multiple dispatch support to Winxed. The
patch is going along surprisingly well, and I should have a pull request
opened up for NotFound to review before the 3.7 release. I don't know if he
will have time to review and merge it before then, but I want to have it
available in any case. The current patch doesn't offer all the flexibility and
power that the Parrot version of it does, but it does get the basics in place
for us to expand upon later.

As a Winxed user, I haven't made a heck of a lot of use of Parrot's MMD
features. I've used it in NQP, but the details are sufficiently abstracted in
that language that you don't really get the feel for what is occuring at the
lower levels. Since the feature is so messy, I've made some effort to avoid
using it at the PIR level. Let me rephrase that. I've made some effort to
avoid PIR entirely.

To really get a handle on multiple dispatch, I did what I always do: I went
right to the source. I opened up the MultiSub PMC, which is the primary
user-visible entry way into the multiple dispatch system. What I found there
was...underwhelming. The MultiSub PMC is not descended from the Sub PMC. It's
basically an array which does basic type-checking on insert operations
(push_pmc, set_pmc_keyed_int, etc) to ensure that objects being added to the
array are indeed Sub PMCs. Actually, it wasn't even consistent, some of the
insert vtables checked whether the PMC was a Sub, other insert vtables checked
whether the PMC satisfied the "invokable" role. While similar, the two checks
will allow different PMCs. I found several vtables that were redundant and
unnecessary, and I found a few other problems as well.

Combine that with what I know about the shortcomings of the Sub PMC, and
nasty code in the associated subsystems (`src/sub.c`, `src/multidispatch.c`,
etc), and I think we have a major problem on our hands. In this post, which
could potentially turn into a long series of posts, I'm going to talk about
some of the problems with Subs and the way I plan to fix them. MultiSub is
one of the pieces that is going to come along for the ride.

Want me to prove to you that we have a major problem? Here is the complete,
unadulterated list of attributes for the Sub PMC:

    ATTR PackFile_ByteCode *seg;        /* bytecode segment */
    ATTR size_t             start_offs; /* sub entry in ops from seg->base.data */
    ATTR size_t             end_offs;
    ATTR INTVAL             HLL_id;         /* see src/hll.c XXX or per segment? */
    ATTR PMC               *namespace_name; /* where this Sub is in - this is either
                                             * a String or a [Key] and describes
                                             * the relative path in the NameSpace */
    ATTR PMC               *namespace_stash; /* the actual hash, HLL::namespace */
    ATTR STRING            *name;            /* name of the sub */
    ATTR STRING            *method_name;     /* method name of the sub */
    ATTR STRING            *ns_entry_name;   /* ns entry name of the sub */
    ATTR STRING            *subid;           /* The ID of the sub. */
    ATTR INTVAL             vtable_index;    /* index in Parrot_vtable_slot_names */
    ATTR PMC               *multi_signature; /* list of types for MMD */
    ATTR UINTVAL            n_regs_used[4];  /* INSP in PBC */
    ATTR PMC               *lex_info;        /* LexInfo PMC */
    ATTR PMC               *outer_sub;       /* :outer for closures */
    ATTR PMC               *eval_pmc;        /* eval container / NULL */
    ATTR PMC               *ctx;             /* the context this sub is in */
    ATTR UINTVAL            comp_flags;      /* compile time and additional flags */
    ATTR Parrot_sub_arginfo *arg_info;       /* Argument counts and flags. */
    ATTR PMC               *outer_ctx;       /* outer context, if a closure */

Have you gone cross-eyed yet? Are you as infuriated by this as I am? If not,
continue reading. If so, continue reading for the lulz.

Here's a question for you: How does Parrot implement closures? Parrot
implements closures by taking a Sub, cloning it, and setting a pointer to the
parent Sub's active CallContext in the `->outer_ctx` field of the child Sub.
In other words, a Closure is not it's own type of thing. It's a Sub, but with
one extra field set in it.

What's the difference between an ordinary Sub and a vtable override? Well, the
vtable override has an index value set in `->vtable_index`. What if we have a
single Sub that we would like to use for two separate vtable slots? What if we
have a single Sub, for something like `set_pmc_keyed` and `set_pmc_keyed_str`,
and we want PCC to automatically coerce arguments from string to PMC to share
a single implementation? The result is major fail.

Let me ask you another question: What is the `namespace_stash`? And, more
importantly, why does the Sub need to know where or how it's being stored?
Keeping track of it's own contents is the business of the NameSpace PMC, not
the job of the Sub PMC. What if we want to reference a single Sub from
multiple namespaces? Or, what if we want to reference a single Sub by multiple
names within a single namespace? What if we want the Sub not to be
automatically stored in *any* namespace at all? Suddenly, `namespace_name`
isn't looking too smart either.

Similarly, isn't it the job of the MultiSub to keep track of the Subs and
their corresponding signatures? I mean, what if I have this Sub:

    .sub 'Foo'
        .param pmc all_args :slurpy
        ...
    .end

...And I want that Sub to be called from a MultiSub for multiple different
signatures, only redirecting certain variants to specific alternatives? If the
MultiSub were some sort of hash or search tree instead of a dumb array, and if
it kept track of the signatures associated with each Sub instead of asking the
Sub to keep track of it's own signature, we gain all that flexibility. Also,
I suspect, there are performance wins to be had if we break a signature
key up into a search tree or search graph and traverse it instead of doing an
in-place manhattan sort on sig lists every time we call the MultiSub.
Basically, I'm saying we should change this:

    push multi_sub, my_sub

To this:

    multi_sub[ ... ] = my_sub

The user can pick the signature, and can reuse a single sub for multiple ones.

When you compile a PIR file with IMCC, IMCC collects all the relevant
information together and jams it all into a single place: The Sub. When Parrot
loads in a packfile, it reads each Sub entry, uses the namespace information
therein to recreate the NameSpace tree, and inserts Subs into the proper
namespaces. Then, when we create a class, the Class searches for the NameSpace
with the same stringified name, and pulls all the methods out of it. Keep in
mind that
[namespaces aren't supposed to hold methods at all][namespace_cleanups], so
the list of methods in the namespace has to be kept separate and hidden until
the class (and only the Class) asks for it. At that point, since the NameSpace
is itching to dump off the responsibility, it deletes its own copy of the list
as soon as it is read.

Similarly, when Parrot loads in a packfile and inserts Subs into the
NameSpace, it's the job of the NameSpace to automatically and invisibly insert
Subs with similar names and the `:multi` flag set into new MultiSub
containers. If you set a Sub with the same name but without the flag set, the
NameSpace overwrites the old one. But if the flag is set, the two are merged
together into a single MultiSub. So here's yet another question for you:

    .local pmc my_namespace     # a NameSpace PMC
    ...
    my_namespace["foo"] = $P0

In this short example, assuming we don't know where `my_namespace` comes from
or what it previously contains, what happens? Well,

1. If `$P0` is any type of PMC *except* a Sub, it's stored as a global,
   overwriting any existing global by the name `"foo"`.
2. If `$P0` is a user-defined subclass of Sub, it's treated differently in
   ways I don't seem to understand.
3. If `$P0` is a Sub with the `:method` flag, it will be stored in a separate,
   secret hash of methods, to be added to the Class of the same name when the
   class is created. *UNLESS* the type in question is a built-in type with a
   PMCProxy metaobject instead of a Class metaobject, then the exact sequence
   of events is mysterious and uncertain, because built-ins can be
   instantiated and used *before* the associated PMCProxy is ever created, so
   there is no single way to fetch all the methods from the NameSpace at once.
4. If `$P0` has the `:multi` flag set, it will get merged into a MultiSub,
   together with a previous Sub of the same name, if any. Unless the previous
   entry is not also a `:multi`, then it overwrites.
5. If `$P0` has the `:vtable` flag set, it will also get stored away in a
   super-secret location, to be grabbed by the Class when necessary, with all
   the same caveats as I mentioned for the `:method` flag, above.
6. If `$P0` is a `:method` or a `:vtable` with the extra `:nsentry` flag set,
   Then it *is* stored in the namespace anyway, in addition to being stored in
   a way that is fetchable by the Class or PMCProxy
7. If `$P0` is a NameSpace, it's stored in the `my_namespace` as a child, and
   becomes a searchable part of the NameSpace tree.

This all sounds like the best, most well-thought-out, best designed and best
implemented solution, doesn't it? And there isn't a hint of magic or confusion
anywhere in sight.

All of that crap, every last bit of it including the problems with the Sub
PMC containing too many unnecessary attributes, is all the symptom. The
single underlying problem that necessitates all of it is that the packfile
loader automatically creates NameSpaces and automatically inserts Sub PMCs
into them when a packfile is loaded. Take that away, force the packfile loader
to stop jamming data where it doesn't belong, and suddenly all the crap I
mentioned above goes away. Piles and piles of the foulest, most garbage-ridden
code I have ever seen evaporates away into a fine mist of unicorn farts. I
say good riddence.

So what's the alternative? Well, 6model doesn't use the NameSpace as
intermediate storage for methods. When you create a class with 6model, you
get individual references to the Subs you want and you insert them, by name,
into the Class. Reuse the same Sub as much as you want. We can extend this
idea even further too, by applying it to MultiSubs. If you want a MultiSub,
create one yourself and insert the functions you want into it. For that
matter we can extend the idea all the way to NameSpaces themselves. Parrot
shouldn't automatically create or populate a NameSpace, the user can create
and populate it. For all these things we can either do the creation at
runtime, or we can do the creation at compile time and serialize the whole
Class/MultiSub/NameSpace into the packfile as well. If we don't want it,
Parrot won't force it upon us automagically.

HLLs like Winxed or NQP-rx or anything else can be modified to generate the
necessary instantiation code in the generated PIR output, instead of relying
on Parrot to do it for us. There's a good chance that this approach could be
more performance-friendly, because we would do less moving of data, less
calling of methods on startup and packfile loading, and less of other
unnecessary operations as well.

I think this sounds like a much better system, personally, and it's a
direction I want to start working towards. The ultimate goals are as follows:

1. Sub PMC should be slimmed down to be a simple wrapper around a section of
   bytecode. It should contain a little bit of information necessary to be
   invoked and to execute bytecode only. It shouldn't hold any other
   bookkeeping information.
2. Other behaviors of Sub, like Closures and Methods can be broken out into
   separate subclasses, if such is warranted. For Closures I think we
   definitely want it. For Methods, it might be unnecessary. I'm not really
   convinced that Methods want to be treated any differently from ordinary
   Subs, but maybe HLLs think differently.
3. Class, NameSpace, and MultiSub should all be given improved interfaces for
   populating their contents at runtime. NameSpace in particular needs to be
   dramatically cut, sliced, and refactored to remove all the magic. NameSpace
   should be, essentially, little more than a Hash with a name and the
   ability to recursively search contents.
4. IMCC needs to be changed to remove code that fills Sub full of unnecessary
   information. Also, if we have time, IMCC needs to be deleted.
5. The MMD system is going to need to be re-thought. Specifically, I don't
   like the idea of keeping around a bunch of signature lists and needing to
   sort them on every invocation. Instead, I think wins can be had if we use
   some variety of a search tree instead. Also, we don't assume every Sub
   is associated 1:1 with a single signature. The MultiSub should provide the
   signature to Sub mappings for each candidate
