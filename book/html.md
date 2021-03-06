---
title: Constructing a Document Tree
chapter: 4
cur: html
prev: text
next: layout
...

So far, your web browser sees web pages as a stream of open tags,
close tags, and text. But HTML is actually a tree, and though the tree
structure hasn't been important yet, you'll need it to draw
backgrounds, add margins, and implement CSS. So this chapter adds a
proper HTML parser and converts the layout engine to use it.


A tree of nodes
===============

Right now, the browser sees web pages as a flat sequence of tags and
text, which is why the `Layout` object's `token` method includes code
for both open and close tags. But HTML is a tree, and each open and
close tag pair are one node in the tree, as is each text token. We
need to convert from tokens to nodes.

We'll need a new `parse` function to do that:

``` {.python}
if __name__ == "__main__":
    # ...
    nodes = parse(lex(body))
    browser = Browser()
    browser.layout(nodes)
    # ...
```

Let's start by defining the two types of nodes.[^1] Element nodes
correspond to tag pairs and form a tree:

[^1]: In reality there are other types of nodes too, like comments,
    doctypes, and `CDATA` sections, and processing instructions. There
    are even some deprecated types!

``` {.python expected=False}
class ElementNode:
    def __init__(self, tag, parent):
        self.tag = tag
        self.parent = parent
        self.children = []
```

That tree can also contain text at the leaves:

```
class TextNode:
    def __init__(self, text, parent):
        self.text = text
        self.parent = parent
        self.children = []
```

Element nodes start empty, and our parser fills them in. The idea is
simple: keep track of the currently open elements, a list from top to bottom:

``` {.python}
def parse(tokens):
    currently_open = []
    for tok in tokens:
        # ...
```

The *end* of this list---the most recently opened element---is the
parent of any new nodes we come across. Any time we finish a node (at
a text or end tag token) the last thing in the `currently_open` list
is its parent:

``` {.python}
for tok in tokens:
    parent = currently_open[-1] if currently_open else None
    # ...
```

Inside the loop, we need to figure out if the token is text, an open
tag, or a close tag, and do the appropriate thing. `Text` tokens are
the easiest: create a new `TextNode` and add it to the bottom-most
open element.

``` {.python indent=8}
if isinstance(tok, Text):
    node = TextNode(tok.text, parent)
    parent.children.append(node)
```

End tags are similar, but instead of making a new node they take the
bottom-most open element:

``` {.python indent=8}
elif tok.tag.startswith("/"):
    node = currently_open.pop()
    currently_open[-1].children.append(node)
```

Finally, for open tags, we need to create a new `ElementNode` and add
it to the list of currently open elements:

``` {.python indent=8 replace=parent)/parent%2c%20tok.attributes)}
else:
    node = ElementNode(tok.tag, parent)
    currently_open.append(node)
```

The core of this logic is about right, but what and when does the
parser return? Try parsing

``` {.html}
<html><body><h1>Hi!</h1></body></html>
```

and the parser will read the `</html>` element, pop the last open
element off the list of open elements, and then crash since there's no
open element to append it to. So in this case we actually want to
return that root element:

``` {.python indent=8}
elif tok.tag.startswith("/"):
    node = currently_open.pop()
    if not currently_open: return node
    currently_open[-1].children.append(node)
```

Time to test this parser out!

::: {.further}
HTML derives from a long line of document processing systems. Its
predecessor, [SGML][sgml] traces back to [RUNOFF][runoff] and is a
sibling to [troff][troff], now used for Linux man pages. The
[committee][jtc1-sc34] that standardized SGML now works on the `.odf`,
`.docx`, and `.epub` formats.
:::

[sgml]: https://en.wikipedia.org/wiki/Standard_Generalized_Markup_Language
[runoff]: https://en.wikipedia.org/wiki/TYPSET_and_RUNOFF
[troff]: https://troff.org
[jtc1-sc34]: https://www.iso.org/committee/45374.html


Self-closing tags
=================

