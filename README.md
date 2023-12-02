# Installation d'un BigBlueButton dockerisé

Voici les principales étapes d'installation et de configuration de BigBlueButton:
 - Clonage du repos Github
 - Configuration de l'environnement
 - Docker Compose
 - Gestions des erreurs

## Prérequis:
	

	- Posséder un nom de domaine publique
	- 8go de Ram
	- 4vCPU
	- 50Go de Storage 
	- Git et Docker (docker compose version >= 1.28)


## Step 1 - Clonage du repos officiel du dossier compose

	git clone --recurstrong textse-submodules https://github.com/bigbluebutton/docker.git bbb-docker
	cd bbb-docker

	# use the more stable main branch (sometimes older)
	git checkout main
	
## Step 2 - Paramétrage de l'environnement
BigblueButton fournis un script de setup le deployer selon l'environnement de notre infrasctructure. 

Dans le dossier bbb-docker:

	sudo ./scripts/setup
![](http://93.90.205.194/docs/bbb_install/screen/Greenlight.png)
### GreenLight
> Greenlight est une interface web front-end, elle vous permettra d'utiliser simplement les fonctionnalités de BigBlueButton si vous ne possédez pas déjà votre propre site web.

### HTTPS Proxy
![](http://93.90.205.194/docs/bbb_install/screen/Https_proxy.png)
> L'HTTPS Prxoy va se connecter à notre nom de domaine pour y générer un certificat SSL (Let's Encrypt) sur ce dernier. Il est donc nécessaire que notre domaine ne soit pas local mais accesible publiquement. Il est important de souligner que la connection en HTTPS peut-être primordial pour les sessions WebRTC car le navigateur peut la refuser si la session n'est pas sécurisé et par conséquent empécher BigBlueButton de fonctionner 

### CoTURN
![](http://93.90.205.194/docs/bbb_install/coturn.png)
> CoTURN est une implémentation open source de TURN et STUN Server, permettant de gérer le traffic média SIP (VOIP) derrière un NAT.
> Vous devez inclure CoTURN si vôtre machine n'est pas directement accessible depuis l'exterieur (via IP Publique) mais derrière un routeur ou une box.

Un moyen simple de le savoir, vérifier les interfaces IPv4 notre machine:

	ip -4 -o a
![](http://93.90.205.194/docs/bbb_install/ip_a.png)
> Ici l'ip publique est bien relié à notre interface (VPS), CoTURN n'est pas nécessaire.

Autre exemple:
![](http://93.90.205.194/docs/bbb_install/ip_a_local.png)

> Dans ce cas de figure notre IP est NATé par nôtre box, CoTURN sera donc nécessaire (Même si l'IP Publique de vôtre box redirige vers votre hôte BigBlueButton).


### Domaine
![](http://93.90.205.194/docs/bbb_install/domain.png)
> Entrez votre domaine Publique (Accessible depuis internet).


### Recording
![](http://93.90.205.194/docs/bbb_install/recording.png)
> Choisissez si vous souhaitez ou non activer l'enregistrement de conférence.


### Prometheus exporter
![](http://93.90.205.194/docs/bbb_install/exporter.png)
> Exportez ou non vos metrics avec Prometheus (Monitoring)

### Si oui, optimisation pour Prometheus exporter
![](http://93.90.205.194/docs/bbb_install/exporter-optimize.png)
### Supprimer automatiquement les anciens recording
![](http://93.90.205.194/docs/bbb_install/recording-remove.png)
### Définir l'adresse IP du Serveur
![](http://93.90.205.194/docs/bbb_install/ip2.png)
> [ Attention ! ] Comme défnit précedemment avec CoTURN, ne mettez vôtre Ip Publique seulement si l'hôte est directement accessible par celle-ci !
> Si vous êtes derrière un NAT (Box ou routeur), alors définissez l'Ip local de vôtre hôte (192.168.. ou autre).

## Step 3 - Docker Compose
Votre configuration est maintenant terminée, un fichier .env a été créer pour définir l'environnement de vos containers.

> Si vous avez fait une erreur et souhaitez relancer le script de setup, il suffit de supprimer le fichier .env:

	sudo rm -f ./.env

> Vous pouvez également éditer le fichier directement si vous le souhaiter, dans ce cas il faudra exécuter par la suite le script pour régéner le docker-compose:

	./scripts/generate-compose

> Il ne reste plus qu'à lancer conteneurisation:

	docker compose up -d

> Si vous utilisez greenlight, vous pouvez créer un compte admin comme ceci:

	docker compose exec greenlight bundle exec rake admin:create

## Gestion des erreurs

### Page Blanche 
Si lors du chargement de la page d'accueil de BigBlueButton (Greenlight), vous obtenez une page blanche, cela peut-être dû à une mauvaise configuration de la base de données PostgreSQL.

Vérifiez les logs générer par le conteneur correspondant à Postgre:

	docker ps -a

![](http://93.90.205.194/docs/bbb_install/docker-ps-a.png)
Puis:

	docker logs [Nom_du_conteneur]
	docker logs docker-postgres-1
> Si vous constatez des erreurs d'accès, de mot de passe incorrect ou de connexions refusé, il faudra se connecter au conteneur pour tester l'accessibilité.

	docker exec -it -u root docker-postgres-1 bash
Allez ensuite dans la config de gestion des accès hôtes:

	find / -name pg_hba.conf
![](http://93.90.205.194/docs/bbb_install/postgre-debug/docker-exec-cat-pg_hba.png)![](http://93.90.205.194/docs/bbb_install/postgre-debug/cat_pg_hba_md5.png)
> [Attention] Vous devrez gérer vôtre propre configuration pour qu'elle soit sécurisé au mieux et adapté à vos besoins. Il ne s'agit là que de débug pour s'assurer que le problème vienne bien d'ici.
> Si vous avez une METHOD qui n'est pas à trust mais à MD5, c'est qu'un mot de passe a été défini et que le hash n'est pas correct, pour s'en assurer remplacer MD5 par trust (à remettre à l'état initial après test pour des raisons évidentes de sécurité)

	# Remplace md5 par trust dans toutes les occurrences qu'il rencontre
	sed -i "s/md5/trust/g" /var/lib/postgresql/data/pg_hba.conf

Relancer ensuite votre compose et réessayer:

	docker compose restart

### Périphérique Média (Micro, Audio, Vidéo) non fonctionnel 
Si tous fonctionne sauf l'utilisation de vos périphériques médias lors de réunions. C'est certainement un problème lié au routage UDP et donc les services telles que WebRTC, Coturn et éventuellement Freeswitch qui sont impacté. Cela peut signifier que vous avez mal configuré vôtre environnement en fonction de l'IP Publique, du NAT, et de CoTURN.
Comme pour PostrgeSQL juste au dessus, vérifier les logs de ces conteneurs pour voir si ils indiquent une erreur vous permettant d'éclaircir la problématique. Dans la plupart vous pouvez utiliser la commande "jq" qui formatera vos logs de façon bien plus lisible.

	sudo apt install jq
	docker logs docker-webrtc-sfu-1 | jq
	# Si une erreur de formatage pour webRtc survient,
	# vous pouvez essayer de n'afficher que les derniers logs
	docker logs docker-webrtc-sfu-1 --tail 20 | jq

> Logs sans jq
> 
> ![](http://93.90.205.194/docs/bbb_install/coturn-debug/log-sans-jq.png)

> Logs avec jq
> 
> ![](http://93.90.205.194/docs/bbb_install/coturn-debug/log-avec-jq.png)
