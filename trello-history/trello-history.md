# How we extended the trello experience with power-ups

[Trello](https://trello.com/) is a collaborative tool we use here at KNP to
split projects in little tasks and share their progression between developers
and clients.

If you're working for a few years in software development, you're aware that
even before architectural or implementation decisions, to define what does your
client really **needs** stays on top of every project challenges. Let's imagine
you and one of your collaborator (1) finally agreed on a feature definition (i.e
it's scope, what the feature will **do**, not how it will be achieved from a
more technical point of view).

In an agile project's workflow, there is chances some of these steps will
happend:
1. You create a new ticket in your favorite collaborative tool,
2. you write the feature definition down in this card,
3. you play poker with your teammates to give this card an estimation,
4. you work the hell out of your sweat and blood to get the job done,
5. you're proud of yourself and push the whole stuff in production,
6. because you're a good developper, the production is not down and you can
notify your client,
7. he says it's not what he needed in the end because he surreptitiously changes
the feature's definition in the card, and nobody realized (see step 4),
8. you table flip the entire open space and think of extreme measures, like
expatriating yourself in the Loir-Et-Cher.

This little story brought to light the emergency for us to keep the history of
every changes made against a project's card. That way, we could have the chance
to point an accusing "AHA!" finger on the client, in case the previous scenario
occures once more.

## From the idea to the POC

We first approached the problem with the idea to use a
[web extension](https://developer.mozilla.org/fr/docs/Mozilla/Add-ons/WebExtensions).
They offer a convenient way to enhance the navigation experience and could be
written using traditionnal browser side web technologies (HTML, CSS, JS). So we
created a POC during our late winter hackathon in January, to demonstrate what
we intended to do was possible:
- Access a card history,
- render it on the back of that card _with markdown support_,
- randomly insert unicorns in the background.

No black magic behind the scene. We simply used the Trello native
[client](https://developers.trello.com/docs/clientjs) to access their API with a
good old XmlHttpRequest. To be called, the API requires explicit authentication:
1. An auto generated API key,
2. a token that is generated once the end user granted the API access, on behalf
of the user identified by this API key.

Although it quite worked well, it also had several drawbacks. Because Trello is
an SPA and we decided to use a web extension, we had trouble dealing with
asynchronicity. Indeed, the Trello API only let us perform actions on domain
oriented objects (card, boards, members...). That is, we still had no way to
directly interact with the Trello front application, nor to listen for events
such as card loading success to know **when** we should get and display it's
history.

We considered to get arround this problem by observing
[DOM Mutations](https://developer.mozilla.org/fr/docs/Web/API/MutationObserver),
and detect when a specific element entered the DOM, identified by it's class or
id. Oh listen, what's that sound ? It looks like the WTF alarms are ranging
afar ! Relying on HTML attributes to identify elements, which are very likely
to change over time, is a 100% chance for this project to end in the cemetery of
repositories nobody dares to debug. We felt the time for a perspective change
has come.

## An implementation change

Trello offers another possibility to build features upon their application, with
what they so called [Power ups](https://trello.com/power-ups). See them as
external libraries which communicates with third services like Google, Facebook
or the one you built with your little fingers. An important limitation though,
these power ups are tied to specific locations opened by Trello, such as
`card-back`, `card-buttons`... These locations are designated by  _capabilities_
in the their vocabulary.

![](https://github.com/jaljo/articles/raw/710f8e2d3561a5fa65bc6422b99aad88695871f3/trello-history/images/pu-setup.png)

**Capabilities are set in the Trello Power-ups administration board. Notice only
administrators can create and expose power ups to their teams.**

On bottom of that form, the _iframe connector url_ targets where the application
of the power up is hosted. As you can see, it must be served over HTTPS, which
can be a bit tricky when working in development environment.

Here comes [Glitch](https://glitch.com/) to the rescue ! In a nutshell, Glitch
provides a friendly ecosystem to collaborate on code, is easily linkable to
github and offers instant hosting and automated deployment, which was handy for
testing purpose. So all we have to do was to create a new project from the
[power up skeleton project](https://glitch.com/edit/#!/trello-power-up-skeleton)
gracefully proposed by Trello and remix it to create our own service.

Here is file structure of the project:
```
- views
- public
  |_ css
  |_ image
  |_ js
  |_ translations
- server.js
```

Nothing fancy here. The entry point of the application is served by an express
instance. From here, we perform an authorization check, then render the history
of the card if the user is authenticated, or a button to ask authorization if
he's not:

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

Some explanations on this piece of code: The `initialize` method takes an object
as first parameter, which associates each capability with the callback it should
execute (in this case, as soon as the back of a card is revealed).

Keep in mind that power up capabilities are rendered in iframes. Yes. Please
don't leave. The `renderHistory` and `askAuthorization` at the end of the
promise flow are simple functions that both returns descriptive objects of how
these iframes should be displayed. The important thing to understand is here:
Each iframe targets a specific html page whitch runs its own js script. That is,
how can we possibily **share data** between those scripts ?

See that `t` argument ? It's the power up toolbox, exposing helpful methods to
interact with the context of the Trello board. For example, the `t.set()` method
is to be used to store scalar values in the context (translations, for example,
are a perfect use case for it). Once stored, a `t.get()` method can be called
from any other script to get our translations back. Piece of cake !

This can be missleading, though. Tricked by our daily habits, we spent some time
trying to over decouple things (i.e get the history data from the API in the
main power up script above, store it in the context and only use other scripts
for templating). The Trello buffer has a limit size of 4096 characters, so no
need to say we didn't have any chance to successfullly store our stringified
history there ! We wandered towards locale storage and string compression to
bypass that limitation but finally figured out this is not how it's was supposed
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

![](https://github.com/jaljo/articles/raw/710f8e2d3561a5fa65bc6422b99aad88695871f3/trello-history/images/history.png)

## Going further

A react impl ?
Dockerize the project and push it to digital ocean
Release this power up to trello marketplace

!connection with slack

(1) This magic word includes clients, teammates, and even agile coaches !
