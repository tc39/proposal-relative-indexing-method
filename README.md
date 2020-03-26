# Proposal for a `.item()` method on all the built-in indexables

A TC39 proposal to add a .item() method to all the basic indexable classes (Array, String, TypedArray)

**Stage: 0**

**Champions: Tab Atkins, Shu-yu Guo**

Rationale
---------

For many years, programmers have asked for the ability to do "negative indexing" of JS Arrays, like you can do with Python. That is, asking for the ability to write `arr[-1]` instead of `arr[arr.length-1]`, where negative numbers count backwards from the last element.

Unfortunately, JS's language design makes this impossible. The `[]` syntax is not specific to Arrays and Strings; it applies to all objects. Referring to a value by index, like `arr[1]`, actually just refers to the property of the object with the key "1", which is something that any object can have. So `arr[-1]` already "works" in today's code, but it returns the value of the "-1" property of the object, rather than returning an index counting back from the end.

There have been many attempts to work around this; the most recent is a restricted proposal to make it easier to access just the last element of an array (<https://github.com/tc39/proposal-array-last>) via a `.last` property.

This proposal instead adopts a more common approach, and suggests adding a `.item()` method to Array, String, and TypedArray, which takes an integer value and returns the item at that index, with the negative-number semantics as described above.

This not only solves the long-standing request in an easy way, but also happens to solve a separate issue for [various DOM APIs, described below](#dom-justifications).

### Existing Methods

Currently, to access a value from the end of an indexable object, the common practice is to write `arr[arr.length - N]`, where N is the Nth item from the end (starting at 1).  This requires naming the indexable twice, additionally adds 7 more characters for the `.length`, and is hostile to anonymous values; you can't use this technique to grab the last item of the return value of a function unless you first store it in a temp variable.

Another method that avoids some of those drawbacks, but has some performance drawbacks of its own, is `arr.slice(-N)[0]`.  This avoid repeating the name, and thus is friendly to anonymous values as well. However, the spelling is a little weird, particularly the trailing `[0]` (since `.slice()` returns an Array). Also, a temporary array is created with all the contents of the source from the desired item to the end, only to be immediately thrown away after retrieving the first item.

Note, however, the fact that `.slice()` (and related methods like `.splice()`) already have the notion of negative indexes, and resolve them exactly as desired.

### DOM Justifications

A recent addition to the WebIDL spec is `ObservableArray<>` (thanks @domenic!), a proxy over an Array that allows web APIs to expose something that to page authors looks exactly like an Array, but still allows the browser to intercept get/set/delete/etc of indexed properties, enforcing type checks and other requirements exactly like they do today with named properties.

We plan to start using this for most APIs that want to expose a list of something, but we'd also like to, when possible, upgrade *older* APIs to use this as well; the fact that many older APIs use bespoke interfaces that badly and incompletely copy the Array interface is a consistent source of frustration for web authors.

(For example, `document.querySelectorAll()` returns, not an Array, but a NodeList, which supports indexed properties and `.length`, and so can be treated as an Array in basic ways, but has only a tiny selection of the Array prototype methods: just `.forEach()` and the iterator methods `.keys()`, `.values()`, and `.entries()`. Popular methods like `.map()` are missing, requiring authors to write code like `[...document.querySelectorAll("a")].map(foo)`.)

This upgrade can *almost* be done in-place, just swapping the various bespoke interfaces with ObservableArray, avoiding breaking anything that doesn't explicitly test the value's type.  There is *one* exception: all of them have a `.item()` method, which returns the value at the passed index.

(This is a remnant of the very old (1990s-era) belief that Java was a reasonable language to use on the web, and so APIs were designed in a "lowest common denominator" style for use in both JS and Java. Java didn't have the ability to indexed properties at the time unless you were actually a Java array, so the .item() method was a compromise that worked identically in both languages.)

It's highly likely there is code that relies on using `.item()` on these interfaces, and we don't want to risk breakage there.

We *could* address this by subclassing ObservableArray and adding `.item()` in the subclass. However, that would mean the values aren't of type Array; various type-checking methods in the community looking for an Array would fail.

Or we could just add `.item()` to ObservableArray itself, as it's a proxy wrapper around Array. This would be confusing and weird however, making it appear that Array had such a method even tho it's not on the prototype.

The ideal solution for us, instead, is to add `.item()` to the Array prototype itself,
and for completeness/consistency, to the other indexable types that support the same general suite of index-related properties like `.slice()`.

Proposed Edits
--------------

TODO, but basically just steal the existing code from `.slice()` on those classes and simplify. Literally all the work we want `.item()` to do is already done by `.slice()`.