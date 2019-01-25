# Rich Text Editor

## Context

We're working for an information company. They publish many contents on their
website, most of the time represented as articles or as a collection of
articles (they call it `topics`). As the company is working in the information
business, they also post some `news`. News are small textes, which may contain
a tweet or a link to any article.

All these kinds of content are requiring a tool to be able to create and edit
them. These contents are saved in our database as an HTML string.

There alreay has many text edition tools online, but we wanted a tool which
looks close to the modern text editor of medium.com . Some open source
libraries are offering such solution, but as most of the time when you have
specific needs, these libraries they do not fulfill all your requirements,
so we'd have to tweak them or to develop additional features on top of them.
Instead of spending time on adapting an existing library, we decided to
implement our own tool as we could make it behave exactly like we want.

It is also important to say that our text editor development took place
when we already built the public website in charge of displaying the contents.
The contents are fetched from an HTTP REST API (our backend).
As some of these contnts are coming from a legacy application, they may have
some broken HTML semantic, so we decided to develop a parser for the received
HTML markup of the content. The parser will then create React components
for each tag that we know. Doing so, we only render tags that we're sure
to support, and we can also easily stylize them.

We wanted to keep the same parsing logic for our text editor as we wanted it
to be WYSIWYG, ie display the edited content as it would be displayed in the
public website, and also be able to edit the content.

## The specs

The text editor sould provide the basic fatures of text edition, ie be able to :
- apply some style to a portion of text (bold, italic, underlined)
- create an HTTP link
- create some paragraphs
- create titles

There are also some media features :
- be able to insert an image (upload it or pick it from the image collection
of the company)
- be able to insert a video (from the video collection of the company)
- be able to insert a tweet (by specifying the tweet URL)
- be able to insert a Youtube video (by specifying the video URL)

Such features may seems very common, but they are however tighten to our
current application as images and videos are dealing with our image and videos
collection.

This is why such requirements pushed us to develop our own text editor from
scratch instead of using an existing library.

## Text edition POC

### First attempt : AST

We first wanted to modelize the text to edit using an AST (abstract syntax
tree). The HTML string was parsed and represented as a tree, in the same way
the DOM nodes are represented.
We started to write some simple mutation functions to stylize the text :
make it bold, underlined, etc... We focused only on the AST part fisrt, the
rfendering would come later. Unit tests would ensure that the mutations were
working as expected.
We managed to update the AST correctly when applying mutations,
eg the following text node

```js
{
    type: "text",
    text: "lorem ipsum",
    children: [],
}
```

was succesfully embed into an other node when we wanted to make it bold :

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

However, things were getting way more complicated when we wanted to update
some selected text that were starting in a paragraph and ending on an other,
or also when the selection was overlapsing on different tags, such as

```
<b>Hello <em>John Doe</em></b>, how are you ?
    |---------------|
selection start   selection end
```

We realized that applying such changes on the AST would have beed difficult
for us. In the mean time, the developer who started to work on the AST was
required to work on an other project so we decided not to use this AST anymore
as it would have been very time consuming to develop and maintain (and also
the learning surve was not easy for newcomers)

We took an other approach to the problem, by keeping the tree representation,
but this time by using directly the DOM API to manipulate the view in order
to easily add/remove HTML elements.

### Second attempt : manipulating the DOM

We took a new start by studying the DOM api. We were able to direclty have a
rendered view (what we haven't so far with the AST) and to edit it. The idea
behind this approach was to use the DOM API instead of reimplementing such
node parsing and manipulation ourselves.
The drawback of this approach is that we're working directly on the view, so
we do not have a model representation anymore that we could render. The view
was our model.

Once more, the MDN was our friend for this study :
- the [DOM Node API](https://developer.mozilla.org/en-US/docs/Web/API/Node)
permitted us to navigate through the node tree
- we were able to [get the selected text](https://developer.mozilla.org/en-US/docs/Web/API/Selection)
(returns a [range](https://developer.mozilla.org/en-US/docs/Web/API/Range))
- we were simply creating elements to insert with
[`document.createElement`](https://developer.mozilla.org/en-US/docs/Web/API/Document/createElement)

We had to be very careful with the DOM manipulation as some Node methods are
directly mutating the node itself. We should update the node only we have
finished to compute the changes to apply. During this computation, we had
to make copies of some node so we could edit them without chabge the actual
view.

We managed to have a working system for text mutations (bold, italic) across
a single or multiple paragraphs, even when the selection was overlapsing
multiple HTML tags (see the above example).

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
This sanitization was done in a pass once the new `b` tag was added to the DOM.

In the end, this was working fine but the number of lines of code started to
grow, and we were discovering more and more edge cases which were painful to
cover.

Finally, we discovered a method which would have saved us all.

### The holy grail : document.execCommand

And then we discovered
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
by the browser :p).
Checkout [all the available commands](https://developer.mozilla.org/en-US/docs/Web/API/Document/execCommand#Commands)
to see what's possible to do !
Needless to say that when we discovered this, we immediately stopped the
previous attempt which was about manipulating the DOM by ourselves, and
it was definitely time for a beer of victory !

Considering the commands list, we had all the operators we needed to build
our text editor. We only had to execute the right command when clicking on
the right button.

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
clicking on the `image` button, it will open a media picker. We can then
search the image to include from our database, or upload a new image.

The `video` button behaves almost the same way than the `image` button :
it will open the media picker for videos.

For the tweet and youtube video insertion, a `text` input is displayed
on the empty paragraph when such buttons are clicked to enter the URL of the
tweet or the youtube video to insert. When submitting the form, the tweet
or the video is inserted.

### Text toolbox

The text toolbox allows some text modifications :
- turn the curent paragraph to a title
- enable / disable italic
- bold / unbold
- enable / disable underline
- insert a link
- quote the selection

## Implementation

We're using [react](https://reactjs.org/) with
[redux](https://react-redux.js.org/)
and [redux-observable](https://redux-observable.js.org/)

In the following of this article, we assume that you're comfortable with the
above technologies. If not, please take some time to understand the key
principles of these libraries.

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
