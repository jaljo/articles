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

Technically speaking, articles are fetched from an HTTP REST API (our backend).
Some of them were coming from a legacy application, and may contain some broken
HTML semantic. To address this problem, we decided to develop a parser for the
received HTML markup to be cleaned. This parser creates React components for,
and only for HTML tags that we allow. Doing so, only tags that we're sure to
support are rendered, which also simplifies design and stylization.

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

Although such features may seems very common, complexity shows up because they
are tighten to our current application (images and videos collections are parts
of the application state).

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
it was the corner stone of the backend application, keeping it at that level of
complexity and abstraction was too risky, time consuming to develop and maintain,
and the learning curve was giving nightmares to newcomers.

We took an other approach to the problem, by using the DOM API to keep the tree
representation and directly manipulate the view in order to easily add/remove
HTML elements.

### Second attempt : manipulating the DOM

So we started digging in the DOM API documentation. We direclty obtain a
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
icing on the cake: conventional shortucts work as well (ctrl+B for bold, etc).

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

We're using [react](https://reactjs.org/) along with
[redux](https://react-redux.js.org/)
and [redux-observable](https://redux-observable.js.org/)

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
`children`.

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
ne used on the page.

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
const didMount = ({ initialize }) => initialize()

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
