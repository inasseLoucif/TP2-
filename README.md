# TP 2 – Réseau AWS : Création et appairage de 2 VPCs  
Trigramme : ILF  

## Objectif du TP
L’objectif de ce TP est de mettre en place deux réseaux privés virtuels (VPC) distincts dans AWS, chacun contenant un sous-réseau public et un sous-réseau privé avec des instances EC2.  
Les instances des sous-réseaux publics servent de bastions SSH pour accéder aux instances privées, et un peering entre les deux VPC permet ensuite aux instances privées de communiquer entre elles en HTTP sur le port 80 uniquement.

## Partie 1 – Création des VPCs
La valeur obtenue dans le tableau de correspondance est `x = 11`.  
Deux VPC ont été créés dans la région `sa-east-1` (South America / São Paulo) :  
- `ILF_VPC1` avec le CIDR `10.11.0.0/16`  
- `ILF_VPC2` avec le CIDR `10.111.0.0/16`  

Pour chaque VPC :  
- Un sous-réseau public `10.11.1.0/24` (respectivement `10.111.1.0/24`) et un sous-réseau privé `10.11.2.0/24` (respectivement `10.111.2.0/24`) ont été créés dans une même zone de disponibilité de São Paulo.  
- Une Internet Gateway a été attachée au VPC et la table de routage du subnet public a été configurée pour sortir vers Internet.  
- Une NAT Gateway a été déployée dans le subnet public afin de permettre aux instances privées d’accéder à Internet tout en restant injoignables depuis l’extérieur.

Les tables de routage ont été ajustées pour que :  
- Les sous-réseaux publics routent vers l’Internet Gateway pour le trafic sortant.  
- Les sous-réseaux privés routent vers la NAT Gateway pour les destinations hors du VPC.

![VPC1 – ILF_VPC1 (10.11.0.0/16)](vpc1.jpg)
![VPC2 – ILF_VPC2 (10.111.0.0/16)](vpc2.jpg)
![Subnets publics/privés et NAT](vpc-and-subnet.jpg)

![Création VPC et subnets](instances-and-ip.jpg)
![Routes et peering VPC2](routes-and-peering.jpg)

## Partie 2 – Instances EC2 et bastions
Dans `ILF_VPC1`, une instance privée `ILF_InstanceVPC1` a été créée dans le subnet `10.11.2.0/24`, et dans `ILF_VPC2`, une instance privée `ILF_InstanceVPC2` a été créée dans le subnet `10.111.2.0/24`.  
Ces instances utilisent une AMI Amazon Linux 2 avec Apache HTTPD déjà préparé comme dans le TP1, afin de disposer rapidement d’un serveur web fonctionnel.

Deux bastions publics ont été créés : `ILF_BastionVPC1` dans le subnet `10.11.1.0/24` et `ILF_BastionVPC2` dans le subnet `10.111.1.0/24`, chacun avec une adresse IP publique.  
Les Security Groups ont été configurés pour :  
- Autoriser SSH (port 22) vers les bastions depuis l’adresse IP d’Ynov.  
- Autoriser SSH uniquement depuis le bastion vers l’instance privée correspondante.  
- Autoriser HTTP (port 80) sur les instances privées, en vue de la communication inter‑VPC.  

La configuration du client SSH a ensuite été modifiée pour effectuer un rebond via les bastions (ProxyJump / ProxyCommand) afin de se connecter aux instances privées sans exposer celles‑ci directement.  
Les connexions SSH aux deux instances privées ont été testées et validées, puis Apache HTTPD a été installé ou démarré si nécessaire pour servir une page sur le port 80.

![Liste des instances et IP](instances-and-ip.jpg)

## Partie 3 – Peering, routage inter‑VPC et tests
Un lien de peering VPC a été créé entre `ILF_VPC1` et `ILF_VPC2`, puis accepté dans la console.  
Ce peering permet aux deux VPC de s’échanger du trafic privé dans la région de São Paulo sans passer par Internet.

Les tables de routage des subnets privés ont été mises à jour pour prendre en compte les CIDR du VPC distant :  
- Dans la table de routage privée de `ILF_VPC1`, une route vers `10.111.0.0/16` pointe vers la connexion de peering.  
- Dans la table de routage privée de `ILF_VPC2`, une route vers `10.11.0.0/16` a été ajoutée via le même peering.

Les Security Groups des instances privées ont ensuite été ajustés pour autoriser le trafic HTTP (port 80) en provenance du CIDR privé de l’autre VPC, tout en gardant les autres ports fermés.  
Les tests ont été réalisés en se connectant en SSH sur `ILF_InstanceVPC1` via `ILF_BastionVPC1`, puis en lançant des requêtes HTTP (par exemple avec `curl`) vers l’adresse privée de `ILF_InstanceVPC2`.  
Le même test a été effectué dans l’autre sens depuis `ILF_InstanceVPC2`, confirmant que les serveurs Apache des deux VPC pouvaient communiquer en HTTP grâce au peering et au routage configuré.

![Routes VPC1 avec peering](vpc&-peering.jpg)
![Describe route tables VPC2](routes-tables-vpc2.jpg)
![Describe instances VPC2](instances-vpc2.jpg)
![Test curl HTTP / peering OK](curl-http.jpg)
