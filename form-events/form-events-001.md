### Deal with form submission from multiple events & animation
##### React / Redux

« The worst mistake a mathematician can make, when struggling with a problem, is to look for a solution into a book. » Although quite provocative, those words by French mathematician Alain Connes sound relevant when handling development problems, just replace « book » by « internet ». What’s the underneath idea ? The best way to learn something is to be stuck, search, be stuck again and finally ask for help. This is why this article tries to focus on conception rather than on code (don’t worry, code comes afterwards!).

My teammate and I have recently encountered a simple but interesting problem on a client project here at KNP, which mainly consists in an administration interface for a news & current affairs website. Nothing fancy at first sight, editors need to write and edit all kinds of contents (mostly articles) and their metadata, for search engine optimization concerns. A picture beeing worth a thousand word, here is a little cinematic of what the UI/UX looks like for this feature:

![form events 1](https://raw.githubusercontent.com/jaljo/articles/master/images/fe_1.png)

![form events 2](https://raw.githubusercontent.com/jaljo/articles/master/images/fe_2.png)

![form events 3](https://raw.githubusercontent.com/jaljo/articles/master/images/fe_3.png)

![form events 4](https://raw.githubusercontent.com/jaljo/articles/master/images/fe_4.png)

We ended up with two panels being part of the content editor interface, where editors can edit metadata. As you can see in the third screen, each panel can be closed from multiple events :
* by clicking on the same button that triggered the panel opening,
* by clicking on the save button inside the open panel,
* by opening another panel (it makes sense that there can only be one panel opened at a time).

Therefore, we have to submit the form data of the panel we’re closing, regardless _where we’re closing it from_. Luckily enough, we’re using React along with Redux, so it shouldn't be such a big deal to solve this. A traditionnal flow would be something like :
* all three events dispatch a generic CLOSE action,
* this action reduces the state of the panel, say it sets a boolean to false,
* this boolean is used in a React component to mount or unmount the panel,
* using lifecycle hooks, we can dispatch a SUBMIT action when the component is about to unmount. This action gathers and carries the form data in its payload, so we can do whatever we want with it later in our program.

![form events 4](https://raw.githubusercontent.com/jaljo/articles/master/images/sigourney_small.jpg)

Sounds great, doesn't it? Sigourney Weaver approves.

What we have forgotten is, our lovely designer friend thought of beautiful animations for the panel to play whenever it’s opening or closing, and bam ! The above flow no longer works. Because we need to unmount the panel component to submit the form data, it will not always been present in the DOM, and so playing animation with it becomes much more complicated.

The key idea here is _the time_. We have to think of intermediate actions that will let us identify the moment when the panel is opening or closing, so we can delay the moment it will be unmounted.

Schematically, here is the flow we’re aiming to achieve :
* a user clicks on an element to open the panel. It dispatches an OPEN action,
* based on its state, the panel can be mounted in the DOM,
* a lifecycle hook dispatches a MOUNTED action,
* the state is updated, we can dynamically append a CSS class to the panel view component, the sliding animation can be played.

The same logic applies when closing the panel, with a subtlety however:
* a user clicks on one of the three elements that fire the CLOSE action,
* the state updates, the CSS class is removed from the view, the panel gently returns to its initial position,
* _here is the magic trick_ : we need to dispatch a CLOSED action, so the panel can be unmounted, but _after_ the animation is complete. This is where epics come into play ! The CLOSE action is observed by a little epic, which _delays_ this action, just the time we need to play the closing animation. Then, it finally dispatches the CLOSED action,
* from there, we can take back our initial flow : the state is updated and the panel (which is now hidden, thanks to epics) is unmounted,
* a lifecycle hook triggers the SUBMIT action when the panel is unmounted.

And voila ! Just to make things clearer, below are some examples of what such implementation may look like (the complete POC can be seen [here](https://github.com/jaljo/form-events)). Here is an extract of the container component of a panel, with its lifecycle hooks :

```
// /src/Component/Container/MetaPanel.js

// mapStateToProps :: State -> Props
const mapStateToProps = state => ({
  isOpening: state.MetaPanel.isOpening,
})

// mapDispatchToProps :: (State, Action *) -> Props
const mapDispatchToProps = dispatch => ({
  mounted: compose(dispatch, mounted),
  close: pipe(
    // prevent form submission default behavior, which reloads the page
    tap(e => e.preventDefault()),
    compose(dispatch, close)
  ),
  submitForm: pipe(
    () => document.querySelector('.meta-panel form'),
    getFormData,
    submitForm,
    dispatch,
  ),
})

// getFormData :: Element -> Object
const getFormData = formElement => ({
  slug: formElement.slug.value,
  shortTitle: formElement.shortTitle.value,
})

// didMount :: Props -> Action.MOUNTED
const didMount = ({ mounted }) => mounted()

// willUnmount :: Props -> Action.SUBMIT_FORM
const willUnmount = ({ submitForm }) => submitForm()

// MetaPanelLifecycle :: React.Component -> React.Component
const MetaPanelLifecycle = compose(
  componentDidMount(didMount),
  componentWillUnmount(willUnmount),
)(MetaPanel)

// MetaPanel :: Props -> React.Component
export default connect(
  mapStateToProps,
  mapDispatchToProps,
)(MetaPanelLifecycle)
```

Here is the panel view component. Just notice how we use the isOpening prop to set the opening CSS class in the section tag.

```
// /src/Component/View/MetaPanel.js

// MetaPanel :: Props -> React.Component
export default ({
  close,
  isOpening,
}) =>
  <section className={`panel meta-panel ${isOpening ? 'opening' : ''}`}>
    <form onSubmit={close}>
      <div className="field">
        <label>Slug</label>
        <input type="text" name="slug"/>
      </div>
      <div className="field">
        <label>Short title</label>
        <input type="text" name="shortTitle"/>
      </div>
      <button className="close">Save</button>
    </form>
  </section>
```

The reducer takes care of updating the panel state. We’re using two distinct booleans : one to determine whether the component should be mounted or not, the other to identify if we are playing the animation.

```
// /src/Redux/State/MetaPanel.js

// MetaPanel :: (State, Action *) -> State
export default createReducer(INITIAL_STATE, {
  [OPEN]: state => ({
    ...state,
    isOpened: true,
  }),
  [MOUNTED]: state => ({
    ...state,
    isOpening: true,
  }),
  [CLOSE]: state => ({
    ...state,
    isOpening: false,
  }),
  [CLOSED]: state => ({
    ...state,
    isOpened: false,
  }),
})
```

And the icing on the cake, a little epic that delays our CLOSED action :

```
// /src/Epic/MetaPanel.js

// delayMetaPanelClosingEpic :: Observable Action Error -> Observable Action _
const delayMetaPanelClosingEpic = action$ => action$.pipe(
  ofType(CLOSE),
  // wait for css transition to complete
  delay(200),
  map(closed),
)
```
