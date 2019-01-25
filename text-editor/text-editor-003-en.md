## Insertion of a tweet : a case study

In this chapter, we will try to give a better insight of how the whole thing is
working, by breaking down the insertion of a tweet case and the components
involved in it's workflow.

### Getting the HTML markup from twitter

Keep in mind that our main goal here is to obtain the simplest possible HTML
representation of our tweet, so it can lately be saved to the API. That way it
will be easy for us to parse it as a component, and finally display it to the
end user in the front end application.

History begins as soon as an user submits the little tweet insertion form.
Thanks to `redux` action creators, the url of the tweet he wants to insert is
boxed along with the `INSERT_TWEET` action, which is observed by a first epic.

The HTML markup of a tweet can be obtained by using the twitter API,
(see here)[https://developer.twitter.com/en/docs/twitter-for-websites/embedded-tweets/overview].
Because of Cross Origin restriction imposed by twitter API, we cannot directly
call that API from a browser application. To solve this issue, we created a
dedicated endpoint in our API (which has been mimiced in the `server` folder of
the POC project).

So the first epic is only responsible to call that endpoint and dispatch a
`EMBED_TWEET_FETCHED` action that carries out the obtained HTML markup in its
payload. Its always convenient to isolate API calls in dedicated epics so errors
can be properly catched and displayed to the end user when something goes wrong.
Cool thing is, keeping epics as atomic as possible (observe an action, do one
thing, then dispatch another action) allow them to be easily tested. That is,
your fellow developpers dont want to break your legs when the prod crashes
(which will happend, but that's another story).

### Inserting the tweet into the DOM

That last action is observed by another epic which will perform the actual tweet
insertion into the DOM (as epic are meant to deal with side effects, it's the
proper place in the project to do that). How is that ?

(See gist here)[https://gist.github.com/jaljo/4c8f83acc48766b930d7d584fe9ed22b]

As you can see, the `insertTweetNode` function returns a Promise, so it provide
us a nice way to handle failures at that step of the process. What does that
function do ?

- Using a combination of react `renderToString` utility, one of our parsers, and
the HTML markup provided by the twitter API in the previous observable, it
creates an HTML representation of the tweet **formatted as we need it to be.**
- Then, using an `insertNewNodeAtIndex` utility, we replace the empty paragraph
from where the tweet insertion form was shown by the new node created at the
previous step (that paragraph index has early been saved into the paragraph
toolbox state when the `SHOW_PARAGRAPH_TOOLBOX` action has been reduced).
- The promise is finally resolved with data such as the editor name, because we
need to carry these information out during the whole process.

Once the tweet has successfully been inserted, a `TWEET_INSERTED` action is
dispatched  by the epic. Otherwise, an error is catched and displayed to the
user. Notice that the kind of error we handle here is not of the same type than
in the previous observable. Here errors that may occur could only be DOM related.

### Rendering the inserted tweet and going back to the original state

The `TWEET_INSERTED` action is observed by four epics (and here we gain full
benefit of working with observable streams) :
- One that takes care of systematically insert and focus a new paragraph after a
media has been inserted (for pure UX concerns),
- One that will hide the paragraph toolbox (we no longer need it to be displayed
as a consequence of the previous action: a new paragraph has been inserted),
- One that will close the tweet insertion form so that when the paragraph toolbox
is lately shown the user is not annoyed,
- The last wich only maps the payload of the `TWEET_INSERTED` action payload to
a new `RENDER_TWEET` one. Although it can sound a bit overkill to have an epic
that does apparently 'nothing', it help us keep our action stream crystal clear.

Guess what ? That action is observed too !

(See gist here)[https://gist.github.com/jaljo/006cab23716b9076dfcd5b2032d03081]

That epic check for the twitter SDK presence in the window object using the
`filter` operator. Noticed how we pass the window object as a dependency of
the Epic ? This life saving feature is given by rxjs 6 and help us to keep our
epics as pure as possible. By letting us the ability to mock the window object
(or our fetchApi, which alos comes as a dependency), testing is way simpler.
