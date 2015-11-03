## StringID project

Reflexions sur une proposition de projet pour le HUB.

Developper une lib generique de StringID, accompagné d'un utilitaire.
Plusieurs fonctionnalités :

La lib :

- Hash au runtime
- Multi plateforme (Windows / Linux)
- Header only (pas sur encore)
- Option : enregistre les strings/hash dans une database
- Option : garde un pointeur sur une const char* pour le debug
- Option : assert si collision

L'utilitaire (.exe) :

- Hash un string donné
- Parse des fichiers de code et y ecrit le hash des strings
- Export de database
- Output status + error (ex : collisions)
- Mode graphique ou non (affichage des string / id, log, menu, options)
- Multi-threadé pour meilleures perfs sur les grosses bases de code
- Multi plateforme (Windows / Linux)
- Compilation avec Fasbuild (generation de projets + compilation distribuée)
- Profiler intégré pour mesurer les performances
- Search and hash : possibilité de definir un regex a chercher dans certains types de fichier et a hasher (in place ou dans database)