# 🧰 Comandos úteis da AWS CLI para EC2

Referência rápida dos comandos mais usados para gerenciar instâncias EC2 pelo terminal.

> Pré-requisito: AWS CLI instalada e configurada com `aws configure`.

## Listar e consultar instâncias

```bash
# Listar todas as instâncias (ID, tipo, estado e IP público)
aws ec2 describe-instances \
  --query "Reservations[].Instances[].[InstanceId,InstanceType,State.Name,PublicIpAddress]" \
  --output table

# Filtrar apenas instâncias em execução
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running"
```

## Criar uma instância

```bash
aws ec2 run-instances \
  --image-id ami-xxxxxxxx \
  --instance-type t3.micro \
  --key-name minha-chave \
  --security-group-ids sg-xxxxxxxx \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=servidor-dio}]' \
  --user-data file://user-data.sh
```

## Gerenciar o ciclo de vida

```bash
aws ec2 stop-instances      --instance-ids i-xxxxxxxx
aws ec2 start-instances     --instance-ids i-xxxxxxxx
aws ec2 reboot-instances    --instance-ids i-xxxxxxxx
aws ec2 terminate-instances --instance-ids i-xxxxxxxx
```

## Security Groups

```bash
# Criar um security group
aws ec2 create-security-group --group-name web-sg --description "Servidor web"

# Liberar HTTP para todos
aws ec2 authorize-security-group-ingress --group-id sg-xxxxxxxx \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

# Liberar SSH apenas para o seu IP
aws ec2 authorize-security-group-ingress --group-id sg-xxxxxxxx \
  --protocol tcp --port 22 --cidr SEU.IP.AQUI/32
```

## Key Pairs

```bash
aws ec2 create-key-pair --key-name minha-chave \
  --query "KeyMaterial" --output text > minha-chave.pem
chmod 400 minha-chave.pem
```

## Volumes, Snapshots e AMIs

```bash
# Criar snapshot de um volume
aws ec2 create-snapshot --volume-id vol-xxxxxxxx --description "Backup diário"

# Criar AMI a partir de uma instância
aws ec2 create-image --instance-id i-xxxxxxxx --name "minha-ami-dio"

# Listar snapshots da sua conta
aws ec2 describe-snapshots --owner-ids self
```

## Elastic IP

```bash
aws ec2 allocate-address
aws ec2 associate-address --instance-id i-xxxxxxxx --allocation-id eipalloc-xxxxxxxx
aws ec2 release-address --allocation-id eipalloc-xxxxxxxx   # liberar para evitar custos
```
