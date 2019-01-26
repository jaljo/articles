# Rich Text Editor

## Context

We're working for an information company. They publish many contents on their
website, most of the time represented as articles or as a collection of
articles (they call it `topics`). As the company is working in the information
business, they also post some `news`. News are small texts, which may contain
tweets or links to other articles.

Editors and web managers needed a tool to be able to create and edit these kinds
of contents. These contents are saved in our database as an HTML string.

There alreay is many text edition tools online, but we wanted a tool which
looks close to the modern text editor of [medium.com](https://medium.com/).
Some open source libraries are offering such solution, but most of the time,
when specific needs arise, these libraries no longer fulfill all your
requirements. And here you are on the highway of awfull tweaks... Instead of
spending time to develop additional features on top of an existing library (with
all drawbacks implied), we decided to implement our own tool. That way, we could
make it behave exactly the way we want.

Keep in mind that our text editor development took place whereas the frontend
website used to publish informations was close to be put online.

Technically speaking, articles are fetched from an HTTP REST API (our back
office). Some of them were coming from a legacy application, and may contain
some broken HTML semantic. To address this problem, we decided to develop a
parser for the received HTML markup to be cleaned. This parser creates React
components for, and only for HTML tags that we allow. Doing so, only tags that
we're sure to support are rendered, which also simplifies design and stylization.

The same parsing logic had to be kept for our text editor as we wanted it to be
WYSIWYG, i.e. display the edited content like it would be displayed in the
public website, with the ability to directly edit it.

## The specs

The text editor sould provide the basic fatures of text edition :
- apply some style to a portion of text (bold, italic, underlined)
- create an HTTP link
- create some paragraphs
- create titles
- create quotes

Our client also needed to insert some more specific medias :
- regular images (uploaded or picked from their image collection)
- videos (also picked from their video collection)
- tweets (by pasting the tweet URL)
- Youtube videos (by pasting the video URL)

Although such features may seems very common, complexity shows up because our
text editor fits in a more global administration system. Unlike in a regular
text editor, where an user can insert images from it's computer, we had to
direclty rely on images and videos collection used in the back office
application so that editors can reuse early uploaded / edited medias (e.g. on a
previous article writing).

These requirements pushed us forward to develop our own text editor from scratch
rather than using an existing library.

## Text edition POC

### First attempt : AST

We first wanted to modelize the text to edit using an AST (abstract syntax
tree). The HTML string was parsed and represented as a tree, the same way the
DOM nodes are represented. Some simple mutation functions were written to
stylize the text : make it bold, underlined, etc... At that time, we only
focused on the AST part first, the rendering part beeing easier to handle could
come later.
Relying on unit tests, we ensured that mutations were working as expected. We
succeeded to update the AST correctly when applying mutations, eg the following
text node

```js
{
    type: "text",
    text: "lorem ipsum",
    children: [],
}
```

was succesfully transformed as a child of an other node when trying to make it
bold :

```js
{
    type: "b",
    text: "",
    children: [
        {
            type: "text",
            text: "lorem ipsum",
            children: []
        }
    ]
}
```

However, things were getting way more complicated from the moment we worked on
more edgy cases, i.e. update some text that was starting in a paragraph and
ending in an other. Selection overlapsing on different and nested tags, such as

```
<b>Hello <em>John Doe</em></b>, how are you ?
    |---------------|
selection start   selection end
```

was a daunting too. Applying such changes on the AST would have been difficult
for us. In the mean time, the developer who started to work on the AST was
required on an other project, so we decided not to use this AST anymore. Because
it was the corner stone of the back office application, keeping it at that level
of complexity and abstraction was too risky and time consuming to develop also to
maintain, plus the learning curve was giving nightmares to newcomers. The whole
project beeing based on functional programing principles, AST development should
have embraced these principles as well. Thing is, AST manipulation relies mostly
on one ability to identifiy ancestors of a node, find, update or even remove it
down through a collection of other nodes. This implies a lot of recursion and
make immutabilty very hard to preserve. Although it would have been possible to
achieve such result using advanced FP concepts like algebraic data types, we
felt that it was definitely not a good bet to take for the project's sake.

We took an other approach to the problem, by using the DOM API to keep the tree
representation and directly manipulate the view in order to easily add/remove
HTML elements.

### Second attempt : manipulating the DOM

So we started to dig in the DOM API documentation. We direclty obtained a
rendered view (what we hadn't so far with the AST) and were able to edit it. The
idea behind this approach was to use the DOM API instead of reimplementing such
node parsing and manipulation ourselves. Of course, that apporach has drawbacks:
working directly on the view, we no longer had a model representation that we
could render. The view became our model (insert dramatic music here for a better
effect).

Once more, MDN was our friend for this study :
- the [DOM Node API](https://developer.mozilla.org/en-US/docs/Web/API/Node)
permitted us to navigate through the node tree
- we were able to [get the selected text](https://developer.mozilla.org/en-US/docs/Web/API/Selection)
(returns a [range](https://developer.mozilla.org/en-US/docs/Web/API/Range))
- we were simply creating elements to insert with
[`document.createElement`](https://developer.mozilla.org/en-US/docs/Web/API/Document/createElement)

We had to be very careful with the DOM manipulation, as some Node methods are
directly mutating the node itself. Update on the node should be done only after
we have finished to compute the changes to apply. To perform this computation,
we worked on copies of these nodes so we could edit them without changing the
actual view.

We reached a working system for text mutations (bold, italic) across a single or
multiple paragraphs, even when the selection was overlapsing multiple HTML tags,
which was quite a big step (see the above example).

The hard part was to produce the cleanest HTML markup as possible, eg for a
`Hello world` text, when `Hello ` is already bold and we make `world` bold
too, we initialy had

```html
<b>Hello </b><b>world</b>
```
two `b` tags following each other, but this should have been simplified to
```html
<b>Hello world</b>
```
That sanitization part was performed once the new `b` tag was added to the DOM.

This was working fine, but the number of lines of code started to perilously
increase, and we were discovering more and more edge cases which were a PITA to
cover.

Finally, down the corner of a murky russian Stackoverflow thread we found it.
The method which would have saved us all.

### The holy grail : document.execCommand

We discovered
[`document.execCommand`](https://developer.mozilla.org/en-US/docs/Web/API/Document/execCommand) \o/
This function applies some text transformations to the current selection. Eg
to bold / unbold the selection, just execute :
```js
document.execCommand('bold')
```
and that's it ! Your selection is wrapped inside a `b` tag, the produced
markup is very clean (remember the previous step where we had to sanitize the
markup manually).

This is very neat, it enables rich text edition directly in the browser (and
by the browser :p). And look at that compatibility table, green everywhere ! The
icing on the cake: because we're working on `contentEditable` tags, conventional
shortucts work as well (ctrl+B for bold, etc).

Checkout [all the available commands](https://developer.mozilla.org/en-US/docs/Web/API/Document/execCommand#Commands)
to see what's possible to do ! Needless to say, when we discovered this, we
immediately stopped the previous attempt, thrown our POC to the lions and it was
definitely time for a beer of victory !

Considering the commands list, we almost had all needed operators to build our
text editor in a maintainable way. We only had to execute the right command when
clicking on the right button.

## Design

The text editor is composed of two distinct sets of tools:
- the paragraph toolbox (displayed when the cursor is on an empty paragraph)
- and the text toolbox (displayed when a text is selected)

![paragraph toolbox](/images/text-editor/paragraph-toolbox.png)
*The paragraph toolbox*

![text toolbox](/images/text-editor/text-toolbox.png)
*The text toolbox*

### Paragraph toolbox

The paragraph toolbox enables some media insertion in a new paragraph. When
clicking on the `image` button, it opens a media picker. We can then search the
image to include from our database, or upload a new image, edit it, etc.

The `video` button behaves almost the same way than the `image` button :
it opens the media picker for videos.

For tweets and youtube videos insertion, a simple `text` input is displayed
on the empty paragraph so the user can type the URL of the media to insert. The
tweet or the video is inserted when this little form is submitted (see the last
part of that article for a detailed workflow on this feature).

## Implementation

We're using
- [react](https://reactjs.org/)
- [redux](https://react-redux.js.org/)
- [react-functional-lifecycle](https://github.com/Aloompa/react-functional-lifecycle)
- [redux-observable](https://redux-observable.js.org/)
- [ramda](https://ramdajs.com/)

In the following part of this article, we'll assume that you're comfortable with
these technologies. If you're not, please take some time to understand the core
principles of them.

### View

The text editor is a react component :

```jsx
// TextEditor :: Props -> React.Component
export default ({
  click,
  editorName,
  keyDown,
  paste,
  selectText,
  children,
}) =>
  <div className="text-editor">
    <ParagraphToolbox editorName={editorName} />
    <TextToolbox editorName={editorName} />
    <article
      data-editor-name={editorName}
      className="edited-text-root"
      contentEditable={true}
      suppressContentEditableWarning={true}
      onClick={e => click(editorName, e.target)}
      onKeyDown={keyDown}
      onPaste={paste}
      onSelect={() => selectText(editorName)}
    >
      {children}
    </article>
  </div>
```

As you can see, the view component is pretty simple. We have the two toolboxes
which are also components. The `article` tag is where all the bindings are
applied. Inside the `article`, we render the actual text to edit using the
`children`. We'll first explain how the `TextEditor` component itself works,
and then we'll dig into the `ParagraphToolbox` and `TextToolbox` components.

As we can have many TextEditor instances on a single page, we have to
differenciate them. To do so, we simply give them a name (notice the
`editorName` prop).
About the other params :
- `contentEditable={true}` applies the
[`contenteditable`](https://developer.mozilla.org/en-US/docs/Web/Guide/HTML/Editable_content)
attribute to the `article` node so we can edit its content.
- `suppressContentEditableWarning={true}` indicates to react not to print a
warning in the console about content editable node.
- `onClick={e => click(editorName, e.targe)}` the handler to call when we click
on the edited text. It will be used to display the paragraph toolbox when
clicking on an empty paragraph for instance.
- `onKeyDown={keyDown}` the handler to call when we press a key down. It will
be used to hide the paragraph toolbox when we start to type into an empty
paragraph.
- `onPaste={paste}` the handler to call when we want to paste some text from
the clipboard. We will read the clipboard content to fetch its text.
- `onSelect={() => selectText(editorName)}` the handler to call when a text
inside this node is [selected](https://reactjs.org/docs/events.html#selection-events)

All the above handlers are dispatching redux actions. This allows us to
listen to these actions inside epics (thanks to `redux-observable`) and
to apply some changes.

In the end, this is how we use this component :

```jsx
<TextEditor editorName="article-editor" main>
  {isEmpty(componentInfos)
    ? <p></p>
    : indexedMap((componentInfo, index) =>
        <Widget
          component={componentInfo}
          id={index}
          key={index}
        />
      )(componentInfos)
  }
</TextEditor>
```

The `TextEditor` is wrapping a `Widget` component tree, which is what would
be edited. The `Widget` component is our component responsible to render all
the king of HTML tags we support (remember what's explained in the
[context](#context) section).

The `main` property indicates if this instance of the TextEditor is the main
instance used on the page.

### Container

```js
// mapDispatchToProps :: (Action * -> State, Props) -> Props
const mapDispatchToProps = (dispatch, props) => ({
  click: compose(dispatch, click),
  keyDown: pipe(
    when(
      compose(equals(13), prop('keyCode')), // the <enter> key
      tap(e => e.preventDefault()),
    ),
    e => keyDown(props.editorName, e.keyCode),
    dispatch,
  ),
  clear: compose(dispatch, clear),
  paste: pipe(
    tap(e => e.preventDefault()),
    paste,
    dispatch,
  ),
  initialize: compose(dispatch, initialize),
  selectText: compose(dispatch, selectText),
})

// didMount :: Props -> Action.INITIALIZE
const didMount = ({ initialize, main }) => !isNil(main) && initialize()

// willUnmount :: Props -> Action.CLEAR
const willUnmount = ({ clear, main }) => !isNil(main) && clear()

// lifecycles :: React.Component -> React.Component
const lifecycles = compose(
  componentDidMount(didMount),
  componentWillUnmount(willUnmount),
)(TextEditor)

// TextEditor :: Props -> React.Component
export default connect(
  null,
  mapDispatchToProps,
)(lifecycles)
```

As we want our component to be connected to the redux store (in order to
dispatch some actions and to get state values), we're using the
[`connect`](https://react-redux.js.org/api/connect) method of the `react-redux`
library.
We're also hooking into the [component's lifecycles](https://reactjs.org/docs/react-component.html#the-component-lifecycle)
using the [react-functional-lifecycle](https://github.com/Aloompa/react-functional-lifecycle)
library to dispatch some redux actions when the component is mounted and will
be unmounted :

- `didMount` : dispatches an `INITIALIZE` action for the main text editor
to indicate that it is ready. We have some epics which are listening to this
event to register some events listeners on the `window.mousedown` event so
we carefully dispatch this action only once in the case where there are
multiple text editor instances on the page. The `window.mousedown` listener
is here to hide the paragraph and text toolboxes when we click outside of
a text editor.
- `willUnmount` : dispatches the `CLEAR` action for the main text editor when
it will be unmounted. The previously `window.mousedown` event listener will
no longer be used once this action is dispatched.

### Epics

Our epics are exposed with
[redux-observable](https://redux-observable.js.org/docs/basics/Epics.html).

All our DOM manipulations and media insertion logic is delared here. There
are also some event listeners to window events.
To have a better understanding of what's inside the epics, we'll present
the paragraph and text toolboxes.

## Text toolbox workflow

The text toolbox contains a set of buttons to apply text mutations to the
selected text :

![text toolbox](/images/text-editor/text-toolbox.png)
*The text toolbox*

The component's view basically consist into a list of buttons :

```jsx
<ul>
  <li>
    <button
      className={`ttbx-mutation icomoon-font ${isBold ? 'active' : ''} ${isTitle ? 'disabled' : ''}`}
      onClick={() => mutate('BOLD')}
      disabled={isTitle}
    >4</button>
  </li>
</ul>
```
*Example of the bold / unbold button*

As for the `TextEditor` component, the `TextToolbox` component is connected to
the redux store so we can dispatch actions and get state values. On the above
example, we're using `isBold` and `isTitle` properties which are coming from
the state.
The `isTitle` flag is used to disable the  bold / unbold button when the
selected text is a title as we always want titles to be bold.
The `isBold` flag is used to whether highlight or not the bold / unbold button
when the selected text is aleady bold or not (this button behavior is similar
to the one you'll find on any wording processor).
When clicked, the button dispatches a `MUTATE` action which would trigger the
selected text mutation.

**How the flags are determined ?**
Such flags are dermined by an epic. The epic is executed when we open the
text toolbox, and also when we apply a text mutation. We simply grab the
status of the selected text using the
[`document.queryCommandState`](https://developer.mozilla.org/en-US/docs/Web/API/Document/queryCommandState)
function. This function returns a boolean indicating whether the given command
has been executed on the selection range, in other world, it indicates if the
selection is of the given type :

```js
const isBold = document.queryCommandState('bold');
```

Now that we know that, it's pretty easy to determine the state of the text
toolbox flags by writing an epic :

```js
// refreshTextToolboxStateEpic :: Observable Action Error -> Observable Action _
export const refreshTextToolboxStateEpic = (action$, state$, { window }) => action$.pipe(
    ofType(SHOW_TEXT_TOOLBOX, MUTATE),
    map(action => [ action, window.getSelection().getRangeAt(0) ]),
    map(([ action, range ]) => [ action.editorName, ({
      isBold: document.queryCommandState('bold'),
      isItalic: document.queryCommandState('italic'),
      isUnderline: document.queryCommandState('underline'),
      isTitle: isInParent('h2')(range),
      isQuote: isInParent('blockquote')(range),
      isLink: isInParent('a')(range),
    })]),
    map(apply(refreshButtonsState)),
    logObservableError(),
  )
```

**How the text is muted ?**
Once again, we're using an epic for that! As described above, the click on a
text toolbox button dispatches an action :
```js
onClick={() => mutate('BOLD')} // will dispatch a `MUTATE` action having the `BOLD` mutation
```
and an epic is listening to the `MUTATE` actions to apply the mutation :

```js
// mutationEpic :: Observable Action Error -> Observable Action _
const mutationEpic = action$ =>
  action$.ofType(MUTATE).pipe(
    map(prop('mutation')),
    filter(complement(equals('LINK'))),
    tap(cond([
      [equals('TITLE'), () => document.execCommand('formatBlock', false, 'h2')],
      [equals('PARAGRAPH'), () => document.execCommand('formatBlock', false, 'p')],
      [equals('ITALIC'), () => document.execCommand('italic')],
      [equals('BOLD'), () => document.execCommand('bold')],
      [equals('UNDERLINE'), () => document.execCommand('underline')],
      [equals('QUOTE'), () => document.execCommand('formatblock', false, 'blockquote')],
      [equals('UNLINK'), () => document.execCommand('unlink')],
    ])),
    ignoreElements(),
  )
```

In this epic, we're using the
[`document.execCommand`](https://developer.mozilla.org/en-US/docs/Web/API/Document/execCommand)
function which is applying the mutation on the selected text.

And voilà ! Muting the text is as simple as that !

## Paragraph toolbox workflow

![paragraph toolbox](/images/text-editor/paragraph-toolbox.png)
*The paragraph toolbox*

The paragraph toolbox is only displayed when the cursor is in an empty
paragraph. If you remember, we are dispatching a `SELECT_TEXT` action when a
text is selected inside the `TextEditor` component :

```jsx
{ /* TextEditor component's view */ }
<article
  onSelect={() => selectText(editorName)}
>
  {children}
</article>
```

This action is also dispatched when moving the cursor, so it's easy to
identify when we're on an empty paragraph to display the paragraph toolbox.
Again, such identification is done in an epic :

```js
// showParagraphToolboxEpic :: Epic -> Observable Action.SHOW_PARAGRAPH_TOOLBOX
export const showParagraphToolboxEpic = (action$, state$, { window }) =>
  action$.pipe(
    ofType(SELECT_TEXT),
    map(action => [ action.editorName, window.getSelection()]),
    filter(compose(isEmptyParagraph, last)),
    map(([ editor, selection]) => [
      editor,
      getParagraphTopPosition(selection),
      getNodeIndex(editor, selection.anchorNode),
    ]),
    map(apply(showParagraphToolbox)),
    logObservableError(),
  )
```

This epic indicates to show the paragraph toolbox for the editor having the
given `editorName`. We also determine the absolute position at which to
show the paragraph toolbox. The node index corresponding to the cursor
is also stored in the redux state to be able to identify where to insert
medias chosen with the paragraph toolbox.

The paragraph toolbox buttons are not as similar to each other than the ones
of the text toolbox. They all behave differently and we won't explain all
the behaviors in this article as it would be too long. However, we'll still
spend some time on one of these buttons : the tweet insertion.

**<TWEET INSERTION EXPLANATION HERE>**

## Clipboard access

Using `contentEditable` tags rather than regular form elements such as `input`
has drawbacks. Say that our end users need to paste text from other sources to
write articles. Doing so in an input element, everything works just great. You
get rid of non textual elements, and end up with a raw string ready to be
submitted. When pasting in contentEditable element, things are way different:
The browser will try to render the pasted text as close as possible from of the
original copied text, resulting in massive and ugly inline style everywhere.
This completely messed up our editor because it violated the sanitization logic
it's based on.

We had to find a way to sanitize (once more !) the user clipboard, to ensure
only raw text is pasted from external sources. Problem is, clipboard interaction
is one of the most obvious security breach in web application, because it
implies a direct communication between javascript and the external world with
almost no way to control what's comming from it. In the past, several workaround
involving extreme methods such as `iframe` and even `flash` (brrrr) were used.
Lately, the well known `document.execCommand('paste')` came to the rescue, but
was still very limitated due to security concerns.

A more recent approach is to use the new clipboard API, which provide an elegant
and secured way to access the user clipboard (as soon as he gaves the app his
permission to do so) and let us work with Promises in return. This feature is
still at experimental state and the API is likely to evolve in the future. We
use it anyway, because it's what fits our user need the best, and we were lucky
enough for the compatibility tables to match his environment.

If you recall, an `onPaste` handler was binded to the article editor. It is
triggered either when user pastes text from the shortcut key combination CTRL+V
or via contextual menu. That handler prevents the browser default behavior and
like every other handlers of this component, dispatches a `PASTE` action which
is observed by an epic:

```js
// checkClipboardAccessEpic :: Epic -> Observable Action.PASTE_GRANTED Action.DISPLAY_CLIPBOARD_WARNING
const checkClipboardAccessEpic = action$ => action$.pipe(
  ofType(PASTE),
  mergeMap(() => navigator.permissions.query({'name': 'clipboard-read'})),
  mergeMap(ifElse(
    isClipboardAccessGranted,
    () => navigator.clipboard.readText().then(pasteGranted),
    () => [displayClipboardWarning()],
  )),
  logObservableErrorAndTriggerAction(displayClipboardSupportError),
)
```

This first epic check for the clipboard access and dispatches an error message
if that permission is not granted. When everything is fine, a `PASTE_GRANTED`
action carrying the pasted text read from the clipboard is dispatched. Another
epic observes that action and will insert the pasted text at the right place.

```js
// pasteCopiedTextEpic :: Epic -> Observable Action.TEXT_PASTED
const pasteCopiedTextEpic = (action$, state$, { window }) =>
  action$.pipe(
    ofType(PASTE_GRANTED),
    map(prop('textToPaste')),
    tap(textToPaste => ifElse(
      isEmptyParagraph,
      pasteTextInParagraph(textToPaste),
      pasteTextInExistingText(textToPaste),
    )(window.getSelection())),
    map(textPasted),
    logObservableError(),
  )
```

Two cases are to be distinguised:
- When pasting in an empty paragraph, things are quite simple. We only replace
the paragraph `innerHTML` by the user text.
- When pasting in an already existing paragraph, it's content has to be updated
so the new text is inserted **between** existing one (if the selection is empty)
or **replace the current selection**. String manipulation is to be done here
that are delegated to an external `pasteTextInExistingText` function to kee the
epic as clear as possible.

Once the text has been pasted, a `TEXT_PASTED` action is dispatched so the
observable stream is resolved :).

## Save the edited content

To save the edited content, we simply grab the HTML string inside the DOM node
of the TextEditor component having the `contentEditable={true}` attribute.
This HTML string is out editeed text, so we cant send it to our backend to save
the edited text.

However, there is an extra step before sending anything to the backend. As
we can insert some third party medias (such as tweets and youtube videos),
the rendered HTML markup of these medias are present in the HTML string
to send. We definitely do not want to save the rendered representation
of these medias, but instead their embed form. For instance, to render tweets,
we're using the embed HTML markup and then we make a call to the twitter SDK
which would render a nice and complete tweet from the embed code.
The rendered forms are always more verbose than the markup forms, so that's
why we prefer to store only the embed forms in our DB.
Aditionaly, the legacy text edition application was already working with
embed code to insert tweets, so we wanted to make this TextEditor
compatible with the contents edited by the legacy app.

This brings us to the `sanitization` step, where we replace any rendered
form of complex inserted media by their embed form. To do so, we have to
keep the embed representation of the media in the application state. We also
have to identify to which rendered form correspond an embed markup (eg for
tweets, we're using the tweet id).
