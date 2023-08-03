# Tutoriel ReactJS, Docker et Kubernetes

Technologies
- Node et NPM
- Docker
- kubectl
- minikube

Objectifs prioritaires
- [x] Créer une application ReactJS bidon en vue d'être déployé sur un cluster Kubernetes
- [x] Utiliser Docker pour le développement local
- [x] Créer des images hébergés sur Docker Hub
- [x] Déployer l'application avec minikube sur un environnement de pré-production

Objectifs secondaires
- [x] Avoir un suivi de santé des nodes Kubernetes avec Prometheus et Grafana
- [x] Créer un environnement local de développement incluant Prometheus, Grafana, MySQL, ExpressJS React
- [x] Avoir un accès aux métrics de MySQL et de l'application React
- [] Avoir un outil de stress-test et déclencher les stress tests
- [] Définir les ressources matériels des pods
- [] Intégrer le CI/CD pour un environnement de test

Objectifs à venir
- [] Envelopper l'application avec NextJS 
- [] Intégrer des machine learning


# Procédure aux objectifs prioritaires

## Structure de fichiers

- K8s : Contient les fichiers de configurations YAML pour le déploiement kubernetes
- react : Contient l'application ReactJS

## Créer l'application ReactJS

Il est nécessaire d'avoir Node et NPM installé sur votre ordinateur

1. Application ReactJS

Créer une instance simple de reactJS sans utiliser TypeScript. L'objectif est de le garder le plus simple possible.

```
cd react
npx create-react-app my-app
cd my-app
npm start
```

Accéder l'URL suivant avec le navigateur : [http://localhost:3000](http://localhost:3000)

2. Variable d'environnement 

Créer un fichier d'environnement local pour seulement modifier le port utiliser par l'application

```
nano .env.local
```

Voici le contenu texte du fichier d'environnement

```
PORT=3002
```

