---
title: "Rust is a Scripting Language Now"
date: 2024-03-12T20:36:38+01:00
author: Martin Kauppinen
---

I was talking with a friend, poking fun at a job ad that said they wanted
experience in scripting languages such as `bash`, `python`, and _`golang`_.

After some more banter about compilied languages not being scripting languages,
we realized that technically you _could_ make golang _seem_ like a scripting
language. All you have to do is have a properly crafted
[shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) at the top of the file.

And of course I wanted to see if I could do something like that to make Rust
_seem_ like a scripting language. After some poking around, here is the final
result:

```rust
#!/usr/bin/env -S sh -c 'rustc ${0} --out-dir $(dirname ${0}) && ${0%%.rs} ; rm -f ${0%%.rs}'

fn main() {
    println!("Hello world!");
}
```

Saving that somewhere as a `.rs` file and running `chmod +x` on it, you can run
it and pretend like it's a shell script. It will compile into a binary with the
same name (sans the `.rs` suffix), run it, then clean up after itself by
removing the binary.

Of course, there are some limitations. Since it's not a `cargo` project, there
is no way to add dependencies to other crates. Everything is should be contained
in this one file. You do have access to the standard library, though.

### Quick walkthrough of the shebang
Since a shebang line passes the path to the file it's contained in as the first
argument to the interpreter, we can simply use `sh -c "commandstring" [arg]` as
our interpreter. Inside of the `sh` command string, `${0}` refers to the first
argument (the path to our "script").

So now we can pass `${0}` in this subshell context to `rustc`, making sure to
set the `--out-dir` argument to make sure the binary ends up next to the
"script"[^maybe-tmp]. The reason for this is so that the script can be run from
anywhere, no matter what your current working directory is. Just type the path
and it will run. Or just the script name if the script is in your `$PATH`!

[^maybe-tmp]: An alternative is to pass `/tmp` as the output directory. Then the
  cleanup step could be skipped too. Although then the running part would be a
  bit more complicated.

Then, we run the resulting binary by stripping the `.rs` suffix from the
"script" path, if compilation succeeded (hence `&&`). Finally, we remove the
binary (with `-f` to suppress any "file not found" errors). We can't use `&&`
or `||` here, as the script could have different exit codes, and we always want
to remove the binary no matter what the exit code was. Hence just a simple `;`
sequence point.

Should you do all this? Probably not.
