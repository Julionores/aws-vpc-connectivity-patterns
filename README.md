# AWS VPC Connectivity Patterns

[![CI](https://github.com/Julionores/aws-vpc-connectivity-patterns/actions/workflows/ci.yml/badge.svg)](https://github.com/Julionores/aws-vpc-connectivity-patterns/actions/workflows/ci.yml)

Trois façons de connecter des VPC entre eux ou à des services AWS **sans passer par
Internet** — VPC Peering, Transit Gateway et VPC Endpoints — comparées dans un seul
dépôt, chacune déployée et vérifiée en réel. Les trois templates de cours d'origine
créaient les VPC et leurs instances, mais **jamais la ressource de connectivité elle-même** :
les réseaux restaient totalement étanches malgré leur apparente proximité.

> Projet réalisé par **Junior Tsafack Megnekeu** ([blog.jtmcloud.com](https://blog.jtmcloud.com) ·
> [GitHub](https://github.com/Julionores) ·
> [LinkedIn](https://www.linkedin.com/in/junior-tsafack-megnekeu-b673151b9)) — pièce d'un
> portfolio technique orienté Cloud/DevOps. Voir aussi
> [`dynamodb-streams-cdc-pipeline`](https://github.com/Julionores/dynamodb-streams-cdc-pipeline),
> [`aws-troubleshooting-challenge`](https://github.com/Julionores/aws-troubleshooting-challenge),
> [`s3-cross-region-replication`](https://github.com/Julionores/s3-cross-region-replication) et
> [`aws-alb-deployment-patterns`](https://github.com/Julionores/aws-alb-deployment-patterns).
> Côté DevSecOps/Full Stack, voir aussi
> [`devsecops-pipeline-reference`](https://github.com/Julionores/devsecops-pipeline-reference),
> [`securebank-api`](https://github.com/Julionores/securebank-api),
> [`postgresql-ha-repmgr`](https://github.com/Julionores/postgresql-ha-repmgr) et
> [`iso27001-isms-toolkit`](https://github.com/Julionores/iso27001-isms-toolkit).
> Côté Machine Learning, voir aussi [`gradientforge`](https://github.com/Julionores/gradientforge),
> [`radar-risque-impaye`](https://github.com/Julionores/radar-risque-impaye),
> [`collecte-agricole-planner`](https://github.com/Julionores/collecte-agricole-planner) et
> [`ticket-tide`](https://github.com/Julionores/ticket-tide), une prévision de série
> temporelle (famille ARMA).

## Comparatif

| | VPC Peering | Transit Gateway | VPC Endpoints |
|---|---|---|---|
| **Relie** | 2 VPC entre eux | N VPC en étoile (hub central) | Un VPC à un service AWS (S3, SSM...) |
| **Passe par Internet ?** | Non | Non | Non |
| **Coût** | Gratuit (hors transfert de données inter-AZ/région) | Facturé à l'heure par attachment + par Go transféré | Gateway Endpoint gratuit ; Interface Endpoint facturé à l'heure |
| **Bonne échelle pour...** | 2-3 VPC | De nombreux VPC (évite le maillage complet en O(n²) du peering) | Accès privé à un service AWS spécifique, sans NAT Gateway |
| **Limite notable** | Pas de routage transitif (A↔B et B↔C ne donnent pas A↔C) | Toutes les routes passent par un point central (bien dimensionner) | Un endpoint par service utilisé |

## Pattern 1 — VPC Peering

`cloudformation/template-vpc-peering.yaml` : deux VPC (`10.0.0.0/16` et `10.1.0.0/16`),
chacun avec sa propre instance web, reliés par une connexion de peering **et** les routes
qui l'exploitent réellement (une connexion de peering ne crée aucune route automatiquement).

### Vérification réelle

Depuis l'instance du premier VPC, une requête vers l'**IP privée** (non routable
publiquement) de l'instance du second VPC, via AWS Systems Manager (aucun SSH nécessaire) :

```bash
$ aws ssm send-command --instance-ids <instance-vpc1> --document-name AWS-RunShellScript \
    --parameters 'commands=["curl -s --max-time 5 http://10.1.1.99/"]'
...
"StandardOutputContent": "<h1>Serveur Web dans VPC2</h1>\n"
```

La requête aboutit uniquement parce que la connexion de peering et ses routes existent —
la preuve que le trafic privé transite réellement entre les deux réseaux.

## Pattern 2 — Transit Gateway

`cloudformation/template-transit-gateway.yaml` : 3 VPC (`10.5.0.0/16`, `10.6.0.0/16`,
`10.7.0.0/16`) rattachés à un unique Transit Gateway central, avec propagation de route
automatique (`DefaultRouteTableAssociation`/`DefaultRouteTablePropagation: enable`) — chaque
VPC apprend automatiquement les routes des deux autres, sans configuration manuelle de
tables de routage Transit Gateway dédiées.

C'est l'alternative qui **évite l'explosion combinatoire du peering** : relier 3 VPC en
peering demande 3 connexions ; en relier 10 en demanderait 45 (n(n-1)/2). Avec un Transit
Gateway, chaque VPC n'a besoin que d'**un seul** rattachement, quel que soit le nombre total
de VPC.

### Vérification réelle

Voir [docs/verification.md](docs/verification.md) pour les résultats détaillés (résultats
capturés lors du test réel sur AWS, section mise à jour après chaque exécution).

## Pattern 3 — VPC Endpoints

`cloudformation/template-vpc-endpoint.yaml` : une instance EC2 **totalement privée** (aucune
Internet Gateway, aucun NAT Gateway, aucune route `0.0.0.0/0`) qui :

- accède à Amazon S3 via un **Gateway Endpoint** (gratuit) ;
- reste administrable via **AWS Systems Manager Session Manager**, grâce à 3 **Interface
  Endpoints** (`ssm`, `ssmmessages`, `ec2messages`) — sans exposer le moindre port à Internet.

### Vérification réelle

```bash
# Accès S3 malgré l'absence totale de route Internet
$ aws ssm send-command ... --parameters 'commands=["aws s3 ls"]'
"StandardOutputContent": "2026-08-24 16:26:01 polly-s3-junior\n2026-05-21 15:00:27 transcribeawstest1\n"

# Confirmation qu'aucun accès Internet général n'existe
$ aws ssm send-command ... --parameters 'commands=["curl --max-time 6 https://www.google.com"]'
"StandardOutputContent": "HTTP_CODE=000 TIME=6.001093CURL_FAILED_EXIT_28\n"
```

Le code de sortie curl `28` (timeout) et `HTTP_CODE=000` confirment l'absence totale de
route Internet, tandis que `aws s3 ls` réussit et retourne les vrais buckets du compte —
la preuve que **seul** le trafic vers S3 passe, via le Gateway Endpoint, sans aucune autre
sortie possible.

## Ce qui a été corrigé par rapport aux templates de cours d'origine

| Pattern | Ressource manquante ajoutée |
|---|---|
| VPC Peering | `AWS::EC2::VPCPeeringConnection` + routes dans les deux tables de routage |
| Transit Gateway | `AWS::EC2::TransitGateway` + 3 `AWS::EC2::TransitGatewayAttachment` + routes |
| VPC Endpoints | `AWS::EC2::VPCEndpoint` (1 Gateway + 3 Interface) |

Dans les trois cas, les security groups ont aussi été resserrés (accès restreint aux CIDR
des VPC pairs plutôt qu'ouverture large), et l'authentification SSH a été remplacée par AWS
Systems Manager Session Manager — aucune des instances de ce dépôt n'expose le port 22.

## Déploiement

Chaque pattern est indépendant et peut être déployé séparément :

```bash
aws cloudformation deploy --template-file cloudformation/template-vpc-peering.yaml \
  --stack-name vpc-peering-demo --capabilities CAPABILITY_IAM --region eu-west-1

aws cloudformation deploy --template-file cloudformation/template-transit-gateway.yaml \
  --stack-name transit-gateway-demo --capabilities CAPABILITY_IAM --region eu-west-1

aws cloudformation deploy --template-file cloudformation/template-vpc-endpoint.yaml \
  --stack-name vpc-endpoint-demo --capabilities CAPABILITY_IAM --region eu-west-1
```

Nettoyage : `aws cloudformation delete-stack --stack-name <nom>`.

## Validation

```bash
pip install cfn-lint
cfn-lint cloudformation/template-vpc-peering.yaml
cfn-lint cloudformation/template-transit-gateway.yaml
cfn-lint cloudformation/template-vpc-endpoint.yaml
```

## Limites assumées

- Une seule zone de disponibilité par VPC dans les trois patterns : suffisant pour
  démontrer la connectivité, pas représentatif d'une architecture haute disponibilité réelle.
- Le pattern Transit Gateway utilise la table de routage par défaut avec association et
  propagation automatiques : adapté à une démo « full mesh », mais une architecture de
  production voudrait généralement des tables de routage Transit Gateway dédiées par
  segment (production / non-production, par exemple) pour un contrôle plus fin.
- Le pattern VPC Peering ne couvre que 2 VPC : au-delà de 3-4 VPC, le nombre de connexions
  nécessaires croît en O(n²) — c'est précisément la limite que le pattern Transit Gateway
  résout.

## Licence

MIT — voir [`LICENSE`](LICENSE). Projet à but pédagogique et de démonstration.
