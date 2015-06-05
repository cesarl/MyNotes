# Idees pour un meilleur culling, plus efficace sur AGE

## Que le culling (Octree) soit realise sous la form de job

Cette amelioration serait efficace dans le cadre du multi-scene ou du multi-culling (par exemple pour le culling des lights).

Je n'ai pas benchmark en detail le culling existant donc je parle un peu "a l'aveugle". Haha !

En gros aujourd'hui AGE fonctionne grosso modo comme ca :

    Gameplay thread : f0 S1;S2|f1 S1;S2|f2 S1;S2  wait  |f3
    Prepare thread  : --------|f0 C1;C2|f1 C1....;C2....|f2
    Render thread   : -----------------|f0 R1....;R2....|f1

Argf ! J'ai mis 20 minutes a faire le graphique ci dessus mais il illustre mal l'idee. Surtout que le Prepare thread prend plus de temps que le rendu, ce qui est loin d'etre le cas aujourd'hui.

Mais l'idee est de dire :

Le `main thread`, update toute les scenes, puis une fois cela fait, il passe le buffer de commande au `prepare render thread`, qui s'occupe de mettre a jour tout les octrees et de les cull, puis une fois tout ca fait, il passe les buffers des objets to draw au `render thread`.

L'idee consisterais a virer ce `Prepare render thread` et de le remplacer par des jobs.

En verite, aujourd'hui la preparation du rendu est tout a fait sommaire. C'est un simple culling a partir du frustum de la camera dans l'octree, puis on ecrit dans un vector tout les objets a draw avant de l'envoyer au `render thread`.
Mais a terme nous pourrions ajouter des fonctionnalites a cette etape de preparation du rendu, et justement, decouper ces etapes en jobs.

Voila un exemple des etapes de preparation possible :
- Mise a jour de l'octree
- Culling des objets visibles pour chaque camera
- Culling des lights visibles pour chaque camera
- Tri des objets a dessiner (back to front / front to back / instancied)
- Creation des async draw call pour le `render thread`
- ...

Prenons l'exemple d'un jeu avec 1 scene, ayant 2 cameras, et 60 lights.

Voila ce que nous pourrions faire

    T1 Update scene's octree | Cull cam1 | Sort objects Cam1 | Prepare draw call Cam1
    T2                       | Cull cam2 | Sort objects Cam2 | Prepare draw call Cam2
    T3                       | Cull lights | Prepare draw call for lights Cam1 |
    T4                                     | Prepare draw call for lights Cam2 |

C'est un exemple.

