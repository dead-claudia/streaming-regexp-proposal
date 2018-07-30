# Streaming regexp proposal

A proposal for streaming regexps, originally proposed in [this ESDiscuss thread](https://esdiscuss.org/topic/streaming-regexp-matching)

## Introduction

Regular expressions are very powerful, but they do have their limits. In particular, there's three scenarios that are needlessly complicated by them lacking a feature:

- Partial matching: [I'll just let the long list of search results for this generic search speak for itself.](https://www.google.com/search?q=javascript+regexp+partial+matching) If you need a concrete use case, interactive form validation is one of them.
- Streaming matching and replacement: [Atom has already found itself in an issue where it needs streaming matching to compensate for the necessary buffering](https://github.com/atom/scandal/issues/5), and if you're working with large files, [you can't properly match them to replace them](https://github.com/eugeneware/replacestream/issues/31) without running into memory issues.
- Reading duplicate group matches: You pretty much have to use a `while`/`for` loop + `re.exec`, which gets [messy in a hurry](https://github.com/MithrilJS/mithril.js/blob/32b319d140dfba14379d1969af929842dcb7cf58/render/hyperscript.js#L15-L29). It's very common when you're parsing simple grammars with regexps, but it'd be much easier to just specify a single regexp to match in a loop.

## Proposal

So, my proposal is pretty simple:

```js
const matcher = /regexp/.matcher(startIndex = re.lastIndex)
```

Create a streaming regular expression matcher. This reads off `lastIndex` for where to "start" if `startIndex` isn't given, but that's it.

```js
const results = matcher.consume(iter)
```

Consume several characters all at once, specified in an iterable or array-like, and return a flattened array of match results. If you've hit the end, pass `iter == null`, in which the matcher dereferences all memory it held and all future calls after this one to `matcher.consume` unconditionally return `undefined`. This is the preferred method for parsing, since the engine can vectorize its testing more easily.

Note that for native RegExps, if you've got a UTF-8 buffer, [you'll need to re-encode it as UTF-16LE](https://nodejs.org/api/buffer.html#buffer_buffer_transcode_source_fromenc_toenc) for it to work.

```js
const results = matcher.consumeChars(...charCodes)
```

Sugar for `matcher.consume([...charCodes])`, but it can avoid some intermediate overhead in the process.

```js
const index = matcher.index
```

Get the current index counter for the matcher.

```js
const group = result.group
```

Get the group name for this result, either an integer if it's an indexed group, a string if it's a named group, or `undefined` if it's a global match.

```js
const start = result.start
const end = result.end
```

Get the start and end indices for this result as integers. This exists so you can do your own slicing of the string.
