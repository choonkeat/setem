# 🔸 setem 🔸
[![sanity_check](https://github.com/ymtszw/setem/actions/workflows/sanity_check.yml/badge.svg)](https://github.com/ymtszw/setem/actions/workflows/sanity_check.yml)
[![npm](https://img.shields.io/npm/v/setem)](https://www.npmjs.com/package/setem)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](#)
[![Twitter: gada\_twt](https://img.shields.io/twitter/follow/gada\_twt.svg?style=social)](https://twitter.com/gada\_twt)

> Set'em: Elm record setter generator

## What is this?

"Getters just get, setters just set!"

`setem` generates **all possible record setters** from your Elm source files into a single module.

## Prerequisites

- Reasonably new `nodejs`
- (Preferably `yarn`)

## Install

```sh
yarn add --dev setem
(npm install --save-dev setem)
```

## Usage

```sh
# Generates for an Elm project (including dependencies)
yarn setem --output src/                                 # For Elm project cwd. "elm.json" file must exist
yarn setem --output src/ --elm-json sub_project/elm.json # For non-cwd Elm project, targeted by "elm.json" file path

# Only generates from specific files (NOT including dependencies)
yarn setem --output src/ src/Main.elm                    # From a single source file
yarn setem --output src/ src/Main.elm src/Data/User.elm  # From multiple source files
yarn setem --output src/ src/**/*.elm                    # From multiple source files resolved by glob in your shell
yarn setem --output src/ src/                            # From multiple source files in a specific directory, expanded recursively
```

(`npx` or global install also work)

It generates `src/RecordSetter.elm` file like this:

```elm
-- This module is generated by `setem` command. DO NOT edit manually!


module RecordSetter exposing (..)


s_f1 : a -> { b | f1 : a } -> { b | f1 : a }
s_f1 value__ record__ =
    { record__ | f1 = value__ }


s_f2 : a -> { b | f2 : a } -> { b | f2 : a }
s_f2 value__ record__ =
    { record__ | f2 = value__ }

...
```

All `s_<field name>` setter function works for any records with matching field names!

## But Why?

As you all Elm developers have probably known, Elm innately provides getters for any record fields:

```sh
$ elm repl
---- Elm 0.19.1 ----------------------------------------------------------------
Say :help for help and :exit to exit! More at <https://elm-lang.org/0.19.1/repl>
--------------------------------------------------------------------------------
> .name
<function> : { b | name : a } -> a
```

These getters are concise, pipeline-friendly (i.e. you can `x |> doSomething |> .name |> doElse`), therefore composition-friendly (there are packages with "lift"-ing functions that work in tandem with getters, such as [elm-form-decoder] or [elm-monocle]).

[elm-form-decoder]: https://gihtub.com/arowM/elm-form-decoder
[elm-monocle]: https://github.com/arturopala/elm-monocle

On the other hand, it does not provide setters of the same characteristics.
Standard record updating syntax is:

```elm
{ record | name = "new name" }
```

...which is,

* Not pipeline-friendly. You have to combine it with anonymous function like:
  ```elm
  x |> doSomething |> (\v -> { record | name = v }) |> doElse
  ```
* Not nest-friendly. You have to combine either `let in` or pattern matches:
  ```elm
  let
      innerRecord =
          record.inner
  in
  { record | inner = { innerRecord | name = doSomething innerRecord.name } }
  ```

Now, as discussed
[many](https://groups.google.com/g/elm-discuss/c/46TJImv3LAg/m/4KPo8R6f5QEJ)
[many](https://discourse.elm-lang.org/t/a-record-update-function-operator/4083)
[times](https://discourse.elm-lang.org/t/proposal-record-setters/5920)
in the community, it is somewhat deliberate choice in the language design,
to NOT provide "record setter/updater" syntax.

* It encourages to create nicely named top-level functions rather than relying on verbose anonymous functions.
* Also it encourages to design flatter, simpler data structures (and possibly using custom types) to more precisely illustrate our requirements.
* For unavoidable situations where setters are in strong demand, we can create "data" module with necessary setters exposed, which actually leads us to think about proper boundary of data and concerns. Never a bad thing!

But. A big BUT.

In our everyday programming we yearn for setters time to time.

* When we work with foreign record data structures from, say, packages
* When we have tons of records to work with, from code generation facilities such as [elm-graphql]
* When it is more natural to work with nested records as-is.
  For example when we use external/existing JSON APIs.
* **Simply when we do not have much time**.
  * Requirement of writing many setters in order to leverage composition-centric logics, is tedious.
  * It is a discouragement for us before writing proper data modules, when we forsee many boilerplate works. Even if it is one time thing.

[elm-graphql]: https://gihtub.com/dillonkearns/elm-graphql

For that, `setem` is born.

## What you get

As the slogan says, "setters just set!"

* `setem` just generates setters, and setters only
  * It is up to you how you utilize those setters.
    For instance, use `setem`-generated setters as building blocks for [Monocle][elm-monocle] definitions!
* It does not provide "updaters" in the sense of `(a -> a) -> { b | name : a } -> { b | name : a }`
  * It might prove useful, though my current intuition says it sees less usages than setters.
* A single importable module. Just `import RecordSetter exposing(..)` in your code and that's it! All setters are always available.

With generated setters it is possible to:

* Pipeline-d set:
  ```
  x |> doSomething |> s_name v |> s_anotherField (updater x.anotherField) |> doElse
  ```
* Nested set:
  ```elm
  { record
      | inner =
          record.inner
              |> s_name (doSomething record.inner.name)
              |> s_anotherField v
      , anotherInner =
          record.anotherInner
              |> s_number 1
              |> s_moreNest
                  (record.anotherInner.moreNest
                      |> s_yetAnotherField vv
                      |> s_howFarCanWeGet [ 2, 3, 4 ]
                  )
  }
  ```

Setters are ordinary functions. Pass them to any high-order functions as needed!

## Tips

### Q. Should we add setem-generated file to git?

A. Up to you. Personally I do. If you are irritated by cluttered diffs every time you generated `setem`,
create a `.gitattributes` file with a following entry:

```ini
src/RecordSetter.elm linguist-generated=true
```

Paths set as `linguist-generated=true` are collapsed by default on GitHub pull request diffs, reducing visual clutter.
See https://github.com/github/linguist/blob/master/docs/overrides.md#generated-code

## Implementation note

It is based on [tree-sitter-elm](https://github.com/Razzeee/tree-sitter-elm). Quite fast!

The generator looks for both record type *definitions* and record data *expressions* from your source files.
It generates setters of not yet used fields (or even, ones you are not going to use at all.)
Unused ones are expected to be sorted out by Dead-Code Elimination feature of `elm make --optimize`.

If you do not give explicit `paths` as command line arguments, `setem` reads your `elm.json` file
and generates setters not only from your `"source-directories"` but from your dependencies as well.
In this scenario, tokens from your dependencies are cached in your `elm-stuff/setem/` directory.

## Development

Install reasonably new node. If you are using `asdf`,

```
git clone git@github.com:ymtszw/setem.git
cd setem/
git submodule update --init --recursive
asdf install
yarn
yarn test
yarn test:cli
```

In GitHub Actions, sanity checks are performed against recent LTS node versions (12,14,16)

## Author & License

MIT License (c) **Yu Matsuzawa**

* Twitter: [@gada\_twt](https://twitter.com/gada\_twt)
* Github: [@ymtszw](https://github.com/ymtszw)

## Show your support

Give a ⭐️ if this project helped you!


***
_This README was generated with ❤️ by [readme-md-generator](https://github.com/kefranabg/readme-md-generator)_
