# Editeur de texte
## React / Redux / RxJS

Proposition de plan

### Le besoin

Poser un peu le contexte, peut être parle indirectement (sans les citer) d'i24.
Nos utilisateurs finaux sont éditeurs, donc dans un souci d'ergonomie on a cherché
à réduire le plus possible la différence de rendu / présentation entre le
moment ou le texte est édité et lorsqu'il est publié sous sa forme définitive.

Le besoin de nos utilisateurs
* Manipulations standards de texte : gras, italic, souligné, insérer un lien,
transformer en titre ou en citation.
* Insérer une image, un tweet ou une vidéo.
* Une référence : Medium

### Décision technique

Pourquoi faire nous même alors qu'il existe plein d'éditeurs WYSIWYG ?

### Prototypage et recherche

* Deux problématiques principales :
    * Appliquer une ou plusieurs mutations a du texte selectionné.
    * Déterminer le / les styles appliqués sur le texte selectionné.

* AST: le contenu HTML provenant du BO legacy n'etant pas fiable (i.e. balises mal
  fermées),  necessité de sécuriser ce contenu avant de l'afficher pour pallier
  à la fois a des problemes de sécurité et de rendu.
    * Transformation de la chaîne HTML en arbre de composants maîtrisés (paragraphes,
  liens, images, etc...).
    * Utiliser cet AST non plus uniquement en lecture, mais aussi en écriture.
  De cette anière on assurait une parfaite synchronisation entre l'etat (redux)
  de l'application et ce qui etait affiché, en s'affranchissant totalement du DOM.
    * Difficulté à couvrir tous les cas d'usage, qui peuvent être complexes (i.e.
  lien a cheval entre un élément en gras et un autre en italique, mettre en gras
  une selection qui court sur deux paragraphes...)
    * Besoin de développer des fonctions complexes avec des problématiques de
  récursion pour pouvoir parcourir et opérer des changements sur l'arbre pour
  refléter les changements de style (i.e. remplacer un enfant par un autre,
  retrouver le plus haut parent commun de deux enfants dans la hierarchie...)
    * Toutes ces opérations existent déjà en natif dans l'API DOM de javascript.

* Tentative de manipuler directement le DOM grâce à l'API native de javascript.
    * Idée de s'abstraire totalement des libraires tierces de notre projet et
  d'open sourcer l'éditeur une fois terminé.
    * Séparation du travail en deux : Nico sur la partie manipulation de texte,
  Joris sur l'UI, l'affichage des différentes boites à outils, etc.
    * Avantages: API très complète qui permet de manipuler simplement une
  hiérarchie de noeuds et d'éléments, on est assez vite arrivés à un résultat.
    * Les problèmes sont revenus quand il a fallu gérer des cas particuliers,
  surtout les chevauchements de style. Idem pour déterminer l'etat du texte,
  selectionné. On est retombé dans les mêmes problématiques qu'en utilisant
  l'AST.

* En cherchant un moyen de savoir si du texte sélectionné était dans un état
donné, découverte de deux autres fonctions natives javascript:
  * `execCommand` pour effectuer des manipualations standards directement sur
  le texte sélectionné (mettre en gras, transformer en citation...)
  * `queryCommandState` qui permet de déterminer si la sélection est dans un état
  donné.
  * Très grande compatibilité inter navigateurs.
  * Décision de revenir à une application redux / react / rxjs.
  * Limite principale : plus de découpage entre l'état de l'application et sa
  projection qui ne font plus qu'un.

### Architecture
#### Initialisation

La chaîne HTML reçue de l'API est parsée et sécurisée, seules les composants
que nous maîtrisons sont rendus : création d'un ensemble de parsers qui à une
balise HTML associent un composant React.

#### Affichage des boites a outils

Deux boites a outils permettent d'agir sur le texte :
* Une pour les changements de style, elle doit s'afficher lorsque du texte est
sélectionné, que ce soit avec la souris ou bien avec le clavier (shift + fleches).
* L'autre pour les insertions (image, tweet, etc), elle doit s'afficher lorsque
l'on sélectionne un paragraphe vide, ou bien lorsque l'on clique sur un élément
préalablement inséré (tout doit être remplaçable).
* Ces composants doivent être masqués lorsque aucun texte n'est sélectionné, ou
que l'editeur n'est pas utilisé.

