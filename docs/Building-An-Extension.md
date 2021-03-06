Building Your First Rasa Extension
==================================

Glad you could join us! This will be a quick walkthrough of the steps involved
in building a simple extension. After completing this tutorial you should feel
comfortable building something of your own! Rasa is moving fast, so it's likely
this will go out of date from time to time, if you notice that something seems
a bit off; feel free to open an issue
[here](https://github.com/ChrisPenner/rasa/issues), or hop into our [gitter
chat](https://gitter.im/rasa-editor/Lobby)!

Downloading and Setting up Rasa
-------------------------------

Rasa is a Haskell project, configured in Haskell; so we're going to need a
basic Haskell working environment if we're going to get anywhere. This guide
assumes at least a basic familiarity with both Haskell &
[Stack](http://seanhess.github.io/2015/08/04/practical-haskell-getting-started.html).
Go ahead and install Stack now, I'll wait! (Psst! I'd recommend
`brew update && brew install haskell-stack`). Make sure you run `stack setup`
before we get started.

You'll also want to install 'nix' which is a tool which stack uses to guarantee
reproducible builds. You can find installation instructions for nix
[here](https://nixos.org/nix/).

Okay, so first clone the Rasa repository; go to your favourite project's
directory and run `git clone https://github.com/ChrisPenner/rasa`; this will
create a new `rasa` directory, go ahead and `cd rasa` into there.

Instead of going through the trouble of making a new package for our extension,
we'll just build the new functionality into our user-config. You can extract it
later if you like!

Let's just edit the existing configuration for now, you can find it in
`rasa-example-config/app/Main.hs`. It should look something like this:

```haskell
module Main where

import Rasa (rasa)
import Rasa.Ext
-- ... some other imports
main :: IO ()
main = rasa $ do
  viewports
  vim
  files
  cursors
  logger
  slate
  -- some more plugins...
```

To get a feel for the different things we can do we'll be building
a simple copy-paste extension.

Hello World
-----------

To get started let's add a simple action and see if we can even get it running.
Let's make a new action to write "Hello world" to a logging file so we can see
if it runs!

```haskell
helloWorld :: Action ()
helloWorld = writeFile "hello-world.txt" "Hello, World!"

main :: IO ()
main = rasa $ do
  -- some plugins...
  -- Add the new action here!
  helloWorld
```

Okay let's try again! `stack build && stack exec rasa` (you may want to alias
that to something, we'll be running it a lot!)

```haskell
rasa/rasa-example-config/app/Main.hs:28:14: error:
- Couldn't match expected type ???Action ()??? with actual type ???IO ()???
- In the expression: writeFile "hello-world.txt" "Hello, World!"
    In an equation for ???helloWorld???:
        helloWorld = writeFile "hello-world.txt" "Hello, World!"
```

Oh of course! We've declared `helloWorld` to be type `Action ()`, which can only
contain statements of type `Action`. Luckily `Action` implements MonadIO so we can
embed IO actions in it by using the function `liftIO`. Don't worry too much if you don't
fully understand what's going on here, you can just follow along for now!

```haskell
-- new import at the top!
import Control.Monad.Trans

-- Let's lift the writeFile call into an `Action`
helloWorld :: Action ()
helloWorld = liftIO $ writeFile "hello-world.txt" "Hello, World!"
```

If we've done this right, then when we rebuild the editor should compile and
start up! If you get an error stating that Haskell couldn't find the
`Control.Monad.Trans` module then you may need to add 'mtl' to the list of
build-depends in `rasa-example-config.cabal`. Okay, so how can we see whether
it worked? If you're using the Vim key-bindings then you can quit using
`Ctrl-C` and check your directory to see if you've got any new files!

```sh
$ cat hello-world.txt
Hello, World!
```

Awesome! It's working! Okay, we can run an action! what's next? I suppose it
would be nice to run an action in response to some keypress right?

Listening for Keypresses
------------------------

Let's change `helloWorld` into a function which
takes a `Keypress` and writes the pressed key to a file!

```haskell
helloWorld :: Keypress -> Action ()
-- Print out the characters we type!
helloWorld (Keypress char _) = liftIO $ appendFile "keylog.txt" ("You pressed: " ++ [char] ++ "\n")
-- Ignore other types of keypresses (arrows, escape sequences, etc.)
helloWorld _ = return ()
```

Okay cool! Now it should record each keypress to the `keylog.txt` file!
Let's run it!

```
rasa/rasa-example-config/app/Main.hs:28:10: error:
- Couldn't match expected type ???Action ()???
                with actual type ???Keypress -> Action ()???
```

Hrmm, right! Now that we're listening for keypress events we need to set up an
event-listener! Rasa provides a way to register listeners for different events;
we'll learn how to listen for any arbitrary event later; but for now it's time
for `onKeypress`!

```haskell
onKeypress :: (Keypress -> Action ()) -> Action ListenerId
```

So we've got our function from our event type (Keypress), so let's try
registering it using `onKeypress`; The onKeypress function returns a reference
to the newly created listener so that we could cancel the listener using
`removeListener` later if we wanted to; but since we don't need to do that;
we'll just ignore it by using `void` from `Control.Monad`.

```haskell
import Control.Monad
main = rasa $ do
  -- other extensions
  void $ onKeypress helloWorld
```

Okay, let's build that and run it, now in a separate terminal we'll run
`tail -f keylog.txt` to watch the key events come in! Go back to the editor and
hit a few keys and you should see them come streaming in! See how easy it is
to get things working?

Interacting With Other Extensions
---------------------------------

A key part of extensions in rasa is that they can interact with other
extensions! This means we can take advantage of the work that others have done
before us! The main way that we interact with other extensions is to use
`Action`s that they expose for us.

We're doing a copy-paste sort of thing so we'll want to look at what the
cursors extension provides for us to use. If we take a look at the
documentation for `rasa-ext-cursors` we'll see a few things that we can do.

Out of the actions they've exposed for us to use one thing that looks promising
is `rangeDo`, it lets us run a function over each range that the `cursors`
extension has selected. `cursors` handles many selections at the same time,
which makes things a bit more complicated, but let's just pretend for now that
it only handles one at a time. `rangeDo` has the following signature:

```haskell
rangeDo :: (CrdRange -> BufAction a) -> BufAction [a]
```

A `CrdRange` is a construct that's part of Rasa, it represents a range of text
within a buffer from a starting 'coordinate' to an ending 'coordinate'. In Rasa
file positions are usually represented using the type `Coord Row Column` where
Row and Column are integers representing the row and column of the position in
the text. 

We can see that `rangeDo` takes a function which accepts a range and returns
a `BufAction`, and runs it on each range, returning a list of the results
inside a `BufAction`. We can use this for our copy-paste extension to go
through and copy the text in each range so we can paste it later! To see if we
can get it working, first let's write a basic version that just writes out the
contents of each range to a file! Things get a bit tougher here, but we're
going to power through it!

We need a few other packages soon, so we'll have to tell stack that we want
to include them in the build! We can do this by going to the
`rasa-example-config.cabal` file and we'll add `data-default` and `yi-rope` to
the `build-depends` list there.

```haskell
-- Make sure we have these imports
import Rasa.Ext.Cursors
import Rasa.Ext.Views
import qualified Yi.Rope as Y

main = rasa $ do
  -- other extensions
  cursors
  void $ onKeypress copyPasta

copyPasta :: Keypress -> Action ()
copyPasta (Keypress 'y' _) = focusDo_ $ rangeDo_ copier
  where
    copier :: CrdRange -> BufAction ()
    copier r = do
      txt <- getRange r
      liftIO $ appendFile "copy-pasta.txt" ("You copied: " ++ Y.toString txt ++ "\n")
copyPasta _ = return ()
```

Okay let's unpack it a bit! We're now listening specifically for the 'y'
character to be pressed, if anything else is pressed we'll just do a simple
`return ()` which does nothing. When someone presses 'y' then we we run
`focusDo` which lifts a `BufAction` into an `Action` which operates over only
the focused buffer(s). The `BufAction` that we're running is handled by
`rangeDo_`. You'll notice we're using `rangeDo_`, which is a version of
`rangeDo` which discards return values resulting in just `BufAction ()`.

As we discovered earlier `rangeDo` takes a function over a range, which we've
defined as `copier`. The next line `txt <- getRange r` uses a rasa command
`getRange` which just gets the text of a specific range from the buffer.
There's a trick though, the text which comes out is actually a `YiString` which
comes from the `Yi.Rope` library, we need to convert it to a string from the
rope so we'll use `Y.toString` from the rope library so we can write it to
file. We then append the text to the `copy-pasta.txt` file.

Let's try it out. Hopefully it builds for you and you can test it out by moving
the cursor around and pressing 'y' to write out the current character to the
`copy-pasta.txt` file (you can watch it with `tail -f copy-pasta.txt`).

If you're new to Vim bindings (these are a bit different after all) you can
move the cursor around the screen by using the `hjkl` keys.

Storing Extension State for Later
---------------------------------

We've got a good start at this point! If you like you can add multiple cursors
using the cursors extension (for example `J` will add a new cursor on the line
below) and should see that each cursor's contents is copied to the file! The
next thing we'll need to do is to store the last thing we've copied so that we
can paste it back later, we could just store it in a file, but this gets tricky
and inefficient for more complicated data-structures, so we'll do it another
way. Rasa provides a way for each extension to store state, in fact you can
store either state global to the entire editor, or we can store state local to
each buffer. In this case we want the latter, that way we can keep a different
copy-paste register for each buffer.

Let's see how we can store some persistent state in the Buffer that we can
access later. We need to define a custom data-type for this so that Rasa can
tell it apart from all the other extension's states. Make sure that you don't
directly store basic data-types like Strings or Ints or they might conflict
with other extensions! Always either wrap your type with a `newtype` for your
extension, or define a new `data` type.

In this case we really only need to store a String which we'll paste later, so
we'll just wrap `String` in a newtype called `CopyPasta`

```haskell
newtype CopyPasta = CopyPasta String
  deriving Show
```

That was easy, note that we're deriving a `Show` instance here, Rasa requires a
`Show` instance for extension states because it makes debugging them much
easier! There's one more typeclass we need to implement in order to store this,
it's the `Data.Default` typeclass. Basically it just states that our
Extension's state has a default value which it can be initialized to. Though
it's not always easy to come up with a Default state for your extension, Rasa
requires it because it simplifies initializing the extensions by a great deal.

Okay, let's write the Default instance!

```haskell
-- New import at the top of your file!
import Data.Default

instance Default CopyPasta where
  def = CopyPasta ""
```

For now we'll just define the default to be an empty string.

With that, we've got everything set up to store custom state! Let's change our
copyPasta code so it copies text into the extension state rather than into a file.

```haskell
copyPasta :: Keypress -> Action ()
copyPasta (Keypress 'y' _) = focusDo_ $ rangeDo_ copier
  where
    copier :: CrdRange -> BufAction ()
    copier r = do
      txt <- getRange r
      setBufExt $ CopyPasta (Y.toString txt)
copyPasta _ = return ()
```

`setBufExt` is a special function of the following type:

```haskell
setBufExt :: (Typeable ext, Show ext, Default ext) => ext -> BufAction ()
```

This shows us that for a value of ANY type `ext`; so long as it can be shown, typed,
and has a default, then we can store that information for later. This is also why 
we must wrap our types up into something unique to our extension; if we used setBufExt
on an unwrapped Int it would be overwritten any time another extension stored an Int!
Using a custom type ensures that no-one else can alter our state unless
they have access to the `CopyPasta` type, so if we don't expose it, then we're
the only ones who can make changes! 

So even if this builds, how can we tell if it's actually working? Well if you
like you can include the `logger` extension (it should be included in
the config already). This extension will print out the state of the editor after
every event that it processes, so it's great for debugging! It prints them out
to a file called 'logs.log' (original I know...). If `logger` is in the config
then we can `tail -f logs.log` to see how our editor state changes as we perform
events. Give it a try now!

If you hit 'y' when a character is selected in the editor, then look at the
logs in `logs.log` and squint a little, you might be able to see that one of
the buffers has a `CopyPasta` object stored in with its extensions! Since we
derived a `Show` instance for `CopyPasta` we should also be able to see the
string that was copied!

Good stuff! If you've gotten stuck on anything, please make an [issue](https://github.com/ChrisPenner/rasa/issues) for it!.

Accessing Stored State
----------------------

Now for the fun part! Let's make it paste! We'll add another case to the
copyPasta function that listens for the user to press the 'p' key instead, if
they press 'p', then we'll paste whatever we have stored!

```haskell
copyPasta :: Keypress -> Action ()
copyPasta (Keypress 'y' _) = focusDo_ $ rangeDo_ copier
  where
    copier :: CrdRange -> BufAction ()
    copier r = do
      txt <- getRange r
      setBufExt $ CopyPasta (Y.toString txt)
copyPasta (Keypress 'p' _) = focusDo_ paster
  where
    paster :: BufAction ()
    paster = do
      CopyPasta str <- getBufExt
      insertText (Y.fromString str)
copyPasta _ = return ()
```

Firstly; notice that `getBufExt` lets us get back any state we've set with
`setBufExt`. If we've never set a value of that type it will return the default
instead. `getBufExt` is also able to return a value of any type; luckily by
pattern matching on the left hand side `CopyPasta str <- getBufExt` the
compiler infers which state to retrieve for us! Pretty cool eh?

You'll see that this time we don't have to run something over every range
because there's actually a useful `insertText` function exposed by the
`cursors` extension! Here's the signature:

```haskell
insertText :: YiString -> BufAction ()
```

It takes a YiString and inserts it at the beginning of every range! How
convenient! That means we can just a write a single `BufAction` that does what
we need and embed it using `focusDo_`. 

We'll do a simple pattern match on the left of the arrow
`CopyPasta str <- getbufExt` to get the string we stored earlier. Now we can
use the nifty `insertText` function! It returns a `BufAction ()`, so we can
just us it in-line with the other `BufAction` where we extract the state. Like
most parts of Rasa `insertText` accepts a YiString; so we'll need to convert
our string. At this point we'd usually just change our extension to store a
`YiString` rather than a string since it's more efficient anyways; but we've
come this far so let's just convert it using `Y.fromString` to turn it into
the YiString we need.

At this point we have a working (albeit simple) copy-pasting extension which
has a separate copy-register for each buffer! We've learned how to listen for
and respond to events, and store state to use later! We've also learned how to use
utilities that are exported from other extensions!

Dispatching Custom Events
-------------------------

Let's add just one last feature, other extensions may want to know when the
user copies something. Let's expose an event so that they can 'listen' for when
something gets copied and do something in response! This is actually really easy
to do since it's a core part of how rasa operates!

First we'll define a new type that represents the behaviour. Each unique action
that someone may want to listen for should have its own `newtype` or `data` type.

```haskell
newtype Copied = Copied String
```

Okay! So there's a type that not only specifies what happened, but also
includes the necessary data for someone to do something useful (we'll include
the string that the user copied as part of the event)!

If we were writing this as an extension in a separate package we'd expose the
type (and probably the constructor so that people can pattern match on it). Then
other folks could use the functions we expose to listen for the event just like we
did with `Keypress` events!

Now that we've defined the type, we need to dispatch an event each time
the user copies something! Say hello to `dispatchBufEvent`!

```haskell
dispatchBufEvent :: forall result eventType. (Monoid result, Typeable eventType, Typeable result) => (eventType -> BufAction result)
```

Whoah! What's going on there!?

Well; because Rasa is built to be extensible; the event system is polymorphic;
which means it can accept arguments of any type. The important part of the type
is: `eventType -> BufAction result`. This tells us that we can use the function
to dispatch any event and get a `BufAction` of the result. The first part of
the signature tells us that it works for any such event or result (`forall`)
where the result is a Monoid, and both can are Typeable. We won't be using the
`result` of the dispatch; so we'll mostly ignore it from now. Just know that
it's possible to use events to 'ask' extensions to provide you with a value in
response to an event; it turns out that `()` is a Monoid; so we'll
just use that when we don't want a response.

Note that we're using `dispatchBufEvent` here; there's also a `dispatchEvent`;
the difference is that `dispatchBufEvent` will only trigger listeners registered on
the buffer where it's dispatched; and `dispatchEvent` only triggers global listeners.

```haskell
newtype Copied = Copied String

copyPasta :: Keypress -> Action ()
copyPasta (Keypress 'y' _) = focusDo_ $ rangeDo_ copier
  where
    copier :: CrdRange -> BufAction ()
    copier r = do
      txt <- getRange r
      setBufExt $ CopyPasta (Y.toString txt)
      -- We'll dispatch the copied event with the text we copied!
      dispatchBufEvent $ Copied (Y.toString txt)
copyPasta (Keypress 'p' _) = focusDo_ paster
  where
    paster :: BufAction ()
    paster = do
      CopyPasta str <- getBufExt
      insertText (Y.fromString str)
copyPasta _ = return ()
```

This is what we'd expect to do, but uh-oh! We build this and we get an error!

Here's a version that works!

Now we're alerting any listeners each time we copy something! Don't believe me?
Okay fine! Let's listen for the events so we can see them coming through!

At this stage will create a utility function for other extensions to listen for
our `Copied` event. We do this with the provided `addListener` or
`addBufListener` functions. These are a bit confusing like `dispatchEvent`; but
we'll get through it!

```haskell
addBufListener :: forall result eventType. (Typeable eventType) => (eventType -> BufAction result) -> BufAction ListenerId
```

Similar to the dispatch functions this is a polymorphic function; except notice
that this time the first argument is a function! `addBufListener` takes a
function which accepts some event and builds a `BufAction` which ends in some
result. We need to make sure that the results and event types that people use
when adding listeners matches the types that we expect when we dispatch
otherwise listeners won't trigger properly; so the best idea is to constrain
the types and re-export the function with the proper signature. We can do that
just like this:

```haskell
addCopyListener :: (Copied -> BufAction ()) -> BufAction ListenerId
addCopyListener = addBufListener

copyListener :: Copied -> BufAction ()
copyListener (Copied str) = liftIO $ appendFile "copied.txt" ("Copied: " ++ str ++ "\n")

newBuf :: BufAdded -> Action ()
newBuf (BufAdded bufRef) = bufDo_ bufRef (addCopyListener copyListener)

main = rasa $ do
  -- other extensions
  onBufAdded_ $ newBuf
```

Okay so this works; but there was a bit of boiler-plate to get it going!
The biggest problem is that the `Copied` event is dispatched to the Buffer where
it occured; so we'll need to listen for it on that buffer. But how do we register
a listener not only for all buffers; but all FUTURE buffers too? We can use
the `onBufAdded` function to register an event listener which triggers each
time a new buffer is added; the BufRef of the new buffer will be passed into
the function via the `BufAdded` event; which we can then use with `bufDo` to
run BufActions against. This is probably pretty confusing right now; that's fine. 
As you read the docs and use other extensions it will hopefully become more clear!

So we wrote a function `newBuf` which will be run each time a new buffer is
added; inside we use the BufRef we get from the event and call `bufDo_` which
runs a BufAction against the buffer referred to by the given `BufRef`. The `_`
means we want to ignore the result (in this case it's a ListenerId). The
BufAction we pass to `bufDo` is `addCopyListener copyListener` which registers
our listener for the copied event in the given buffer.

### Extracting the Extension to its own Package

Now that we've built an extension we should share it with the community!

If we extract it into its own package then other extensions and users can
depend on it and use it in their own extensions and configurations. To do this
we can use stack to create a new project for us. `cd` into the top-level rasa
directory where the rest of the `rasa-ext` folders are and run
`stack new rasa-ext-copy-pasta simple-library`.

It's good practice to prefix your rasa extension package name with `rasa-ext`
just so people can search for them easily. Running that stack command should
have made a new package folder for you with a simple library package template
inside. Go ahead and `cd rasa-ext-copy-pasta`.

Inside here you'll see some config files and a `src` folder. You'll want to
open up the `rasa-ext-copy-pasta.cabal` and make sure the info in there is
correct and has the libraries we need, You'll need to move over the things from
'build-depends:' of `rasa-example-config.cabal` into this cabal file. You'll
also see a `stack.yaml` inside the folder, we won't need that since the entire
rasa git repository is a single stack project with its own `stack.yaml`, so go
ahead and `rm stack.yaml`.

At this point we can move our code over, you can delete the `Lib.hs` that's in
`src` and instead make some folders inside `src`: `Rasa/Ext`. Add a new file inside
`Ext` called `CopyPasta.hs`. Your structure should look something like this:

```
rasa-ext-copy-pasta
????????? src
??????? ????????? Rasa
???????     ????????? Ext
???????         ????????? CopyPasta.hs
????????? rasa-ext-copy-pasta.cabal
????????? Setup.hs
????????? LICENSE
```

All extensions should be stored as a new module inside `Rasa.Ext`. Now we can
go ahead and copy those functions we wrote inside `Main.hs` into our new
`CopyPasta.hs` and add a module definition at the top. We'll want to only
export things we're okay with other people using, so how about we export the
`addCopyListener` function which sets up a custom listener, and also the
`Copied` type so that users can write their own event listeners and unpack the
string inside. We'll also pack up all of the setup for the key-listeners into a single
Action which the user can put into their config.

We'll end up with something like this:

```haskell
-- CopyPasta.hs

module Rasa.Ext.CopyPasta (
    copyPasta
  , Copied(..)
) where

import Rasa.Ext
import Rasa.Ext.Cursors

import Control.Monad.Trans
import Data.Default
import qualified Yi.Rope as Y

newtype CopyPasta = CopyPasta String
  deriving Show

instance Default CopyPasta where
  def = CopyPasta ""

newtype Copied = Copied String

-- We've renamed things so we can export a single 'Action'
-- that the user can embed in their config.
copyPasta :: Action ()
copyPasta = void $ onKeypress keyListener

keyListener :: Keypress -> Action ()
keyListener (Keypress 'y' _) = do
  focusDo_ $ rangeDo copier
  where
    copier :: CrdRange -> BufAction String
    copier r = do
      txt <- getRange r
      setBufExt $ CopyPasta (Y.toString txt)
      dispatchBufEvent $ Copied (Y.toString txt)
keyListener (Keypress 'p' _) = focusDo_ paster
  where
    paster :: BufAction ()
    paster = do
      CopyPasta str <- getBufExt
      insertText (Y.fromString str)
keyListener _ = return ()
```

Lastly we need to tell cabal which modules we want to export, so we'll add
`Rasa.Ext.CopyPasta` to the list in rasa-ext-copy-pasta.cabal:

```yaml
  exposed-modules: Rasa.Ext.CopyPasta
```

Aaaaaaand since we're doing this inside one big stack project we'll need
to tell stack about it in the `stack.yaml`; add your new package to the `packages`
list:

```yaml
packages:
# ... other packages
- ./rasa-ext-copy-pasta
```

If you really wanted to do this right, you'd make a new package outside of the
rasa repo; this requires setting up your own stack.yaml dependencies, which can
be a bit tricky, so if you run into any trouble come make an issue and we'll
see if we can sort it out! But conceptually, we've made our own package that
anyone could use! All that would be left is to
`stack upload rasa-ext-copy-pasta` to upload it to hackage (but please don't do
that or we'll have a ton of copies of this tutorial uploaded).

That's all for this episode! Go and make some of your own extensions!
I'm excited to see what you come up with!

There are a few other extensions included in this repo, so go ahead and take a
look at how they're doing things to get a few more ideas on how to do the
tricky stuff!

Cheers!
