Differentes passes possibles :
- Pass classique opaque
	- Skybox pass
		- Uniform : VP, Texture
	- Deferred pass write to buffers (normals, diffuse, depth, etc) Opaque
		- Draw static geometry
			- Uniforms : MVP, NormaTex, DiffuseTex, DiffuseColor (completez svp)
		- Draw skinned geometry
			- Meme uniforms + : array<mat4>bones
	- Deferred pass write to buffers (normals, diffuse, depth, etc) NON Opaque
		- Draw static geometry
			- Uniforms : MVP, NormalTex, DiffuseTex, DiffuseColor (completez svp)
		- Draw skinned geometry
			- Meme uniforms + : array<mat4>bones
	- Bilboard pass (vegetation / autre)
			- Uniforms : MVP, NormalTex, DiffuseTex, DiffuseColor
	- Particle pass
			- Uniforms : don't know
	- Reflection pass (reflection de la skybox sur des objets)
			- Uniforms : don't know
	- Spotlight pass lightning
		- Draw each spotlight
			- Uniforms : MVP (a completer)
	- Spotlight shadows
		- For each spotlight
			- Draw each geometry
				- Static geometry
			        - Uniforms : MVP, NormalTex, DiffuseTex, DiffuseColor (completez svp)
				- Skinned geometry
			        - Meme uniforms + : array<mat4>bones
	- Pointlight lightning
		-Draw each pointlight
			- Uniforms : MVP (a completer)
	- Pointlight shadows
		- For each pointlight
			- Draw each geometry
				- Static geometry
			        - Uniforms : MVP, NormaTex, DiffuseTex, DiffuseColor (completez svp)
				- Skinned geometry
			        - Meme uniforms + : array<mat4>bones
	- Directionnal lightning
		- Uniforms : MVP (a completer)
	- Directionnal shadow
		- Uniforms : MVP (a completer)
	- Water pass (ocean, lake)
		- Uniforms : MVP (a completer)
	- Water reflection pass
		- Uniforms : MVP (a completer)
	- Merge deferred pass
	- Highlight object (ajoute une silouhette a l'objet)
		- Uniforms : Color, epaisseur, ...
	- Postfx HDR
		- Uniforms : MVP (a completer)
	- PostFx antialiasing
		- Uniforms : MVP (a completer)
	- PostFx glow
		- Uniforms : MVP (a completer)
	- PostFx motion blur
		- Uniforms : MVP (a completer)

Certains materiaux ne sont pas affectés par les shadow (pour faker un material qui emet de la lumiere - genre un ecran)
![Imgur](http://i.imgur.com/VpHwaol.png)

Certains materiaux ne castent pas de shadow (genre les semi-transparent ou autre)

--------------------------

Donc on pourrait se dire :
- Un mesh peut avoir un ou plusieurs materials
- Un material a un ID unique des qu'il change de type ou d'information constante
- Chaque instance d'un meme type de material, si il contient des informations variables (ex : bones matrix array) a un ID unique
- Un material n'a pas de render pass d'attaché (sauf exception plus bas). Pourquoi ? Ca serait tres pratique pourtant.
	- Je suis bien d'accord mais imaginons que tu rajoute une pass X, tu fais comment pour tout tes materiaux existants, tu les reedite a la mains ? Et si tu es a une centaine, mille, tu prends un stagiaire ? Hahahah !
- Un material a des informations (bitset) qui serviront a determiner si oui ou non ils sont utilisé par une pass. Parmis ces informations :
	- Skinned
	- Cast shadow
	- Touched by shadows
	- Opaque
	- IsUsedBySpecificPasses
	- Reflective
	- Instanciable

Pour chaque camera (parallel)
- Une fois un mesh cull et occlusion cullé, on regarde chacun de ses materials (parallel)
    - si il n'est pas instanciable, on cherche directement les pass auquel il correspond et on l'y ajoute avec le mesh
    - si il est instanciable, on l'ajoute a son bucket avec le mesh, dans les pass correspondantes
- Pour chaque point light cullé on l'ajoute aux pass correspondantes a la light (parallel)
	- On cull chaque mesh qui rentre dedans (parallel)
		- On l'ajoute aux pass correspondantes a la light qui necessite des mesh
- Pour chaque spotlight cullé on l'ajoute aux pass correspondantes a la light (parallel)
	- On cull chaque mesh qui rentre dans le frustum de la spotlight (parallel)
		- On l'ajoute aux pass correspondantes a la light qui necessite des mesh

Si on a 1000 meshs, 10 point lights qui castent des shadow et 10 spot lights qui castent des shadow et une camera
- On fait 1000 tests de frustum pour les mesh vs camera, 100 passent le test
- On fait 100 tests d'oclusion pour les mesh
- On fait 1000 tests * 10 pour les mesh vs le frustum des spotlights (10 000 tests)
- On fait 1000 tests * 10 pour les mesh vs la bounding sphere des point lights (10 000 tests)
En tout on fait deja 21 100 tests !!!
Mais en vertite les objets a cull sont reunis dans des blocks de 64, et le culling d'un block prend environ 16us.
Ca fait donc 16 tests (1000 / 64) de 16us pour les meshs vs frustum -> 256 us
Plus 2 fois 16us pour les lights -> 32us
Donc en tout 288us. Et si tu te dis que tout ca est bien dispatché sur 8 thread, ca te fais du 36us

Ensuite :
-> on lance une tache de sort par pass pour qu'elle trie les objets non instanciables a sa sauce (tout ca en parallel)
-> Pour chaque pass :
	-> Si c'est une pass qui gere les instancied
		-> Pour chaque material (on le bind)
			-> Pour chaque type de mesh
				-> On fait un instancied draw call
	-> autrement
		-> Pour chaque material (on le bind)
			-> Pour chaque mesh (on le draw)
			...

------------------------------------------------------------------------------

### Les principes généraux :

- Limiter au maximum les bindings (shaders, uniforms, etc)
    + Logiquement BGFX nous aide pour ca, mais plus on rationnalise nos appels mieux c'est.
- Automatiser la relation : mesh, material, pipeline. Un juste milieu entre flexibilité (nouveau effets facilement implementable - avec conservation des performances) et automatisation.
- Instancied rendering au coeur du rendu
    + Pouvoir faire facilement de l'instancied rendering sans se prendre la tete (l'automatiser en quelque sorte)
- Etroite relation entre shaders et materials. Pour permettre une automatisation et des gains de performance.
    + Meme nommage pour les uniforms.
    + Mise en cache des appels aux binding.
- Relation etroite entre culling, occlusion et render pass
    +   Tout les cas de figures doivent etre pris en compte (culling de frustum, de spot, de pointlight, des whatever)
- Paralellisation de la preparation du rendu au maximum et performances optimales.
    +   La preparation du rendu et les appels a BGFX ne doivent pas depasser 2ms pour une scene tres relativement complexe (1000 objets, dont certain instantiables, 2 points lights, 2 spotlights, une directionnal, occlusion culling, un objet skinné)
    +   Pas de sync entre les thread (lock free au maximum)
    +   Pas d'allocation. Utilisation de containers pre-alloués

### Les pass

Que fait une pass ? Elle a en inputs des informations, ex :
- Une liste de spotlight avec une liste de mesh qui entrent dans chacun des frustums
- Une liste de pointlight avec une liste de mesh qui entrent dans chacun de leur bounding sphere
- Une liste de material avec pour chaque material une liste de mesh
- Une liste de material avec pour chaque material, un mesh et une liste de transformations pour faire de l'instancied
- ...

A partir de ces inputs, une pass output des commandes de rendu (via BGFX)

On le voit bien, ces inputs sont de sortes differentes : liste, listes de listes, listes de listes de listes etc
Une pass peu aussi aussi prendre d'autres inputs, comme par exemple des informations de camera ou autre.
Ces inputs vont donc entretenir une relation tres etroite avec la preparation du rendu, qui lui outputera ces listes.
Ca sera donc les pass qui commanderont le comportement de la preparation du rendu.

Je propose que les pass n'ai que peu de choix quand a leur reclamations d'inputs :
- Liste des spotlights
    + Avec liste des mesh dedans
    + Ou sans
- Liste des pointlights
    + Avec liste des mesh dedans
    + Ou sans
- Liste des meshs dans un frustum

Le filtrage et la preparation des ces listes se fait ensuite en interne dans les pass.

.... a quoi que ....

En fait on pourrait diviser l'archi selon cette grossiere idee :
- A) On a les BFC manager, chacun contiennent un type de BFC (camera, meshs, spotlights, pointlights, whatever). Ces elements contenus sont de type :
    + Cullable
    + et / ou
    + Able to cull :)
- B) On a le Cull manager qui prend des commandes. Le but est de ne pas faire deux fois le meme culling pour rien et de reutiliser les resultats des memes commandes. Du genre, je veux :
    + Cull Camera3D VS Nothing (donc toute les cameras)
        * For each Camera3D cull Pointlight
            - For each PointLight cull Meshs
            - For each PointLigh cull Billboards
    + Cull Camera3D VS Nothing
        * For each Camera3D cull Meshs
    + Cull Camera3D VS Nothing
        * For each Camera3D cull Meshs (Res A)
        * For each Camera3D cull Spotlights
            - For each Spotlights cull from Res A
- (Notes concernant le point B : Ca pu le truc hyper compliqué a developper, autre approche proposée ci dessous)
- B) deuxieme approche) On a le cull manager qui prend des commandes de culling :
    + Meshs dans camera
    + Meshs dans spotlight
    + Spotlight dans camera
    + ................................ nan ca marche pas ...
- C) On a les sort manager qui prennent le resultat des cull manager et les sort en fonction des exigences des pass, du genre :
    + Groupe moi les mesh par material
    + Groupe moi les mesh par instance de mesh par material
    + File moi juste les mesh avec leur material

... rien de tout ca n'est implementable tel quel a reflechir demain

