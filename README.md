# ssh-ec2
Un utilitaire pour augmenter la puissance de sa machine, grâce à AWS.

## Qu'est-ce que c'est ?
`ssh-ec2` est un script `bash`, permettant de faciliter l'utilisation du cloud AWS pour la réalisation de
calculs complexes avec des instances puissantes, équipées de GPU ou de beaucoups de mémoire. L'outil se charge de :
- créer et lancer une instance éphémère EC2
- répliquer le répertoire courant sur l'instance
- lancer un SSH dans la copie du répertoire
    - soit avec un `bash`
    - soit pour déclencher un traitement
- de récupérer tous les résultats
- et de détruire l'instance.

## Fonctionnalités
Cet outil offre :
- la possibilité d'indiquer le cycle de vie de l'instance à la fin du traitement
    - l'interrompre et la détuire (pas de frais supplémentaires)
    - la laisser vivante (des frais supplémentaires sont appliqués par Amazon),
    - la sauvegarder et l'arreter (des frais moins élevés sont appliqués par Amazon)
- de choisir l'image AMI à utiliser, le type de l'instance, la clé SSH, la région, etc.
- Offre différents moyens de gérer le multi-session sur l'instance
    - pas de multi-sessions (par défaut)
    - Tmux
    - Screen
- Synchronise
    - Avant le traitement, le répertoire courant (hors les fichiers et répertoires indiqués dans `.rsyncignore`)
sur l'instance EC2, avant de lancer le traitement directement dans le répertoire synchronisé
    - Après le traitement, pour récupérer tous les nouveaux fichiers ou les fichiers modifiés
- De limiter les droits accordés à l'instance
- De permettre un accès direct aux fichiers S3 (et au autres services AWS),
sans devoir injecter les *credentials* dans l'instance (`aws s3 ls s3://mybucket`)
- D'ajouter des tags aux instances
- ...

Il permet d'ignorer :
- l'AMI spécifique à la région pour l'image à utiliser
- l'adresse IP de l'instance
- le `user` à utiliser pour se connecter (`ubuntu`, `ec2-user`, ...)
- le changement de répertoire un fois connecté (`cd prj`)
- l'activation de `tmux` ou de `screen` pour reprendre la main (`tmux a`)
- la synchronisation des fichiers

## Installation
Pour installer l'outil, il faut cloner le repo
```bash
$ git clone https://gitlab.octo.com/pprados/ssh-ec2.git
$ cd ssh-ec2
```
Sous MacOS, assurez-vous d'avoir un programme `make` de version 4+.
```bash
make --version
```
Si ce n'est pas le cas, mettez la version à jour.
```bash
brew install make
```

Puis installez:
- soit un lien symbolique vers le source (`make install-with-ln`) pour
bénéficier des mises à jours du repo (mais il ne faut plus supprimer les sources)
- soit faire un copie dans `/usr/local/bin` (`make install`)

Une fois cela effectué, l'outil est disponible dans tous les projets, sans rien installer de plus.
```bash
$ cd mon_projet
$ ssh-ec2 # Create EC2 instance, duplicate directory, start ssh, synchronize result, terminate instance
```

## Pré-requis sur AWS pour l'utilisateur
Voir [utilisateur](user.md)

## Pré-requis sur AWS pour l'administrateur
Voir [administrateur](admin.md)

## Utilisation
Le programme est paramétrable via la ligne de commande ou à l'aide de variables d'environements.

Une fois installé correctement dans l'OS (voir plus haut), la commande est disponible dans tous les répertoires.
Depuis un répertoire racine de projet, vous pouvez l'invoquer sans autre modification.

