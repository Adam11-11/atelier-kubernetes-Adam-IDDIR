# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :** IDDIR Adam
**Groupe :** 

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > Il affiche **3 nœuds** :
   > - `k3d-lab-server-0` : le **control-plane** (master), qui pilote le cluster (API Server, scheduler, gestion de l'état).
   > - `k3d-lab-agent-0` et `k3d-lab-agent-1` : les deux **workers** (agents), qui exécutent les conteneurs applicatifs (pods).
2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   > `docker ps` affiche **4 conteneurs** : les 3 nœuds du cluster (`server-0`, `agent-0`, `agent-1`) ainsi qu'un conteneur `k3d-lab-serverlb` (load balancer proxy qui expose les ports du cluster). Concrètement, dans **k3d**, chaque nœud Kubernetes est un simple **conteneur Docker** exécutant une instance légère de k3s.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   > La commande `kubectl create deployment` a créé :
   > - un **Deployment** (`deployment.apps/web`),
   > - un **ReplicaSet** (`replicaset.apps/web-68d995574f`),
   > - un **Pod** (`pod/web-68d995574f-l6jck`).
   > *(Note : `service/web` a été créé ensuite par `kubectl expose`, et `service/kubernetes` existe par défaut).*
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   > Dès que le pod `web-68d995574f-l6jck` est passé en état `Terminating`, un nouveau pod avec un suffixe aléatoire différent (`web-68d995574f-znzl7`) a immédiatement été créé (`Pending` → `ContainerCreating` → `Running`). C'est le **ReplicaSet** qui assure l'auto-réparation via la **boucle de réconciliation** de Kubernetes pour maintenir en permanence le nombre de réplicas désiré (ici 1).

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   > **Non**, la page par défaut (*Welcome to nginx!*) est revenue. Le système de fichiers d'un conteneur est **éphémère** : lorsqu'un pod est supprimé, toutes les modifications locales faites à l'intérieur sont perdues, et le nouveau pod recréé par le ReplicaSet redémarre à partir de l'image `nginx` vierge d'origine.
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   > Bien que la commande cible le Service (`svc/web`), `kubectl port-forward` établit en réalité un tunnel réseau direct vers **un pod précis** derrière ce Service. Quand ce pod a été supprimé, son espace réseau (*network namespace*) a été fermé, ce qui a coupé la connexion (`error: lost connection to pod`).

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   > - `kubectl create` est **impératif** : il crée une nouvelle ressource sur le cluster. Si la ressource existe déjà, la commande échoue.
   > - `kubectl apply` est **déclaratif** : il lit l'état désiré décrit dans un fichier YAML, crée la ressource si elle n'existe pas encore, ou la **met à jour** (*patch*) si elle existe déjà pour l'aligner avec le fichier.
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Le `selector` permet au Service d'identifier dynamiquement **quels pods** doivent recevoir le trafic réseau grâce à leurs labels (ex: `app: site`). S'il ne correspond à aucun pod, le Service n'a aucun *Endpoint* associé (`kubectl get endpoints site` affiche `<none>`) et toute requête vers ce Service échoue (erreur de connexion / *Connection refused* ou *503*).

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   > Le scheduler Kubernetes a réparti un pod sur chacun des **3 nœuds** du cluster : `k3d-lab-server-0`, `k3d-lab-agent-0` et `k3d-lab-agent-1`.

   NAME                    READY   STATUS    RESTARTS   AGE     IP          NODE               NOMINATED NODE   READINESS GATES
site-5cc664db9c-6smnn   1/1     Running   0          3m59s   10.42.1.6   k3d-lab-server-0              
site-5cc664db9c-v2hb9   1/1     Running   0          4m7s    10.42.2.9   k3d-lab-agent-1               
site-5cc664db9c-znq55   1/1     Running   0          4m12s   10.42.0.7   k3d-lab-agent-0

10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
 > Le nouveau pod est passé en état **`ErrImagePull`** puis **`ImagePullBackOff`**. La commande `kubectl describe pod -l app=site` (section *Events*) a donné la cause exacte : le tag `nginx:9.99` est introuvable sur Docker Hub (*manifest for nginx:9.99 not found*).
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
 > **Oui**, le site est resté 100 % accessible. Grâce à la stratégie de **Rolling Update** par défaut de Kubernetes, les anciens pods fonctionnels (`nginx:1.28`) ne sont pas supprimés tant que le nouveau pod n'est pas passé à l'état `Ready`. Le Service `site` continue donc de router le trafic uniquement vers les pods sains.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
 > https://ominous-zebra-5gr9g7645j593vrvr-8585.app.github.dev/
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
 > **Oui**, Portainer et `kubectl` communiquent exactement avec le même composant central du *control plane* : l'**API Server** de Kubernetes (`kube-apiserver`). Portainer est un simple client graphique de l'API Kubernetes (au même titre que `kubectl` en ligne de commande) ; la seule différence visible dans `kubectl get deploy mario -o yaml` est l'ajout automatique de métadonnées/labels propres à Portainer (`io.portainer.kubernetes...`).

## Bonus (facultatif)

14. Bonus réalisés et remarques :
 > Non réalisés.

---

**Score obtenu avec `./scripts/check.sh` :** ... / ...