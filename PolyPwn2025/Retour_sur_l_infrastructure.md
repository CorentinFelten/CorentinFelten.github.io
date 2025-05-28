# CTF PolyPwn 2025 - Retour sur l'infrastructure
Avant de commencer, quelques petits points de mise en contexte. Il s'agit de mon premier déploiement d'infrastructure. J'ai déjà manipulé quelques serveurs Linux par le passé, notamment dans le cadre d'un projet de développement avec un ami, mais jamais à cette échelle et pour un projet aussi demandant en ressources. Quand je me suis engagé à produire une infrastructure, je n'avais jamais entendu parlé de CTFd, je connaissais à peine l'univers des CTFs et n'avais encore jamais participé à une compétition. Entre temps, j'ai eu l'occasion de participer à deux CTFs en personne, à savoir le HackFest 2024 et le @Hack 2025. Dans ces deux CTFs, l'infrastructure a eu quelques problèmes, à différents moments et d'ampleurs différentes, qui m'ont fait prendre conscience de la difficulté de la tâche de concevoir une infrastructure robuste et adaptée à la compétition. En rétrospect, et au vu de comment s'est déroulé le PolyPwn 2025, je suis heureux d'avoir pu m'engager pour la conception et le déploiement des services pour le premier CTF en personne organisé par PolyCyber. 

## Avant toute chose, un peu de jargon
Dans ce write-up, je vais mentionner quelques termes qui ne parleront peut être pas à tous, donc voici un petit glossaire :

