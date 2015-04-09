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

Le seul probleme, concernant l'archi existante est le passage des commandes depuis le `main thread` vers le `render thread`, car jusqu'ici le `prepare render thread` faisait office de proxy.

J'en parlerais plus tard ! ^^


