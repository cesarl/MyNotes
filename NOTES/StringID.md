## StringID project

Reflexions sur une proposition de projet pour le HUB.

Developper une lib generique de StringID, accompagné d'un utilitaire.
Plusieurs fonctionnalités :

La lib :

- Hash au runtime ou non (config)
- Multi plateforme (Windows / Linux)
- Header only (pas encore certain mais je pense)
- Option : enregistre les strings/hash dans une database
- Option : garde un pointeur sur une const char* pour le debug
- Option : assert si collision
- Option : gere ou non les std::strings
- Option : alloue ou non
- Option : id uint32 ou uint64
- Option : peut etre plusieurs fonctions de hash au choix - a voir, je penche pour du `fnv` pour le moment

L'utilitaire (.exe) :

- Hash un string donné
- Parse des fichiers de code et y ecrit le hash des strings
- Garde en cache les dernieres modifications de fichier, pour ne pas avoir a reparser des fichiers non modifiés
- Export de database
- Output status + error (ex : collisions)
- Output optionnel du temps passé a chercher et hasher les strings
- Mode graphique ou non (affichage des string / id, log, menu, options)
- Multi-threadé pour meilleures perfs sur les grosses bases de code
- Multi plateforme (Windows / Linux)
- Compilation avec Fasbuild (generation de projets + compilation distribuée)
- Profiler intégré pour mesurer les performances
- Search and hash : possibilité de definir un regex a chercher dans certains types de fichier et a hasher (in place ou dans database) - pas sur encore de son utilité
