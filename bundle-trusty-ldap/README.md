# 5 Minutes Stacks, Episode neuf: LDAP

**Draft - WIP et Image non encore disponible...**

## Episode neuf: LDAP

Alors... LDAP...

En suivant ce tutoriel, vous obtiendrez une instance Ubuntu Trusty Tahr, pré-configurée avec LDAP sur son propre réseau. 
Ce réseau de LDAP (et un autre réseau de votre choix) sera relié par un routeur Neutron, permettant un accès sécurisé et privé depuis vos autres instances à LDAP.
Comme pour l’application GitLab, toutes les données du LDAP seront stockées dans un volume dédié qui pourra être sauvegardé sur demande.
Par défaut, l’application LAM (LDAP Access Manager) sera également exécutée sur l'instance et accessible par port 443 via HTTPS (automatiquement redirigé depuis le port 80).
A travers l’application LAM vous serez en mesure de modifier votre base de données LDAP rapidement et efficacement grâce à une interface intuitive même pour les moins aguerris à LDAP.

## Préparations

### Les versions

* LDAP (slapd, ldap-utils) 2.4.31-1
* LAM (ldap-account-manager) 4.4-1

### Les pré-requis pour déployer cette stack

Ce devrait être la routine maintenant :
* un accès internet
* un shell Linux
* un [compte Cloudwatt](https://www.cloudwatt.com/authentification), avec une [paire de clés existante](https://console.cloudwatt.com/project/access_and_security/?tab=access_security_tabs__keypairs_tab)
* les outils [OpenStack CLI](http://docs.openstack.org/cli-reference/content/install_clients.html)
* un clone local du dépôt git [Cloudwatt applications](https://github.com/cloudwatt/applications)
* The ID of an Neutron Subnet containing servers who need to connect to your LDAP instance.

### Taille de l'instance

Par défaut, le script propose un déploiement sur une instance de type " Small " (s1.cw.small-1) en tarification à l'usage (les prix à l'heure et au mois sont disponibles sur la [page Tarifs](https://www.cloudwatt.com/fr/produits/tarifs.html) du site de Cloudwatt). Bien sûr, vous pouvez ajuster les paramètres de la stack et en particulier sa taille par défaut.
En outre, notre stack LDAP est adaptée pour faire bon usage du stockage bloc Cinder. Cela garantit la protection de vos projets et vous permet de ne payer que pour l'espace que vous utilisez. La taille du volume peut être ajustée dans la console. La stack GitLab peut supporter des dizaines à des teraoctets d'espace de stockage.
Les paramètres de la stack sont bien sur modifiables à volonté.

### Au fait...

As with the GitLab Bundle, the `.restore` heat template and `backup.sh` script enable you to manipulate Cinder Volume Storage. With these files, you may create Cinder Volume Backups: Save states of your LDAP stack's volume for you to redeploy with the `.restore` heat template when needed.

Both normal and 'restored' stacks can be launched from the [console](#console), but our nifty `stack-start.sh` also allows you to create both kinds of stacks easily from a [terminal](#startup).

Backups must be initialized with our handy `backup.sh` script and take a curt 5 minutes from start to full return of functionality. [(More about backing up and restoring your LDAP...)](#backup)

## Tour du propriétaire

Une fois le répertoire cloné, vous trouvez, dans le répertoire `bundle-trusty-ldap/`:
* `bundle-trusty-gitlab.ldap.yml` : Template d'orchestration HEAT, qui va servir à déployer l'infrastructure nécessaire.
* `bundle-trusty-ldap.restore.heat.yml` : Template d’orchestration HEAT. Il déploie l’infrastructure nécessaire et restaure vos données depuis un backup !
* `backup.sh` : Script de création de sauvegarde. Ce script magique permet la sauvegarde de vos données dans un volume de stockage bloc prêt à l’emploi en cas de malheur (redéploiement en seulement 5 minutes)
* `stack-start.sh` : Script de lancement de la stack. C'est un micro-script pour vous économiser quelques copier-coller.
* `stack-get-url.sh` : Script de récupération de l'IP d'entrée de votre stack.

## Démarrage

### Initialiser l'environnement

Munissez-vous de vos identifiants Cloudwatt, et cliquez [ICI](https://console.cloudwatt.com/project/access_and_security/api_access/openrc/). Si vous n'êtes pas connecté, vous passerez par l'écran d'authentification, puis vous le téléchargement d'un script démarrera. C'est grâce à celui-ci que vous pourrez initialiser les accès shell aux API Cloudwatt.
Source the downloaded file in your shell and enter your password when prompted to begin using the OpenStack clients.

~~~ bash
$ source COMPUTE-[...]-openrc.sh
Please enter your OpenStack Password:

$ [whatever mind-blowing stuff you have planned...]
~~~

Une fois ceci fait, les outils ligne de commande OpenStack peuvent interagir avec votre compte Cloudwatt.

### Ajuster les paramètres

Dans le fichier `bundle-trusty-gitlab.ldap.yml` vous trouverez en haut une section `parameters`. Les deux seuls paramètres obligatoires à ajuster sont 
* celui nommé `keypair_name` dont la valeur `default` doit contenir le nom d'une paire de clés valide dans votre compte utilisateur
* et celui nommé `subnet_id`

C'est dans ce même fichier que vous pouvez ajuster la taille de l'instance, la taille du volume, et le type du stockage en jouant sur les paramètres `flavor`, `volume_size` et `volume_type`.

Par défaut, un réseau et un sous-réseau sont automatiquement créés sur lesquels réside le serveur LDAP. Ce comportement peut être modifié aussi en éditant le fichier ` bundle-trusty-ldap.heat.yml`. Toutefois cela peut poser des problèmes de sécurité.

~~~ yaml
heat_template_version: 2013-05-23


description: All-in-one LDAP stack with LDAP Account Manager


parameters:
  keypair_name:
    default: my-keypair-name                <-- Indiquer ici votre paire de clés par défaut
    label: Paire de cles SSH - SSH Keypair
    description: Paire de cles a injecter dans instance - Keypair to inject in instance
    type: string

  flavor_name:
    default: s1.cw.small-1                  <-- Indiquer ici la taille de l’instance par défaut
    label: Type instance - Instance Type (Flavor)
    description: Type instance a deployer - Flavor to use for the deployed instance
    type: string
    constraints:
      - allowed_values:
        [...]

  subnet_id:
    label: Subnet ID
    description: Subnet to attach to router to allow connection to LDAP
    type: string

  volume_size:
    default: 10                             <-- Indiquer ici la taille du volume par défaut
    label: LDAP Volume Size
    description: Size of Volume for LDAP Storage (Gigabytes)
    type: number
    constraints:
      - range: { min: 2, max: 1000 }
        description: Volume must be at least 2 gigabytes

  volume_type:
    default: standard                       <-- Indiquer ici le type du volume par défaut
    label: GitLab Volume Type
    description: Performance flavor of the linked Volume for LDAP Storage
    type: string
    constraints:
      - allowed_values:
          - standard
          - performant

resources:
  network:
    type: OS::Neutron::Net

  subnet:
    type: OS::Neutron::Subnet
    properties:
      network_id: { get_resource: network }
      ip_version: 4
      cidr: 10.27.27.0/24
      allocation_pools:
        - { start: 10.27.27.100, end: 10.27.27.199 }
[...]
~~~

<a name="startup" />

### Démarrer la stack

Dans un shell, lancer le script `stack-start.sh` :

~~~ bash
$ ./stack-start.sh DAPPER «subnet-id» «my-keypair-name»
+--------------------------------------+------------+--------------------+----------------------+
| id                                   | stack_name | stack_status       | creation_time        |
+--------------------------------------+------------+--------------------+----------------------+
| xixixx-xixxi-ixixi-xiixxxi-ixxxixixi | DAPPER     | CREATE_IN_PROGRESS | 2025-10-23T07:27:69Z |
+--------------------------------------+------------+--------------------+----------------------+
~~~

Attendez **5 minutes** que le déploiement soit complet (vous pouvez utiliser la commande `watch` pour voir le statut en temps réel).

~~~ bash
$ watch -n 1 heat stack-list
+--------------------------------------+------------+-----------------+----------------------+
| id                                   | stack_name | stack_status    | creation_time        |
+--------------------------------------+------------+-----------------+----------------------+
| xixixx-xixxi-ixixi-xiixxxi-ixxxixixi | DAPPER     | CREATE_COMPLETE | 2025-10-23T07:27:69Z |
+--------------------------------------+------------+-----------------+----------------------+
~~~

### Enjoy

Une fois tout ceci fait, vous pouvez lancez le script `stack-get-url.sh`.

~~~ bash
$ ./stack-get-url.sh DAPPER
DAPPER  http://70.60.637.17
~~~

qui va récupérer l'IP flottante attribuée à votre stack. Vous pouvez alors attaquer cette IP avec votre navigateur préféré et confirmer votre intérêt en **acceptant le certificat de sécurité**.

## Dans les coulisses

Le script `start-stack.sh` s'occupe de lancer les appels nécessaires sur les API Cloudwatt pour exécuter le script heat :

* démarrer une instance basée sur Ubuntu Trusty Tahr
* attaché une IP flottante pour l’application LAM
* démarrer, attacher et formater un volume de stockage pour LDAP ou en restaurer un depuis <<< a provided backup_id>>>
* reconfigurer LDAP pour que le stockage des données se fasse sur le volume
* Ecrire un guide simple dans sorties de la stack (comment utiliser LDAP depuis un server d’un sous-réseau) <<< how to use the LDAP server from the provided subnet's servers>>>

<a name="console" />

### Bash is what I do with my head against a wall when I think of shell.

That's okay! But not actually. Sorry folks, but LDAP is not for kids: if you want to play, better get used to that blocky cursor. The actual creation of the stack can be done entirely from our console, but to allow other servers in the provided subnet to connect, you're going to have to connect to those servers in SSH.

En utilisant la console, vous pouvez déployer votre serveur LDAP :
1.	Allez sur le Github Cloudwatt dans le répertoire [applications/bundle-trusty-ldap](https://github.com/cloudwatt/applications/tree/master/bundle-trusty-ldap)
2.	Cliquez sur le fichier nommé `bundle-trusty-ldap.heat.yml` ou `bundle-trusty-ldap.restore.heat.yml` pour [restaurer depuis une sauvegarde](#backup))
3.	Cliquez sur RAW, une page web apparait avec le détail du script
4.	Enregistrez-sous le contenu sur votre PC dans un fichier avec le nom proposé par votre navigateur (enlever le .txt à la fin)
5.	Rendez-vous à la section « [Stacks](https://console.cloudwatt.com/project/stacks/) » de la console.
6.	Cliquez sur « Lancer la stack », puis cliquez sur « fichier du modèle » et sélectionnez le fichier que vous venez de sauvegarder sur votre PC, puis cliquez sur « SUIVANT »
7.	Donnez un nom à votre stack dans le champ « Nom de la stack »
8.	Entrez votre keypair dans le champ « keypair_name »
9.	Confirmez la taille du volume de stockage (en Go) dans le champ « LDAP Volume Size »
10.	Choisissez la taille de votre instance parmi le menu déroulant « flavor_name » et cliquez sur « LANCER »

La stack va se créer automatiquement (vous pouvez en voir la progression cliquant sur son nom). Quand tous les modules deviendront « verts », la création sera terminée. Vous pourrez alors aller dans le menu « Instances » pour découvrir l’IP flottante qui a été générée automatiquement ou simplement recharger la page et allez dans l’onglet « vue d’ensemble ». Ne vous reste plus qu’à lancer votre IP dans votre navigateur.

C’est (déjà) FINI ! A vous de jouer maintenant.

Is what I wish I could say. At this point LDAP will be running, and LAM will work and modify the LDAP database, but no other server has access! The next section covers how to gain access to LDAP from a server in the provided subnet.

## So you wanna see my LDAP?

Thankfully, I made it easy. In the output of the stack, either in the Overview tab of your stack's page on the console, or with:

~~~ bash
$ heat output-show «stack-name» --all
~~~

You will find three steps, not necessarily in the correct order, to aid you on your quest for booty.
Booty, of course, being access to your new LDAP database from a server in the provided subnet.
The steps should look similar to this:

##

**router-interface-ip** : Find given subnet's router-interface IP with:

~~~ bash
$ neutron router-port-list «ldap-router-name» | grep «provided-subnet-id» | cut -d\"\\\"\" -f8
~~~

**ldap_ip_address_via_router** : Once access to LDAP has been configured as shown here,

LDAP will then be accessible from ldap://«ldap-through-router-ip»:389

**ldap_access_configuration** : From the SSH terminal of any server in the subnet (passed as parameter)add access to LDAP with:

~~~ bash
$ sudo ip route add «ldap-through-router-ip» via «router-interface-ip»
~~~

**floating_ip_url** : LDAP Account Manager External URL

http://«lam-public-ip»/

##

For those who like a little more instruction in their instructions:

1.  First, from the terminal on your computer, run the command to obtain the *router-interface-ip*, which you will need in the next few steps. The variables will already be inserted, so you can just copy-paste the line into the terminal. Neat right?
2. Armed with the *router-interface-ip*, connect with SSH to a server in the provided subnet and enter the *ldap_access_configuration* command. Running `sudo ip route` will show you the route that was just added, if you're curious.
3. Verify that you can actually connect to LDAP by running `curl «ldap_ip_address_via_router»` with the *ldap_ip_address_via_router* found in the output.

Voila! You can now securely access LDAP from your other servers. LAM is public, however, and can be used through any browser from the *floating_ip_url*.

<a name="backup" />

## LAM (Light Attack Munitions)

Except not really: LDAP Account Manager presents your LDAP database as if it were a user management tool, one of it's most common uses. It also provides a more development/broad-use oriented interface, but for most users the main panel is best, as it is much more intuitive and requires little to no knowledge of LDAP.

By default, the password for the main login (username: Administrator), the *master* password, and the *server preferences* password are all **c10udw477**, but each can be changed individually.

Try it out, tweak it's configuration, read it's manual if you wish: LAM is very modular and I'm sure it can be easily structured to meet any needs you may have in an LDAP GUI. However if you don't want it, or if fool-proof security is very important to you, can always just disassociate its floating-IP, either from the console in the «Access & Security» tab, or from the command line.

~~~ bash
$ nova help floating-ip-disassociate
~~~

<a name="backup" />

## Sauvegarde et Restauration

Sauvegarder votre travail semble une bonne idée, n’est-ce pas ? Nous avons développé une méthode vous permettant de sauvegarder votre travail rapidement et facilement.

~~~ bash
$ ./backup.sh DAPPER
~~~

Et 5 minutes plus tard, vous retrouvez votre environnement LDAP.
La restauration est aussi simple que reconstruire une nouvelle stack mais cette fois avec le fichier heat `.restore.heat.yml`, et en indiquant l’ID de la sauvegarde que vous souhaitez restaurer. Naturellement, la taille du nouveau volume ne doit pas être plus petite que l’original pour éviter la perte de données. Vous pouvez voir la liste des sauvegardes dans la console dans l’onglet « Sauvegardes de Volume » du menu « Volume » ou par les lignes de commandes :

~~~ bash
$ cinder backup-list

+------+-----------+-----------+-----------------------------------+------+--------------+---------------+
|  ID  | Volume ID |   Status  |                Name               | Size | Object Count |   Container   |
+------+-----------+-----------+-----------------------------------+------+--------------+---------------+
| XXXX | XXXXXXXXX | available | ldap-backup-2025/10/23-07:27:69   |  10  |     206      | volumebackups |
+------+-----------+-----------+-----------------------------------+------+--------------+---------------+
~~~

Toutefois, notez que même si cette méthode permet de restaurer facilement votre service, <<<your local Git tools>>>  ne prendra pas en compte le changement d’adresse IP. Votre paire de clés SS reste valide mais vous devez vous assurer de **corriger any hosts and project remote addresses ** avant de continuer votre travail.

## So watt?

Ce tutoriel a pour but d'accélérer votre démarrage. A ce stade **vous** êtes maître(sse) à bord.

Vous avez un point d'entrée sur votre machine virtuelle en ssh via l'IP flottante exposée et votre clé privée (utilisateur `cloud` par défaut).

Les chemins intéressants sur votre machine :

- `/dev/vdb`: Volume mount point
- `/mnt/vdb/`: `/dev/vdb` mounts here
- `/mnt/vdb/ldap/`: Mounts onto `/var/lib/ldap/` and contains all of LDAP's data
- `/var/lib/ldap/`: LDAP stores its database here; `/mnt/vdb/ldap/` mounts here
- `/etc/ldap`: LDAP Configuration; only the `ssl/` directory is backed up in the volume here, however this only contains the config database: LDAP accounts and other user-created data are stored in the database at `/var/lib/ldap/`, and are properly stored in the volume
- `/etc/ldap/ssl/`: Apache's '.key' and '.crt' for HTTPS
- `/etc/ldap/ldap-volume.sh`: Script run upon reboot to remount the volume and verify LDAP is ready to function again
- `/usr/share/ldap-account-manager`: Root of LAM's site; some LAM configuration
- `/etc/ldap-account-manager`: More LAM configuration files
- `/var/lib/ldap-account-manager`: LAM's working directory (config, sessions, temporary files)
- `/var/lib/ldap-account-manager/config/`: Another set of LAM configuration files
- `/etc/ldap-account-manager/apache.conf`: `/etc/apache2/conf-available/ldap-account-manager.conf` redirects here; contains Apache-relevant configuration for LAM
- `/etc/apache2/sites-available/ssl-lam.conf`: Apache site configuration for LAM through SSL. Enabled by default.
- `/etc/apache2/sites-available/http-lam.conf`: Apache site configuration for LAM through HTTP. Disabled by default.
- `/mnt/vdb/stack_public_entry_point`: Contains last known IP address, used to replace the IP address in all locations when it changes

Quelques ressources qui pourraient vous intéresser :

* [Ubuntu OpenLDAP Server Guide](https://help.ubuntu.com/lts/serverguide/openldap-server.html)
* [LAM Manual](https://www.ldap-account-manager.org/static/doc/manual-onePage/index.html)
* [OpenLDAP Documentation Catalog](http://www.openldap.org/doc/)
* [OpenLDAP  Independent Publications](http://www.openldap.org/pub/)
* [Online OpenLDAP Man Pages](http://www.openldap.org/software/man.cgi)


-----
Have fun. Hack in peace.