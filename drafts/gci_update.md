The Google Code-In (GCI) program is coming up on it's last week and things are
reaching a fevered pitch. Students are circling like hungry sharks in the
water, snapping at every task we throw their way. Frankly, we're having
trouble keeping up with some of the more prolific participants. As of last
night we had crossed 150 completed tasks, and there is no indiciation that
things are slowing down at all.

I think now is a good time to reflect on the program, talk about what we've
learned, what we can do better next year, and what I think can change about
the program as whole.

Over the course of this program Parrot has offered a number of tasks for it's
students. In the beginning we had plenty of time to prepare an initial
selection, and there was much diversity in the first crop of tasks: We had
tasks involving code, testing, documentation, translations, user outreach, and
several other areas. When those all were completed, we started looking for
other areas that could be mined for new tasks. After all, the best play for
the Parrot Foundation was to provide as many tasks as possible, and try to get
as much "free" work out of the students as we could.

After the initial batch of tasks was emptied, we started looking elsewhere.
The first thing we did was offer several translation tasks to get our various
documents translated into several languages. What we ended up with was a huge
pile of translated documents of various levels of quality, but for which we
didn't even have an organizational infrastructure in place to manage. In fact,
thats a problem that the Parrot Foundation still hasn't solved: Where do we
put all these translated documents, how do we evaluate them, and how do we go
about keeping them up to date? At the time of writing this blog post, The
Parrot repository contains several branches which have translated documents
for which there are currently no plans to merge until we get this issue sorted
out.

We went through the ticket queue, several times, and turned any tasks that
appeared suitable into GCI tickets. Those went relatively quickly. These I
think were the most natural tasks: They were things that we already wanted,
and the tickets themselves provided a nice place for students to discuss their
results, post patches, receive feedback, etc. Parrot does have a very large
queue of outstanding tickets, but many of them are open precisely because they
are very difficult or even open-ended. While we still have hundreds of more
tickets, most of them are probably not suitable for a GCI task.

Later we started making several tasks to include code coverage of our test
suite. For some things, like the new Embedding API, this brough a huge influx
of valuable tests for the API which I am sincerely grateful for, but we also
got accused of "spamming" tasks and allowing certain students to inflate their
scores by rapidly completing tasks that were too easy. I'm happy to explain
why this is not the case, but it's still a little bit annoying to be accused
of it.

On a related note, improving our code coverage is a very nice thing to do and
can be a very tedious chore. It's a perfect thing for a contributor like a GCI
student to do because it is a task that is necessary but which rarely draws
sufficient attention from core developers. It's not a task that I personally
perform too often. On the other hand, having all these extra helpers around
can really help us get moving on core development tasks such as fixing bugs,
adding new features, cleaning/refactoring code, etc. We do want improved code
coverage, but we don't want to be feeding those kinds of tasks to GCI students
exclusively. We do need to keep a nice balance.

So what have we learned this year in GCI?

The first lesson I've discovered is
that the students will eat up code-related tasks as quickly as we create them.
Their desire to write code is insatiable. To be ready for GCI next year we
should have at least 100 code-related tasks available up front, and be
prepared to create several more for the duration of the program.

Translation tasks can be easy to do and are relatively popular with students,
but don't always yield results. We have received several good submissions of
documentation translations, but we also received several which were not good
and almost could not become good after cycles of feedback and resubmission.
This can be a bit of a waste of time for the student and mentor alike. Also,
it's not always easy to find a developer to be a mentor or proof-reader for
translations in all languages. On at least one occasion one of our developers
had to call in a favor with a colleague to get a translation proof-read.
Luckily for us the translation was high-quality and didn't require much
feedback or modification.

Other types of tasks such as documentation, research, and outreach tasks, are
less popular. They are also much harder for us to come up with in sufficient
numbers. Students will do these if there are no other options, but it's
becoming clear that these kinds of tasks will sit in the queues while students
focus on the code. Even if students loved these tasks, there's no way we could
create enough of them to keep up with demand.

Students have been very willing and even eager to jump into tasks for various
ecosystem projects such as [Parrot-Linear-Algebra][pla], [ParrotSharp][ps],
and [Winxed][]. Several new features (and accompanying tests) were added for
PLA. Several new features and an entire new test suite were added for
ParrotSharp. I find these things to be very valuable, and I would sincerely
hope that in future years the ecosystem projects represent a much higher
percentage of tasks for students to perform.

[pla]: http://github.com/Whiteknight/parrot-linear-algebra
[ps]: http://github.com/Whiteknight/parrotsharp
[Winxed]: http://code.google.com/p/winxed