Try running this parser on this page, and you'll find that `parse`
doesn't return anything; let's find out why:

``` {.python expected=False}
def parse(tokens):
    # ...
    print(currently_open)
    raise Exception("Reached last token before end of document")
```

Python prints a list of `ElementNode` objects, meaning that there were
open HTML elements still around when it reached the last token:

```
[<__main__.ElementNode object at 0x101399c70>,  ...]
```

Ok, that's not too helpful. Python needs a method called `__repr__` to
be defined to print things a little more reasonably:

``` {.python}
class ElementNode:
    # ...
    def __repr__(self):
        return "<" + self.tag + ">"
```

This produces a more reasonable result:

``` 
[<!DOCTYPE html>,
 <html lang="en-US" xml:lang="en-US">,
 <head>,
 <meta charset="utf-8" />,
 <meta name="generator" content="pandoc" />,
 <meta name="viewport" content=... />,
 <link rel="prev" href="text" />,
 <link rel="next" href="layout" />,
 <link rel="stylesheet" href="../book.css" />]
```

Why aren't these open elements closed? Well, most of them (like
`<meta>` and `<link>`) are what are called self-closing: you don't
ever write `</meta>` or `</link>`. These tags don't need a close tag
because they never surround content. Let's add that to our parser:


``` {.python indent=8 replace=parent)/parent%2c%20tok.attributes)}
# ...
elif tok.tag in SELF_CLOSING_TAGS:
    node = ElementNode(tok.tag, parent)
    parent.children.append(node)
```

Use the following `SELF_CLOSING_TAGS` list, straight from the
[standard][html5-void-elements]:[^void-elements]

[html5-void-elements]: https://html.spec.whatwg.org/multipage/syntax.html#void-elements

[^void-elements]: A lot of these tags are obscure or obsolete, but
    it's nice that there's a complete list.

``` {.python}
SELF_CLOSING_TAGS = [
    "area", "base", "br", "col", "embed", "hr", "img", "input",
    "link", "meta", "param", "source", "track", "wbr",
]
```

Test your parser on this page to see if that helped.

::: {.further}
Putting a slash at the end of self-closing tags, like `<br/>`,
became fashionable when [XHTML][xhtml] looked like it might replace
HTML. But unlike in [XML][xml-self-closing], in HTML self-closing tags
are identified by name, not by some special syntax.
:::

[xml-self-closing]: https://www.w3.org/TR/xml/#sec-starttags
[xhtml]: https://www.w3.org/TR/xhtml1/

Attributes
==========