Une fois tout les draw calls de prepares (pour chaques camera et les lights) on peut submit au `Render thread` ces commandes.
On peut meme imaginer les soumettre des que possible (tout en conservant l'ordre requis par la pipeline)

De plus, le "schema" ci dessus n'illustre qu'une seule frame. En double bufferisant, on pourrait avoir la preparatio du rendu de la frame t+1 alors que celle de la t+0 n'est pas terminee.
... je dis des conneries, on ne peux pas modifier l'octree alors qu'on est en train de lire dessus dans un autre thread. Pour ca il faudrait soit le dupliquer (beurk) soit on update l'octree, et on cull tout ce dont on a besoin avant...

Le seul probleme, concernant l'archi existante est le passage des commandes depuis le `main thread` vers le `render thread`, car jusqu'ici le `prepare render thread` faisait office de proxy.

J'en parlerais plus tard ! ^^

----------------------------------------------

----------------------------------------------

# 1 mois plus tard : BFC

Une entité a une collection de components (Dynamique mesh, static mesh, life, weapon holder, sound emitter, etc etc). L'entité est orientée gameplay et game design.

Une entité a un link.
Le link fait le lien entre l'entité (le game object coté gameplay), et sa representation graphique.

Dans le `Link`, on va stocker plein d'informations necessaires au gameplay et impactant le rendu.

Ce link herite de `BFCLink`. `BFCLink` est une data structure beaucoup plus legere que le `Link`. Elle contient :
- Une liste des ressources drawable attachées a l'entité via leurs components (static mesh, dynamique mesh, skinned mesh, point light, spotlight, billboard, whatever, etc).
- Quelques trucs qui ne vous interessent pas qui permettent d'updater le culling que pour celles qui ont bougees (3 pointeurs)

Les drawable listés par le link sont représentés par des `BFCCullableHandle`.
Ces handles contiennent un pointeur sur le cullable (ou drawable, je ne suis pas encore sur du nom haha), et un pointeur (c'est pas un pointeur en vrai mais pareil) sur sa representation dans le system de culling.

Parlons du system de culling, il est relativement simple :
Les elements a culler sont stocker dans des BFCItem.
Un BFCItem contient :
- un pointeur sur le drawable 64b
- un void* pour peut etre un truc plus tard 64b
- un vec 4 : position + radius de la BBSphere 128b

Ce sont ces item que l'on va tester contre le frustum.
Ces items sont rangés contigue dans des BFCBlock.
Un block aujourdhui contient 64 Items (peut etre plus un jour).
Un BFCBlockManager gere la creation et la destruction des Item

Donc : Un BFCBlockManager == l'equivalent d'un octree.

Cependant, je veux specialiser les "Octree", de maniere a ce qu'ils ne contiennent qu'un seul et meme type de drawable (pour arreter de faire du switch case degueulasse)
D'autant plus qu'aujourd'hui nous n'avons que tres peu de type different (nous en avons 1 seul : meshDrawable)

Les fameux drawable (ou cullable) heritent tous de `BFCCullableObject`.
Ce dernier ne contient rien, si ce n'est :
- une `virtual CullableTypeID getBFCType()` qui permet de distinguer dans quel BFCBlockManager le chercher, et plus simplement de quel type est le cullable. (Il retourne un enum quoi : StaticMesh, DynamicMesh, etc etc).
- une mat4 de world transform pour le draw (pas sur encore que ca sera directement cette data structure qui servira au draw, ou une autre qu'elle contiendrait)

#### Comment fonctionne le moteur

Je decris ici une premiere etape, des choses seront amenees a etre changee plus tard (ex : virer le concept de main thread)

On a le main thread qui va updater non petites scenes les unes apres les autres.

Lorsque l'on bouge une entite elle enregistre son link aupres du `BFCLinkTracker`. Elle ne le fait qu'une seule fois par frame, meme si elle est bougee plusieurs fois dans la meme frame. Cela permettra a la fin de la frame de n'updater que les links ayant bougés.

A la fin de la scene. Le BFCLinkTracker :
-> itere sur les link
   -> itere sur les drawable contenus par les links
      -> mets a jour la bounding sphere des BFCKItem utilises pour le culling

A ce moment la se pose une question !
Nous pourrions ajouter une `virtual glm::vec4 ComputeTransform(const glm::mat4 &linkTransform)` a `BFCCullableBase`. Cette virtual prendrait en argument la world transformation du link et :
- Calculera sa veritable World Transform pour le draw
- Calculera sa veritable bounding sphere et la renvera pour la stocker dans l'item qui servira au culling.
Faire ca permettrait de ne pas rescale les mesh (et surtout les meshs animés !). Ont aurait simplement une matrice de transformation pour inverser le scale.
Par contre ca fait apperler une virtual en plus. Bref, a voir.

Donc, nos BFCKItems ont ete mis a jour : leur bounding sphere a ete calculee.

Notez que cette update peut etre faire de maniere parallelisee sans besoin de synchronisation.

Nous passons ensuite au culling ! Pour cela c'est tres simple et tres modulable, c'est ca qui est interessant. Prenons un exemple debile :

Nous avons 3 spotlights, une camera, 1000 objets drawable dans notre scene.
Nous voulons faire une liste de drawable par spotlight, et par camera. Donc 4 listes de drawable.

Nous avons 2 types de drawables : les mesh de type X et d'autre de type Y. Et disons que les spotlights ne veulent pas des de type Y.

Voila ce qui va se passer. On va dire :
- BlockManagerFactory remplis moi cette liste des drawable X et Y pour ce frustum de camera, puis, quand c'est fini, fait l'occlusion (callback quand tout les jobs ont termines).
- BlockManagerFactory remplis moi cette liste des drawable X pour ce frustum de spotlight1.
- BlockManagerFactory remplis moi cette liste des drawable X pour ce frustum de spotlight2.
- BlockManagerFactory remplis moi cette liste des drawable X pour ce frustum de spotlight3.
- Puis quand tout est termine, envoie ca au render thread (ou alors pourquoi pas envoie au fur et a mesure que les listes sont terminees)

Ce que le blockmanagerfactory va faire c'est qu'il va lancer 1 job, par block. Ce job va tester les 64 items du block vs le frustum et si le test est bon, push dans la liste le pointeur sur le drawable.

Les listes dans lesquelles on push sont bien evidemment tread safe (je pense que des listes lock free peuvent etre pas mal)

On pourrait meme imaginer que l'update d'une scene est un job. Que quand une scene a fini d'etre updatee, on lance en meme temps la preparation du rendu, et la scene suivante en parallelle

La vous allez me dire : "Oui mais avant avec le prepare render thread on pouvait faire une update physique pendant qu'on preparait le rendu ! C'est pas ca que tu nous avait vendu cesar ?"

En effet je vous avais vendu qu'on double bufferisait, ce qui est le cas, mais quand on regarde ce qu'il se passe on voit bien que  la double bufferisation ne sert a rien des qu'un thraed rattrape l'autre il est obligé de l'attendre et hop on perd du temps. Donc au final ca revenait au meme : quand on passait 6ms dans le prepare, on etait obligé de passer 6ms (dont 5 d'attente) dans le main thread.
On etait bloque car on ne pouvait pas se permettre de rater une frame entre les deux thread, sinon on pouvait perdre des infos telles que :
- on a ajouté une entite
- on a ajouté un mesh
- on a bougé une entite
etc etc

La, avec ma nouvelle proposition, plus de probleme !
On a tout dans le meme thread : creation / deletion d'entite et de component / ajout suppression de components graphique / mise a jour des positions. Ensuite on peut dispatcher le culling !
On pourrait meme double bufferiser les BFCBlock tres simplement si jamais on voulait.

Voili voilou !
