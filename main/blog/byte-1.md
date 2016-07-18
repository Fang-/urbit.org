---
date: ~2016.7.14
type: post
title: Urbyte 1: atoms, auras, types
author: The Urbit Team
layout: urbit,post
comments: true
hide: true
navmode: navbar
navdpad: false
navselect: blog
navpath: /
navhome: /
navclass: urbit
---
<br /><br />
Thanks for reading Urbyte 0!  In Urbyte 1, we'll look a little
more deeply at atoms and other simple noun types.

Let's restart our engines:

```
$ urbit -FI ~zod myship
[...]
ames: on localhost, UDP 31337.
http: live (insecure, public) on 8080
http: live ("secure", public) on 8443
http: live (insecure, loopback) on 12321
~zod:dojo>
```

## The atom question

It seems limiting to have only one kind of atom, the unsigned
decimal.  How are signed integers represented?  Symbols?
Floating-point values?  Dates?  IP addresses?  Even just hex?

The Lisp S-expression, with cons cells and atoms, is very similar
to Hoon's nouns.  But languages in the Lisp family mark each atom
with a dynamic tag that defines the atom's meaning.  These tags
are effectively dynamic types.

Hoon also tracks the meaning of atoms.  But Hoon is statically
type, so an atom description is obviously a type.  An atomic type
in Hoon is called an *aura*.  The aura syntax is just an ASCII
string starting with `@`, like `@ud` for unsigned decimal.

## Examples

With the dojo command `? expression`, you tell the dojo to print
both the product of the expression, and the product's type.

Let's type in some aura examples:

### Unsigned integers

```
> 42
42

> ? 42
  @ud
42

> ? 0x42
  @ux
0x42

> ? `@ud`0x42
  @ud
66

> ? (add 0x42 42)
  @
108
```

So decimal is aura `@ud`, hexadecimal is `@ux`.  Note the `@ud`
cast syntax, which lets us change the type of an atom, and
converting hex `0x42` to a decimal.

An aura is soft type.   The programmer can always insist on how
to interpret an atom.  Hoon doesn't have "dependent types"; it
can't enforce any constraints the aura puts on the atom's value.

### Signed integers

```
> -7
-7

> ? -7
  @sd

> --7
--7

> ? --7
  @sd
--7

> ? (sum:si -7 --7)
  @s
--0

> `@ud`-7
13

> `@ud`--7
14
```

`--7` means "positive 7."  `+7` might have been better, but `+`
is not URL-safe.

Hoon needs `--` to distinguish positive signed numbers because
Hoon's type system doesn't feed back into its parser.  The
expression itself needs to control whether it's `@sd` or `@ud`.

With unlimited atom width, traditional sign extension makes no
sense.  The sign bit is the low bit.  Even atoms are positive,
odd atoms are negative.

And Hoon has no overloading or type detection; we need to use a
different function to add signed integers.  We'll always choose
slightly clunkier syntax in exchange for much simpler semantics.

### Symbols

```
> ?
  @t
'Ürbit'

> ? `@ux`'Ürbit'
  @ux
0x7469.6272.9cc3

> ? `@ux`'foo'
  @ux
0x6f.6f66
```

The `@t` aura is UTF-8 text, with the first byte low.  Again,
Hoon doesn't actually have the power to check statically that
your UTF-8 is valid.  Auras don't tell you what an atom is, just
what you want it to be.

### Dates, floats, IP addresses...

```
> ? now
  @da
~2016.7.11..01.32.04..2c0d

> ? @~h6
  @dr
~h6

> `@ux`~h6
  @ux
0x5460.0000.0000.0000.0000

> ? (add ~h6 now)
  @
170.141.184.502.236.089.414.812.327.632.537.911.296

> `@da`(add ~h6 now)
~2016.7.11..07.32.28..842f

> ? .3.14
  @rs
.3.14

> ? .127.0.0.1
  @if
.127.0.0.1
```

`@da` is a 128-bit absolute date (64 bits for fractions of a
second, 64 bits for seconds); `@dr` is a relative date;  `@rs` is
a single-precision IEEE real; `@if` is an IPv4 address.  This is
not all Hoon's aura syntax, but it's a good range of examples.

`\`@da\~(add ~h6 now)` is a good example of the use and limits of
the aura system.  Unlike the signed-integer sum, offsetting a
date can use plain atomic addition.  But the product of `add` is
a flavorless atom with no aura, so we have to cast it back to a
date aura.