Pour gérer tous ces événements, nous avons fait le choix de travailler avec des
actions très simples et de travailler directements dans des epics avec RxJS :
* Les événements du clavier sont déclenchés par deux SyntheticEvent de React,
`onKeyUp` et `onKeyDown` qui dispatchent deux actions Redux éponymes. Des
Observables sont ensuite créés à partir de ces actions.
* Pour gérer les événements de la souris, nous créons simplement des Observables à
partir de l'evenement `mouseUp` de l'objet `window`.

Le gros du travail derrière a consisté à filtrer ces événements en fonction
des différents comportements que nous souhaitions et pour éviter les interractions
avec d'autres parties de l'application. Deux interfaces javascript nous ont été
particulièrement utiles pour cela :
* `Selection`, a partir duquel il est aisé de déterminer quelle partie du texte
est sélectionnée, avec son noeud de départ et de destination.
* `Range` obtenu depuis un objet `Selection` et avec lequel il est possible de
manipuler, voir de créer artificiellement une sélection de toute pièce.

#### Analyse et détermination d'état de la sélection

Quand du texte est sélectionné, ou qu'un changement de style à lieu, la selection
est analysée avec `document.queryCommandState`. Cette fonction s'appuie sur le
style CSS appliqué à l'élément sélectionné pour déterminer son état.

Plus de détail sur la documentation officielle de [MDN](https://developer.mozilla.org/en-US/docs/Web/API/Document/queryCommandState)

Néanmoins, il est a noter que cette fonction ne permet pas de couvrir tous les
cas. Déterminer si le texte est un titre ne peut se faire que par analyse des
ancêtres de l'élément de départ et de fin de la sélection courante. Ces éléments
nous sont fournis par l'interface `Range`.

[Range](https://developer.mozilla.org/en-US/docs/Web/API/Range)

Une fois l'analyse terminée, une action `REFRESH_BUTTONS_STATE` est dispatchée et
met à jour l'état de tous les boutons de la boite à outils lorsqu'elle atteint
son reducer. C'est également cette étape qui permet de déterminer le _sens_ de
certaines mutations (par exemple un titre doit pouvoir être converti de nouveau
en texte simple et vice versa).

#### Mutation de texte

Une action générique `MUTATE` est dispatchée lorsqu'un utilisateur clique sur un
des bouttons d'action de la boite à outil. Cette action porte le type de
mutation à appliquer, ainsi que des options facultatives. On en a par exemple
besoin pour gérer le cas particulier de la transformation de la sélection en
lien (l'utilisateur doit renseigner l'url du lien).

Cette action n'a aucune conséquence sur le state de l'application, elle est
interceptée par une epic qui se charge d'exécuter la bonne commande en fonction
du type de mutation demandée, par exemple :

`document.execCommand('italic')`

Plus de détail sur la documentation officielle de [MDN](https://developer.mozilla.org/en-US/docs/Web/API/Document/execCommand)

#### Insertion d'éléments

Les insertions sont gérées par une autre boite à outils dédiée. A chaque fois
que l'utilisateur change de paragraphe, on enregistre l'index de ce paragraphe
dans le state de l'application. Ainsi, l'utilisateur peut ensuite interagir avec
l'interface sans risque de perdre l'information de _l'endroit_ ou devra finalement
être inséré l'élément qu'il aura choisi. En effet, dans le cas de l'insertion
d'une image par exemple, nous ne maitrisons pas tous les scénarios par lesquels
l'utilisateur peut passer pour aboutir finalement à son insertion dans le texte.
Dans notre cas, nous laissons à l'utilisateur la possibilité d'uploader de
nouvelles images, de les redimentionner et ce sans ordre pré-établi.

Une fois l'image choisie nous pouvons faire appel a un de nos composants générique
pour construire la structure que nous avons besoin d'insérer. Nous utilisons pour
cela la fonction utilitaire de React `renderToString` en combinaison avec le
`DOMParser` natif de javascript, qui transforme une chaîne HTML en hiérarchie
d'objets de type `Element`, qui peuvent ensuite être rendus par le DOM.

#### Insertion d'éléments : cas du tweet (ou youtube / facebook...)

@TODO

### Ouverture / conclusion

Même si on manipule directement le DOM, rien n'empêche d'enregistrer la nouvelle
chaîne HTML obtenue après manipulation, dans le state et de refaire une nouvelle
projection à partir de celle ci. De cette façon on retrouve la synchronisation
entre React / Redux et on limite les effets de bords ou la perte de données (en
cas de changement de route dans le contexte d'une PWA par exemple).
