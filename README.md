# Guide de déploiement d'un lab ADCS intégrant ESC12, ESC13 et ESC14

Ces publications sont fournies à titre éducatif et à des fins de recherche uniquement.  
Leur utilisation à des fins malveillantes, illégales ou non autorisées est strictement prohibée. 
Deloitte décline toute responsabilité pour tout dommage ou conséquence découlant de l'utilisation de ces publications à d'autres fins que celles décrites ci-avant. 
L'utilisateur est seul responsable de se conformer aux lois et réglementations applicables.

Le guide ci-dessous décrit la procédure à suivre afin de déployer **manuellement** un lab Active Directory permettant l'exploitation de l'ESC12, l'ESC13 et l'ESC14.

Il est bien sûr possible d'automatiser ce déploiement, mais monter manuellement son lab permet à mon sens de mieux comprendre les vulnérabilités que l'on y introduit.

## Prérequis

Avant de pouvoir introduire des vulnérabilités dans notre lab, nous avons d'abord besoin d'un Active Directory fonctionnel.
Nous ne rentrerons pas dans le détail de son déploiement ici, de nombreuses ressources disponibles en ligne l'expliquant déjà très bien.

Le lab que j'utilise est composé d'un contrôleur de domaine Windows Server 2022 DC-ESC.esc.local (192.168.1.2) virtualisé sur VirtualBox. Ce serveur est également configuré comme serveur DHCP, serveur DNS et autorité de certification afin de simplifier l'architecture.

## ESC12

