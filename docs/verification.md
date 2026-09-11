# Vérification réelle -- Transit Gateway

Ce document capture les résultats du test réel effectué sur AWS (région `eu-west-1`) pour
le pattern Transit Gateway (`cloudformation/template-transit-gateway.yaml`).

## Déploiement

```bash
$ aws cloudformation deploy --template-file cloudformation/template-transit-gateway.yaml \
    --stack-name transit-gateway-test --capabilities CAPABILITY_IAM --region eu-west-1
Successfully created/updated stack - transit-gateway-test
```

## Test de connectivité complète (maillage à 3 VPC)

Avec `DefaultRouteTableAssociation`/`DefaultRouteTablePropagation` activés, chaque VPC
attaché doit pouvoir joindre les deux autres sans configuration additionnelle. Le test
consiste à interroger, depuis l'instance de VPC1, les IP privées des instances de VPC2 et
VPC3 via AWS Systems Manager (aucun SSH) :

```bash
$ aws ssm send-command --instance-ids i-0a7b42b5308f5b141 --document-name AWS-RunShellScript \
    --parameters 'commands=["echo --- VPC2 ---; curl -s --max-time 5 http://10.6.1.103/","echo --- VPC3 ---; curl -s --max-time 5 http://10.7.1.173/"]'

$ aws ssm get-command-invocation --command-id <id> --instance-id i-0a7b42b5308f5b141
{
  "Status": "Success",
  "StandardOutputContent": "--- VPC2 ---\n<h1>Serveur dans VPC2</h1>\n--- VPC3 ---\n<h1>Serveur dans VPC3</h1>\n"
}
```

Depuis l'instance de **VPC1** (`10.5.0.0/16`), les instances de **VPC2** (`10.6.0.0/16`) et
**VPC3** (`10.7.0.0/16`) sont toutes deux joignables sur leur IP privée -- confirmant que la
propagation de route automatique du Transit Gateway relie bien les 3 VPC en étoile, sans
qu'aucune connexion de peering point à point n'ait été nécessaire.

## Nettoyage

```bash
aws cloudformation delete-stack --stack-name transit-gateway-test --region eu-west-1
```

Stack supprimée après test -- aucune ressource (Transit Gateway, VPC, instances,
attachments) ne reste active.
