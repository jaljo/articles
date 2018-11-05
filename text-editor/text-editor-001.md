### Editeur de texte
#### React / Redux / RxJS

Proposition de plan

##### Le besoin

Poser un peu le contexte, peut être parle indirectement d'i24.
Nos utilisateurs finaux sont éditeurs, donc dans un souci d'ergonomie on a cherché
à réduire le plus possible la différence de rendu / présentation entre le
moment ou le texte est édité et lorsqu'il est publié sous sa forme définitive.

##### Décision technique

Pourquoi faire nous même alors qu'il existe plein d'éditeurs WYSIWYG ?

##### Prototypage et recherche
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

##### Architecture
