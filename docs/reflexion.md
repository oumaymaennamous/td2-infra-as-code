# TD2 — Réponses aux 4 questions de réflexion

## Q1 — Pourquoi utilise-t-on des data sources pour `asl-net` et pas des resources ? Que se passerait-il si on inversait ?

Le réseau `asl-net` existe déjà : il fait partie du socle OpenStack préparé au TD1. Une **data source** permet de retrouver ses attributs, notamment son identifiant, afin de connecter notre VM à ce réseau sans le recréer ni en prendre la gestion.

Dans `terraform/data.tf`, `data "openstack_networking_network_v2" "asl_net"` référence ce réseau. La VM définie dans `compute.tf` utilise ensuite cette référence.

Une **resource** décrit au contraire un objet que Terraform doit gérer dans son état. Remplacer la data source par une resource sans importer le réseau existant ferait tenter à Terraform de créer un nouveau réseau ; selon les contraintes OpenStack, cela produirait un doublon ou une erreur. Terraform ne reconnaît pas automatiquement un objet existant à partir de son seul nom.

Si nous importions le réseau dans une resource, il entrerait dans le périmètre de gestion de ce projet et pourrait être supprimé lors d’un `terraform destroy`. Utiliser une data source préserve donc le socle partagé du TD1.

## Q2 — Quelle est la frontière exacte entre ce que fait Terraform et ce que fait Ansible ? Donnez un exemple de tâche qui ne peut pas aller dans l’autre outil.

**Terraform provisionne les ressources OpenStack ; Ansible configure le système à l’intérieur de la VM.**

| Outil | Responsabilité dans ce TD | Exemple du dépôt |
|---|---|---|
| Terraform | Créer et relier les ressources cloud | Créer la VM, le Security Group, les règles et l’IP flottante ; associer cette IP au port Neutron |
| Ansible | Préparer Ubuntu et gérer ses logiciels et services | Configurer le DNS, installer K3s, écrire sa configuration et démarrer son service |
| Helm | Déployer les couches applicatives dans Kubernetes | Installer les releases `postgres`, `backend` et `frontend` |

Par exemple, **associer une IP flottante à un port Neutron** nécessite une opération sur l’API OpenStack : ce n’est pas une tâche de configuration du système invité. Inversement, **écrire `/etc/rancher/k3s/config.yaml` puis démarrer le service K3s** ne relève pas des ressources du provider OpenStack : c’est le rôle d’Ansible.

Cette frontière décrit la répartition choisie dans le TD, et non une impossibilité absolue : Ansible possède aussi des modules cloud et Terraform peut lancer des provisioners. Nous évitons ce mélange pour garder des responsabilités et des cycles de vie clairs.

## Q3 — Pourquoi un StatefulSet et pas un Deployment pour PostgreSQL ? Citez 2 différences concrètes.

PostgreSQL conserve des données qui doivent rester associées à une instance identifiable. Le **StatefulSet** fournit deux propriétés utiles dans notre déploiement :

1. **Une identité stable.** Le pod de la base s’appelle `postgres-0`, y compris après sa recréation. Le StatefulSet est associé au Service headless `postgres`, qui permet une identité réseau stable. Les pods d’un Deployment sont interchangeables et leurs noms ne constituent pas une identité stable.
2. **Un volume associé à cette identité.** Le bloc `volumeClaimTemplates` crée le PVC `data-postgres-0`. Lorsqu’un pod de remplacement reprend l’identité `postgres-0`, il réutilise ce PVC et retrouve les données. Un Deployment ne crée pas automatiquement un PVC propre à chaque réplique avec `volumeClaimTemplates` ; il faudrait gérer le stockage autrement.

Dans le chart du dépôt, PostgreSQL utilise un seul réplica, un volume de **1 Gi**, la StorageClass `local-path` et `PGDATA=/var/lib/postgresql/data/pgdata`.

Le StatefulSet ne suffit pas à rendre PostgreSQL hautement disponible : il ne configure pas de réplication de la base. De plus, le stockage `local-path` repose sur le disque de la VM ; il ne protège pas les données contre la destruction de cette VM.

## Q4 — Expliquez en 3 lignes pourquoi on utilise un tunnel SSH plutôt que d’ouvrir le port 6443 dans le SG.

Le port 6443 donne accès à l’API d’administration Kubernetes : le laisser fermé dans le Security Group réduit son exposition au réseau extérieur.  
Le tunnel SSH transporte les requêtes de Helm et kubectl depuis `127.0.0.1:6443` sur DevStack vers `127.0.0.1:6443` sur la VM, via le port SSH 22 déjà autorisé.  
Nous conservons ainsi un accès d’administration chiffré et authentifié par SSH, ainsi que l’authentification et la vérification TLS Kubernetes du kubeconfig, sans ouvrir un port supplémentaire.

