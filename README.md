# TP Docker

BTS SIO SISR

Nom : Cuvellier
Date : 18/11/2026

On a fait le TP sur une VM Debian.

---

## 1. Installation

On installe Docker :

```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo usermod -aG docker $USER
```

On se déconnecte et on se reconnecte.

On vérifie les versions :

```bash
docker -v
docker-compose -v
```

On lance notre premier conteneur :

```bash
docker run hello-world
```

<img width="679" height="446" alt="image" src="https://github.com/user-attachments/assets/c7737b2a-8cc6-4045-a865-8455b445f107" />


<br><br>

---

## 2. Images et containers

On lance un serveur nginx :

```bash
docker run -p 8080:80 -d nginx
```

- `run` : lance le conteneur
- `-p 8080:80` : le port 8080 de notre machine va vers le port 80 du conteneur
- `-d` : le conteneur tourne en arrière-plan
- `nginx` : l'image

"curl localhost:8080"

<img width="466" height="83" alt="image" src="https://github.com/user-attachments/assets/e14b73c4-fba8-40a6-b07d-92d70ac196ec" />


<br><br><br>

### À quoi servent les commandes ?

| Commande | Ce qu'elle fait |
|---|---|
| `docker ps` | Montre les conteneurs qui tournent |
| `docker ps -a` | Montre tous les conteneurs (même arrêtés) |
| `docker images` | Montre les images qu'on a |

<img width="1208" height="260" alt="image" src="https://github.com/user-attachments/assets/ba49b49e-2ba7-4818-88d0-98edeb5a804a" />

<br><br><br>

### Arrêter et supprimer

| Question | Commande |
|---|---|
| Arrêter un container | `docker stop nom_ou_id` |
| Supprimer un container | `docker rm nom_ou_id` |
| Supprimer une image | `docker rmi nom_image` |

### Apache

On télécharge l'image :

```bash
docker pull httpd
```

On la lance sur le port 8081 :

```bash
docker run -d -p 8081:80 httpd
```

curl localhost:8081` : on voit "It works!".

<img width="730" height="78" alt="image" src="https://github.com/user-attachments/assets/b0eea1b7-1596-4b49-9d98-0f445d79c544" />


<br><br><br>

---

## 3. Créer notre image

### Image Debian

```bash
mkdir lab0
cd lab0
nano Dockerfile
```

Dans le fichier `Dockerfile` :

```dockerfile
FROM debian:stable-slim
LABEL maintainer="BTS SIO2 SISR"
RUN apt-get update \
 && apt-get install -y vim git htop mc tree \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

La dernière ligne supprime les fichiers inutiles pour que l'image soit plus petite.

On construit l'image (il ne faut pas oublier le point) :

```bash
docker build -t btssio:v1.0 .
```

On regarde la taille :

```bash
docker images
```

Taille de l'image : ... Mo

<img width="625" height="144" alt="image" src="https://github.com/user-attachments/assets/7b9e7718-ca3f-4dfb-9b41-5309659bc20d" />


<br><br><br>

On lance le conteneur et on rentre dedans :

```bash
docker run -tid --name sisr btssio:v1.0
docker exec -ti sisr bash
```

On teste les outils (vim, git, htop, mc, tree), puis on sort avec `exit`.

<img width="174" height="14" alt="image" src="https://github.com/user-attachments/assets/3a8e0bf2-08e8-49c8-bb0b-cee365e39182" />


<br><br><br>

### Image plus légère : Alpine

L'indice "France en Formule 1" c'est **Alpine** (l'écurie française). Alpine est un Linux très léger.

```bash
cd ..
mkdir lab1
cd lab1
nano Dockerfile
```

Dans le fichier `Dockerfile` :

```dockerfile
FROM alpine:latest
LABEL maintainer="BTS SIO2 SISR"
RUN apk add --no-cache vim git htop mc tree bash
CMD ["bash"]
```

```bash
docker build -t btssio:v2.0 .
docker images
docker run -tid --name sisr2 btssio:v2.0
docker exec -ti sisr2 bash
```

| Image | Taille |
|---|---|
| btssio:v1.0 (Debian) | ... Mo |
| btssio:v2.0 (Alpine) | ... Mo |

L'image Alpine est beaucoup plus petite et on a les mêmes outils.

<img width="532" height="47" alt="image" src="https://github.com/user-attachments/assets/a2a3c916-1b68-43ba-adb1-a13b9b1ff9df" />


<br><br><br>

### Nettoyage

```bash
docker rm -f sisr sisr2
docker rmi btssio:v1.0 btssio:v2.0
```

---

## 4. Docker Compose

On crée un dossier avec 4 fichiers :

```bash
mkdir projet
cd projet
```

**requirements.txt**

```
Flask==1.1.4
redis==3.5.3
```

**app.py**

```python
from flask import Flask
from redis import Redis, RedisError
import os
import socket

redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = ("<h3>Hello {name}!</h3>"
            "<b>Hostname:</b> {hostname}<br/>"
            "<b>Visits:</b> {visits}")
    return html.format(name=os.getenv("NOM", "world"),
                       hostname=socket.gethostname(),
                       visits=visits)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
```

**Dockerfile**

```dockerfile
FROM python:2.7-slim
WORKDIR /app
COPY . /app
RUN pip install -r requirements.txt
EXPOSE 80
ENV NOM="BTS SIO2 SISR"
CMD ["python", "app.py"]
```

On construit l'image :

```bash
docker build -t monimage .
```
<img width="736" height="404" alt="image" src="https://github.com/user-attachments/assets/0c70588d-21cd-4956-b877-061b0701325c" />


Si on lance l'image toute seule, le compteur ne marche pas, car Redis n'est pas là :

```bash
docker run -d -p 80:80 --name test monimage
```
<img width="605" height="32" alt="image" src="https://github.com/user-attachments/assets/1bb0e634-c072-4e87-966c-104cda82843b" />

<img width="949" height="47" alt="image" src="https://github.com/user-attachments/assets/2486a989-5610-4711-9dad-ce010d1ecd09" />

<img width="1280" height="161" alt="image" src="https://github.com/user-attachments/assets/6f449bd5-02a9-4086-94b5-aa8082e09bcb" />


<br><br><br>

```bash
docker rm -f test
```

**docker-compose.yml**

```yaml
version: "3"
services:
  monapp:
    image: monimage
    depends_on:
      - redis
    ports:
      - "80:80"
    networks:
      - monreseau
    environment:
      - NOM=les SISR
  redis:
    image: redis
    networks:
      - monreseau
networks:
  monreseau:
```

On lance :

```bash
docker-compose up
```

On va sur `localhost` et on fait F5 : le compteur augmente.


<br><br><br>
### Changer le hostname en SaintLuc

On ajoute une ligne `hostname: SaintLuc` dans le service `monapp` :

```yaml
  monapp:
    image: monimage
    hostname: SaintLuc
```

On relance :

```bash
docker-compose down
docker-compose up -d
```

La page affiche `Hostname: SaintLuc`.

**Capture : page avec SaintLuc**

<br><br><br><br><br><br>

### Nettoyage final

```bash
docker-compose down
docker rmi monimage redis nginx httpd hello-world
```

---

## Conclusion

On a appris à lancer des conteneurs, à créer nos images avec un Dockerfile, à faire une image plus légère avec Alpine, et à lancer plusieurs conteneurs ensemble avec Docker Compose.