Par la suite, vous pouvez créer un fichier `.env` (voir plus loin)
avec les paramètres spécifiques (région, type d'instance, etc.) et/ou
un fichier `.rsyncignore` pour indiquer les répertoires à ne pas dupliquer sur l'instance EC2
lors de l'invocation de l'outil (pour des raisons d'optimisations uniquement).

Voici les valeurs par défaut des principaux paramètres proposés par l'outil.

| Paramètre                                 | Valeur par défaut           |
|:------------------------------------------|:----------------------------|
| Region (`AWS_REGION`)                     | eu-central-1                |
| Type d'instance (`AWS_INSTANCE_TYPE`)     | p2.xlarge                   |
| Image (`AWS_IMAGE_NAME`)                  | Deep Learning AMI (Ubuntu)* |
| Profile (`AWS_IAM_INSTANCE_PROFILE`)      | EC2ReadOnlyAccessToS3       |
| Initialisation de la VM (`AWS_USER_DATA`) | ''                          |

Notez que si vous avez indiqué une région préféré lors de l'installation de AWS CLI, elle est utilisé
par défaut (fichier `~/.aws/config`) en lieu et place de `eu-central-1`.

Notez que pour ajouter un script d'initialisation de la VM, exécuté lors de sa création, il faut :
- soit utiliser un fichier (`file://....sh`)
- soit utiliser un script complet **AVEC** shebang

```bash
$ export AWS_USER_DATA='#!/usr/bin/env bash
source activate $(VENV_AWS)
conda install 'make>=4' -y'

$ echo -e "$AWS_USER_DATA"
```
Vous pouvez modifier les valeurs par défaut en déclarant des variables d'environnements
```bash
export AWS_INSTANCE_TYPE=p2.xlarge
export AWS_IMAGE_NAME="Deep Learning AMI (Amazon Linux)*"
export AWS_REGION=eu-west-1
```
Ces variables sont prioritaires aux autres paramètres.

Vous pouvez également valoriser ces variables dans un fichier `.env`
```bash
# File .env
AWS_INSTANCE_TYPE=t2.small
AWS_IMAGE_NAME=Deep Learning AMI (Amazon Linux)*
AWS_REGION=eu-west-1
```
Dans ce cas, ces variables sont moins prioritaires que les paramètres valorisés
en dehors. Par exemple :
```bash
AWS_REGION=eu-east-1 ssh-ec2
```
permet de forcer une autre région.

Plus d'informations sont présentes dans le source de `ssh-ec2`.

Pour avoir un rapide rappel des paramètres de la ligne de commande `--help` :
```bash
$ ssh-ec2 --help
ssh-ec2 [-lsr|--leave|--stop|--terminate] [-daf|--detach|--attach|--finish] [--no-rsync] \
 [--multi <tmux|screen|none>] \
  [-v|--verbose] \
 [-i <pem file>] [-[LR] port:host:port] [cmds*]
```

### Fin de vie de l'instance EC2
Un simple lancement de `ssh-ec2` permet d'avoir une session `bash`
sur une instance EC2 qui sera détruite à la sortie du terminal.

#### Utilisation synchrone
Les paramètres sont interprétés comme une commande à exécuter (comme avec un `ssh` classique).
```bash
ssh-ec2 who am i # Invoke 'who am i' on EC2

----------------------------------------------------------
EC2 Name: PPR-ssh-ec2
Region:   eu-central-1
Type:     p2.xlarge
Image Id: ami-002b6c63ff04afa5f (Deep Learning AMI (Ubuntu)*)
Key name: PPR
Tags:     User=pprados, Name=PPR-ssh-ec2, Trigram=PPR, Hostname=PPR-OCTO
----------------------------------------------------------
Synchronizes current directory (except files in .rsyncignore)...
ssh ubuntu@52.57.165.31 ...

ubuntu   pts/0        2019-03-11 15:31 (82.238.92.100)

Synchronizes result... done
```

Pour indiquer le comportement que doit avoir l'instance EC2 à la fin de la session,
il faut utiliser
- `--leave` (ou `-l`) pour laisser instance vivante
- `--stop` (ou `-s`) pour la sauvegarder et l'arréter
- ou `--terminate` (ou `-t` - par défaut) pour la supprimer.
```bash
ssh-ec2 --stop "source activate cntk_p36 ; VENV=cntk_p36 make train" # Sauve et arrète l'instance après le traitement
```

Il est possible de rattraper une instance qui à vocation a être interrompu avec un `--stop` ou un `--terminate`.
Pour cela, il faut lancer en parallèle une autre session avec une autre règle de terminaison,
avant d'interrompre la première.
Le programme applique la gestion du cycle de vie de l'instance EC2
s'il est le dernier à y avoir accès.

#### Utilisation asynchrone
Il est possible de lancer une commande et de détacher imédiatement le terminal. L'instance reste vivante.
```bash
ssh-ec2 --detach 'while sleep 1; do echo $(date) thinking ; done' # Return immediately
```
un appel avec `--attach` permet de se rattacher au terminal
```bash
ssh-ec2 --attach
```
Suivant l'utilitaire de multiplexage, il faut utiliser `Ctrl-B d` (tmux - défaut) ou `Ctrl-A d` (screen) pour
se détacher de la session en la laissant vivante.
Référez-vous à la documentation de ces outils pour en savoir plus.
`ssh-ec2` se charge d'utiliser au mieux ces outils.
Par défaut, lors de l'invocation avec un `--detach`, `tmux` est utilisé.

Pour récupérer les résultats d'une instance détachée, il faut s'y rattacher avec
```bash
ssh-ec2 --finish # Synchronize results at end
```
au terme de cette session, les fichiers sont récupérés et l'instance EC2 suit
son cycle de vie.

| Objectif                                                                   | Procédure                             |
|----------------------------------------------------------------------------|---------------------------------------|
| Lancer un `make` et attendre le résultat pour continuer                    | `ssh-ec2 make`                        |
| Lancer un `make` puis sauver l'état de l'instance et l'arreter             | `ssh-ec2 --stop make`                 |
| Lancer une session SSH sur une instance et la garder vivante               | `ssh-ec2 --leave`                     |
| Lancer un `make` sur une instance et la garder vivante avec tmux           | `ssh-ec2 --leave --multi tmux make`   |
| Lancer un `make` et le laisser continuer détaché                           | `ssh-ec2 --detach make`               |
| Jeter un oeil sur un traitement détaché                                    | `ssh-ec2 --attach`                    |
| Récupérer le résultat d'un traitement détaché puis terminer l'instance EC2 | `ssh-ec2 --finish`                    |
| Récupérer le résultat d'un traitement détaché et garder l'instance EC2     | `ssh-ec2 --finish --leave`            |
| Rattraper une session qui va être tuée à la fin du traitement              | Dans un autre shell `ssh-ec2 --leave` |
| Forcer la suppression d'une instance                                       | `ssh-ec2 :`                           |

### Synchronisation
Parfois, il n'est pas nécessaire de synchroniser les fichiers.
Le paramètre `--no-rsync` permet cela.

Pour avoir un suivi des fichiers synchronisés, il faut ajouter `--verbose` pour voir les détails.

### Paramètres supplémentaires
Il est également possible de faire du reroutage de port.
```bash
ssh-ec2 -L 8888:localhost:8888 "jupyter notebook --NotebookApp.open_browser=False"
```
ou d'indiquer le fichier local de clé à utiliser.
```bash
ssh-ec2 -I my_key.pem "who am i"
```
D'autres paramètres plus subtiles sont documentés dans les sources. Vous ne devriez pas
en avoir besoin.

Pour résumer

| Paramètre     | Impact                                                                                 |
|:--------------|:---------------------------------------------------------------------------------------|
| _rien_        | Lance une session SSH.                                                                 |
| --help        | Affiche un rapide rappel des paramètres.                                               |
| --terminate   | Supprime l'instance EC2 après la session SSH (par défaut).                             |
| --leave       | Garde l'instance EC2 vivante après la session SSH.                                     |
| --stop        | Arrete l'instance EC2 après l'avoir sauvegardée, après la session SSH.                 |
| --detach      | Lance une commande et rend la main immédiatement.                                      |
| --attach      | Se rebranche sur le traitement en cours.                                               |
| --finish      | Se rebranche sur le traitement en cours et synchronise les résultats à la fin.         |
| -L <p:host:p> | Transfert un port local sur un port distant de l'instance, le temps de la session SSH. |
| -R <p:host:p> | Transfert un port distant sur un port local de l'instance, le temps de la session SSH. |
| --no-rsync    | Ne synchronise pas les fichiers du répertoire local avant et après le traitement.      |
| --verbose     | Affiche les fichiers synchronisés                                                      |

## Utilisation dans un Makefile
Sous MacOS, assurez-vous d'avoir un programme `make` de version 4+.
```bash
make --version
```
Si ce n'est pas le cas, mettez la version à jour.
```bash
brew install make
```

Il suffit de quelques recettes complémentaires dans un fichier `Makefile` pour pouvoir exécuter
tous les traitement sur AWS. Consultez le fichier [Makefile](https://gitlab.octo.com/pprados/ssh-ec2/raw/master/Makefile?raw=true).
```bash
ec2-%: ## call make recipe on EC2
	$(VALIDATE_VENV)
	ssh-ec2 $(EC2_LIFE_CYCLE) "source activate $(VENV_AWS) ; LC_ALL="en_US.UTF-8" VENV=$(VENV_AWS) make $(*:ec2-%=%)"
...
```
Avec ces règles, il suffit de préfixer les recettes d'origine pour les exécuter
sur AWS. Par exemple, s'il existe une recette `train` dans le `Makefile`,
```bash
make ec2-train # Start 'make train' on EC2 ephemeral instance.
```
permet d'exécuter la recette `make train` sur l'instance EC2.

Pour rappel, les AMI de machines learnings possèdent déjà des environnements pré-installés
avec des versions optimisées des différents frameworks.

Les recettes précédentes activent l'environnement conda `cntk_p36` avant le traitement.
Les recettes doivent compléter l'environnement si nécessaire (`pip install ...`).
Il est conseillé de rédiger des règles capablent de le détecter
pour installer le nécessaire avant le traitement.
Inspirez vous du [Makefile](https://gitlab.octo.com/pprados/ssh-ec2/raw/master/Makefile?raw=true)
d'exemple fourni avec le projet.

# Faq

## J'ai une erreur de connexion ou de privilège

Assurez-vous :
- d'avoir le Trigram valorisé (`echo $TRIGRAM`)
- d'avoir bien utilisé le bon 'tenant' (le compte [AWS](https://eu-central-1.signin.aws.amazon.com)),
- d'avoir pour chaque [région]((https://eu-central-1.signin.aws.amazon.com)) une clé identifié par votre trigram

Avec la console AWS, créez une instance, puis connectez vous avec
```bash
ssh -i ~/.ssh/$TRIGRAM ubuntu@<adresse ip de l'instance>
```
Tant que cela ne fonctionne pas, il n'est pas nécessaire d'aller plus loin.

## Je n'arrive toujours pas à me connecter
- Essayez d'indiquer le fichier de clé lors du lancement
```bash
$ ssh-ec2 -i ~/.ssh/$TRIGRAM --leave
```
Si cela fonctionne, c'est une bonne nouvelle.
Il faut juste ajouter la clé en mémoire pour qu'elle soit disponible.
- Sous Linux, il n'y a rien à faire ou executez un `ssh-add ~/.ssh/$TRIGRAM`
- Sous Mac, voir [ici](https://apple.stackexchange.com/questions/48502/how-can-i-permanently-add-my-ssh-private-key-to-keychain-so-it-is-automatically)
ou executez un `ssh-add ~/.ssh/$TRIGRAM`

# Bonus

## Mise à jour de `bash`
Pour avoir une version plus à jour de bash sur MacOS:
```bash
$ brew install bash
$ sudo bash -c 'echo /usr/local/bin/bash >> /etc/shells' then chsh -s /usr/local/bin/bash
$ bash --version
GNU bash, version 4+
```

# Contribution
Toutes les contributions et suggestion sont les bienvenues.
Contactez moi (ppr@octo.com) ou soumettez un pull-request (doc comprise).


PPR TODO: tester AWS_USER_DATA=pip install -e .
