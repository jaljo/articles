# How we extended the trello experience with power-ups

[Trello](https://trello.com/) is a collaborative tool we use here at KNP to
split projects in little tasks and share their progression between developers
and clients.

If you've been working in software development for a few years, you're aware
that even before architectural or implementation decisions, defining what your
client really **needs** stays on top of every project's challenges. Let's
imagine you and one of your collaborators finally agreed on a feature definition
(i.e it's scope, what the feature will **do**, not how it will be achieved from
a more technical point of view).

In an agile project's workflow, chances are some of these steps will happen:
1. you create a new ticket in your favorite collaborative tool,
2. you write the feature definition down in this card,
3. you play planning poker with your teammates to give this card an estimation,
4. you work your ass off to get the job done,
5. you're proud of yourself and push the whole stuff in production,
6. because you're a good developper, the production is not down,
7. the feature's definition has changed in the card, and nobody realized it
until now (see step 4),
8. you table flip the entire open space and consider extreme measures, like
relocating to Loir-et-Cher.

This little story brought to light the urgent need for us to log the history of
every changes made against a project's card. That way, we could have the
opportunity to point an accusing "AHA!" finger at the unfortunate one who
surreptitiously changed that card's description.

## From the idea to the POC

We approached the problem with the idea of using a
[web extension](https://developer.mozilla.org/fr/docs/Mozilla/Add-ons/WebExtensions).
They offer a convenient way to enhance the navigation experience and could be
written using traditional browser-side web technologies (HTML, CSS, JS). So we
created a POC during our last winter hackathon in January, to demonstrate that
what we intended to do was possible:
- access a card history,
- render it on the back of that card _with markdown support_,
- randomly insert unicorns in the background.

No black magic behind the scene. We simply used the Trello native
[client](https://developers.trello.com/docs/clientjs) to access their API with a
good old XmlHttpRequest. In order to be called, the API requires explicit
authentication:
1. an auto generated API key,
2. a token that is generated once the end user granted the API access, on behalf
of the API key-identified user.

Although this first version worked quite well, it also had several drawbacks.
Because Trello is an SPA and we decided to use a web extension, we had trouble
dealing with asynchronicity. Indeed, the Trello API only lets us perform actions
on domain-oriented objects (card, boards, members...); that is, we still had no
way to directly interact with the Trello front application, nor could we listen
for events such as card loading success to know **when** we should get and
display its history.

We considered getting around this problem by observing
[DOM Mutations](https://developer.mozilla.org/fr/docs/Web/API/MutationObserver),
and detecting when a specific element entered the DOM, identified by its class
or id. Oh listen, what's that noise ? It sounds like the WTF alarms are ranging
afar ! Relying on HTML attributes to identify elements, which are very likely
to change over time, is a 100% chance for this project to end in the cemetery of
repositories nobody dares to debug. We felt the time for a perspective change
has come.

## An implementation change

Trello offers another possibility to build features upon their application, with
what they so-called [Power ups](https://trello.com/power-ups). You can see them
as external libraries which communicate with third-party services like Google,
Facebook or the one you built with your little fingers. An important limitation
though, these power ups are tied to specific locations opened by Trello, such as
`card-back`, `card-buttons`... These locations are designated by _capabilities_
in their vocabulary.

![](https://github.com/jaljo/articles/blob/master/trello-history/images/pu-setup.png)

**Capabilities are set in the Trello Power-ups administration board. Please note
that only administrators can create and expose power ups to their teams.**

At the bottom of that form, the _iframe connector url_ targets where the
application of the power up is hosted. As you can see, it must be served over
HTTPS, which can be a bit tricky when working in a development environment.

Here comes [Serveo](https://serveo.net/) to the rescue ! In a nutshell, Serveo
lets you expose any local server to the Internet, which was handy for our
testing purposes. So all we had to do was to create a new project from the
[power up skeleton project](https://glitch.com/edit/#!/trello-power-up-skeleton)
graciously offered by Trello and remix it to create our own service. This
skeleton is hosted on (Glitch)[https://glitch.com/], which provides a friendly
ecosystem to collaborate on code and offers instant hosting with automated
deployment.

Here is the file structure of the project:
```
- public
  |_ css
  |_ image
  |_ js
  |_ translations
```

Nothing fancy here. First, we perform an authorization check, then we render the
history of the card if the user is authenticated, or we display a button to ask
authorization to access the API if they're not:

```js
window.TrelloPowerUp.initialize({
  'card-back-section': (t, options) => getTranslations(locale)
    .then(response => response.json())
    .then(trans => t.set('organization', 'shared', 'trans', trans))
    .then(() => t.getRestApi()
      .isAuthorized()
      .then(isAuthorized => isAuthorized
        ? renderHistory(t)
        : askAuthorization(t)
      ),
    )
}, {
  appKey: 'no-you-wont-have-my-key',
  appName: 'KNP Trello History',
})
```

Here are some explanations on this piece of code: the `initialize` method takes
an object as its first parameter, which associates each capability with the
callback it should execute (in this case, as soon as the back of one card is
revealed).

Keep in mind that power up capabilities are rendered in iframes. Yes. Please
don't leave. The `renderHistory` and `askAuthorization` at the end of the
promise flow are simple functions that both return descriptive objects of how
these iframes should be displayed. The important thing to understand is here:
Each iframe targets a specific HTML page which runs its own JS script. That is,
how can we possibly **share data** between those scripts ?

See that `t` argument ? It's the power up toolbox, exposing helpful methods to
interact with the context of the Trello board. For example, the `t.set()` method
is to be used to store scalar values in the context (translations, for example,
are a perfect use case for it). Once stored, a `t.get()` method can be called
from any other script to get our translations back. Piece of cake !

This can be misleading, though. Tricked by our daily habits, we spent some time
trying to over decouple things (i.e get the history data from the API in the
main power up script above, store it in the context and only use other scripts
for templating). The Trello buffer has a limit size of 4096 characters, so no
need to say we didn't have any chance to successfully store our stringified
history there ! We wandered towards locale storage and string compression to
bypass that limitation but finally figured out this is not how it was supposed
to be done.

One script per iframe / page to be shown, simple as that. Let's have a quick
look on the history script:

```js
t.render(() => t.get('organization', 'shared', 'translations')
  .then(renderHistory(t))
)

// renderHistory :: Object -> String -> _
const renderHistory = t => translations => t.getRestApi().getToken()
  .then(R.pipeP(
    getCardHistory(t.getRestApi().appKey, t.getContext().card),
    response => response.json(),
    R.tap(() => document.getElementById('history').innerHTML = ''),
    R.ifElse(
      R.compose(R.gt(2), R.length),
      R.tap(() => noHistory(translations)),
      // drop the first element as it is exactly the same as the description
      R.compose(R.map(renderCard), R.drop(1)),
    ),
  ))
  .catch(error => openAuthorizeIframe(t))
```

The `t.render()` method ensures the history section is refreshed each time the
user modifies the description of a card (so the previous description will be
added to the history).

Here you can see how we get the translations from the context. We're also using
[ramda](https://ramdajs.com/docs/) to get full benefit of promise composition
and fetch the card history from Trello. The `renderCard`  function does nothing
more than transforming a `card` object returned from the API in a renderable
HTML string, and that's all !

![](https://raw.githubusercontent.com/jaljo/articles/master/trello-history/images/history.png)

## Going further

Of course, accessing a card's history could have been achieved in many other
ways: by connecting Trello to Slack for example. That said, we liked the idea
of not depending of another service and having an "out of the box" solution
that would work anywhere.

In the end, it was quite a pleasant journey to discover the Trello API and how
to build power ups with it. Although their documentation is pretty sized,
information is sometimes scattered, tedious to find and lacks of complete
examples. Power ups still are a new thing in the Trello landscape (< 100 power
ups were released as of late 2018), and we learned it the hard way when we faced
problems with the authentication process.

Speaking of which, please remember that once the auth token is obtained, Trello
will save it **server-side**. So if you're facing authentication problems or
weird behavior, cleaning cookies or the cache browser side won't be of any help.
We almost went nuts because of this, until we found out there was a special
button made to get rid of that token:

![](https://raw.githubusercontent.com/jaljo/articles/master/trello-history/images/reset-token.png)

The final version of the Power-up was done during our last summer hackathon,
and we are pretty proud of it. We didn't had the time to go for a better, more
decoupled architecture but it may be worth giving a try on an FRP implementation
with React and Redux. We're also considering to releasing this power up to the
Trello marketplace.

If you want to use it for your own Trello boards, you can either:
- host the [Docker image of the project](https://cloud.docker.com/u/knplabs/repository/docker/knplabs/trello-history-powerup)
- directly use [our hosted service](https://trello-history-powerup.s3.eu-west-3.amazonaws.com/index.html)

Feel free to browse the complete codebase on
[GitHub](https://github.com/KnpLabs/trello-history-powerup) !