Il existe un [guide d'utilisateur](https://docs.yubico.com/hardware/yubihsm-2/hsm-2-user-guide/hsm2-adcs-deploy.html) mis à disposition par Yubi afin de configurer une PKI s'appuyant sur la YubiHSMv2.
Ce guide est relativement détaillé et fournit toutes les informations qui nous seront nécessaires afin de déployer notre lab. Une version résumée et imagée des actions à réaliser est toutefois disponible ci-dessous.


La première étape consiste à connecter la YubiHSMv2 à notre contrôleur de domaine.  
Dans le cadre d'un DC virtualisé à l'aide de VirtualBox, il est d'abord nécessaire d'installer le "Virtual Box Extension Pack" associé à notre version de VBox pour permettre à notre machine virtuelle de communiquer via USB.


Il nous faut ensuite installer les éléments nécessaires à la gestion de l'HSM sur notre contrôleur de domaine.  
Pour cela, il nous faut exécuter les `.msi` `yubihsm-cngprovider-windows-amd64.msi`, `yubihsm-connector-windows-amd64.msi` et `yubihsm-shell-x64.msi` disponibles depuis la page suivante : https://developers.yubico.com/YubiHSM2/Releases/.


Si l'installation se déroule correctement, l'entrée de registre `HKLM/Software/Yubico/YubiHSM` devrait être créée et remplie avec des valeurs par défaut.

On peut ensuite re-configurer le rôle Active Directory Certificate Service sur notre contrôleur de domaine en lui spécifiant qu'il doit utiliser l'HSM pour le stockage de la clé privée de l'autorité de certification.
Il faut pour cela suivre les étapes suivantes :

- Ajouter le rôle Active Directory Certificate Service depuis le menu "*Add Roles and Features*" et ajouter la feature "*Certification Authority*"
  
  <img src="/images/ESC12%20-%201.png" alt="ESC12 AD CS setup" width="600">

- Ouvrir le menu *AD CS Configuration* et sélectionner les options suivantes : *Enterprise CA*, *Root CA*, *Create a new private key*
- Lorsque l'option *Select a cryptographic provider* apparaît, sélectionner *RSA#YubiHSM Key Storage Provider* et cocher l'option "*Allow administrator interaction when the private key is accessed by the CA*". Le choix de cette option est recommandé par Yubi car elle permet d'exporter la clée privée de l'autorité de certification sans qu'il soit nécessaire de couper le service au préalable
  
  <img src="/images/ESC12%20-%202.png" alt="Choice of Cryptographic provider" width="600">


*Note : Si l'autorité de certification a déjà été configurée sans YubiHSMv2, il peut être plus simple de retirer le rôle AD CS du serveur et suivre les étapes ci-dessus.*

Si jamais vous rencontrez une erreur de type *KDC_ERR_PADATA_TYPE_NOSUPP*, vous pouvez vous référer aux liens suivants afin de résoudre le problème : 
- https://www.github.com/ly4k/Certipy/issues/64#issuecomment-1199251655
- https://http418infosec.com/ad-certificate-services-the-basics#KRB-ERROR_16_KDC_ERR_PADATA_TYPE_NOSUPP

## ESC13

Il nous faut tout d'abord créer un nouveau groupe de sécurité **universel** vide, par exemple *Universal Admins*.
Pour cela, nous devons ouvrir l'Active Directory Users and Computers et spécifier les options suivantes :

<img src="/images/ESC13%20-%201.png" alt="Creation of Universal Admins group" width="600">

On peut ensuite faire un clic-droit sur `Universal Admins` et sélectionner *Add to a group* pour l'ajouter au groupe `Enterprise Admins`.


Il nous faut maintenant ajouter une stratégie d'émission sur l'un des modèles de certificat :

- Lancer `certsrv.msc`, clic-droit sur *Certificate Templates* et sélectionner l'option *Manage*
- Dupliquer le modèle de certificat `User` en modifiant son nom, par exemple ESC13_Template
- Faire un clic-droit sur `ESC13_Template` et choisir les options suivantes : *Extensions*, *Issuance Policies*, *Edit*, *Add*, *New*. On peut ensuite donner un nom à notre OID, par exemple OID_ESC13
  
	<img src="/images/ESC13%20-%202.png" alt="Creation of ESC13 Template" height="500">

- Faire un clic-droit sur *Certificate Templates* dans la fenêtre de `certsrv.msc` et sélectionner *New*, *Certificate Template to Issue*, `ESC13_Template`

La stratégie d'émission est ajoutée sur le modèle de certificat `ESC13_Template`, mais il nous faut maintenant la lier à notre groupe `Universal Admins`.

- Ouvrir l'ADSI Edit depuis le contrôleur de domaine, faire un clic-droit sur *ADSI Edit* sélectionner *Connect to* en laissant les valeurs par défaut, et noter le *distinguished name* dans les propriétés de notre groupe `Universal Admins`
- Retrouver notre stratégie d'émission et identifier son `Name` en utilisant la commande suivante (à adapter en fonction du nom donné au domaine) :
  ```Get-ADObject -Filter * -SearchBase "CN=OID,CN=Public Key Services,CN=Services,CN=Configuration,DC=esc,DC=local" -Properties DisplayName,msPKI-Cert-Template-OID```
   
- Ouvrir à nouveau l'ADSI Edit et sélectionner *Connect to* en remplaçant le *Default naming context* par *Configuration*. Identifier notre stratégie d'émission à partir de son `Name` puis faire un clic-droit, sélectionner l'option *Properties* et modifier l'attribut `msDS-OIDToGroupLink` en y spécifiant le `distinguished name` du groupe `Universal Admins`

	<img src="/images/ESC13%20-%203.png" alt="Modification of OID Group Link" height="600">

*Si l'ESC13 n'est pas détectée lors de l'utilisation de `certipy find`, redémarrez votre contrôleur de domaine.*

## ESC14 - Scénario A (permissions d'écriture sur altSecurityIdentities)

Afin de pouvoir mettre en place ce scénario, nous avons besoin de créer 2 utilisateurs :
- Un utilisateur correspondant à un compte compromis par l'attaquant, par exemple `compromised_user`
- Un utilisateur cible, par exemple `user_esc14_A`. Cet utilisateur doit faire partie d'un groupe à privilèges afin de rendre l'impact de sa compromission plus visible

Il nous faut donner à notre utilisateur `compromised_user` les permissions d'écrire sur l'attribut altSecurityIdentities de `user_esc14_A` :

- Ouvrir l'Active Directory Users and Computers et activer l'affichage des *Advanced Features* dans le menu *View*
- Ouvrir les propriétés de `user_esc14_A` et accéder à l'onglet *Security*
- Cliquer sur *Add* et ajouter l'utilisateur `compromised_user`. Celui-ci devrait à présent avoir des permissions en lecture seule
- Ouvrir le menu *Advanced*, sélectionner la ligne propre à l'utilisateur `compromised_user`, cliquer sur *Edit*, cocher *Write altSecurityIdentities* et valider les modifications
