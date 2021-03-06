# Occlusion culling on AGE

I plan to implement some depth buffer occlusion culling on AGE.

------------------------------------

References :

http://rastergrid.com/blog/2010/10/hierarchical-z-map-based-occlusion-culling/

Un outil pour generer des occluders a partir des meshs
http://nickdarnell.com/oxel/

http://www.nickdarnell.com/hierarchical-z-buffer-occlusion-culling/

--------------------------------------

## Descriptions grossieres des techniques que j'ai en tete

Elles peuvent se completer ou non les unes les autres.

### Utiliser la depth de la derniere passe de rendu

C'est un peu post moderne comme technique. Et il y a plein d'inconnues non resolues (dans ma tete).

Avant le draw de la frame 1 :
- tu prends la depth de la frame 0
- tu la transforme avec la nouvelle MVP (celle de la frame 1)
- tu te sers de cette depth transformee pour faire ton occlusion culling

Il y a pas mal d'inconnues :

Imaginons qu'un objet tombe tres proche du frustum de la camera a la frame 0, mais qu'a la frame 1 cette objet sois tombé plus bas, et donc hors champs.
Lors du rendu de la frame 1, au moment de culler sur la depth, tres peu d'objets vont passer le test. Or le resultat sera faux puisqu'en verite l'objet n'occludera plus puisqu'il n'existe pas dans la frame 1.

Il existe parait-il des solutions pour remedier a ce probleme. Dans le cas de notre objet physiqué, on peut ecrire dans un buffer la direction et velocite de l'objet, et donc par consequent calculer (je ne sais pas vraiment comment) si il est la pour de vrai ou pas.
Cette solution qui a un nom savant que j'ai oublié (directional vector ou truc du genre) ne me semble pas pertinente et un peu tirée par les cheveux.

Cependant, on peut imaginer quelques choses en s'en inspirant :

Les objets susceptibles de bouger, ou de disparaitre entre deux frames, sont notés par exemple avec un 0 dans le buffer, et les autres, immuables, un 1. Du coup on ne prend en compte pour l'occlusion que des objets 1.
Tout ca revient a distinguer les geometrie dynamiques de celles statiques.

Ce qui nous amene a une question :

Peut etre que l'on pourrait tout simplement rendre les objets statique uniquement dans une depth map specifique pour s'en servir ensuite a la frame t+1 en squizzant les objet dynamique ? Yo no se.

Cela nous amene a la deuxieme solution.

### Rendre dans une premiere batch les objets statiques

Voila les etapes :
- 1) On rend dans un premier temps les objets statiques (en deferred ou autre, on s'en fout tant qu'on a un depth buffer)
- 2) On mip map notre depth buffer
- 3) On rend dans un second temps les objets dynamiques en faisant un test d'occlusion sur la depth de la premiere passe.

La, il y a plusieurs solutions et approches (qui concerne aussi le point precedent mais dont je n'ai pas parlé).

Lors de l'etape 3, ou l'on tente de draw les objets dynamiques en testant sur la depth des objets statiques :
- Est ce que l'on utilise un compute shader, qui s'occupe de tester des AABB ou des Spheres dans un premier temps, puis on attends qu'il nous ecrive son resultat dans un buffer pour le lire et n'appeler que le draw des objets visibles ?
- Ou l'on peut faire la meme chose avec les objets directement dans le draw call, en passant en uniform leur AABB par exemple ? Dans le VS on fait le test, on passe ensuite par un Geometry shader (GS) qui discard ou nom le draw en fonction du resultat outputé par le VS...

Dans cette solution, ce qui me semble positif :
- On rend les geometries statiques en meme temps que l'on ecrit leur depth (pas de depth pre-pass)

Ce qui me semble negatif :
- Si on a une grande quantite de geometrie statique qui s'occludent potentiellement mutuellement (du genre un bout de ville, des immeubles en cachant d'autres) bah on peut etre amene a rendre un nombre plus ou moins grand de geomtries plus ou moins complexes.

La solution / approche 3 peut potentiellement apporter une solution au point negatif de cette derniere :

### Faire une pre passe de depth

On peut faire 3 type a mon sens de pre-passe de depth :
- On fait une pre-passe degueulasse avec les geometries (sans LOD ou autre).
- On a des LOD, on fait une prepasse avec le LOD le plus bas (en trichant et le downscalant un peu par securite)
- On a pas de LOD mais on veut quand meme pas faire une pre-pass avec les mesh finaux car c'est debile, donc on fais notre prepass avec des AABB ou des sphere. Mais du coup, on prend le risque d'avoir par exemple une voiture derriere un arbre. Elle devrait etre partiellement visible hors, elle risque d'etre occludée par la AABB de l'arbre.
- On a des occluder meshs ! Et pous ca, on a ca : Oxel (cf les references en haut de la page) Fuck yeah !

Si on se concentre uniquement sur la derniere proposition.

On fait une prepasse des objets occludants en utilisant leurs occludeurs (composition de box).
Puis, a determiner, comme evoqué plus haut : soit on draw les mesh en les accompagnant d'une uniforme de leur AABB pour tester sur la depth, ou alors on fait tourner un compute shader qui nous output la liste de ce qui est visible.

Le probleme d'utiliser un compute, c'est qu'il faut trouver quelque chose a faire faire au Render thread pendant que le compute fait son affaire. Sinon on va perdre du temps CPU pour rien... J'aurais du coup plutot tendance a pencher pour la solution on balance le draw call avec une uniform pour sa AABB on fait le test dans le VS et on le discard dans le GS si il est occlude.


