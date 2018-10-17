### Deal with form submission from multiple events & animation
##### React / Redux

« The worst mistake a mathematician can make when struggling
with a problem is to look for a solution into a book. »
Although quite provocative, those words by french
mathematician Alain Connes sounds relevant when handling
development problems, just replace « book » by « internet ».
What’s the underneath idea ? The best way to learn something
is to be stucked, search, be stucked again and finally ask
for help. This is why this article try to emphasis on
conception rather than on code (don’t wory, code comes
afterwards !).

We’ve recently encountered a simple but interesting problem on
a client project at KNP, which mainly consist in an
administration interface for a news & current affairs website.
Nothing fancy at first sight, editors need to write and edit
all kind of contents (mostly articles) and their metadatas,
for search engine optimization concerns.  A picture beeing
worth a thousand word, here is a little cinematic of what the
UI/UX looks like for this feature :

![form events 1](https://raw.githubusercontent.com/jaljo/articles/master/images/fe_1.png")

![form events 2](https://raw.githubusercontent.com/jaljo/articles/master/images/fe_2.png")

![form events 3](https://raw.githubusercontent.com/jaljo/articles/master/images/fe_3.png")

![form events 4](https://raw.githubusercontent.com/jaljo/articles/master/images/fe_4.png")

We ended up with two panels beeing part of the content editor
interface, where editors can edit metadatas. As you can see in
the third screen, each panel can be closed from multiple
events :
* by clicking on the same button that triggered the panel
  opening,
* by clicking on the save button inside the opened panel,
* by opening an other panel (it makes sense that there can
  only be one panel opened at a time).

Therefore, we have to submit the form data of the panel we’re
closing, regardless from where we’re closing it. Luckily
enough, we’re using React along with Redux so it should’nt be
such a big deal to solve this. A traditionnal flow would be
something like :
* all three events dispatch a generic CLOSE action,
* this action reduces the state of the panel, say it sets a
  boolean to false,
* this boolean is used in a React component to mount or unmount
  the panel,
* using lifecycle hooks, we can dispatch a SUBMIT action when
  the component is about to unmount. This action gather and
  carries the form data in its payload, so we can do whatever we
  want with it later in our program.
