# Specular [![GithubActions](https://github.com/github/docs/actions/workflows/main.yml/badge.svg)](https://github.com/restaumatic/purescript-specular)

Specular is a library for building Web-based UIs in PureScript, based on
Functional Reactive Programming (FRP).

The API and DOM interaction is heavily inspired by [Reflex][reflex] and [Reflex-DOM][reflex-dom].
The FRP implementation is based on [Incremental](https://github.com/janestreet/incremental) (although the algorithm differs in some important ways).

## API

### FRP types

To use Specular effectively, you need to be familliar with some basic types.

#### [Dynamic](https://pursuit.purescript.org/packages/purescript-specular/docs/Specular.FRP.Base#t:Dynamic)

`Dynamic a` represents a read-only reference to a changing value of type `a`.

```purescript
-- | Read the current value of a `Dynamic`.
readDynamic :: forall m a. MonadEffect m => Dynamic a -> m a

-- | Execute the given action for the current value, and each new value when it changes.
subscribeDyn_ :: forall m a. MonadEffect m => MonadCleanup m => (a -> Effect Unit) -> Dynamic a -> m a
```

`Dynamic` is a `Monad`.

```purescript
-- `pure` creates a Dynamic that never changes.
pure "foo" :: Dynamic String

-- An applicative combination of Dynamics changes whenever one of them changes.
d1 :: Dynamic Int
d2 :: Dynamic Int
(+) <$> d1 <*> d2 :: Dynamic Int

-- Using the power of Monad we can choose which Dynamic to observe.
which :: Dynamic Bool
(which >>= if _ then d1 else d2) :: Dynamic Int
```

We can introduce new root Dynamics using `newDynamic`. Root Dynamics are read-write
and will be replaced by [Refs](https://pursuit.purescript.org/packages/purescript-specular/docs/Specular.Ref#t:Ref) in the future, since they are almost the same.


```purescript
-- | Construct a new root Dynamic that can be changed from `Effect`-land.
newDynamic :: forall m a. MonadEffect m => a -> m { dynamic :: Dynamic a, read :: Effect a, set :: a -> Effect Unit, modify :: (a -> a) -> Effect Unit }
```

#### [Event](https://pursuit.purescript.org/packages/purescript-specular/docs/Specular.FRP.Base#t:Event)

`Event a` represents a source of occurences. Each occurence carries a value of type `a`.

`Event` is a `Functor`.

We can construct a trivial event `never :: forall a. Event a`, which never occurs.

Events can be combined:

```purescript
-- | An Event that occurs when any of the events occur. If some of them occur simultaneously, the occurence value is that of the leftmost one.
leftmost :: forall a. Array (Event a) -> Event a
```

Events can be transformed:

```purescript
-- | Retain only the occurences of the event for which the given predicate function returns `true`.
filterEvent :: forall a. (a -> Boolean) -> Event a -> Event a

-- | Map the given function over an Event, and retain only the occurences for which it returned a Just value.
filterMapEvent :: forall a b. (a -> Maybe b) -> Event a -> Event b

-- | Retain only the occurences of the Event which contain a Just value.
filterJustEvent :: forall a. Event (Maybe a) -> Event a
```

We can observe `Event`s by being notified of their occurences.

```purescript
-- | Execute the given action for each occurence of the Event.
subscribeEvent_ :: forall m a. MonadEffect m => MonadCleanup m => (a -> Effect Unit) -> Event a -> m a
```


#### [Ref](https://pursuit.purescript.org/packages/purescript-specular/docs/Specular.Ref#t:Ref)

`Ref a` represents a read-write reference to a mutable observable variable.

We can think of a `Ref` as of `Effect.Ref`, but with additional functions:
- the ability to notify subscribers about changes to the value,
- the ability to focus using a lens.

`Ref a` consists of:
- `Ref.value :: Ref a -> Dynamic a` to observe the value
- `Ref.modify :: Ref a -> (a -> a) -> Effect Unit` to modify the value using a function

As a shortcut we have `Ref.write :: Ref a -> a -> Effect Unit` to replace the value completely,
and a `Ref.read :: forall a. => Ref a -> Effect a` to read the current value of a `Ref`.

Creating a Ref:

```purescript
Ref.new :: forall a. a -> Effect (Ref a)
```

`Ref` is not a `Functor`, because it's read-write. It's `Invariant`, that is, it can be mapped over using a bijection.

This API will also likely change in the future, so that our interface resembles a standard [Ref](https://pursuit.purescript.org/packages/purescript-refs/5.0.0/docs/Effect.Ref#t:Ref)

### Building DOM content

#### [Widget](https://pursuit.purescript.org/packages/purescript-specular/docs/Specular.Dom.Widget#t:Widget)

`Widget a` is a computation which can perform `Effects`, produce DOM nodes, subscribe to Events and Dynamics and returns a value of type `a`.

`Widget`s can be executed using `runMainWidgetInBody` - their contents will be inserted into the `document.body` element.

#### [Prop](https://pursuit.purescript.org/packages/purescript-specular/docs/Specular.Dom.Element#t:Prop)

`Prop` is a modifier attached to a DOM element. Specific ways to construct a `Prop` are presented below.

#### [Attrs](https://pursuit.purescript.org/packages/purescript-specular/docs/Specular.Dom.Browser#t:Attrs)

`Attrs` is a map of HTML attributes.

```purescript
-- A singleton map can be constructed using the `:=` operator.
"type":="button" :: Attrs

-- Attrs can be combined using the Monoid instance.
"type":="button" <> "name":="btn" :: Attrs
```

#### Static DOM

```purescript
import Specular.Dom.Element

-- | Produce a text node.
text :: String -> Widget Unit

-- | `el tag props body` - Produce a DOM Element.
-- |
-- | The elements produced by the `body` widget will be inserted as children of the element.
el :: forall a. TagName -> Array Prop -> Widget a -> Widget a

-- | `el tag props body` - Produce a DOM Element with no props.
el_ :: forall a. TagName -> Widget a -> Widget a

-- | Attach static attributes to the element.
attrs :: Attrs -> Prop

-- | Attach a static attribute to the element.
attr :: AttrName -> AttrValue -> Prop

-- | Attach CSS classes to the element
classes :: [ClassName] -> Prop

-- | Attach a CSS class to the element
class_ :: ClassName -> Prop
```

For example, to produce the following HTML:

```html
<div class="alert alert-warning alert-dismissible fade show" role="alert">
  <strong>Holy guacamole!</strong> You should check in on some of those fields below.
  <button type="button" class="close" data-dismiss="alert" aria-label="Close">
    <span aria-hidden="true">&times;</span>
  </button>
</div>
```

One would write the following Specular code:

```purescript
el "div" [classes ["alert", "alert-warning", "alert-dismissible", "fade", "show"], attr "role" "alert"] do
  el_ "strong" $ text "Holy guacamole!"
  text " You should check in on some of those fields below."
  el "button" [class_ "close", attrs ("type":="button" <> "data-dismiss":="alert" <> "aria-label":="Close")] do
    el "span" [attr "aria-hidden"  "true"] do
      text "×"
```

#### Dynamic text, attributes and classes

Most of the `Prop` constructors have their dynamic counterparts. As a convention, their names end in `D`. For example:

```purescript
-- | Attach dynamic attributes to the element.
attrsD :: Dynamic Attrs -> Prop

-- | Attach dynamic CSS classes to the element
classesD :: Dynamic [ClassName] -> Prop
```

`text` also has a dynamic counterpart:

```purescript
-- | Create a text node whose text will reflect the value of the given Dynamic.
dynText :: Dynamic String -> Widget Unit
```

For convenience, utilities for common cases are provided such as:

```purescript
attrWhenD :: Dynamic Boolean -> AttrName -> AttrValue -> Prop
classWhenD :: Dynamic Boolean -> ClassName -> Prop
```

For example: assume you have `name :: Dynamic String`.
The code:

```purescript
let isLong nm = String.length nm >= 5
el "div" [class_ "name", classWhenD (isLong <$> name) "long"] do
  text "Your name is: "
  dynText name
```

when `name` has value `"Jan"`, would produce

```html
<div class="name">Your name is Jan</div>
```

whereas when `name` has value `"Titelitury"`, would produce

```html
<div class="name long">Your name is Titelitury</div>
```

#### Dynamic DOM structure

Sometimes changing text and attributes is not enough. For that there's `withDynamic_`:

```purescript
withDynamic_ :: forall a. Dynamic a -> (a -> Widget Unit) -> Widget Unit
```

Whenever the Dynamic changes, it will re-render a new `Widget` based on the latest value.

Example:

```purescript
-- Assume loading :: Dynamic Boolean

withDynamic_ loading $
  if _ then
    el "div" [class_ "loading"] $ text "Loading..."
  else
    el_ "div" do
      el_ "h1" $ text "Content"
      el_ "p" $ text "Bla bla bla"
```

Warning: Re-rendering a whole DOM block on each change has performance implications. Use with care.

#### Handling events

```purescript
-- | Connect a DOM event on the node to a callback.
on :: EventType -> (DOM.Event -> Effect Unit) -> Prop

-- | Shorthand: `on "click"`
onClick :: (DOM.Event -> Effect Unit) -> Prop

-- | Like `onClick`, but takes a callback which ignores the DOM event.
onClick_ :: Effect Unit -> Prop
```

Example:

```purescript
-- Assume save :: Effect Unit

el "button" [attr "type" "button", onClick_ save] do
  text "Save"
```


For inputs, we have predefined props that make `change` and `input` events handling easier (available in [Specular.Dom.Element](https://pursuit.purescript.org/packages/purescript-specular/docs/Specular.Dom.Element))

```purescript
-- * Input value
-- | Attach dynamically-changing `value` property to an input element.
-- | The value can still be changed by user interaction.
-- |
-- | Only works on `<input>` and `<select>` elements.
valueD :: Dynamic String -> Prop

-- | Set up a two-way binding between the `value` of an `<input>` element,
-- | and the given `Ref`.
-- |
-- | The `Ref` will be updated on `change` event, i.e. at the end of user interaction, not on every keystroke.
-- |
-- | Only works on input elements.
bindValueOnChange :: Ref String -> Prop


-- | Attach dynamically-changing `checked` property to an input element.
-- | The value can still be changed by user interaction.
-- |
-- | Only works on input `type="checkbox"` and `type="radio"` elements.
checkedD :: Dynamic Boolean -> Prop

-- | Set up a two-way binding between the `checked` of an `<input>` element,
-- | and the given `Ref`.
-- |
-- | Only works on input `type="checkbox"` and `type="radio"` elements.
bindChecked :: Ref Boolean -> Prop

```

Example:

```purescript

import Prelude
import Specular.Ref (Ref, newRef)
import Specular.Dom.Browser ((:=))

import Specular.Dom.Element (el, attr, bindValueOnChange)
import Specular.Dom.Widget (emptyWidget)

let description :: Ref String = newRef ""

el "input" [attr "type" "text", bindValueOnChange description] emptyWidget

```


#### A Counter example

```purescript
module Main where

import Prelude
import Effect (Effect)


import Specular.Dom.Element (attr, class_,  el,  onClick_, text, dynText)
import Specular.Dom.Widget (runMainWidgetInBody)
import Specular.Ref (Ref)
import Specular.Ref as Ref


main :: Effect Unit
main = do
  -- | Will append widget to the body
  runMainWidgetInBody do
    counter :: Ref Int <- Ref.new 0

    -- | Subtract 1 from counter value
    let subtractCb = (Ref.modify counter) (add (negate 1))

    -- | Add 1 to counter value
    let addCb =  (Ref.modify counter) (add 1)

    el "button" [class_ "btn", attr "type" "button", onClick_ addCb ] do
      text "+"

    dynText $ show <$> Ref.value counter


    el "button" [class_ "btn", attr "type" "button", onClick_ subtractCb ] do
      text "-"
```

<p class="callout warning">Warning: examples which can be found in this repo which are using "FixFRP" are deprecated !</p>


## Getting started - using starter app

Clone this repository and start hacking: https://github.com/restaumatic/purescript-specular-starter

## Getting started - manually

We will use spago in this example, because spago allows us to override package sets.

Initialize a repository and install purescript

- `npm init`
- `npm install --save-dev purescript@0.13.8`
- `npm install --save-dev spago`

Add `node_modules/.bin` to path:
- `export PATH="./node_modules/.bin:$PATH"`

Initialize `spago`:

- `spago init`

to check if everything is working so far:
- `spago build`


Since `Specular` is not in an official `package-set`, you will have to add it manually,
by appending `with specular` to your `in upstream` block in `packages.dhall` file.

```dhall
-- Something like this will exist in your packages.dhall
let upstream =
  https://github.com/purescript/package-sets/releases/download/psc-0.13.8-20210226/packages.dhall sha256:7e973070e323137f27e12af93bc2c2f600d53ce4ae73bb51f34eb7d7ce0a43ea
in  upstream
  -- Add specular:
  with specular =
  { dependencies =
    [ "prelude"
      , "aff"
        , "typelevel-prelude"
        , "record"
        , "unsafe-reference"
        , "random"
        , "generics-rep"
        , "debug"
        , "foreign-object"
        , "contravariant"
        , "avar"
    ]
    ,
    repo
      =
      "https://github.com/restaumatic/purescript-specular.git"
      ,
    version
      =
      "master"
  }
```

Install specular:
- `spago install specular`
- `spago build`

Replace the content of `src/Main.purs` with the counter example, and run:
- `spago bundle-app`

Create and open `index.html` file.
```html
<html>
  <body>
    <script>window.global = {}</script>
    <script src="index.js"></script>
  </body>
</html>
```

The ugly global is required for now (possibly a browserify artifact).

If everything worked correctly, there should be a Spec(ta)ular counter!  :)

## Why not just use Reflex and GHCJS?

In short: code size. Specular demos are 240K unminified (with DCE - `pulp build
-O`), or 19K minified with `uglifyjs -c -m` and gzipped. In contrast, a a GHCJS
(`0.2.1.9007019`) program that prints `Hello World` (no DOM bindings included,
just `base`) weighs `1.1M` unminified, or 62K minified with Closure Compiler's
`ADVANCED_OPTIMIZATIONS` and gzipped. Supporting Haskell semantics has a cost.

There are also other reasons, of course.

## Why not use other PureScript UI libraries?

See [Motivation](doc/Motivation.md).

## Limitations

Some of the cons of Specular:

- No good way to do server-side rendering. Local state complicates this.

- Performance may be sometimes bad, because it does not use any Virtual DOM -
  the element placement instructions you write translate pretty much directly to
  `createElement`/`appendChild`. There are no (representative) benchmarks yet.

- Time travel debugging, as known from Elm, is not possible.

- Currently no way to bind to React Native.

- Programs written with Specular may be harder to understand for some people who
  prefer the single state variable approach.

- Compared to Reflex, it has way less FRP combinators.

- Creating recursive data flows is more cumbersome than in Reflex, because
  PureScript has eager evaluation and no `RecursiveDo`.

- It's immature and not popular, and may have bugs.

If you think there are more, please open an issue. They should be listed.

[reflex]: https://github.com/reflex-frp/reflex
[reflex-dom]: https://github.com/reflex-frp/reflex-dom

## Who's using it?

- **Restaumatic** - used in production for a signification portion of online ordering frontend, as well as for backoffice apps and our mobile app for restaurants.

## Contact

If you discover bugs, want new features, or have questions, please post an issue using the GitHub issue tracker.

You can also contact `@mbieleck` on [FP Chat](https://fpchat-invite.herokuapp.com/), if you want to chat about Specular.
