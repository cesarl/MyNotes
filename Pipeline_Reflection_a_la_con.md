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

