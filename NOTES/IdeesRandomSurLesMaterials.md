Qu'est ce qu'un material ?

Un material est un ensemble d'uniform concernant une geometrie. Ces uniforms peuvent etre des vector(2/3/4), des float, des int, des matrix, ou des textures.

Ces uniform vont s'addresser a un / des shaders.

Un premier probleme :
 - Les shaders n'ont pas les memes uniforms.
 - Les materials ne sont pas necessairement destinés a etre rendu avec les meme shaders
 - Un objet peut etre ammené a etre dessiné plusieurs fois par frame avec differents shaders (exemple : higlighter un objet -> 2 pass, une classique, une de highlight avec des infos de couleur par exemple)

Du coup on serait tenter de se demander :
- Un material doit-il etre lié a un shader ?
- Un material peut-il etre lié a plusieurs shader ? (exemple des plusieurs pass par frame)

On a deux solutions a mon avis :
- Soit on fait des materiaux statiques, c'est a dire, tout les materiaux ont le meme type d'information. Donc pas d'exception possible, si l'on veut ajouter un uniform X pour un material on est obligé de hacker.
Cepandant, ca nous permet de facilement mettre en place un systeme de hash, cad, generer un id unique en fonction des inforamtions contenues dans ce material, ce qui nous permet de les trier tres simplement et des faire de l'instancied rendering.
- Sinon on fait des materiaux tous differents, avec n'importe quoi comme infos. C'est ce que l'on a aujourd'hui (cf Properties). Mais du coup, impossibilite totale de les trier.

... plus tard

Okay, je vais essayer de decrire une approche possible, surement pleine de trous.

Nos materials heritent tous de `MaterialBase`. La plupart seront de type `DefaultMaterial` (diffuse, specular, bla bla bla).

- Chaque material a un ID unique.
- Chaque material indique si il est instanciable ou non
- Chaque material indique si il est opaque ou non
- Chaque material indique a quelle render pass il participe (bitset)

`MaterialBase`

    class MaterialBase
    {
        MaterialBase()
            : _uniqueId(-1)
            , _instanciable(true) // par defaut est instanciable
            , _opaque(true) // par defaut opaque
            {}
        virtual void bind(Shader &shader, FrameInformations &frameInfos) = 0;
        protected:
        size_t _uniqueId;
        size_t _renderPass; // bitset
        bool   _instanciable; // if it can be used for instancied rendering or not
        bool   _opaque;
    };

`DefaultMaterial` (en guise d'exemple)

    class DefaultMaterial : public MaterialBase
    {
        public:
        DefaultMaterial()
            : MaterialBase()
            {}

        virtual void bind(Shader &shader, FrameInformations &frameInfos)
        {
            shader.bindUniform<vec4>(diffuseColor, "diffuseColor");
            //...
            shader.bindUniform<TexturePtr>(diffuseTexture, "diffuseTexture");
            //...
        }
        protected:
        vec4 diffuseColor;
        vec4 specularColor;
        //...
        TexturePtr diffuseTexture;
        TexturePtr specularTexture;
        //...
    };

`HighlighMaterial` (disons que c'est un material pour highlighter un objet)

    class HighlightMaterial
    {
        protected:
        vec4 highlightColor;
        float highlightBorderWidth;
    };

`FlashingMaterial` (disons que c'est un material qui change l'opacité de l'objet en fonction du temps)
    
    class FlashingMaterial : public DefaultMaterial
    {
        public:
        FlashingMaterial()
        : DefaultMaterial()
        {
            _opaque = false; // il y a de la transparence donc non-opaque
            _instanciable = false; // le niveau d'opacite peut changer entre les instances, donc non-instanciable
        }

        void bind(Shader &shader, FrameInformations &frameInfos)
        {
            DefaultMaterial::bind(shader, frameInfos);

            _opacity += frameInfos.GetDeltaTime();
            if (_opacity > 1.0f)
            {
                _opactity -= 1.0f;
            }
            shader.bindUniform<float>(_opacity, "opacity");
        }
        protected:
        float _opacity;
    }

Un mesh se verra attribuer 1 ou plusieurs materials.

L'exemple ci-dessous, je me suis rendu compte apres l'avoir fait que c'etait de la merde.

- Render Passes
    - Pass A
        - Material 1 // Instanciable + Opaque
            - Mesh1
                - [Trans1, trans2, trans3, trans4] // 4 instances
            - Mesh2
                - [Trans1] // 1 instance
        - Material 2 // Instanciable + Opaque
            - Mesh1
                - [Trans1] // 1 instance
        - Material 3 // NonInstanciable + Opaque
            - Mesh1
                - [Trans1, trans2, trans3, trans4] // 4 instances
    - Pass B
        - ...
    - Pass C
        - ...

Proposition d'un workflow, apres comment l'implementer ... a voir

- On frustum cull tout les meshs [parallel]
    - On occlusion tout les occludable [parallel]
    - On occlude pas les non-occludable [parallel]
    - Pour chaque material on push le mesh dans un bucket (avec sa depth) [parallel]
    - Pour chaque MaterialBucket on fait soit [parallel]
        + On groupe les instancied
        + On


