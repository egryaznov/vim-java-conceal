# Zero-footprint, hassle-free "let" in Java with Vim

This document describes how to reduce boilerplate code in local constant
declaration with a `let` keyword in Java using Vim's _conceal_ feature:

![Showcase][showcase]

As you can see, the actual content of a file is **not affected** in any way!
The `let` keyword is merely a virtual enhancement, *it doesn't exists*.
The benefits of this are obvious:
+ No additional frameworks, dependencies, or libraries are required
+ No support from Java ecosystem, unlike the notorious ``var`` keyword, is
    required
+ No tool like Maven or FindBugs gets broken because sources are *not
    altered* in any way

Notice that `let` appears only in actual code, comments and string stay the
same.

## TL;DR
If you just want to use this feature in your Vim, do the following:

0. Place the following lines in your `.vimrc`:
    ```
    set conceallevel=2
    set concealcursor=
    ```
1. [Download](../java.vim) `java.vim` from the root directory of this repo.
2. In the folder `/syntax` under your vim installation directory locate the
   namesake file and replace it with its downloaded version. By default it
   should reside in:
    * MacVim: `/Applications/MacVim.app/Contents/Resources/vim/runtime/syntax/java.vim`
    * Windows: `C:\Program Files(x86)\Vim\vim8\syntax\java.vim`
3. Restart Vim.
4. Enjoy!

## Issues
Some plugins are known for breaking this feature, here's an incomplete list:

* `Yggdroot/indentLine` -- also uses concealment to put indentation marks.
  Out of the box hides all concealed items, including `let`. You can try to
  tweak the plugin to remedy this.

Your own `syntax` definitions can also break the feature, try to load vim with a
clear `.vimrc` to debug it:
```
vim -u NONE
```

## The Whole Story
Java is notorious for abundance of boilerplate code. Language designers
try to mitigate the problem by introducing various code features like `lambdas`,
`generics` and now `try-with-resources` with every new version.

