# lambduh

[![Build Status](https://travis-ci.org/jedwards1211/lambduh.svg?branch=master)](https://travis-ci.org/jedwards1211/lambduh)
[![Coverage Status](https://coveralls.io/repos/github/jedwards1211/lambduh/badge.svg?branch=master)](https://coveralls.io/github/jedwards1211/lambduh?branch=master)
[![semantic-release](https://img.shields.io/badge/%20%20%F0%9F%93%A6%F0%9F%9A%80-semantic--release-e10079.svg)](https://github.com/semantic-release/semantic-release)
[![Commitizen friendly](https://img.shields.io/badge/commitizen-friendly-brightgreen.svg)](http://commitizen.github.io/cz-cli/)

## Turing-complete JS commandline JSON/text transformer

Can't remember how `sed` works.
Can't remember how `cut` works.
I just don't have to use them often enough to commit them to memory.
When I first read about `jq`, I thought *Hmm, this is cool, but I love ES6.  I know it well.
Can't I just do all of this stuff in JS?*

These days I'm too lazy to read manpages and learn tools like `jq`, `sed`, `awk`, etc. for things that I could
I could write a JS expression for in ten seconds.  All that was lacking was a basic command to handle the I/O.
Just look how frigging easy stuff is:

```
npm install --global lambduh

# Print lines of file in reverse order:
> duh 'lines => lines.reverse()' < somefile

# Swap first and last name:
> duh 'line => line.split(/\s+/g).reverse().join(", ")' < names
Edwards, Andy
Doe, John

# Put all `lodash` functions on global scope for `duh` commands for the next few examples
> cd ~ && npm install lodash
> echo 'Object.assign(global, require("lodash"))' >> ~/.lambduh.js

# Shuffle lines of a file
> duh 'lines => shuffle(lines)' < somefile

# See who's been committing to `react` recently:
> curl 'https://api.github.com/repos/facebook/react/commits?per_page=100' \
  | duh 'json => mapValues(groupBy(json, "commit.committer.name"), size)'
{
  "GitHub": 60,
  "Dan Abramov": 17,
  "Flarnie Marchan": 2,
  "Brian Vaughn": 3,
  "Nathan Hunzaker": 5,
  "Brandon Dail": 8,
  "Ben Alpert": 1,
  "Andrew Clark": 1,
  "Dominic Gannaway": 3
}

# Destructure and restructure JSON subdocuments
> curl 'https://api.github.com/repos/stedolan/jq/commits?per_page=2' \
  | duh 'json => json.map(({ commit: { message, committer: { name } } }) => ({ message, name }))'
[
  {
    "message": "Deal with strptime() on OS X and *BSD (fix #1415)\n\nstrptime() on OS X and *BSDs (reputedly) does not set tm_wday and\ntm_yday unless corresponding %U and %j format specifiers were used.\nThat can be... surprising when one parsed year, month, and day anyways.\nGlibc's strptime() conveniently sets tm_wday and tm_yday in those cases,\nbut OS X's does not, ignoring them completely.\n\nThis commit makes jq compute those where possible, though the day of\nweek computation may be wrong for dates before 1900-03-01 or after\n2099-12-31.",
    "name": "Nicolas Williams"
  },
  {
    "message": "Attempt to fix #1415\n\nOS X (and *BSD) strptime() does not set tm_wday nor tm_yday unless\ncorresponding format options are used.  That means we must call timegm()\nto set them.",
    "name": "Nicolas Williams"
  }
]

## Node version note

The syntax that `duh` accepts for your function depends on the version of Node you're running.  If you're running a
really old version, you may not be able to use lambdas, and will have to resort to `function () { return ... }` instead.

## Command

You can use `lambduh` or just `duh`.

## Modes

You may have noticed above that you can apply a transformation to each line of a plain text file.
This is pretty much the only magic built into the `duh` command:
* If the first argument of your function is `lines` (ignoring case), it runs in **lines mode**.
* Otherwise if the first argument of your function starts with `l` or `L`, it runs in **line mode**.
* If the first argument of your function starts with `t` or `T`, it runs in **text mode**.
* Otherwise, it runs in **json mode**.

:star: :+1: **Nice!!** :punch: :sparkles:

### JSON Mode

`JSON.parse`s `stdin`, calls your function on it, writes `JSON.stringify` to `stdout`.  Duh!

### Line Mode

Reads lines from `stdin` one-by-one, calls your function on each with `(text, lineNumber)`, and writes what it returns
to `stdout`, except:

* If your function returns `false`, it quits reading lines.
* If your function returns `null` or `undefined`, it skips to the next line without printing anything.

### Lines Mode

Calls your function with an array of lines from stdin.  If your function returns an array of lines, it writes them
in order to `stdout`.  Otherwise, it calls `String` on it and writes that string to `stdout`.

### Text Mode

Calls your function with the entire text from stdin in a single string, and writes what your function returns to
`stdout`.

## Customization

`lambduh` requires any `.lambduh.js` files in the working directory or its ancestors (from root to deepest), as well
as `.lambduh.js` in your home directory.  You can use these files to put functions on `global` scope so that you can
use them in your function argument to `lambduh`.  For example, here are some of the things I put in my `~/.lambduh.js`:

```js
Object.assign(global, require('lodash')) // put all lodash functions on context
global.replace = require('preserve-case') // case-preserving string.replace, often handy
```

With this I can do powerful stuff like:
```
> echo '{ "a": "foo bar", "b": "FOO BAR", "c": "it's all about the fooBar", "d": "foo-bar" }' \
  | duh 'json => mapValues(json, value => replace(value, "foo bar", "baz qux"))'
{
  "a": "baz qux",
  "b": "BAZ QUX",
  "c": "it's all about the bazQux",
  "d": "baz-qux"
}
```

