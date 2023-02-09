---
title: "CSS Overhaul"
date: 2023-02-09T20:56:52+01:00
author: Martin Kauppinen
draft: true
---

New year, new look! The [risotto](https://github.com/joeroe/risotto) theme
served me well, but I eventually got bored of it and decided to do a complete
overhaul and write it all myself, from scratch. I feel like I have much more
control this way to present the content exactly how I want. It was also an
excuse to learn about `dart` and SASS.

I opted to make it a bit more spacious and legible. I had two main gripes with
`risotto` as it was before: the text was pretty dense; and the contrast was a
bit too low. I feel like I've struck a good balance with this new theme.

The repo for the theme can be found
[here](https://github.com/martinkauppinen/markau-theme). I don't know how
reusable it is, but if someone happens to stumble across it, they can fork it
and give making it work for them a shot.

It took me quite a bit longer than I'd like to admit to cobble it all together.
I had never used `dart` before, but it's a nice change of pace to vanilla CSS.
The code is a bit all over the place as it's a result of lots and lots of
iteration, with no clear goal in mind. I just sat down and started styling until
I was happy with the result. However, using `dart` on Netlify, which is my
hosting service of choice, is not currently supported. There are ways around it
with [GitHub Actions
shenanigans](https://www.brycewray.com/posts/2022/05/using-dart-sass-hugo-github-actions-edition/),
but I didn't feel like doing anything like that, so for now I'm manually
transpiling the theme to CSS before pushing changes to the repo. Changes to the
theme will be much less frequent than blog posts anyway, and this way I still
get the nice automatic deployment on Netlify without having to configure the
main blog repo with API keys and secrets.

## Showcase
Here's a quick showcase of theme elements that didn't appear in the above text:

Code blocks:
```rust
fn main() {
    println!("Hello world!");
}
```

Blockquotes:

> You miss 100% of the shots you don't take.
> - Wayne Gretzky
> - Michael Scott

Images:

{{< figure src="http://placehold.it/640x400&text=Placeholder" caption="Image caption"
alt="Placeholder image" >}}

Some `<b>inline code</b>`. And
[links](https://youtu.be/dQw4w9WgXcQ), and footnotes[^1]! As you can tell, I
like putting things in little boxes.

[^1]: Here be more explanations.