Strangely, the self-closing tag code in the previous section doesn't
help; why? Because the problem tags aren't just `<meta>`: they have
attributes, like in `<meta charset="utf-8" />`. HTML attributes give
additional information about an element; open tags can have any number
of attributes (though close tags can't have any). Attribute values can
be either quoted, unquoted, or omitted entirely.

Attribute values can be anything, and if they're quoted they can even
contain whitespace. But for simplicity, let's stick to unquoted
attribute values. Then neither tag names nor attribute-value pairs can
contain whitespace, so we can split the tag contents on whitespace to
get the tag name and the attribute-value pairs:

``` {.python}
class Tag:
    def __init__(self, text):
        parts = text.split()
        self.tag = parts[0].lower()
```

Note that the tag name is converted to lower case,[^case-fold] because
HTML tag names are case-insensitive.

[^case-fold]: This is [not the right way][case-hard] to do case
    insensitive comparisons; the Unicode case folding algorithm should
    be used if you want to handle languages other than English. But in
    HTML specifically, tag names only use the ASCII characters where
    this test is sufficient.
    
[case-hard]: https://www.b-list.org/weblog/2018/nov/26/case/

This fixes the problem of identifying self-closing tags, but since
we're already here, let's also turn the attribute-value pairs into a
dictionary:

``` {.python indent=4}
def __init__(self, text):
    # ...
    self.attributes = {}
    for attrpair in parts[1:]:
        key, value = attrpair.split("=", 1)
        self.attributes[key.lower()] = value
```

This code assumes all attributes have a value, but in fact the value
can be omitted, like in `<input disabled>`. In this case, the
attribute value is supposed to default to the empty string:

``` {.python indent=8}
for attrpair in parts[1:]:
    if "=" in attrpair:
        # ...
    else:
        self.attributes[attrpair.lower()] = ""
```

Finally, this code misbehaves on quoted values, since it includes the
quotes as part of the value. We can fix that:

``` {.python indent=12}
if "=" in attrpair:
    if len(value) > 2 and value[0] in ["'", "\""]:
        value = value[1:-1]
    # ...
```

This conditional checks the first character of the value to determine
if it's quoted, and if so strips off the first and last character,
leaving the contents of the quotes.

When we convert `Tag`s to `ElementNode`s, we need to move the
attributes as well. Let's add an `attributes` field on `ElementNode`:

``` {.python}
class ElementNode:
    def __init__(self, tag, parent, attributes):
        self.tag = tag
        self.parent = parent
        self.attributes = attributes
        self.children = []
```

When we create an `ElementNode`, we'll copy the attributes over from
the corresponding `Tag`:

``` {.python indent=12}
node = ElementNode(tok.tag, parent, tok.attributes)
```

::: {.further}
Prior to the invention of CSS, some browsers supported web page
styling using attributes like `bgcolor` and `vlink` (the
color of visited links) and tags like `font`. These [are
obsolete][html5-obsolete], but browsers still support some of them.
:::

[html5-obsolete]: https://html.spec.whatwg.org/multipage/obsolete.html#obsolete


Doctype declarations
====================

Self-closing elements are now handled correctly and the list of open
elements when parsing this page is now shorter:

```
[<!DOCTYPE>]
```

[html5-doctype]: https://html.spec.whatwg.org/multipage/syntax.html#the-doctype

This special tag is called a [doctype][html5-doctype], and it's not
really an element at all. It's not supposed to have close tag, but we
can't mark it a self-closing tag—it's always the very first thing in
the document, so there wouldn't be an open element to append it to.
It's best to throw it away:[^quirks-mode]

[^quirks-mode]: Real browsers also set some flags that switch between
    standards-compliant and legacy parsing and layout modes.

``` {.python indent=8}
elif tok.tag.startswith("!"):
    continue
```

Furthermore, text can appear after the doctype declaration but before
any other elements; in that case `currently_open` is empty so there's
no node to add that text to. Usually that text is just whitespace
between `<!DOCTYPE html>` and `<html>` in the HTML source, so it's
silly to have the parser crash here. Let's handle it just skipping
all text nodes that only contain whitespace:[^ignore-them]

[^ignore-them]: Real browsers retain whitespace nodes: whitespace is
    significant inside `<pre>` tags or in cases like the difference
    between `make<span>up</span>` and `make <span>up</span>`. Our
    browser won't support that, and ignoring all whitespace tags
    simplifies [later chapters](layout.md) by avoiding a special-case
    for whitespace-only text tags.

``` {.python indent=8}
if isinstance(tok, Text):
    if tok.text.isspace(): continue
    # ...
```

With this change, the parser can now parse this page, and most other
valid HTML pages!

Now our browser should *use* this element tree. Let's add a `recurse`
method to replace the current `for` loop inside the `Layout`
constructor. To start, we can just call `token` twice per element
node, emulating the old token-based layout:

``` {.python indent=4 expected=False}
def recurse(self, tree):
    if isinstance(tree, TextNode):
        self.text(tree.text)
    else:
        self.token(Tag(tree.tag))
        for child in tree.children:
            self.recurse(child)
        self.token(Tag("/" + tree.tag))
```

This works but it's a little ridiculous. We've gone through all this
effort to construct a tree, and now we're just emulating tokens? It's
now a little clearer that the old `token` function had three different
parts to it:

- The part that handled `Text` tokens, which now isn't being used;
- The part that handled start tags; and
- The part that handled end tags.

Let's split `token` into two functions, then, `open` and `close`:

``` {.python indent=4}
def open(self, tag):
    if tag == "i":
        self.style = "italic"
    # ...

def close(self, tag):
    if tag == "i":
        self.style = "roman"
    # ...
```

Make sure to update `recurse` to call these two new functions; now it
no longer has to construct `Tag` objects or add slashes to things to
indicate a close tag!

::: {.further}
SGML document type declarations had a URL to define the valid tags.
Browsers use the absense of a document type declaration to
[identify][quirks-mode] very old, pre-SGML versions of
HTML,[^almost-standards-mode] but don't need the URL, so `<!doctype
html>` is the best document type declaration today.
:::

[quirks-mode]: https://developer.mozilla.org/en-US/docs/Web/HTML/Quirks_Mode_and_Standards_Mode

[^almost-standards-mode]: There's also a crazy thing called "[almost
    standards][limited-quirks]" or "limited quirks" mode, due to a
    backwards-incompatible change in table cell vertical layout. Yes.
    I don't need to make these up!

[limited-quirks]: https://hsivonen.fi/doctype/

Handling author errors
======================

The parser now handles HTML pages correctly—at least, pages written by
the sorts of goody-two-shoes programmers who remember the HTML
boilerplate, close their open tags, and make their bed in the morning.
We, or at least I, am not such a person, so browsers have had to adapt
to handle poorly-written, confusing, boilerplate-less HTML.

In fact, modern HTML parsers are capable of transforming *any* string
of characters into an HTML tree, no matter how confusing the markup.[^3]
The full algorithm is, as you might expect, complicated beyond belief,
with dozens of ever-more-special cases forming a taxonomy of human
error, but one of the nicer time-saving innovations is *implicit* tags.

[^3]: Yes, it's crazy, and for a few years in the early '00s the W3C
    tried to [do away with it](https://www.w3.org/TR/xhtml1/). They
    failed.

Normally, an HTML document starts with a familiar boilerplate:

``` {.html}
<!doctype html>
<html>
  <head>
  </head>
  <body>
  </body>
</html>
```

In reality, *all six* of these tags, except the doctype, are optional:
browsers insert them automatically. To do so, they compare the current
token's tag to the list of currently open elements; that reveals
whether any additional elements need to be created:

``` {.python indent=4}
for tok in tokens:
    implicit_tags(tok, currently_open)
    # ...
```

The `implicit_tags` function examines the incoming token and
determines whether any implicit tags need to be added. Importantly,
more than one implicit tag may need to be added, so the function uses
a loop:

``` {.python}
def implicit_tags(tok, currently_open):
    tag = tok.tag if isinstance(tok, Tag) else None
    while True:
        open_tags = [node.tag for node in currently_open]
        # ...
```

That loop needs to handle each possible implicit tag. The implicit
`<html>` tag is easiest, since it must be the root element:

``` {.python indent=8}
if open_tags == [] and tag != "html":
    currently_open.append(ElementNode("html", None, {}))
```

With the `<head>` and `<body>` elements, you need to look at the tag
being handled. Only a few tags are actually supposed to go in the
`<head>` element:

``` {.python}
HEAD_TAGS = [
    "base", "basefont", "bgsound", "noscript",
    "link", "meta", "title", "style", "script",
]
```

That tells you whether to insert `<head>` or `<body>`:[^where-script]

[^where-script]: Note that some tags, like `<script>`, can go in
    either the head or body section of an HTML document. The code
    below places it inside a `<head>` tag by default, but doesn't
    prevent its being explicitly placed inside `<body>` by the page
    author.

``` {.python indent=8}
elif open_tags == ["html"] and tag not in ["head", "body", "/html"]:
    if tag in HEAD_TAGS:
        implicit = "head"
    else:
        implicit = "body"
    parent = currently_open[-1]
    currently_open.append(ElementNode(implicit, parent, {}))
```

If you see an element that's not supposed to go in the `<head>`, you
need to implicitly close the `<head>` section:

``` {.python indent=8}
elif open_tags == ["html", "head"] and tag not in ["/head"] + HEAD_TAGS:
    node = currently_open.pop()
    parent = currently_open[-1]
    parent.children.append(node)
```

Note that the this code doesn't create the `<body>` element itself.
That's for the next iteration of the loop.

The loop ends for all other cases, with no additional implicit tags
being inserted:

``` {.python indent=8}
else:
    break
```

We also want implicit close tags for `</body>` and `</html>`. Usually,
leaving them out will mean the `parse` function reaches the end of its
loop without closing all open tags, and would thus return nothing.
Let's make it return *something* instead:

``` {.python}
def parse(tokens):
    # ...
    while currently_open:
        node = currently_open.pop()
        if not currently_open: return node
        currently_open[-1].children.append(node)
```

These rules for malformed HTML may seem arbitrary, and they are: they
evolved over years of trying to guess what people "meant" when they
wrote that HTML, and are now codified in the [HTML parsing
standard][html5-parsing].

[html5-parsing]: https://html.spec.whatwg.org/multipage/parsing.html

::: {.further}
Thanks to implicit tags, you can often skip the `<html>`, `<body>`,
and `<head>` elements. They'll be implicitly added back for you.
Nor does writing them explicitly let you do anything weird; the HTML
parser's [many states][after-after-body] guarantee that there's only
one `<head>` and one `<body>`.[^except-templates]
:::

[^except-templates]: At least, per document. An HTML file that uses
    frames or templates can have more than one `<head>` and `<body>`,
    but they correspond to different documents.

[after-after-body]: https://html.spec.whatwg.org/multipage/parsing.html#parsing-main-afterbody

Summary
=======

This chapter taught our browser that HTML is a tree, not just a flat
list of tokens. We added:

- A parser to transform HTML tokens to a tree
- Layout operating recursively on the tree
- Code to recognize and handle attributes on elements
- Automatic fixes for some malformed HTML documents

The tree structure of HTML is essential to display visually complex
web pages, as we will see in the [next chapter](layout.md).

::: {.signup}
:::


Exercises
=========

*Comments:* Update the HTML lexer to support comments. Comments in
HTML begin with `<!--` and end with `-->`. However, comments aren't
the same as tags: they can contain any text, including left and right
angle brackets. The lexer should skip comments, not generating any
token at all. Test: is `<!-->` a comment, or does it just start one?

*Paragraphs:* Since it's not clear what it would mean for one
paragraph to contain another, the most common reason for this to
happen in a web page is that someone forgot a close tag. Change the
parser so that a document like `<p>hello<p>world</p>` results in two
sibling paragraphs instead of one paragraph inside another.

*Scripts:* JavaScript code embedded in a `<script>` tag uses the left
angle bracket to mean less-than. Modify your lexer so that the
contents of `<script>` tags are treated specially: no tags are allowed
inside `<script>`, except the `</script>` close tag.[^or-space]

[^or-space]: Technically it's just `</script` followed by a [space,
    tab, `\v`, `\r`, slash, or greater than sign][script-end-state].
    If you need to talk about `</script>` tags inside your JavaScript
    code, split it across multiple strings. I talk about it in a
    video.

[script-end-state]: https://html.spec.whatwg.org/multipage/parsing.html#script-data-end-tag-name-state

*Quoted attributes:* Quoted attributes can contain spaces and right
angle brackets. Fix the lexer so that this is supported properly.
Hint: the current lexer is a finite state machine, with two states
(determined by `in_tag`). You'll need more states.

*Syntax Highlighting:* Implement the `view-source:` protocol as in
[Chapter 1](http.md#exercises), but make it syntax-highlight the
source code of HTML pages. Keep source code for HTML tags in a normal
font, but make text contents bold. If you've implemented it, wrap text
in `<pre>` tags as well to preserve line breaks. Use your browser's
HTML lexer to implement the syntax highlighter.
