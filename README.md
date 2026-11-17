
\## 1. Environnement de travail



\- Système d'exploitation : Windows

\- Outil : Docker Desktop

\- Terminal : Windows PowerShell

\- Image Alpine : `alpine:3.20`

\- Image Nginx : `nginx:1.27-alpine`



\## 2. Travaux pratiques réalisés



\### Étape 1 : Découverte des réseaux Docker



Consultation des réseaux disponibles :



```powershell

docker network ls

docker network inspect bridge

```



\### Étape 2 : Réseau bridge par défaut



Création de deux conteneurs Alpine :



```powershell

docker run -d --name a1 alpine:3.20 sleep 3600

docker run -d --name a2 alpine:3.20 sleep 3600

```



Test de communication par adresse IP :



```powershell

docker exec a1 ping -c 2 172.17.0.16

```



La communication par adresse IP fonctionne, mais la résolution automatique du nom `a2` ne fonctionne pas sur le réseau bridge par défaut.



\### Étape 3 : Réseau bridge personnalisé



Création du réseau :



```powershell

docker network create reseau-tp

```



Création des conteneurs :



```powershell

docker run -d --name b1 --network reseau-tp alpine:3.20 sleep 3600

docker run -d --name b2 --network reseau-tp alpine:3.20 sleep 3600

```



Test DNS et communication :



```powershell

docker exec b1 ping -c 2 b2

docker exec b1 nslookup b2

```



La résolution DNS automatique fonctionne entre les conteneurs du réseau personnalisé.



\### Étape 4 : Publication des ports



Lancement d'un serveur Nginx :



```powershell

docker run -d --name web --network reseau-tp -p 127.0.0.1:8084:80 nginx:1.27-alpine

```



Test depuis Windows :



```powershell

curl.exe http://localhost:8084

```



Test depuis un autre conteneur :



```powershell

docker exec b1 wget -qO- http://web

```



Le serveur Nginx est accessible sur le port interne 80 et le port 8084 de la machine hôte.



\### Étape 5 : Connexion à plusieurs réseaux



Connexion du conteneur a1 au réseau personnalisé :



```powershell

docker network connect reseau-tp a1

```



Vérification :



```powershell

docker inspect -f '{{json .NetworkSettings.Networks}}' a1

```



Déconnexion :



```powershell

docker network disconnect reseau-tp a1

```



\### Étape 6 : Alias réseau



Création d'un conteneur avec un alias :



```powershell

docker run -d --name b3 --network reseau-tp --network-alias serveur alpine:3.20 sleep 3600

```



Tests :



```powershell

docker exec b1 ping -c 2 b3

docker exec b1 ping -c 2 serveur

```



L'alias `serveur` permet d'accéder au conteneur b3 par un nom alternatif.



\### Étape 7 : Réseau none



Test avec un conteneur sans accès réseau :



```powershell

docker run --rm --network none alpine:3.20 ip addr

```



Test de connectivité :



```powershell

docker run --rm --network none alpine:3.20 ping -c 2 8.8.8.8

```



Le réseau none ne fournit aucune interface réseau externe au conteneur.



 

## 8. TP : Cloisonnement réseau avec Docker

### Objectif

Mettre en place une architecture réseau segmentée avec trois conteneurs :

- `proxy-cloison` : serveur Nginx, connecté uniquement au réseau frontend.
- `app-cloison` : conteneur intermédiaire, connecté aux deux réseaux.
- `db-cloison` : serveur PostgreSQL, connecté uniquement au réseau backend.

### 8.1 Création des réseaux

```powershell
docker network create frontend
docker network create --internal backend
```

Le réseau `backend` est interne et ne permet pas l'accès direct à Internet.

### 8.2 Création de la base de données

```powershell
docker run -d --name db-cloison --network backend -e POSTGRES_USER=etudiant -e POSTGRES_PASSWORD=motdepasse-tp -e POSTGRES_DB=cours -v pgdata-reseau:/var/lib/postgresql/data postgres:16-alpine
```

### 8.3 Création du conteneur applicatif

```powershell
docker run -d --name app-cloison --network backend postgres:16-alpine sleep 3600
docker network connect frontend app-cloison
```

### 8.4 Création du proxy Nginx

Le port 8080 étant déjà utilisé par un autre conteneur, le port 8085 a été choisi.

```powershell
docker run -d --name proxy-cloison --network frontend -p 127.0.0.1:8085:80 nginx:1.27-alpine
```

Vérification :

```powershell
curl.exe http://localhost:8085
```

### 8.5 Tests de cloisonnement

| Test | Résultat |
|---|---|
| Proxy vers App | Accessible |
| Proxy vers DB | Inaccessible |
| App vers Proxy | Accessible |
| App vers DB | Accessible |
| DB vers Proxy | Inaccessible |
| DB vers Internet | Inaccessible |
| Proxy vers Internet | Accessible |

Commandes de vérification :

```powershell
docker exec proxy-cloison ping -c 2 app-cloison
docker exec proxy-cloison ping -c 2 db-cloison
docker exec app-cloison ping -c 2 proxy-cloison
docker exec app-cloison pg_isready -h db-cloison -U etudiant
docker exec db-cloison ping -c 2 proxy-cloison
docker exec db-cloison ping -c 2 -W 3 8.8.8.8
docker exec proxy-cloison ping -c 2 -W 3 8.8.8.8
```







