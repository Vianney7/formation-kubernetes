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
    args: ["-c", "echo Hello World > /pod-data/index.html"
```

## Exercice 2