--------------------

## Retour sur experience

J'ai fait une premiere implementation de test :
- Un mur et un sol sont definis en tant qu'occludeur
- Derriere le mur 100 meshs assez complexes sont disposes

### Experience 1

__Echec__ haha !

Lorsque l'occlusion est activee :
- On draw le mur et le sol
- On mipmap la depth (pour le moment ca sert a rien)
- On draw les mesh complexes
    + Vertex shader basic
    + Geometrie shader (qui discard ou non le fragment shader - condition en dure dans le shader pour tester)
    + Fragment shader classique

Voila le resultat du benchmark :

| Method                          | FPS     |
|---------------------------------|---------|
| 2 Pass + Z mipmap + Dicard All  | 148 Fps |
| 1 Pass + Z mipmap + Dicard All  | 149 Fps |
| 1 Pass + Z mipmap + Discard Non | 79 Fps  |
| 2 Pass + Z mipmap + Discard Non | 85 Fps  |
| 2 Pass + Discard All            | 158 Fps |
| 2 Pass + Discard Non            | 87 Fps  |
| 1 classical Pass                | 240 Fps |

Conclusion : le faite de diviser le rendu en 2 pass ralenti considerablement le rendu.
L'impact du mipmaping de la depth est negligeable.

En gros on dirait que c'est l'utilisation du geometry shader qui plombe les perfs. Meme lorsqu'il discard tout les draw calls.
En effet on passe par le GS pour chaque triangle de chaque mesh complexe. De plus on fait tout de meme un draw call.

### Experience 2

__Succes mitigé__ hoho !

Toujours avec la meme scene et les mesh complexes derriere un mur.

- On draw le mur et le sol
- On mipmap la depth
- On get la depth map (au mipmap 3) sur la RAM
- Pour chaque mesh complexe, on test un point (transformé sur sa MVP) dans le z buffer copie.
- Si la depth du point est < a celle de la depth map on fait un draw call pour cet objet.

Bien sur c'est loin d'etre optimisé, pour plusieurs raisons :
- Deja la pipeline n'offre pas d'acces simplifié a la matrice de transformation du mesh dans le render thread. Du coup j'ai du tricxer temporairement pour y acceder (ca ne prend que tres peu de temps, mais ca en prend quand meme plus que rien hehe)
- Je copie la depth map sur la RAM avec un `glGetTexImage`. Je pourrais surement utiliser un pixel buffer object pour gagner du temps.
- Je ne test qu'un seul point. Donc c'est forcement inexacte. Il faudrait que je test les 8 points de la AABB.
- Je n'ai qu'un seul niveau de mipmap, il m'en faudrait plusieurs pour tester la depth sur differentes depth map en fonction de la taille de la AABB.
- Tout ca est fait sur le render thread. Ce qui ne devrait pas etre le cas. Il faudrait deporter ca sur le prepare render thread, qui est designé a cet effet, et qui dispose de beaucoup plus de temps que le render thread.

Voila les resultats en comparaison avec une version qui n'occlude rien :

| Method                          | FPS     | Ms |
|---------------------------------|---------|----|
| Occlusion On + 0 objects a visible | 290 Fps | 3.44 |
| Occlusion On + all objects are visible  | 70 Fps | 14.28 |
| 1 classical Pass                | 240 Fps | 4.16 |

Mine de rien, on gagne 1 milliseconde lorsque tout les objets sont occludes, ca n'est pas rien. Cependant lorsqu'ils sont touts visibles, les performances sont catastrophiques. Sont elles perdues par l'appelle `UniqueDraw` moins optimise que le `DrawAll` qui draw tout sans distinction ... ?
C'est a benchmarquer. Quoi qu'il en soit, rien de tout ca ne devrait etre fait dans le `Render Thread`, c'est un heresie ! Personne ne respecte mon "archi" threadée hehe !

### Experience 3

On a 2 `DepthMap` objects qu'on swap a chaque frame entre le `render1 thread et le `prepare render` thread.

Cet objet contient :
- Un buffer de pixel correspondant a la depth map
- La matrice VP utilisee pour cette depth map

A chaque rendu, le render thread rend les objets Occludant. Get la depth depuis le GPU vers le buffer de l'objet `DepthMap`. Puis rend les objet occludables.

En paralelle. Le `Prepare render` thread test les bounding box de tout les objets occludables dans la `Depth Map` de la frame precedente.

Du coup le resultat est quand meme biaisé car le test est fait sur la depth precedente.

Il va falloir que je rescale les AABB pour les elargir en fonction de la VP de la depth map.

De plus les infos de depth sont comme differentes de celles que je calcule pour les boites (de 0.001) ce qui change tout. Pour le moment j'ai hacké les valeurs pour tester :)


| Method                          | FPS     | Ms |
|---------------------------------|---------|----|
| Occlusion On + 0 objects a visible | 390 Fps | 2.56 |
| Occlusion On + 0 objects a visible | 160 Fps | 2.56 |
| 1 classical Pass                | 200~240 Fps | 4.16 |

Todo list for experiment 3 :
- Ne pas tester les 8 points de la AABB. Mais definir un rectangle sur le z le plus proche et tester chaque pixel de la depth map
- Debuguer la difference de 0.005 f entre le z du pixel et le z de l'objet
- Diviser le travail sur plusieurs threads
- Trouver selon quel ratio aggrandir les AABB


