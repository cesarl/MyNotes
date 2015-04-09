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



