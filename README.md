# Garden

Garden is a library for rendering CSS in Clojure. Conceptually similar to
[hiccup](https://github.com/weavejester/hiccup), it uses vectors to represent
rules and maps to represent declarations. Garden also comes with functions for
working with units.

## Table of contents

* [Installation](#installation)
* [Syntax](#syntax)
  * [Rules](#rules)
  * [Declarations](#declarations)
  * [Units](#units)
* [Thanks](#thanks)

## Installation

Add the following dependency to your `project.clj` file:

```clojure
[garden "0.1.0-alpha"]
```

## Syntax

Garden syntax is very similar to Hiccup. If you're familiar with Hiccup you
should feel right at home working with Garden. If not, don't sweat it! Garden's
syntax is fairly simple.

From your project's root directory start up a new REPL and try the following:

```clojure
user=> (require '[garden.core :refer (css)]))
nil
user=> (css [:body {:font-size "16px"}])
"body{font-family:16px}"
```

First you'll notice the use of the `css` macro. This macro takes an optional
map of compiler flags, any number of rules, and returns a string of compiled
CSS. We'll start off by discussing rules then follow up with declarations and
Garden's unit utilities. Then we'll demonstrate the use of compiler flags.

### Rules

As mentioned, vectors represent rules in CSS. The first _n_ **non-collection**
elements of a vector depict the rule's selector where _n_ > 0. When _n_ = 0 the
rule is not rendered. To produce a rule which selects the `<h1>` and `<h2>` HTML
elements for example, we simply begin a vector with `[:h1 :h2]`.

```clojure
user=> (css [:h1 :h2 {:font-weight "none"}])
"h1,h2{font-weight:none}"
```

To target child selectors nested vectors may be employed:

```clojure
user=> (css [:h1 [:a {:text-decoration "none"}]])
"h1 a{text-decoration:none}"
user=> (css [:h1 :h2 [:a {:text-decoration "none"]])
"h1 a, h2 a{text-decoration:none}"
```

A slightly more complex example demonstrating nested vectors with multiple
selectors:

```clojure
user=> (css [:h1 :h2 {:font-weight "normal"}
             [:strong :b {:font-weight "bold"}]])
"h1,h2{font-weight:normal}h1 strong,h1 b,h2 strong,h2 b{font-weight:bold}"
```

### Declarations

Clojure maps represent CSS declarations where map keys and values represent
CSS properties and values respectively. Garden's declaration syntax is a bit
more involved than rules and understanding it is important to make the most of
the library.

#### Properties

Declaration map keys _should_ either be a string, keyword, or symbol:

```clojure
user=> (css [:h1 {"font-weight" "normal"}])
"h1{font-weight:normal}"
user=> (css [:h1 {:font-weight "normal"}])
"h1{font-weight:normal}"
user=> (css [:h1 {'font-weight "normal"}])
"h1{font-weight:normal}"
```

At the moment Garden will not raise an error if other key types are used and
will happily generate invalid CSS:

```clojure
user=> (css [:h1 {30000 "nom-nom"}])
"h1{30000:nom-nom}"
```

To be on the safe side, just play ball and avoid anything else for now.

#### Values

We've already seen strings used as declaration map values but Garden supports
keywords, symbols, numbers, maps, vectors, and lists in addition to this.

Strings, keywords, symbols, and numbers are rendered as literal CSS values:

```clojure
user=> (css [:body {:font "16px sans-serif"}])
"body{font:16px sans-serif}"
```

Be warned, you must escape literal string values yourself:

```clojure
user=> (css [:pre {:font-family "\"Liberation Mono\", Consolas, monospace"}])
"pre{font-family:\"Liberation Mono\", Consolas, monospace}"
```


In some cases it would be useful target several properties in a "group" of
properties without having to type the same prefix several times. To do this
with Garden we use maps. Maps as declaration values are used to denote a
property suffix (IE. `-family` or `-weight`) and may be nested as deeply as
you like.

Here are a few practical examples of where this technique might be handy:

```clojure
user=> ;; Working with vendor prefixes:
user=> (css [:.box
             {:-moz {:border-radius "3px"
                     :box-sizing "border-box"}}])
".box{-moz-border-radius:3px;-moz-box-sizing:border-box}"
user=> ;; Creating DRY "mixins":
user=> (def reset-text-formatting
         {:font {:weight "normal" :style "normal" :variant "normal"}
          :text {:decoration "none"}})
#'user/reset-text-formatting
user=> (css [:a reset-text-formatting])
"a{font-variant:normal;font-style:normal;font-weight:normal;text-decoration:none}"
user=> (defn partly-rounded
         ([r1] (partly-rounded r1 r1))
         ([r1 r2]
          {:border {:top-right-radius r1
                    :bottom-left-radius r2}}))
#'user/partly-rounded
user=> (css [:.box (partly-rounded "3px")])
".box{border-bottom-left-radius:3px;border-top-right-radius:3px}"
```

Finally we have vectors and lists which are handled in the same manor when used
as a declaration value. The semantics of these values increment the level of
complexity somewhat so be sure you understand their behavior before you use
them. When you use a vector/list as a value you are asking Garden for a
*space* separated list.

```clojure
user=> (css [:p {:font ["16px" "sans-serif"]}])
"p{font:16px sans-serif}"
```

When you nest a vector/list  you are asking for a *comma* separated list.

```clojure
user=> (css [:p {:font ["16px" '(Helvetica Arial sans-serif)]}])
"p{font:16px Helvetica,Arial,sans-serif}"
```

Be warned, this pattern is recursive!

### Units

So far we've got all the core pieces to start building basic stylesheets. But
it would be useful to have something for working with one of CSS's most
fundamental data types: units!

Fortunately, Garden has built in support for working with all the major CSS
units. This includes creation, conversion, and arithmetic. To start using units
use/require the `garden.units` namespace.

```clojure
user=> (require '[garden.units :refer (px in)])
nil
```

For demonstration purposes we're only `refer`ing the `px` and `in` units but
Garden supports all of the usual suspects. Also we'll use the `css` macro to
render the units as strings but note this is not necessary when authoring a
stylesheet.

Unit creation is straightforward:

```clojure
user=> (css (px 16))
"16px"
```

Unit functions take a number _n_ and construct a new `Unit` record with _n_ as
the magnitude. Unit functions also accept other units as values returning their
conversion if possible. This makes working with unit values very flexible.

```clojure
user=> (css (px (px 16)))
"16px"
user=> (css (px (pt 1)))
"1.3333333333px"
```

Unit arithmetic is available via the - spoiler alert- unit arithmetic functions.

```clojure
user=> (require '[garden.units :refer (px+ px* px- px-div)])
nil
user=> (css (px+ 1 2 3 4 6))
"16px"
user=> (css (px-div 2 4))
"0.5px"
user=> (css (px* 2 2))
"4px"
```

Since the arithmetic functions use the primary unit functions in their
definitions, conversion occurs seamlessly (when possible):

```clojure
user=> (css (px* 2 (pt 1)))
"2.6666666666px"
```

You might be wondering, which units can be converted to another? This
depends on the type of unit you are working with. The CSS spec outlines a few
categories of units but only absolute (`px`, `pt`, `pc`, `in`, `cm`, and `mm`),
angle (`deg`, `grad`, `rad`, `turn`), time (`s`, `ms`), and frequency
(`Hz`, `kHz`) units may be freely converted and only between their respective
groups. This means you cannot, for example, convert `px` to `rad` or `Hz` to
`cm`. Doing so will raise an error.

In the future, some exceptions to this rule might apply for working with `em`s
since it's technically possible to compute their contextual value.

## Thanks

A big thank you goes out to @weavejester for creating Hiccup. I learned a ton
from studying the source code and borrowed several ideas. Writing this library
would have been significantly harder without it.

I'd also like to thank @jhardy for putting up with random questions and pushing
me to get this library out there.

Thanks to everyone in `#clojure` on IRC for answering my questions and being
patient with me. If you're looking for an example of a great community, look no
further.