## Auras explained

Does an aura represent the units of the atom, its syntax, its
semantics, its constraints?

It can mean any or all of these.  Programmers can even use their
own aura strings which nothing in Hoon recognizes.  The set of
syntaxes that Hoon knows how to parse and print is fixed, but
there are plenty of reasons to introduce user-defined auras.

If your program uses both fortnights and furlongs, the aura
system will not parse a fortnight syntax or print furlongs in the
dojo.  But it will keep you from accidentally confusing the two,
which is probably most of what you want.

Hoon does interpret the string in one way: auras specialize to
the right.  For example, `@u` is any unsigned integer; `@ux` is
an unsigned integer that likes to be printed as hex.

Hoon's typechecker lets you go up or down the specialization
ladder, but complains if you try to go across.  For example,
`@u` can silently convert into `@ux` or `@ux` into `@u`, but
turning `@ud` into `@ux`, `@ux` into `@da`, etc, requires an
explicit cast.

## Constants, symbols, loobeans and null

An atom type isn't just an aura.  There's another choice: the
type can be *general* and contain all atoms (like the types we've
seen above), or *constant* and contain just one atom.

A type describes a set of values.  While auras may informally
describe a format for which not all atoms are valid, any atom can
be cast into the aura whether valid or invalid.

In a constant, the type contains just one atom.  Hoon actually
enforces this restriction.

The regular syntax for an atomic constant is just to put `%` on
front of the constant.  Like:

```
> ? 42
  @ud
42

> ? %42
  $42
42

> ? %~h6
  $~h6
~h6
```

The `$` on the type indicates the constant.  Note that constants
still have auras; `~h6` (6 hours) is still a `@dr`.

## Common irregulars

There are a few special forms worth learning early: symbols,
null, loobeans, and of course ships:

```
> ? %foo
  $foo
%foo

> `@ux`%foo
0x6f.6f66

> .n
.n

> |
.n

> .y
.y

> &
.y

> `@ud`&
0

> ~
~

> `@ud`~
0

> ? ~zod
  ~zod
@p

> `@ux`~forseg-bolbyn
0x885e.8af9
```

`%symbol` is a constant with aura `@tas` (text, ASCII, symbol).
The symbol rules, which are enforced by the Hoon parser but not
(as we've discussed) the type system: a symbol is lowercase with
infix hyphens ("kebab-case").

Loobeans are not true and false, but yes (`.y` or `&`) and no
(`.n` or `|`).  This is because *yes is zero and no is one*.
This makes sense at a certain mathematical level, though it
was probably a mistake.

`~` is Hoon's nil: constant `0`, aura `@n`.

And of course, `@p` is the phonemic base used for ship names.
(It's useful for any kind of memorable number.)

## Cells and types

Naturally, all these atoms fit neatly into cells, and retain
their types within the cells.

```
> ? [~h6 [.3.14 %foo] 0xdead.beef]
  {@dr {@rs $foo} @ux}
[~h6 [.3.14 %foo] 0xdead.beef]

> `*`[~h6 [.3.14 %foo] 0xdead.beef]
[398.449.671.992.126.314.905.600 [1.078.523.331 7.303.014] 3.735.928.559]
```

Another fun trick is to cast any noun to the `*` type, ie,
generic noun.  We then know nothing about the noun, so we print
all its atoms as decimals.

## And we're done

And we're done!  Press `^D` to exit:

```
~zod:dojo> ^D
$
```

## Questions and/or exercises

As always, these questions are optional and only for fun.

We didn't include anywhere near all auras or syntaxes in this
lesson.  It's just a set of examples.

Try using the dojo's magic error erasure to stumble around the
rest of Hoon's atom syntax.  Any string that can't be extended
into a valid expression will be trimmed back, with a beep, until
it can.  Return will beep if the expression is not complete.
This feedback is harmless and does not use an electric shock,
but it enables stochastic reinforcement learning of Hoon syntax.

If `~h6` is six hours, how do you write forty-five minutes?  Two
days, six hours and forty-five minutes?

What would you expect the syntax for double-precision float to
be?  What about base64?

Does an absolute date have to include every zillisecond, or can
it be pruned to a day or a year?

If there's an IP syntax, what about bitcoin addresses?  What
other auras do you think should be supported?
