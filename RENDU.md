# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :** Koalal Mohamed-Heni
**Groupe :** 

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > 3 nœuds, tous `Ready` :
   > - `k3d-lab-server-0` : `control-plane` (le « master »). Il héberge l'API Server et décide où placer les pods.
   > - `k3d-lab-agent-0` et `k3d-lab-agent-1` : les 2 workers, qui exécutent les pods applicatifs.

2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   > `docker ps` montre 4 conteneurs : `k3d-lab-server-0`, `k3d-lab-agent-0`, `k3d-lab-agent-1` (image `rancher/k3s`) et `k3d-lab-serverlb` (image `k3d-proxy`, le load balancer du port 8081).
   > Dans k3d, un nœud Kubernetes est concrètement **un conteneur Docker** qui exécute k3s : on simule plusieurs machines sur une seule VM.
   > Même « vide », le cluster fait tourner des pods système dans le namespace `kube-system` : coredns (DNS interne), traefik (Ingress), metrics-server, local-path-provisioner (stockage).

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   > Un **Deployment** (`web`), qui a créé un **ReplicaSet** (`web-68d995574f`), qui a créé un **Pod** (`web-68d995574f-knmpq`).
   > Le **Service** `web` (ClusterIP `10.43.54.118`, port 80) vient de `kubectl expose`.

4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   > L'ancien pod `web-68d995574f-knmpq` est passé en `Terminating`, et un nouveau pod `web-68d995574f-5svcr` (nom différent) a été créé et était `Running` en environ 1 seconde.
   > On a créé un Deployment, pas un pod : son ReplicaSet maintient en permanence 1 pod en vie. C'est la **boucle de réconciliation** (état désiré contre état réel) : Kubernetes corrige automatiquement l'écart. C'est l'auto-réparation.

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   > Non, la page « Welcome to nginx! » est revenue. Le système de fichiers d'un conteneur est **éphémère** : le nouveau pod repart de l'image `nginx` d'origine. Une modification faite à la main dans un conteneur est perdue au redémarrage. Il faut décrire le contenu en dehors du conteneur (ConfigMap, volume).

6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   > Un `port-forward` vers un Service s'attache en réalité à **un seul pod**. Quand ce pod est supprimé, le tunnel n'a plus de cible et se termine (`Terminated`). Il faut le relancer vers le nouveau pod.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   > `kubectl create` est **impératif** : on donne un ordre (« crée ceci »). Il échoue si l'objet existe déjà, et rien n'est versionné.
   > `kubectl apply -f` est **déclaratif** : on décrit l'état désiré dans un fichier YAML (versionné dans Git), et Kubernetes crée ou met à jour les objets pour s'y conformer. On peut le relancer autant de fois que voulu (`unchanged`, `configured`).

8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Le `selector` indique au Service **vers quels pods envoyer le trafic**, via leurs labels (ici `app: site`).
   > S'il ne correspond à aucun pod, le Service n'a **aucun endpoint** (`kubectl get endpoints site` est vide) : il existe mais ne mène nulle part, et les requêtes échouent.

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```
   NAME                    READY   STATUS    RESTARTS   AGE     IP           NODE               NOMINATED NODE   READINESS GATES
   site-5688b8f46c-4p8p8   1/1     Running   0          40s     10.42.0.6    k3d-lab-agent-0    <none>           <none>
   site-5688b8f46c-6pkcm   1/1     Running   0          2m13s   10.42.1.14   k3d-lab-agent-1    <none>           <none>
   site-5688b8f46c-ptghw   1/1     Running   0          40s     10.42.2.5    k3d-lab-server-0   <none>           <none>
   ```
   > Un pod par nœud : le scheduler répartit les réplicas pour qu'une panne de nœud ne coupe pas tout le site. Dans k3s, le control-plane accepte aussi des pods applicatifs.

10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    > Statut `ErrImagePull` puis `ImagePullBackOff`. La commande `kubectl describe pod site-57db5878bd-sxg29` (section **Events**) a donné la cause exacte : `Failed to pull image "nginx:9.99": ... docker.io/library/nginx:9.99: not found`. La version 9.99 n'existe pas sur Docker Hub.
    > Correction avec `kubectl rollout undo deploy/site`, puis vérification avec `kubectl diff -f k8s/site/` (aucune différence : cluster et fichiers alignés).

11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    > Oui. Grâce au **rolling update**, Kubernetes n'a lancé qu'un seul nouveau pod et n'a supprimé aucun ancien pod tant que le nouveau n'était pas prêt. Les 3 pods en nginx 1.28 sont restés `Running` et ont continué à servir le site.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    > https://symmetrical-space-enigma-977j4wv6q7gqh9jw-8585.app.github.dev/

13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    > `kubectl get deploy mario -o yaml` montre un Deployment standard (`readyReplicas: 1`, ReplicaSet `mario-5cfb955858`), identique à ceux créés en ligne de commande.
    > Oui, Portainer et `kubectl` parlent au **même composant : l'API Server** du control-plane. Portainer n'est qu'une interface graphique au-dessus de l'API Kubernetes (il tourne dans le cluster avec un ServiceAccount qui a les droits nécessaires).

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    > Aucun bonus réalisé. Remarque : la version récente de Portainer demande un setup token (récupéré via `kubectl logs -n portainer deploy/portainer | grep -i token`) pour créer le compte admin.

---

**Score obtenu avec `./scripts/check.sh` :** 15 / 18 (100 % des exercices obligatoires ; l'exercice 3 n'est pas vérifiable automatiquement)