Relancer l'application et tester à nouveau l'URL avec le nouveau port : [http://localhost:3002](http://localhost:3002)

3. Build de production

Créons le build de production 

```
npm run build
```

## Intégrer docker

Il est nécessaire d'avoir un compte Docker Hub et que Docker soit installé sur votre ordinateur

1. S'authentifier avec la commande suivante

```
docker login
```

Si l'authentification a fonctionné, le message `Login Succeeded` apparaîtra

2. Dockerfile et .dockerignore

Dans le dossier `react`, créer les 2 fichiers `Dockerfile` et `.dockerignore`

Ajouter le code suivant dans le fichier `Dockerfile`. Ceci créera un container docker avec NGINX en utilisant les fichiers statiques du build créer pour reactJS

```
FROM nginx:stable-alpine

COPY my-app/build/ /usr/share/nginx/html
```

Ajouter le code suivant dans le fichier `.dockerignore`.

```
npm-debug.log
.dockerignore
**/.git
**/.DS_Store
**/node_modules
```

3. Créer une image du projet et la rendre accessible sur DockerHub

Toujours dans le dossier `react`, créer une image du projet et testé l'application 

```
docker build -t your_docker_username/k8-test-react .
docker run -d -p 3000:80 your_docker_username/k8-test-react
```

Accéder l'URL suivant avec le navigateur : [http://localhost:3000](http://localhost:3000)

Si la page s'affiche correctement, fermer les containers et pousser l'image sur DockerHub

```
docker stop $(docker ps -a -q)
docker push your_docker_username/k8-test-react:latest
```

## Déployer l'application avec Kubernetes

Il est nécessaire d'avoir `kubectl` et `minikube` installé sur votre ordinateur. Si ce n'est pas fait, suivre les 4 premières étapes du tutoriel suivant : [LinuxTech](https://www.linuxtechi.com/how-to-install-minikube-on-ubuntu/)

1. Préparer minikube et définir un namespace du projet

Accéder au répertoire `K8s` et lancer `minikube`

```
cd ../K8s
minikube start
```

Avec `kubectl`, créer un nouvel espace pour l'application reactJS et définir cet espace par défaut

```
kubectl create namespace k8-test
kubectl config set-context --current --namespace=k8-test
```

2. Fichiers de configuration utilisé par kubectl

Créer un fichier YAML pour le déploiement du node kubernetes avec le nombre de réplication de notre application à 2.

```
nano deployment.yaml
```

Voici le contenu du fichier

```
kind: Deployment
apiVersion: apps/v1
metadata:
  name: react-docker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: react-docker
  template:
    metadata:
      labels:
        app: react-docker
    spec:
      containers:
      - name: react-docker
        image: your_docker_username/k8-test-react
        ports:
        - containerPort: 80

```

Utiliser le fichier de déploiement et vérifier que le déploiement est instancié avec 2 pods

```
kubectl apply -f deployment.yaml
kubectl get deployment -w
```

En ce moment, le déploiement ne permet pas d'accéder à l'application. Il est nécessaire de créer un Load Balancer qui permettra de redistribuer le traffic vers les bons pods. Créer un fichier `load-balancer.yaml` qui servira à créer un service kubernetes.

```
nano load-balancer.yaml
```

Voici le contenu du fichier

```
apiVersion: v1
kind: Service
metadata:
  name: load-balancer
  labels:
    app: react-docker
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    nodePort: 31000
  selector:
    app: react-docker
```

Utiliser le fichier de load balancing et vérifier que le service est instancié

```
kubectl apply -f load-balancer.yaml
kubectl get services -w
```

3. Accéder à l'application Web

Pour accéder à l'application sur le navigateur, il est nécessaire d'obtenir l'adresse IP de minikube et d'utiliser le port 31000 définit par la règle nodePort du load balancer

Obtenir l'adresse IP de minikube avec la commande suivante

```
minikube ip
```

Accéder à l'URL suivant en replacant `minikube_ip` par l'adresse IP retourné de la commande précédente

```
http://minikube_ip:31000
```

## Altérer le nombre de pods en temps réel

Nous avons défini un nombre de pods pour notre application par le biais du fichier de configuration servant au déploiement. Nous pouvons modifier le nombre de pods sans avoir à modifier le fichier de configuration.

Augmenter le nombre de pods à 5 avec l'argument `--replicas` et vérifier l'état du nombre de pods actifs

```
kubectl scale deployment react-docker --replicas=5
kubectl get deployment -w
```

Diminuer ensuite le nombre de pods à 1 et vérifier l'état du nombre de pods actifs

```
kubectl scale deployment react-docker --replicas=1
kubectl get deployment -w
```


# Procédure aux objectifs secondaires

## Installer Helm, Prometheus & Grafana

[Medium Article](https://medium.com/globant/setup-prometheus-and-grafana-monitoring-on-kubernetes-cluster-using-helm-3484efd85891)

1. Installer Helm

```
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt update
sudo apt install helm
```

2. Ajouter le repos Helm pour Prometheus & Grafana

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

3. Installer & Démarrer le service Prometheus

```
helm install prometheus prometheus-community/prometheus
kubectl expose service prometheus-server --type=NodePort --target-port=9090 --name=prometheus-server-ext
minikube service prometheus-server-ext --namespace=k8-test
```

4. Installer & Démarrer le service Grafana

```
helm install grafana grafana/grafana
```

Get the decrypted grafana password

```
kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Expose and start the Grafana service

```
kubectl expose service grafana --type=NodePort --target-port=3000 --name=grafana-ext
minikube service grafana-ext
```

5. Log into Grafana

Obtenir l'URL de Grafana avec son port et y accéder avec un navigateur. Garder en note l'URL de Prometheus.

```
minikube service list
```

Se connecter avec l'utilisateur 'admin' et le mot de passe que nous avons décrypté précédemment

Ajouter notre premier DataSource de Prometheus en copiant son URL ainsi que son port. Sauvegarder & tester.

Une fois le DataSource connecté, créons des graphs. Pour simplifier le processus, importer un dashboard provenant de [Grafana](https://grafana.com/grafana/dashboards/). Rechercher `Prometheus` et obtenir son identifiant unique qui devrait être le '3662'.

Après le chargement des données, attacher le au dashboard Prometheus.


## Ajouter le monitoring de l'application React