- CTF : Capture The Flag, compétition de cybersécurité dont le but est de résoudre des challenges dans différentes catégories (programmation, reverse engineering, web)
- [CTFd](https://github.com/CTFd/CTFd) : framework de déploiement de CTF
- Docker : service de déploiement de conteneurs, petites machines virtuelles spécialisées pour une application spécifique
- Firewall statefull / stateless : Un firewall statefull retient les informations de la connexion en cours à travers les différents messages de la communication, tandis qu'un firewall stateless analyse les paquets de manière indépendantes, même lors d'une communication bidirectionnelle entre un client et un serveur
- VPS : Virtual Private Server, un serveur privé virtuel

## Infrastructure
### Structure générale
L'ensemble du CTF a tourné sur un serveur principal, entouré de quelques VPS supplémentaires. Le serveur principal contenait les bases de données, le CTFd, ainsi qu'un reverse proxy pour un accès en HTTPS, et les VPS additionnels étaient utilisé pour différents challenges quand le déploiement ne se prêtait pas à un déploiement sur le serveur principal (j'y reviendrai plus tard). Ces services étaient déployés via docker, et spécifiés dans un docker compose, l'objectif étant de pouvoir redéployer l'ensemble du serveur rapidement en cas de besoin.

![Schéma du serveur principal](<Serveur principal.png>)

### Détail du serveur en lui même
Le serveur a été choisi pour pouvoir correctement soutenir la charge des 240 participants inscrits, ainsi que pouvoir déployer des challenges à instances dynamiquement. Le choix s'est porté entre deux configurations après beaucoup d'échanges avec les organisateurs du UnitedCTF : 16 coeurs CPU + 16 Gb de RAM, ou bien 8 coeurs CPU + 32 Gb de RAM. N'ayant jamais organisé d'événement avec un tel besoin de puissance, il a été difficile d'estimer correctement la puissance du serveur à l'avance. Ne sachant pas quelle quantité de challenges demanderaient des instances déployées dynamiquement lors de la conception de l'infrastructure, et ne voulant pas risquer de prendre trop peu de RAM, le choix s'est donc porté sur la configuration 8 coeurs + 32 Gb de RAM.

### CTFd et paramétrage du serveur
Pour certains challenges, il était nécessaire de trouver un moyen d'instancier des machines pour chaque équipe du CTF, pour éviter qu'elles ne se marchent dessus (exemple : gain de droits `root` sur la machine à la fin du challenge). Les deux CTFs auxquels j'ai eu la chance de participer avaient chacun une approche différente, mais passaient tous deux par des services en dehors de CTFd. Le déploiement de l'instancer du @Hack m'avait d'ailleurs particulièrement déplu, forçant un passage par VPN qui permettait la connexion à seulement une instance à la fois, les instances ayant des durées de vie excessivement courtes (parfois à peine 15 minutes). 

Afin d'éviter de tomber sur les mêmes problèmes, d'améliorer l'intégration au CTFd sans avoir à réinventer la roue et recoder intégralement un instancer, nous avons choisi de partir sur un plugin CTFd avec des fonctionnalités intéressantes : intégration de l'instancer directement sur les pages des challenges, gestion des instances par équipe / par utilisateur en fonction du type d'événement, durée de vie des instances plus longue, page admin de gestion des instances dans CTFd... Malgré quelques surprises inattendues lors de l'événement, comme un nom d'équipe à plus de 64 caractères qui empêchait de charger la page admin parce que le plugin ne s'attendait pas à ça, les retours des participants ont été très positifs sur l'instancer, et nous n'avons détecté aucun problème ou abus avec les instances dynamiques.

Au total, près de 1050 instances ont été déployées lors de l'événement, et ce sans souci majeur. Elles ont été déployées directement sur le serveur principal, en exposant un port par instance sur le serveur. 

Seul point négatif sur l'instancer : il ne supporte à ce jour pas la possibilité de déployer des instances via docker-compose, et donc ne permet pas le déploiement de réseaux d'instances. C'est une fonctionnalité que nous aurions aimé être en mesure de proposer, mais elle demandera davantage de développement pour permettre des déploiement plus complexes. Egalement, nous aurions aimé rendre plus difficile l'accès aux instances des différentes équipes, afin d'éviter qu'une équipe mal intentionnée découvre les ports ouverts d'autres équipes et perturbe leurs challenges. L'idéal aurait été de trouver un moyen de changer dynamiquement la configuration du reverse proxy, et de lier un port, et donc une instance, à un sous-domaine généré avec le nom / id / hash de l'équipe. Comme précédemment ces changements auraient requis plus de développement et d'adaptation du plugin à nos besoins. 

## Hébergement - avantages et inconvénients
Pour héberger l'ensemble de l'infrastructure, il a été décidé de partir sur un VPS fourni par OVHCloud, tant pour la proximité de leur datacenter avec Montréal, réduisant donc la latence de communication, que pour leurs prix attractifs. D'un point de vue coût, le serveur est revenu à CA$115.09 pour un abonnement d'un mois, ce qui restait largement dans le budget de l'événement de PolyCyber. Le déploiement a été très simple, et le serveur a fonctionné comme attendu tout au long de l'événement, très stable et avec très peu de latence. 

Malheureusement, c'est là que s'arrêtent les compliments, et que le choix d'OVH s'est révélé moins flexible que prévu. Pour commencer, l'option KVM du serveur a subitement décidé d'arrêter de fonctionner en plein événement, et ne répondait plus. Par chance, nous n'avons pas eu de problèmes serveur, donc pas eu besoin d'interagir avec, mais si le serveur avait eu le moindre problème, nous aurions du le redémarrer pour récupérer un accès KVM fonctionnel, coupant ainsi temporairement l'accès à la compétition aux participants. 

En plus de ce point qui ne nous a pas trop affecté, le paramétrage de règles de firewall sur OVH est trop rigide : seulement 20 règles de firewall possibles à définir, pas de moyen de spécifier des plages d'IPs (uniquement des /32), et le plus pénalisant pour nous, le firewall est stateless et non statefull. J'y reviendrai plus tard, mais ce dernier point nous a forcé à désactiver la règle DENY ALL finale de notre configuration de pare-feu. 

Enfin, le point qui nous a le plus impacté lors de l'événement : la protection automatique Anti-DDoS de OVH. Aucun moyen de la désactiver. Aucun moyen de forcer le whitelisting des IPs de Polytechnique Montréal. Aucun moyen de couper la protection automatique une fois démarrée, bloquant certaines IPs pendant 15 minutes à la fois. Après avoir contacté directement deux employés de OVH, on nous a appris que le filtre était interne à leur réseau, et donc impossible à retirer. J'y reviendrai également par la suite dans la section des challenges qui ont posé problème avec l'infrastructure, mais une track de deux challenges s'est retrouvée fortement impactée par cette protection anti-DDoS.

Pour résumer l'expérience avec OVH : serveurs pas chers, et très performants, mais solution autour du VPS n'étant pas à la hauteur du service VPS proposé. Pas de granularité dans les options de firewall, pas de contrôle sur les options de sécurités déployées par OVH. L'hébergeur n'est malheureusement pas le plus adapté à une compétition de cybersécurité.

## Challenges et retour sur l'événement
### Détails des challenges
Au cours des 10 heures de compétition, nous avons proposé un total de 75 challenges, allant de challenges d'introduction à la cybersécurité à des challenges relativement difficiles. Je n'ai pas participé à leur création, mais je remercie énormément tous les créateurs et créatrices pour leur implication dans la conception de l'événement. Comme mentionné plus tôt, quelques challenges sont entrés en conflit avec l'infrastructure et son déploiement pour diverses raisons, parfois évitables :

- Les Aventuriers du Cache Perdu : ce challenge reposait sur l'idée que l'instance ferait une requête à un domaine extérieur. En raison du firewall stateless, nous nous sommes retrouvés dans l'obligation de désactiver la règle DENY ALL sur le trafic TCP, afin de permettre aux réponses des requêtes effectuées par l'instance de lui revenir. Ce souci aurait pu être évité si le firewall de OVH était configurable pour recevoir les réponses de requêtes émises par le serveur.
- Shrak into my castle : ce challenge nécessitait initialement un `nmap` afin de découvrir que le serveur ciblé était un serveur Windows. La requête étant facilement détectable, elle a probablement causé les premières détections Anti-DDoS de la part de OVH. Lorsque le serveur était affecté par la protection d'OVH, il en devenait inacessible, cassant le challenge. Ce souci aurait pu être évité si la protection Anti-DDoS était désactivable pour ce serveur.

### Charge serveur
Au cours de l'événement, le serveur n'a pas dépassé les 10 Gb d'utilisation de RAM, et n'est pas monté au delà de 50% d'utilisation sur tous les coeurs CPU en même temps. De fait, le serveur était surdimensionné en RAM, : il aurait largement été possible de prendre le serveur 16 coeurs + 16 Gb de RAM et l'expérience aurait été encore meilleure. Pour la prochaine édition, à compter que le nombre de participant soit équivalent, il serait plus judicieux de privilégier le nombre de coeurs CPU à la RAM pour plus de stabilité et vitesse de calcul. 

## Quelques chiffres pour conclure
- 0 problème majeur remonté sur l'infrastructure
- 1050 instances déployées sur 10 heures
- Pas plus de 10 Gb de RAM utilisés
- Pas plus de 50% d'utilisation sur tous les coeurs CPU
- 75 challenges
- 93 Gb de données transmises depuis le serveur
- 181 participants sur place
- Et surtout... **0 secondes de downtime sur l'événement** !

## Mot de la fin
Bien que j'ai été officiellement en charge de l'infrastructure, je n'aurais pas pu tout mettre en place sans l'aide précieuse de Maël, Mathias, et Zachary. Je suis très heureux de ce que PolyCyber a réalisé avec ce premier CTF physique, et j'espère avoir l'occasion de contribuer à des projets similaires dans le futur !