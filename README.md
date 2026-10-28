# Labo 09 – Bases de données distribuées et verrous distribués

<img src="https://upload.wikimedia.org/wikipedia/commons/2/2a/Ets_quebec_logo.png" width="250">

ÉTS - LOG430 - Architecture logicielle - Chargé de laboratoire : Gabriel C. Ullmann.

## 🎯 Objectifs d'apprentissage

- Observer et comparer [YugabyteDB](https://docs.yugabyte.com/) et [CockroachDB](https://www.cockroachlabs.com/docs/) en termes de performances sous forte concurrence 
- Comprendre le concept de **verrou distribué** (*distributed lock*) et pourquoi il est essentiel dans les systèmes distribués
- Comparer les stratégies de verrouillage **pessimiste** et **optimiste** en termes de latence et de fiabilité
- Vérifier la **résilience** lors d'une panne de nœud

## ⚙️ Setup
L'application Store Manager a évolué d'un monolithe vers une architecture microservices event-driven. Pour le Labo 09, nous franchissons une étape cruciale : la migration vers une architecture entièrement distribuée, pas seulement dans les services mais également dans la base de données. Cette évolution prépare l'application pour un déploiement nuage à grande échelle.

Dans ce labo, nous allons observer et comparer deux bases de données distribuées gratuites et open source : YugabyteDB et CockroachDB. 
1. Tout d'abord, nous expérimenterons avec différentes approches de verrou distribué avec YugabyteDB (dans le répertoire `yugabyte-db`), et nous terminerons avec un test de charge. 
2. Ensuite, nous allons répéter le test de charge avec CockroachDB (dans le répertoire `cockroach-db`)
3. Finalement, nous allons comparer les deux tests de charge pour déterminer laquelle nous permettra de traiter plus de requêtes simultanées.

Afin de nous concentrer davantage sur le fonctionnement de la base de données, nous utiliserons une version simplifiée de l'application Store Manager. Suivez les étapes dans cette section pour préparer les 2 projets/répertoires.

> 📝 **NOTE** : CockroachDB a récemment [changé](https://www.cockroachlabs.com/blog/enterprise-license-announcement/) à un modèle "source available", ce n'est plus open source. De plus, même si c'est gratuit pour l'utilisation personnelle et éducationnelle, CockroachDB n'est pas gratuit pour une utilisation commerciale.

### 1. Clonez le dépôt

Créez votre propre dépôt à partir du dépôt gabarit (template). Vous pouvez modifier la visibilité pour le rendre privé si vous le souhaitez.

```bash
git clone https://github.com/[votrenom]/log430-labo9
cd log430-labo9
```

### 2. Créez un fichier .env

Dans les répertoires `yugabyte-db` et `cockroach-db`, créez un fichier `.env` basé sur `.env.example`.

### 3. Créez un réseau Docker

```bash
docker network create labo09-network
```

### 4. Préparez l'environnement de développement

Démarrez premièrement le projet dans le répertoire `yugabyte-db`. Suivez les mêmes étapes que pour les derniers laboratoires.

> ⚠️ **ATTENTION** : Le conteneur `db-init` démarrera, initialisera la base de données, puis s'arrêtera automatiquement. Si vous remarquez qu'il est arrêté, c'est tout à fait normal.

## 🧪 Activités pratiques

L'architecture de ce laboratoire repose sur un cluster (grappe) à trois nœuds. Dans le cas de YugabyteDB, les conteneurs s'appelent `yugabyte1`, `yugabyte2`, `yugabyte3`. Contrairement à une base de données classique centralisée, YugabyteDB distribue automatiquement les données et les transactions entre les nœuds. Cependant, il utilise encore le paradigme relationnel, donc il nous permet de créer des tables, des colonnes, et d'utiliser le langage SQL pour consulter la base de données.

Pour en savoir plus sur l'architecture et les décisions de conception, veuillez consulter le document d'architecture sur `/docs/arc42/docs.md` et l'ADR sur `/docs/adr/adr001.md` avant de commencer les activités.

> 📝 **NOTE** : Dans une vraie application en environnement de production, les nœuds d'un cluster seraient déployés sur des serveurs physiques distincts. Par simplicité, dans ce labo, les trois nœuds tournent tous dans des conteneurs Docker sur la même machine.

### 1. Observez le schéma et le cluster YugabyteDB

Commençons par explorer l'interface d'administration de YugabyteDB pour comprendre comment les données sont organisées et répliquées dans le cluster.

1. Ouvrez http://localhost:7000 (**YB Master UI**).

2. Observez la liste des nœuds et identifiez les rôles **leader** et **follower**. Dans une base de données distribuée, il y a toujours un nœud maître (*leader*) qui coordonne les décisions de consensus (quelles transactions sont validées, dans quel ordre) et plusieurs nœuds secondaires (*followers*) qui répliquent les données. Cette séparation garantit la cohérence même en cas de panne d'un nœud.

3. Cliquez sur **Tables > Orders** et observez les **tablets**. Un tablet est l'unité de base de la **fragmentation** ([sharding](https://docs.yugabyte.com/stable/architecture/docdb-sharding/#sharding)) dans YugabyteDB : chaque table est découpée en plusieurs tablets, et chaque tablet est assigné à un nœud différent. Cela permet de distribuer la charge de lecture et d'écriture horizontalement. Pour en savoir plus, consultez la [documentation officielle sur le tablet splitting](https://docs.yugabyte.com/stable/architecture/docdb-sharding/tablet-splitting/).

4. Pour visualiser directement les données stockées, exécutez la commande suivante via l'onglet **Exec** de Docker Desktop sur le conteneur `yugabyte1`. Vous devriez voir une table vide :

```sh
ysqlsh -h yugabyte1 -U yugabyte -c "SELECT * FROM orders;"
```

Exécutez le test de concurrence suivant via l'onglet **Exec** de Docker Desktop. Ne vous inquiétez pas pour la compréhension approfondie de ce test, nous l'étudierons lors de la prochaine activité:

```bash
python tests/concurrency_test.py --threads 5 --product 3
```

Répétez la vérification. Vous devriez maintenant voir de nouveaux enregistrements :

```sh
ysqlsh -h yugabyte1 -U yugabyte -c "SELECT * FROM orders;"
```

> 💡 **Question 1** : Quelle est la sortie du terminal que vous obtenez? Si vous répétez cette commande sur `yugabyte2` et `yugabyte3`, est-ce que la sortie est identique? Illustrez votre réponse avec des captures d'écran ou des sorties du terminal.

### 2. Comparez le verrouillage pessimiste vs. optimiste

Dans un contexte de base de données distribuée, plusieurs instances de la base de données peuvent tenter de modifier les mêmes lignes au même moment. Sans mécanisme de contrôle, cela peut mener à des **conditions de course** (*race conditions*) : par exemple, deux transactions lisent un stock de 1 unité, toutes les deux décident de le décrémenter, et on se retrouve avec un stock négatif.

Dans ce laboratoire, nous étudierons deux approches pour éviter ce type de problème :

- **Verrouillage optimiste** : Chaque ligne de stock dans la table `stocks` possède une colonne `version`. Une transaction lit la version courante, calcule la nouvelle quantité, puis effectue un `UPDATE` uniquement si la version en base de données correspond toujours à celle qu'elle a lue. Si aucune ligne n'est affectée, c'est qu'une autre transaction a déjà modifié la ligne entre-temps. La transaction recommence alors depuis le début, jusqu'à un maximum de tentatives (`max_retries`). Cette approche évite les verrous lourds et est performante quand les conflits sont rares.

- **Verrouillage pessimiste** : La transaction acquiert un verrou au niveau de la ligne dès la lecture (`SELECT … FOR UPDATE`). Toute autre transaction qui tente de toucher la même ligne dans la table `stocks` sera **bloquée** jusqu'à ce que la première transaction soit terminée (commit ou rollback). Cette approche garantit qu'aucun conflit ne peut survenir, au prix d'une latence plus élevée en cas de forte contention.

Lisez le code dans `src/controllers/order_controller.py` pour voir comment les deux approches sont implémentées dans ce projet. Pour comparer les deux approches en conditions réelles, exécutez le test de concurrence suivant via l'onglet **Exec** de Docker Desktop :

```bash
python tests/concurrency_test.py --threads 20 --product 3
```

L'article avec l'ID 3 a un stock initial de 2 unités. Avec 20 threads tentant simultanément de passer une commande d'une unité, seules 2 commandes devraient être acceptées. Si le système en accepte davantage, cela indique que le verrou ne fonctionne pas correctement.

Après l'exécution du test, vérifiez le stock final depuis votre machine hôte. Après le test, le stock de l'article ID 3 devrait être zéro :

```bash
curl http://localhost:5000/stocks
```

> 💡 **Question 2** : Observez la latence moyenne des deux approches affichée dans la sortie du test. Laquelle a la latence moyenne la plus élevée et pourquoi? Illustrez votre réponse avec les sorties du terminal.

> 💡 **Question 3** : Répétez le test avec 5 threads au lieu de 20. Quelle approche a actuellement la latence moyenne la plus élevée et pourquoi? Illustrez votre réponse avec les sorties du terminal.

### 3. Testez la charge sur YugabyteDB avec Locust

Maintenant que nous avons observé les deux stratégies en isolation, nous allons les comparer sous une charge soutenue afin de mesurer leur impact sur le débit (*throughput*) et le taux d'erreurs de l'application. 

Réinitialisez les stocks avant de commencer depuis votre machine hôte :
```bash
curl -X POST http://localhost:5000/stocks/reset
```

Pour exécuter le test, accédez à http://localhost:8089 et appliquez les paramètres suivants :

- **Number of users (nombre total d'utilisateurs)** : 50
- **Spawn rate (taux d'apparition des nouveaux utilisateurs)** : 5 (par seconde)
- **Cliquez sur l'onglet Advanced Options > Run time (temps d'exécution)** : 60s (ou 1m)

Lancez le test et observez les statistiques (onglet `Statistics`) et graphiques (onglet `Charts`) dans Locust. Enregistrez le contenu du tableau `Statistics`, nous l'utiliserons plus tard pour comparer le test suivant (par exemple, vous pouvez copier-coller le tableau dans Excel/Google Sheets ou dans un fichier texte).

> 💡 **Question 4** : En utilisant YugabyteDB, quelle stratégie de verrouillage affiche le plus bas taux d'erreurs et la plus baisse latence moyenne? Illustrez votre réponse avec des captures d'écran ou statistiques de l'interface Locust.

### 4. Observez la résilience du cluster YugabyteDB et la cohérence des données

L'un des avantages majeurs d'une base de données distribuée est sa capacité à continuer de fonctionner même en cas de panne d'un nœud, grâce à la réplication et à [l'algorithme de consensus Raft](https://raft.github.io/). Dans cette activité, nous allons vérifier ce comportement en simulant une panne pendant un test de charge.

> 📝 **NOTE** : Il existe d'autres algorithmes de consensus, tels que [Paxos](https://docs.datastax.com/en/dse/6.9/architecture/database-internals/lightweight-transactions.html). Cependant, YugabyteDB utilise Raft et CockroachDB utilise [MultiRaft](https://www.cockroachlabs.com/blog/scaling-raft/).

1. Lancez un test de charge en continu depuis l'interface Locust (**50 utilisateurs**, un *spawn rate* de **5 utilisateurs/seconde**, sans durée limite).

2. Pendant que le test tourne, arrêtez un nœud secondaire :
```bash
docker stop yugabyte2
```

3. Observez dans Locust si le taux d'erreur augmente et, si oui, combien de temps dure la période de basculement (*failover*) avant que le système se stabilise.

4. Redémarrez le nœud arrêté et observez la reprise :
```bash
docker start yugabyte2
```

> 💡 **Question 5** : Est-ce que le taux d'erreur a augmenté lors de l'arrêt du nœud? Combien de temps a duré le basculement (approximativement)? Illustrez votre réponse avec des captures d'écran et statistiques de l'interface Locust.

### 5. Testez la charge sur CockroachDB avec Locust
Démarrez le projet dans le répertoire `cockroach-db`. Assurez-vous que les étapes de setup ont été exécutées avant le démarrage. Vous devriez voir les conteneurs `cockroach1`, `cockroach2` et `cockroach3` dans la liste. Si vous voulez, vous pouvez arrêter vos conteneurs YugabyteDB pour économiser les ressources.

Ensuite, répétez le test de charge sur CockroachDB avec les mêmes paramètres de l'activité 3. Lancez le test et observez les statistiques et graphiques. Enregistrez le contenu du tableau `Statistics` et comparez avec les résultats de l'activité 3.

> 💡 **Question 6** : En utilisant CockroachDB, quelle stratégie de verrouillage affiche le plus bas taux d'erreurs et la plus baisse latence? Illustrez votre réponse avec des captures d'écran ou statistiques de l'interface Locust.

> 💡 **Question 7** : Quelle base de données affiche le plus bas taux d'erreurs et la plus baisse latence? Est-ce que c'est YugabyteDB ou CockroachDB? Illustrez votre réponse avec des captures d'écran ou statistiques de l'interface Locust.

### 6. Déployez votre cluster de base de données sur Kubernetes

Jusqu'ici, votre cluster à trois nœuds tourne en Docker Compose, sur votre machine. Un cluster de base de données distribuée n'a d'intérêt que s'il survit à la perte d'un nœud, ce qui suppose un orchestrateur. Vous allez donc le redéployer sur la grappe du cours avec un **StatefulSet**.

#### Pourquoi un StatefulSet et pas un Deployment

Un `Deployment` crée des pods interchangeables, aux noms imprévisibles. Ça ne convient pas ici : chaque nœud de base de données a **une identité stable** et **son propre disque**. Le `StatefulSet` apporte les deux :

- des noms ordinaux prévisibles — `cockroachdb-0`, `cockroachdb-1`, `cockroachdb-2` ;
- une entrée DNS stable par pod, fournie par un **Service headless** ;
- un volume persistant par pod, créé par `volumeClaimTemplates`.

C'est ce qui permet au nœud 0 de retrouver le nœud 1 après un redémarrage.

#### Choisissez votre base de données

Les deux sont acceptées, mais **CockroachDB est nettement plus simple à porter**. Comparez les commandes dans les deux `docker-compose.yml` :

- CockroachDB lance ses trois nœuds avec **la même commande**, seul `--advertise-addr` diffère. Un StatefulSet n'a qu'un seul gabarit de pod : c'est exactement ce qu'il faut.
- YugabyteDB démarre le nœud 1 autrement que les nœuds 2 et 3, qui font `--join=yugabyte1`. Exprimer cette asymétrie dans un gabarit unique demande un détour.

Prenez CockroachDB, sauf si vous voulez le défi.

#### Ce que vous devez écrire

Trois ressources, dans un fichier `k8s/cockroachdb.yml` à créer :

```yaml
# 1. Service headless : donne a chaque pod un nom DNS stable
apiVersion: v1
kind: Service
metadata:
  name: cockroachdb
spec:
  clusterIP: None                  # c'est ce "None" qui le rend headless
  publishNotReadyAddresses: true   # indispensable : les noeuds doivent se
                                   # joindre AVANT d'etre prets
  selector:
    app: cockroachdb
  ports:
    - name: sql
      port: 26257
    - name: http
      port: 8080

---
# 2. StatefulSet : 3 noeuds, chacun avec son volume
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cockroachdb
spec:
  serviceName: cockroachdb         # doit pointer sur le Service headless
  replicas: 3
  selector:
    matchLabels:
      app: cockroachdb
  template:
    metadata:
      labels:
        app: cockroachdb
    spec:
      containers:
        - name: cockroachdb
          image: cockroachdb/cockroach:latest
          ports:
            - containerPort: 26257
            - containerPort: 8080
          command: ["/bin/bash", "-ecx"]
          args:
            # TODO: completez la commande de demarrage.
            #   --advertise-addr doit valoir le nom complet du pod : $(hostname -f)
            #   --join doit lister les trois ordinaux :
            #     cockroachdb-0.cockroachdb, cockroachdb-1.cockroachdb, ...
            #   pensez aussi a --insecure, --listen-addr et --http-addr
            - "exec /cockroach/cockroach start TODO"
          volumeMounts:
            - name: datadir
              mountPath: /cockroach/cockroach-data
  volumeClaimTemplates:
    - metadata:
        name: datadir
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 1Gi
        # pas de storageClassName : la classe par defaut de la grappe s'applique

---
# 3. Service normal : le point d'entree de votre application
# TODO: exposez le port 26257 vers les pods portant le label app=cockroachdb
```

Adaptez ensuite la chaîne de connexion de l'application pour qu'elle pointe sur ce dernier Service au lieu de `cockroach1`.

#### Initialisez le cluster

Le `docker-compose.yml` possède un service `cluster-init` qui exécute `cockroach init`. **Vous ne pouvez pas le reproduire avec un Job** : le groupe d'API `batch` vous est refusé sur la grappe du cours. Faites-le à la main, une seule fois, quand les trois pods sont démarrés :

```bash
kubectl exec -it cockroachdb-0 -- /cockroach/cockroach init --insecure
```

#### Vérifiez

```bash
kubectl get statefulset
kubectl get pods -l app=cockroachdb
kubectl get pvc                       # trois volumes, un par pod
kubectl exec -it cockroachdb-0 -- /cockroach/cockroach node status --insecure
```

Les trois nœuds doivent apparaître, avec leur identifiant et leur adresse.

> ⚠️ **Libérez la place quand vous avez terminé.** La grappe est partagée par les 25 équipes sur un seul serveur, et votre StatefulSet y ajoute trois pods et trois volumes. Dès vos captures prises :
>
> ```bash
> kubectl delete statefulset cockroachdb
> kubectl delete pvc -l app=cockroachdb
> ```
>
> Les volumes ne sont pas supprimés avec le StatefulSet : la deuxième commande n'est pas optionnelle.

> 📝 **Si vos pods restent en `Pending` avec `Too many pods`**, la grappe est saturée et ce n'est pas votre faute. Validez votre manifeste sur un cluster local — `kind`, `k3d` ou le Kubernetes de Docker Desktop — et joignez la capture de l'erreur à votre rapport. Un déploiement validé localement reste recevable.

> 💡 **Question 8** : Pourquoi un `Service` headless est-il nécessaire ici, alors qu'un Service ordinaire suffisait au Labo 05 ? Que contient l'entrée DNS d'un pod du StatefulSet, et que se passerait-il si vous utilisiez un `Deployment` à la place ? Illustrez avec la sortie de `kubectl get pods` et d'un `nslookup` depuis un pod.

### 7. Automatisez avec un pipeline CI/CD

Créez un workflow GitHub Actions qui, à chaque push :

1. exécute `concurrency_test.py` — aucun déploiement ne doit partir si le test de concurrence échoue ;
2. construit l'image de votre application et la publie sur GHCR, comme au Labo 00.

Pour le déploiement lui-même, souvenez-vous que les runners de GitHub sont hébergés hors du réseau de l'ÉTS et ne peuvent pas joindre la grappe. Deux voies :

- automatiser jusqu'à la publication de l'image, et déclencher le déploiement depuis votre machine connectée au VPN avec un script versionné ;
- enregistrer un **runner auto-hébergé** sur une machine connectée au VPN, qui exécutera le `kubectl` depuis le workflow.

La seconde ferme la boucle, mais elle ne fonctionne que lorsque la machine est allumée. Documentez votre choix dans le rapport.


## 📦 Livrables

> ⚠️ **La remise se fait au niveau de la PHASE, pas du labo.** Les Labos 06, 07, 08 et 09 forment la **Phase 3**, remise en une seule fois dans un espace Moodle unique, à la fin de la phase.

Vous déposerez **exactement deux fichiers** dans Moodle, pour toute la Phase 3 :

| # | Fichier | Contenu |
|---|---|---|
| 1 | `LOG430-A26-Phase3-EquipeNN.zip` | Le code source des **4 labos**, un dossier par labo (`labo06/`, `labo07/`, `labo08/`, `labo09/`) |
| 2 | `LOG430-A26-Phase3-EquipeNN.pdf` | Un **rapport unique** en 4 sections, une par labo |

> ⚠️ Le rapport PDF doit être déposé **comme fichier séparé**, et non à l'intérieur du zip.

Ce que **ce labo** doit contenir dans la remise :

- Son code source complet, dans le dossier `labo09/` du zip.
- La section correspondante du rapport, répondant aux **8 questions** de ce fichier. Il est **obligatoire** d'illustrer chaque réponse avec du code ou une sortie de terminal.

