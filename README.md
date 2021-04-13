# Luatwine
A Twine 2 story format that outputs Lua

## Overview

With Luatwine, you can write Twine 2 stories with Lua scripts embedded directly into your Passages. The Story can then be exported as a plain Lua file, which can be played in the browser like any other Twine format, or be embedded into any engine that supports Lua - for example, the Luatwine Unity plugin.

### Who is this for?

* Twine 'power users' using the platform to develop games or complex mechanics, who will benefit from Lua's programmer-friendly syntax and structures.
* Developers interested in using their creations in unconventional ways, who will benefit from Lua's embeddability and extensibility.

### Who is this not for?

* Developers focused on linear storytelling, CYOA, and other projects with minimal logic (and are only targeting the browser). Harlowe should have everything you need.
* Developers indending to take advantage of complex browser-based features: media playback, CSS and so on. For any feature, if it's not in [TextMeshPro's Rich Text](http://digitalnativestudios.com/textmeshpro/docs/rich-text/) then Luatwine probably won't support it in the browser.
* Developers who like the 'mutable' model used by Harlowe: hiding/showing parts of a passage, changing words from one to another when the user clicks something and so on. See [Immutability] for more details.

## Syntax

Luatwine supports the following:
* All CommonMark Markdown syntax, excluding standard hyperlinks and HTML
* Links in the form `[[Passage]]`, `[[Text->Passage]]`, and `[[Passage<-Text]]`
* Plain variables, in the form `$myVariable`
* Complex expressions, in the form `<$ foo + bar $>`
* Script blocks, in the form `{$ someCode(); moreCode() $}`
* 'Changer' blocks, in the form `$em[ Text _here_. ]` and `<$color('red')$>[ Text *here*. ]`

### Links

Luatwine supports the 'Twine standard' passage link syntax, which allows the Twine editor to work out what passages link to what, and draw pretty lines between them.

Internally, `[[Text->Passage]]` expands to `<$ link('Passage') $>[Text]`; but note that if you use the expanded form, Twine won't draw the link lines.

### Embedded code

Short-form variables such as `$foo` will look up the value of 'foo' and display it. See [Rules for rendering] for details on what exactly will be outputted here. The variable name must begin with an ASCII letter or underscore, which is followed by a contiguous sequence of ASCII letters, numbers, or underscores. This ensures that monetary values in the text e.g. `$1.10` will display as intended.
> If for some reason you want to look up a variable with a non-identifier name, you can use `<$ _G['1.10'] $>`

Short-form variables are of course limited to a single identifier. To show the result of an expression - a function call, arithmetic, or similar - you can use the expression syntax: `<$ 1 + 2 + 3 $>` will print '6'. You can add whitespace and new-lines anywhere you could ordinarily with Lua, without affecting the output. The same [Rules for rendering] apply here.

Expressions are limited to a single, well, expression. If you want to run code with statements, such as to set variables and define functions, you can use the script block syntax: `{$ x = 5 $}`, which will set the value of 'x' to '5'. Script blocks only generate output if explicit calls to `show()` and friends are made. As with expressions, you can add whitespace anywhere you like.

Internally, short-form variables expand to expressions, which in turn expand to script blocks with an inner call to `show()`. So `$foo` expands to `<$ foo $>` which expands to `{$ show(foo) $}`.

### Changer syntax

If a short-form variable or expression immediately precedes a 'content block' - any content surrounded by `[` and `]` - it will be treated as a 'changer' with the content block applied to it. See [Changers] for more details. Changer blocks can be nested infinitely.

A content block without an attached changer expression will generate a warning (because there is no reason to do this) but will otherwise be shown normally. So `[Text]` will display as `Text`.

## Built-in functions

### Changers

A 'changer' is something that modifies a block of content - for example hiding/showing it, wrapping it with some formatting, or rendering it multiple times. Behind the scenes, changers are higher-order functions; they can be called with a single `content` argument, which itself is a parameterless function that renders the attached content. This means that `$em[Text]` is functionally equivalent to `<$ em(function() text('Text') end) $>`, and `<$color('red')$>[Text]` is the same as `<$color('red')(function() text('Text') end)$>`

#### Conditionals
* `_if(condition)`: Renders its content if `condition` is truthy
* `_else`: Renders its content if the previous `_if` and none of the `_elseif` calls following it were entered
* `_elseif(condition)`: Renders its content if `condition` is truthy *and* the previous `_if` and none of the `_elseif` calls following it were entered

#### Formatting
* `em`: Pushes an 'em' (for emphasis) instruction before the content
* `strong`: Pushes a 'strong' (for strong emphasis) instruction before the content
* `u`: Pushes a 'u' (for underline) instruction before the content
* `s`: Pushes an 's' (for strike-through) instruction before the content
* `color(color)`: Pushes a 'color' (for text color) instruction with the given `color` argument before the content

#### Repetition
* `repeat(count)`: Renders the content `count` times in a row
* `forEach(iterable, key...)`: Renders the content for each item in `iterable`, assigning the value to a variable with the string name of `key`

#### Modification
* `replace(pattern, replacement)`: Executes a [pattern](https://www.lua.org/pil/20.2.html) replacement on the `text()` emissions within the block
* `strip(tag...)`: Causes all `push(tag)` emissions (with their paired `pop()`s) to be dropped. For example, `<$strip('em')$>[ Some $em[text] ]` will render `Some text` without the 'em' instruction of the inner block.

#### Misc
* `name(name)`: Hides the block, and assigns it to a variable named `name`. The block can subsequently be displayed with `$name`.
* `combine(changer...)`: Creates a Changer that combines each of its Changer arguments in order of outer-most to inner-most. For example, `<$combine(em, u)$>[Text]` equates to `$em[$u[Text]]`. 
* `freeze`: Renders the block's content and caches it, so that subsequent calls don't execute any embedded code a second time.

### Emission functions

These are the low-level functions that produce the user-visible text, and need to be implemented by the host.

* `push(tag, arguments...)`: Encloses all further output in `tag`, until `pop()` is called. Valid tags include:
  * `p`: Paragraph
  * `em`: Emphasis
  * `strong`: Strong emphasis
  * `b`: Bold
  * `i`: Italics
  * `u`: Underline
  * `s`: Strikethrough
  * `h1`-`h6`: Heading
  * `color`: Text color (with argument)
  * `click`: Executes the given function when the text is clicked
* `pop()`: Ends the tag from the last un-popped `push()` instruction.  `push()` and `pop()` must be balanced.
* `text(text)`: Outputs the given text string
* `object(tag, arguments...)`: Outputs the given non-text object. Valid tags include:
  * `hr`: Horizontal line
  * `br`: Line break

Note that *any* tag can be given as an argument here; this list is intended as a useful baseline, but the host could support more or less than this.

## Conventions and caveats

Luatwine attempts to make most of its internal operations straightforward and predictable, even if this is on the surface less convenient.

### Rules for rendering

When you `show()` a value, the content that gets outputted (i.e. the sequence of emissions) depends on the value's type:
* Strings are outputted directly: `text(value)`
* Booleans and Numbers are converted to strings: `text(tostring(value))`
* Functions are called with no arguments, which may generate output as a side-effect. Their return value is ignored. This is what allows saved content blocks (with `name()`) to be printable later on.
* Tables are treated like functions (they could have a metatable with call operator)
* Nil values are skipped

### Block repetition

When a block is executed, it is always *actually* executed - any code within will run, even if it has been run before. To stop this from happening, you can use the `freeze` function.

### Immutability

Once content is emitted to the host (with `push()`, `pop()`, `object()`, and `text()`), it stays there 'forever' and cannot be changed by any subsequent code. Think of the host as a typewriter: it can add letters, switch fonts and styles, but can't change what's on the page without tearing it up and starting over - in other words, calling `clear()`.
> It's possible to add custom emission functions to deal with modification if you *really* need it, but that's out of scope for this README!

The advantage here is that the implementation of these 'emission' functions is *much* simpler than if modification was allowed. Let's make a simple HTML emitter now:
```lua
tags = {}

function push(tag)
    tags[#tags+1] = tag
    print('<'..tag..'>')
end

function pop()
    print('</'..tags[#tags]..'>')
    tags[#tags] = nil
end

function text(text)
    print(text)
end

function object(tag)
    print('<'..tag..' />')
end

function clear()
    print('<hr />')
end
```

That's it! The upshot is that whether you're targeting HTML, Rich Text, or direct API calls, integration into your engine of choice should be 'stupid simple'.

Note that in this example, `clear()` doesn't actually clear the screen; if our output is plain HTML, this is of course impossible. In your application you can use the appropriate APIs to clear any existing text - perhaps saving it first to a 'conversation history' system, if desired.

### Text output is not re-parsed

When emitting text through code, e.g. `<$ '*foo*$bar' $>`, it is emitted 'verbatim', without any additional parsing. In the example shown, the literal string `*foo*$bar` will be printed.

The reason for this is that a second parsing step, in addition to placing a lot of implementation burden on the Lua runtime, host, and/or exporter, introduces some subtle and complex edge-cases relating to *when* exactly this parsing takes place (What does `'$'..'foo'` print? What about `'$f'..'oo'`? Or `x = '$'; show('$x'..'foo')`?).

Fortunately, there are a number of options to achieve a very similar effect:
* Use Lua directly, which is almost as terse as Markdown, e.g. `<$ em('foo')..bar $>`
* Wrap the content in a `name()` changer, and refer to it later, e.g. `<$name('x')$>[ *foo*$bar ] ... $x`
* Use the content directly in the passage, and add any code before/after, e.g. `<$ code() $> *foo*$bar <$ moreCode() $>`

### The parser is dumb

The tokens `<$`, `$>`, `{$`, and `$}` were chosen because they are always invalid sequences in Lua and (probably) invalid in Markdown, so the parser just needs to scan for the matching end token and treat the whole match as the Lua code to execute.

However, if you have a Lua string literal that actually contains `$}` or `$>`, you'll run into problems! If for **some reason** you need to operate on these sequences or include them in your text, you'll have to elaborate the expression a bit: `'$'..'}'`. There are no 'escape sequences' or suchlike.

## Extension

By default, Luatwine's built-in functionality is deliberately limited to improve performance and reduce bloat. However, adding in the functionality you need is usually trivial thanks to the flexibility of Lua.

### Number of visits
For example, suppose you need to know if a given passage has been visited before - something like `$firstVisit[ Ask for her name ] $_else[ Ask for her number ]`. You could add the following code blocks:
```lua
-- In a 'startup' passage
visitedSet = {}
function firstVisit(content)
    return iff(visitedSet[passage.name])(content)
end

-- In a 'footer' passage
visitedSet[passage.name] = true
```

Alternatively, you could implement a 'history' system that tracks every passage the player has visited:
```lua
-- In a 'startup' passage
history = {}
function visits(upTo)
    local count = 0
    for _, value in ipairs(history) do
        if value == passage.name then
            count += 1
        end
        if upTo ~= nil and count >= upTo
            break
        end
    end
    return count
end

function firstVisit(content)
    return iff(visits(1) == 0)(content)
end

-- In a 'footer' passage
history[#history + 1] = passage.name
```

Why isn't this just enabled by default? Because, if you expect to have a long play-time, having a list that grows infinitely is probably a bad idea! The `visitedSet` implementation avoids this somewhat, but perhaps you need to know the number of times a passage was visited, or their relative order. The tradeoffs between flexibility and state management are best decided by you, for your specific application.
