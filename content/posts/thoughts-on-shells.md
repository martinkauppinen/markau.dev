---
title: "Thoughts on shells"
date: 2021-11-30T20:16:57+01:00
author: Martin Kauppinen
---

I have been using [`fish`](https://fishshell.com/) as my shell for a while now and
I am a bit conflicted on whether or not to continue using it. This post is
essentially a ramble for me to organize my thoughts and perhaps make a decision
eventually.

{{< toc >}}

## My early days
I started my shell journey with `bash` way back when I didn't even know what a
shell was, nor the distinction between the shell and the terminal (emulator). It
came installed by default on my first Linux distribution (I believe it was
Ubuntu 8.04? It was more than half my lifetime ago, anyway), and basically just
used it for installing some programs and running commands I found on different
fora when I had a problem that needed fixing. Coming from Windows, doing
anything non-graphical was a bit foreign and mystical.

Eventually I grew comfortable with moving around my system and doing things in
the shell sometimes rather than always through a graphical file manager or other
applications, though it took a good while and I often reached for graphical
tools when unsure.

Then one day - I can't say when - something switched and I preferred doing most
things in the shell, only opening graphical programs when needed[^1]. I started
automating some tasks, writing shell scripts to type repetitive or difficult to
remember commands less. I wasn't proficient yet by any stretch, but I wasn't
entirely new at it like when I was fresh from Windows. The shell became
comfortable. Or, to be more precise: `bash` became comfortable.

[^1]: "Need" being defined a bit stronger now - basically meaning "the shell
  equivalent is not good or non-existent".

## There are other shells?
In university a friend introduced me to `zsh`. At this point I hadn't even
realized that there _were_ other shells. I hadn't figured out that I was using a
specific one called `bash`; I was just doing things "in the terminal"! I was
also shown [oh my zsh](https://ohmyz.sh/) and it was just so cool and shiny that
I had to have it. I still base my shell prompts, no matter what shell I use, on
the [ys theme](https://blog.ysmood.org/my-ys-terminal-theme/). I really like
having a multi-line prompt so that any commands I type are always the first
thing on the line (save for the starting `$`).

Of course, `zsh` having a bunch of fancy plugins and eye-candy wasn't the only
good[^2] thing about it. Oh my zsh is often the thing listed online in articles
and blog posts as _the reason_ to switch to `zsh`, but I don't think that's even
the most compelling reason. `zsh` just has some saner defaults and
configurations that `bash` either lacks, or does a poor job recreating.

[^2]: Whether or not eye-candy is good is, of course, subjective.

One thing that comes to mind is how `zsh` handles tab-completion. If a
completion is ambiguous, the possible completions are listed below the prompt
and can be moved around with using `<Tab>` and `<Shift-Tab>`, or the arrow keys.
The auto-completion in `bash`, on the other hand is crude in comparison. When
hitting `<Tab>` when completions are ambiguous, it just spits out the list of
possible completions, shoving your prompt down in the terminal window, and
leaving it up to you to type enough that the match is no longer ambiguous. You
can sort of get slightly nicer tab-completion in `bash` with something like:
```bash
bind 'TAB:menu-complete'
bind 'set show-all-if-ambiguous on'
```
but this just feels like a crude approximation of `zsh`'s nice version.

Another nice feature of `zsh` is the recursive path expansion and automatic
`cd`. To move into a directory, you just need to type the path, and to move deep
into a file tree you don't even have to type (or repeatedly tab-complete!) the
full path. Hitting `<Tab>` after an abbreviated path expands to the full path,
for example `/u/l/s/f` expands to `/usr/local/share/fonts` on my system.

There are many other features that just gel better with who I am and how I use
the shell in `zsh` than in `bash`. This became a long tangent on `zsh`, and I'm
not even planning on talking about `fish` next. Interesting how you can just
continue writing sometimes once you've started.

## POSIX-noncompliance? Ew.
Both `bash` and `zsh` are POSIX-compatible, and their syntaxes similar enough
that switching between the two is in most cases painless. The first
non-POSIX-ish shell I ran into was `tcsh` and my god did (do) I dislike it
strongly. I realize this is mostly inertia and not being used to it, but it
lacking simple things like redirecting stdout and stderr to different places,
and redirection in general being a bit weird. I sometimes run into
`tcsh`-scripts and the syntax is... not horrifying, but bad enough that my eyes
glaze over. It feels like taking C and trying to cram it into some form of shell
language[^3] to make it suck less. But all shell languages suck, and so `(t)csh`
does too.

There were lots of other little things that just bothered me so much about using
`tcsh` interactively that I began believing (wrongly) that POSIX-noncompliant
shells are awful by default.

[^3]: This is exactly what `tcsh`'s predecessor, `csh`, had as a goal.

## POSIX-noncompliance is good, actually.
Enter `fish`. It is not POSIX-compliant, but it is a good shell. A great shell,
even. It corrected my belief that POSIX-noncompliance is bad by default, and it
is a joy to use interactively. It has a ton of the features and eye-candy that
`zsh` has by default, and more! Its completion hinting[^4] where a context-aware
completion is shown dimly after your cursor is very cool. Hitting the right
arrow key will usually complete almost exactly the command you're after, drawing
from the command history.

[^4]: I don't know what it's called and cannot at this time be bothered to look
  it up.

And POSIX-noncompliance is mostly a non-issue. Why should the shell be beholden
to the rules of POSIX anyway? Why should it have to behave like `sh`? Some may
say it's for portability, others may say it's stifling innovation. I'm not going
to make a judgement call on that. I can see the merit of both sides on this
issue.

To summarize, here's a short list of what `fish` does compared to `zsh`:
+ Recursive path expansion like `zsh`? Check.
+ Automatic `cd` like `zsh`? Check.
+ Cool eye-candy like `zsh`? Check.
+ Completion hinting, unlike `zsh`? Pretty cool!
+ A joy to use interactively? Absolutely!
+ Configurable in a nice way, like `zsh`? About that...

## POSIX-noncompliance is bad, actually.
`fish` is very opinionated[^5]. "Configurability is the root of all evil" is one
of the core beliefs of `fish`'s design philosophy. The way you configure `fish`
is via a web interface, for some reason. On its own, not a terrible idea
perhaps. But to me[^6] it feels... opaque? I don't know how to describe it.
Sure, there's text configuration in the bottom of it somewhere if you wanted to
configure it with text, like most other shells, but it feels like it's
discouraged to do so.

[^5]: Granted, so am I.
[^6]: This is an opinion piece, remember.

It also bothers me that aliases as such don't really exist in `fish`, and you
don't just configure them in something like a `.fishrc` file. Instead, you
create functions, and each function gets its own file in
`~/.config/fish/functions/`. Aliases are pretty short and concise; it is
completely reasonable to keep most of them in the same file. And since `fish`
has its own language and syntax, I have to keep looking it up whenever I want to
do some text config. POSIX skills are non-transferrable here. And only useful
with this particular shell. I know I said up above that I wasn't going to make a
judgement call for or against POSIX, but I do feel like I'm leaning slightly
towards POSIX. And again, I know that it's force of habit, but oh boy is that a
strong force.

## Switching back to `zsh`?
Typing all of this out, I think I will be going back to `zsh` pretty soon.
`fish` has been great to use, but I have had some issues with it. The completion
hinting sometimes simply does not work or doesn't do what I expect/want it do to
and is frankly distracting sometimes. Tab-completion sometimes hits some weird
recursion depth that I haven't managed to get to the bottom of. I have to look
up syntax whenever I have to do something that is slightly more complicated
than piping and redirecting some commands, but which isn't complicated enough to
warrant a script.

It is very possible (even likely) that I just don't appreciate `fish` because I
don't understand it yet. It's different, in both good and bad ways. But I
recently learned that [you can bring some `fish` into your
`zsh`](https://github.com/zsh-users/fizsh). With both shells having plugin
systems, making one behave a bit more like the other shouldn't be too hard.

## Conclusion
In the end, which shell you use interactively largely doesn't matter. Use
whatever suits best. The differences are pretty small, but can for exactly that
reason be a bit jarring.  I may revisit `fish` in the future, and try to gain
some more appreciation for it. But until then, I'll be happy in `zsh`. Or maybe
I'll go all out and try out something like
[`nushell`](https://github.com/nushell/nushell). We'll see.

