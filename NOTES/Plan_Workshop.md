## Game engine workshop | Reflexion generale

Tentative de definition des workshops que j'aimerais donner a Epimerde

#### Objectifs

- Objectif principal : introduction technique au travail sur moteur :
    + Presenter les problemes clefs
        + Les illustrer tout au long des seances a travers des exemples concrets (etude d'architectures)
        + Proposer des outils / solutions
- Objectif secondaire : introduction "professionnelle"
    + Organisation humaine dans les studios que j'ai fréquenté
    + Organisation technique dans les studios que j'ai fréquenté
- Objectif bonus : donner envie aux étudiants ...
        + ... de faire du game engine
        + ... de changer certaines de leurs approches techniques
        + ... de partager aux autres participants leurs connaissances

#### Methode

L'objectif etant d'illuistrer des problematiques recurrentes du developpement de moteur, il faut dans un premier temps introduire les participants a celles ci.

Pour ne pas les endormir avec un long cours theorique. Une presentation concise de celles-ci leur sera faite en ouverture du premier workshop. Elle seront ensuite approfondies tout au long des seances.

Nous parlerons de l'organisation telle que je l'ai observé dans les differents studios que j'ai eu la chance de frequenter au long du workshop, de maniere dispersée pour faire des petite pauses techniques.

La presentation des differents points sera abordée a travers differents medium :

- Presentation de snippets de code
- Live code (par moi-meme, ou participant volontaire)
- Eventuellement demander aux participant de coder sur leurs ordis (pas sur, j'ai peur que l'on ai pas le temps)
- Le workshop reposera sur la participation des etudiants

#### Audience

Je ne veux pas proposer un workshop elitiste et pretentieux. C'est pouquoi, meme si il est principalement destinés aux etudiants de deuxieme année et plus (ayant des bases en C++), ceux de premiere année - meme les tek0 - sont chaudement encouragés a venir. Meme si certains ne comprennent pas tout, c'est bien normal, si ca peut affuter leur desir d'apprendre de nouvelles choses, le pari est gagné.

Je souhaiterais limiter le nombre de participant a 20 personnes (peut-etre que je reve ma vie et qu'il n'y aura personne) de maniere a echanger entre nous assez simplement.

#### Materiel necessaire

- Un projo pour afficher des trucs.
- Si vous voulez vous pouvez filmer et retransmettre sur internet pour les etudiants de province. On peut meme faire un live si il y a des interessés.
    + Si vous filmez, vous pouvez le mettre en ligne de maniere public, mais je me reserve le droit de veto si jamais je considere avoir échoué et que j'ai honte :)


#### Points importants

- Rappeler a l'audience de garder a l'esprit que je ne suis qu'un petit stagiaire mediocre. Les informations que je delivre proviennent :
    + Soit de lecture / video / discussion, produitent/avec des personnes qualifiées. Donc je repete.
    +  Soit de mon avis personnel. A prendre donc avec des pincettes. C'est un camarade qui parle avec des camarades (a bas la hierarchie).
- C'est experimental. C'est la premiere fois que je fais ca. Il se peut que le contenu prevu pour un cours soit en realite trop long/court, trop complexe/simple ... je tenterais de rediriger le tire en fonction du feed back des participants.

-------------

### Programme

#### Workshop 1

- Presentation
    + Presentation Cesar Leblic
    + Presentation Workshop
    + Tour de table presentation de participants
- Reflexion sur le profiling.
    + Quels outils / techniques, pouvons nous utiliser ?
        + fps counter
        + memory tracer
        + thread analyser
        + profiler CPU
    + Note : Je donne pour chacun des points ci-dessus des outils / libs pour le faire
- Mettons en pratique le profiling et profilons ensemble des exemples de code (a priori j'aurais deja codé les exemples. Mais peut etre que je le ferais en live ou alors que je demanderais a des participants de le faire, ca peut etre interessant, j'y reflechi). Cherchons a comprendre ce qui prend du temps, pourquoi, et comment l'optimiser.
    + Exemple numero 1 ( solution : cache miss )
        + On embraye sur qu'est ce qu'un cache miss, comment ca marche un CPU (va falloir que je revise hehe), etc
        + On propose une correction que l'on re-profile pour voir la difference en action
    + Exemple numero 2 (solution : object oriented progamming from hell)
        + On embraye sur une introduction au data oriented programming
        + On propose un correction que l'on re-profile pour voir la difference en action
- (Si on a le temps) On finit par une analyse de differents containers de la stl et de leur impact en terme de cache miss / complexité algoritmique. L'idee est d'introduire les premieres années sur l'impact en terme de performance que cela peut-avoir (ca pourra leur servir sur tout les projets de tek).
- Bref présentation du workshop 2. Bien leur expliquer qu'il sera plus drole que le 1 (j'espere), on etait obligé de passer par la pour s'amuser pendant les suivants.
- Leur demander de m'envoyer un mail avec ce qu'ils ont pensé du workshop : si c'est de la merde ou pas ; si ils souhaiteraient aborder des points particulier a l'avenir ; ce que l'on pourrait faire pour ameliorer certains points.

