# Formation Kubernetes

## Exercice 1

Créer un pod avec trois conteneurs :
- un conteneur d'initialisation créant un fichier index.html contenant "Hello World"
- un conteneur avec nginx et un volume permettant de monter le fichier index.html créer par le conteneur d'initialisation dans le répertoire /usr/share/nginx/html
- un conteneur effectuant un wget sur le nginx pour afficher le message "Hello World" normalement retourné

De plus, une sonde ''liveness probe'' doit régulièrement vérifier que le nginx fonctionne normalement.

Réponse:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: three-containers
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5
      timeoutSeconds: 1
      failureThreshold: 3
  - name: busybox-container
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true ; do sleep 5 ; wget -O- http://localhost:80/ ; done"]
  initContainers:
  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello World > /pod-data/index.html"]
```

## Exercice 2

Créer deux déploiements basé sur le pod du premier exercice :
- le premier tc1 qui déploie un replica du pod avec le message "Hello from TC1" retourné par le nginx
- le premier tc2 qui déploie un replica du pod avec le message "Hello from TC2" retourné par le nginx

Créer un service global-pc qui délivre le traffic sur son port 8080 avec le port 80 des pods des déploiement tc1 et tc2.

Vérifier avec un wget exécuté dans un pod basé sur l'image d'ubuntu la requète `wget http://global-pc:8080/` retourne les messages des pods des deux déploiements.

Réponse:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tc1
spec:
  replicas: 1
  selector:
    matchLabels:
      version: tc1
  template:
    metadata:
      labels:
        app: nginx
        version: tc1
    spec:
      volumes:
      - name: shared-data
        emptyDir: {}
      containers:
      - name: nginx-container
        image: nginx
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
          timeoutSeconds: 1
          failureThreshold: 3
      - name: busybox-container
        image: busybox
        command: ["/bin/sh"]
        args: ["-c", "while true ; do sleep 5 ; wget -O- http://localhost:80/ ; done"]
      initContainers:
      - name: debian-container
        image: debian
        volumeMounts:
        - name: shared-data
          mountPath: /pod-data
        command: ["/bin/sh"]
        args: ["-c", "echo Hello from TC1 > /pod-data/index.html"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tc2
spec:
  replicas: 1
  selector:
    matchLabels:
      version: tc2
  template:
    metadata:
      labels:
        app: nginx
        version: tc2
    spec:
      volumes:
      - name: shared-data
        emptyDir: {}
      containers:
      - name: nginx-container
        image: nginx
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
          timeoutSeconds: 1
          failureThreshold: 3
      - name: busybox-container
        image: busybox
        command: ["/bin/sh"]
        args: ["-c", "while true ; do sleep 5 ; wget -O- http://localhost:80/ ; done"]
      initContainers:
      - name: debian-container
        image: debian
        volumeMounts:
        - name: shared-data
          mountPath: /pod-data
        command: ["/bin/sh"]
        args: ["-c", "echo Hello from TC2 > /pod-data/index.html"]
---
apiVersion: v1
kind: Service
metadata:
  name: global-pc
spec:
  ports:
  - name: http-web
    port: 8080
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: ClusterIP
```
## Exercice 3

Créer un pod nginx avec une configuration modifié (les messages de logs commencent tous par la chaîne de caractères "YOLO") en montant un fichier nginx.conf modifié à l'emplacement /etc/nginx/nginx.conf doit 

Solution:

- extraire le fichier original de configuration de l'image docker nginx

```shell
docker run --namespace nginx nginx
docker cp nginx:/etc/nginx/nginx.conf .
docker rm -v -f nginx
```

- modifier le fichier nginx.conf comme suit :

```
[...]
    log_format  main  'YOLO $remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
[...]
```

- créer un config map contenant le fichier nginx.conf:

```shell
kubectl create configmap nginx-extra --from-file=nginx.conf=nginx.conf
```

- déployer le pod nginx avec le montage du volume basé sur le configmap:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-yolo
spec:
  containers:
  - image: nginx
    name: nginx-yolo
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: nginx-extra
        items:
        - key: nginx.conf
          path: nginx.conf
```

## Exercice XX

Déploiement d'une application mongo-express (exposé par un ingress controller) avec une base de données mongo.

- [Déploiement de la base de données mongo dans son namespace mongo](./mongo-mongo.yaml)
- [Déploiement de l'application web mongo-express dans son namespace mongo-express](./mongo-mongo-express.yaml)

## Exercice XX+1

Modifier le précédent exemple pour permettre la sauvegarde des données du MongoDB via un PersistenceVolumeClaim.

Attention: 
- il ne peux exister qu'une instance du pod MongoDB référençant le PVC
- stratégie de mise-à-jour doît être Recreate (pour éviter que deux pods référencent le même PVC pendant la mise-à-jour).

- [Déploiement mongo avec vpc avec accès ReadWriteOnce dans son namespace mongo](./mongo-mongo-pvc.yaml)

Un autre exemple avec le mode d'accès ReadWriteOncePod pour garantir qu'un seul pod soit connecté au PV ([ne gonctionne qu'à partir de la version 1.22 en alpha avec la feature gate activé pour cett option](https://kubernetes.io/blog/2021/09/13/read-write-once-pod-access-mode-alpha/)).

- [Déploiement mongo avec vpc avec accès ReadWriteOncePod dans son namespace mongo](./mongo-mongo-pvc-pod.yaml)