When I read the [JEP 286](http://openjdk.java.net/jeps/286) I was excited about
`var`, but disappointed that there will be no `val` any time soon. So I decided
to add my own version with Vim's `conceal` feature.

At first I thought it would be fairly easy to do, but I soon discovered that you
need to perform various hacks to actually achieve this.

Firstly, let me tell you about `syntax` command. Vim uses it as a part of syntax
highlighting mechanism to define a _group_ of strings which should have the same
style. This command has an additional `conceal` option to virtually replace a
`pattern` with just *one single char*. The syntax is as follows:
```
syntax match Conceal /pattern/ conceal cchar=c
```
Here `pattern` is a regular expression that you want to conceal and `c` is a
char with which your `pattern` will be virtually replaced.

Already you can see a limitation: why only a single char, and not an arbitrary
string? It appears that [Vince Negri](https://sites.google.com/site/vincenegri/concealownsyntaxforvim), the
author of this feature, originally developed it to work with LaTeX documents.

So, unfortunately you cannot do that:
```
syntax match Conceal /final @NotNull var/ conceal cchar=let
```
How to remedy this? Is there a hacky solution? Think for a moment: you want to
replace every occurrence of a string `final @NotNull var` with a word `let`, but
you can only do that **char by char**. Thus, instead of having only **one**
`syntax` command, we need **three** of them for every char. And in each case
replacement should happen not for a whole string, but only for certain part of
it. This is where `ms` and `me` flags come into play. Let me firstly show you
the answer, and then I will explain:
```
syntax match Conceal /final @NotNull var/me=s+1 conceal cchar=l
syntax match Conceal /inal @NotNull var/me=s+1 conceal cchar=e
syntax match Conceal /nal @NotNull var/ conceal cchar=t
```
The `me` flag after `/` is, in Vim's parlance, a *pattern offset*, which specifies
the `offset for the end of the matched text`. With `/patter/me=s+1` we can tell Vim to
match only the very first letter in `pattern`. The `s+1` part means *the first
symbol from the start of a match*. So, we can read the first `syntax` command
like the following:
```
Whenever you encounter the string "final @NotNull var", match only its first
letter and virtually replace it with "l".
```
Similarly, `ms` signifies the *start* of a match, and `ms=e-1` can be used to
exclude last letter from it.

So far so good, but why `f` and `fi` are omitted in the next two commands?
Firstly because they will be concealed, and no concealed item can be matched.
And secondly because we still need some way to group these three commands
together, and the only way to do that is to ensure that the next match will
happen as close as possible to the current one.

We not done yet, there is another obstacle in the way: if `pattern` contains a
substring, which appears in a command like `syntax keyword someGroup
substring`, then the concealment won't happen, because `syntax keyword` is made
to overrule any other `syntax match` command. Thus, if Vim loads the following,
concealed strings will reappear:
```
syntax keyword javaStorageClass  final
```
Luckily there is a solution. Any time you read a `.java` file,
Vim will load a default highlighting from `/syntax/java.vim`, which is
located in Vim's installation directory. Remove `final` and `var` from the `syn
keyword` commands and you will be okay.

Moreover, if still want to highlight those strings, add a few lines like this:
```
syntax match javaType "var"
syntax match javaStorageClass "final"
```

After that, save file and restart Vim, you should see the concealment working
like in GIF above.

##### See also:
```
:help syntax
:help conceal
:help syn-cchar
:help syn-pattern-offset
:help matchadd
:help ownsyntax
```

## FAQ

##### Is any other editor/IDE has a similar feature?
Yes, Emacs and IntelliJ IDEA have full support of it, although in IDEA it's
called *Folding*.

##### How did you write `let` so quickly?
I used a snippet expansion feature of [this](github.com/Shougo/neosnippet.vim)
plugin.

##### Can I use `val` instead of `let`?
Put the following in your .vimrc:
```
let g:java_concealment_use_let = 0
```

##### I don't use `@NotNull` in my code.
Put the following in your .vimrc:
```
let g:java_concealment_use_notnull = 0
```

##### What about other primitive types?
Put the following in your .vimrc:
```
let g:java_concealment_primitive = 1
```

##### Can I conceal ALL constant declarations with ALL possible types?
Put the following in your .vimrc:
```
let g:java_concealment_all = 1
```

##### I want the feature to behave differently.
You should be able to write your own `syntax` definitions for concealment
if you thoroughly read **The Whole Story** section.

##### Can I use regular expressions instead of plain string?
~~No.~~ Yes, if you are smart about it. The only problem of using them lies in
the fact that you generally cannot predict the length of a matched string.
Therefore, you don't know how to delineate a specific part of expression you want
to conceal with a single char. But if the first `k` letters, where `k` equals
the length of a string you want to conceal with (e.g. with `let` `k = 3`), are
hard-coded, then it should be fairly easy to do.

For example, here's how you can enable the feature for all non-primitive types:
```
syntax match Conceal "final @NotNull \w\+"me=s+1 conceal cchar=l
syntax match Conceal "inal @NotNull \w\+"me=s+1 conceal cchar=e
syntax match Conceal "nal @NotNull \w\+" conceal cchar=t
```
Here we needed only first three hard-coded letters of the word `final` to make
everything work.

##### Is there any other, non-hacky way to do this?
Not to my knowledge. As was explained in **The Whole Story** section, there are
a couple of limitation in the Vim's concealment mechanism that forces us to
resort to an expedient solution. One possible alternative is to implement this
directly in Vim itself by making the `conceal` mechanism to accept strings of
any length and then send a push request to Bram Moolenaar. Well, *good luck with
that*.

# Acknowledgment
Thanks to [Marko Trosi](https://github.com/marcotrosi) for support, check out
his Vim Telegram chat group: https://t.me/VimUserGroup.

[showcase]: https://media.giphy.com/media/jb3yeutGEpFyU4oGae/giphy.gif
