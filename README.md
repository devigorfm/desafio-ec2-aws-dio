# ☁️ Gerenciamento de Instâncias EC2 na AWS

Repositório criado como entrega do desafio da **DIO** para consolidar os conhecimentos sobre **Amazon EC2 (Elastic Compute Cloud)**. Aqui estão minhas anotações, os processos que pratiquei e os principais insights adquiridos durante as aulas.

---

## 📑 Sumário

1. [O que é o Amazon EC2](#-o-que-é-o-amazon-ec2)
2. [Conceitos fundamentais](#-conceitos-fundamentais)
3. [Ciclo de vida de uma instância](#-ciclo-de-vida-de-uma-instância)
4. [Passo a passo: criando uma instância](#-passo-a-passo-criando-uma-instância)
5. [Armazenamento: EBS, Snapshots e AMIs](#-armazenamento-ebs-snapshots-e-amis)
6. [Segurança e acesso](#-segurança-e-acesso)
7. [Modelos de compra e custos](#-modelos-de-compra-e-custos)
8. [Escalabilidade e alta disponibilidade](#-escalabilidade-e-alta-disponibilidade)
9. [Monitoramento](#-monitoramento)
10. [Boas práticas](#-boas-práticas)
11. [Insights e aprendizados](#-insights-e-aprendizados)
12. [Referências](#-referências)

---

## 🖥️ O que é o Amazon EC2

O **Amazon EC2** é o serviço da AWS que fornece **capacidade computacional redimensionável na nuvem**. Na prática, é uma máquina virtual (servidor) que você cria em minutos, paga pelo uso e pode aumentar, diminuir ou desligar quando quiser — sem precisar comprar hardware.

**Casos de uso comuns:**
- Hospedar aplicações web e APIs
- Servidores de banco de dados autogerenciados
- Ambientes de desenvolvimento e testes
- Processamento em lote, machine learning e renderização

---

## 📚 Conceitos fundamentais

| Conceito | Descrição |
|---|---|
| **Instância** | A máquina virtual em si, rodando na infraestrutura da AWS. |
| **AMI (Amazon Machine Image)** | "Molde" usado para criar a instância: contém o sistema operacional e, opcionalmente, softwares e configurações. |
| **Tipo de instância** | Define CPU, memória, rede e armazenamento (ex.: `t2.micro`, `t3.medium`, `m5.large`). |
| **EBS (Elastic Block Store)** | Disco persistente anexado à instância. |
| **Instance Store** | Armazenamento temporário físico do host — os dados são perdidos ao parar a instância. |
| **Security Group** | Firewall virtual que controla o tráfego de entrada e saída da instância. |
| **Key Pair** | Par de chaves (pública/privada) usado para acesso seguro via SSH ou para obter a senha no Windows. |
| **Região e Zona de Disponibilidade (AZ)** | Região é a localização geográfica; AZs são data centers isolados dentro de uma região. |
| **Elastic IP** | IP público estático que pode ser associado/reassociado a instâncias. |
| **User Data** | Script executado na primeira inicialização da instância (útil para automatizar configurações). |

### Famílias de tipos de instância

| Família | Foco | Exemplos |
|---|---|---|
| **Uso geral** | Equilíbrio entre CPU, memória e rede | `t3`, `t4g`, `m5`, `m7i` |
| **Otimizada para computação** | Processamento intenso | `c5`, `c7g` |
| **Otimizada para memória** | Grandes volumes de dados em memória | `r5`, `x2idn` |
| **Computação acelerada** | GPUs / chips de ML | `p4`, `g5`, `inf2` |
| **Otimizada para armazenamento** | Alto I/O em disco local | `i3`, `d3` |

> 💡 **Lendo o nome do tipo:** em `m5.large`, `m` é a família, `5` a geração e `large` o tamanho. Letras extras indicam variações, como `g` (processador Graviton/ARM), `a` (AMD) ou `d` (disco NVMe local).

---

## 🔄 Ciclo de vida de uma instância

```
            launch
              │
              ▼
          [pending] ──────► [running] ◄────────┐
                              │   │            │
                     stop     │   │ reboot     │ start
                              ▼   └──► (running)│
                          [stopping] ──► [stopped]
                              
          terminate (a partir de running ou stopped)
                              │
                              ▼
                   [shutting-down] ──► [terminated]
```

| Estado | Cobrança de computação | Observação |
|---|---|---|
| `pending` | Não | Instância sendo preparada. |
| `running` | **Sim** | Instância em uso. |
| `stopping` / `stopped` | Não | Volumes EBS continuam sendo cobrados. |
| `rebooting` | Sim | Mantém IP público, dados e host. |
| `terminated` | Não | Instância excluída definitivamente. |

**Diferenças importantes:**
- **Stop:** desliga a instância, preserva o volume EBS. Ao iniciar novamente, ela pode ir para outro host e **o IP público muda** (exceto com Elastic IP).
- **Reboot:** reinicia o sistema operacional sem trocar de host; IP é mantido.
- **Terminate:** exclui a instância. Por padrão, o volume raiz também é apagado (`DeleteOnTermination = true`).
- **Hibernate:** salva o conteúdo da RAM no volume EBS, permitindo retomar rapidamente de onde parou.

---

## 🛠️ Passo a passo: criando uma instância

1. Acessar o **Console da AWS** → serviço **EC2** → **Launch instance**.
2. **Nome e tags:** definir um nome (ex.: `servidor-web-dio`). Tags ajudam a organizar e rastrear custos.
3. **AMI:** escolher o sistema operacional (ex.: Amazon Linux 2023 ou Ubuntu Server).
4. **Tipo de instância:** selecionar um tipo elegível ao nível gratuito, como `t2.micro` ou `t3.micro`.
5. **Key pair:** criar ou selecionar um par de chaves e **guardar o arquivo `.pem` em local seguro** (ele só pode ser baixado uma vez).
6. **Configurações de rede:** escolher VPC/sub-rede e configurar o **Security Group** (ex.: liberar SSH porta 22 apenas para o meu IP e HTTP porta 80 para todos).
7. **Armazenamento:** definir o tamanho e o tipo do volume EBS (ex.: 8 GiB `gp3`).
8. **Detalhes avançados (opcional):** adicionar um script de **User Data**.
9. Revisar e clicar em **Launch instance**.

### Exemplo de User Data (servidor web Apache)

```bash
#!/bin/bash
dnf update -y
dnf install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Olá da minha instância EC2 - Desafio DIO</h1>" > /var/www/html/index.html
```

### Conectando via SSH

```bash
chmod 400 minha-chave.pem
ssh -i minha-chave.pem ec2-user@<IP-PUBLICO>   # Amazon Linux
ssh -i minha-chave.pem ubuntu@<IP-PUBLICO>     # Ubuntu
```

> Também é possível usar o **EC2 Instance Connect** ou o **Session Manager (Systems Manager)** direto pelo console, sem precisar abrir a porta 22.

---

## 💾 Armazenamento: EBS, Snapshots e AMIs

### Tipos de volume EBS

| Tipo | Categoria | Uso indicado |
|---|---|---|
| `gp3` / `gp2` | SSD de uso geral | Maioria das cargas de trabalho, volumes de boot |
| `io2` / `io1` | SSD de IOPS provisionadas | Bancos de dados críticos com alta exigência de I/O |
| `st1` | HDD otimizado para throughput | Big data, logs, processamento sequencial |
| `sc1` | HDD frio | Dados acessados raramente, menor custo |

**Pontos-chave sobre EBS:**
- Um volume EBS fica **em uma única AZ** e só pode ser anexado a instâncias da mesma AZ.
- Pode ser desanexado de uma instância e anexado a outra.
- É possível **aumentar o tamanho** e alterar o tipo sem parar a instância (Elastic Volumes).

### Snapshots
- São **backups incrementais** de um volume EBS armazenados no Amazon S3.
- Permitem restaurar volumes, **copiar dados entre AZs e regiões** e criar AMIs.
- Podem ser automatizados com o **Amazon Data Lifecycle Manager**.

### AMIs personalizadas
- Depois de configurar uma instância (softwares, ajustes, etc.), é possível gerar uma **AMI própria** a partir dela.
- Essa AMI serve para criar novas instâncias idênticas rapidamente — base para padronização e para o Auto Scaling.

---

## 🔐 Segurança e acesso

### Security Groups
- Funcionam como **firewall no nível da instância**.
- São **stateful**: se o tráfego de entrada é permitido, a resposta sai automaticamente.
- Só possuem regras de **permissão** (não existe regra de "negar").
- Por padrão: **todo tráfego de entrada é bloqueado** e todo tráfego de saída é liberado.

Exemplo de regras de entrada para um servidor web:

| Tipo | Protocolo | Porta | Origem |
|---|---|---|---|
| SSH | TCP | 22 | Meu IP (`x.x.x.x/32`) |
| HTTP | TCP | 80 | `0.0.0.0/0` |
| HTTPS | TCP | 443 | `0.0.0.0/0` |

> ⚠️ Nunca liberar SSH (22) ou RDP (3389) para `0.0.0.0/0` em ambientes reais.

### IAM Roles para instâncias
- Em vez de guardar chaves de acesso (`access keys`) dentro do servidor, associa-se uma **IAM Role** à instância.
- A aplicação recebe credenciais temporárias automaticamente para acessar outros serviços (ex.: S3).

### Outras práticas de segurança
- Usar o **IMDSv2** (Instance Metadata Service v2) para proteger os metadados da instância.
- Habilitar **criptografia nos volumes EBS**.
- Manter o sistema operacional atualizado (Systems Manager Patch Manager ajuda nisso).

---

## 💰 Modelos de compra e custos

| Modelo | Como funciona | Quando usar |
|---|---|---|
| **On-Demand** | Paga por segundo/hora, sem compromisso | Cargas imprevisíveis, testes, curto prazo |
| **Savings Plans** | Compromisso de gasto ($/hora) por 1 ou 3 anos, com desconto | Uso contínuo com flexibilidade de tipo/região |
| **Reserved Instances** | Compromisso de 1 ou 3 anos com configuração específica | Cargas estáveis e previsíveis |
| **Spot Instances** | Usa capacidade ociosa da AWS com grandes descontos; pode ser interrompida com 2 minutos de aviso | Tarefas tolerantes a falhas, processamento em lote |
| **Dedicated Hosts / Instances** | Hardware físico dedicado | Requisitos de licenciamento ou conformidade |

**O que gera custo no EC2:**
- Tempo de execução da instância (`running`)
- Volumes EBS e snapshots (mesmo com a instância parada)
- Elastic IPs e endereços IPv4 públicos
- Transferência de dados para fora da AWS

> 💡 **Dica de ouro para estudantes:** ao terminar os laboratórios, **encerre (terminate)** as instâncias, apague volumes e snapshots que não usa e libere Elastic IPs. Configure também um **alerta de orçamento no AWS Budgets** para evitar surpresas na fatura.

---

## 📈 Escalabilidade e alta disponibilidade

- **Escalabilidade vertical:** trocar o tipo da instância por um maior (requer parar a instância).
- **Escalabilidade horizontal:** adicionar mais instâncias para dividir a carga.

### Elastic Load Balancing (ELB)
Distribui o tráfego entre várias instâncias e AZs, verificando a saúde de cada uma (health checks).
- **Application Load Balancer (ALB):** camada 7 (HTTP/HTTPS), roteamento por caminho ou host.
- **Network Load Balancer (NLB):** camada 4 (TCP/UDP), altíssimo desempenho.

### Auto Scaling Group (ASG)
- Mantém um número **mínimo, desejado e máximo** de instâncias.
- Cria ou remove instâncias automaticamente com base em métricas (ex.: CPU acima de 70%).
- Substitui instâncias com falha automaticamente.
- Usa um **Launch Template** (AMI, tipo, Security Group, User Data) como modelo.

```
          Usuários
             │
             ▼
   ┌───────────────────┐
   │   Load Balancer   │
   └───────────────────┘
       │           │
       ▼           ▼
  ┌────────┐   ┌────────┐
  │ EC2    │   │ EC2    │   ◄── Auto Scaling Group
  │ AZ-a   │   │ AZ-b   │
  └────────┘   └────────┘
```

---

## 📊 Monitoramento

- **Amazon CloudWatch:** coleta métricas como uso de CPU, tráfego de rede e leitura/escrita em disco.
  - Monitoramento básico: métricas a cada 5 minutos.
  - Monitoramento detalhado: métricas a cada 1 minuto (custo adicional).
  - Memória e uso de disco exigem o **CloudWatch Agent** instalado na instância.
- **Status checks:** verificam a saúde do sistema (hardware AWS) e da instância (sistema operacional).
- **Alarmes CloudWatch:** podem enviar notificações (SNS) ou executar ações como parar, reiniciar ou recuperar a instância.

---

## ✅ Boas práticas

- [ ] Usar **tags** em todos os recursos (nome, projeto, ambiente, responsável).
- [ ] Escolher o **tipo de instância adequado** (right-sizing) e revisar periodicamente.
- [ ] Restringir os Security Groups ao **mínimo necessário**.
- [ ] Usar **IAM Roles** em vez de credenciais fixas nos servidores.
- [ ] Criptografar volumes EBS e fazer **snapshots regulares**.
- [ ] Distribuir instâncias em **múltiplas AZs** para alta disponibilidade.
- [ ] Ativar **proteção contra encerramento** (termination protection) em instâncias críticas.
- [ ] Monitorar custos com **AWS Budgets** e **Cost Explorer**.
- [ ] Encerrar recursos de teste ao fim de cada estudo.

---

## 💡 Insights e aprendizados

- **Parar ≠ encerrar:** parar uma instância interrompe a cobrança de computação, mas o disco EBS continua gerando custo. Para zerar o custo, é preciso encerrar e remover volumes/snapshots.
- **O IP público muda:** descobri que, ao parar e iniciar a instância, o IP público é trocado. Para manter um endereço fixo, é necessário um Elastic IP.
- **A chave `.pem` é única:** se o arquivo for perdido, não é possível baixá-lo de novo — o acesso precisa ser recuperado por outros caminhos (como Session Manager ou trocando o volume de instância).
- **User Data economiza tempo:** automatizar a instalação de pacotes na criação evita repetir configurações manuais.
- **AMIs e Auto Scaling se complementam:** com uma AMI pronta e um Launch Template, novas instâncias sobem já configuradas.
- **Segurança começa no Security Group:** liberar apenas as portas necessárias e restringir o SSH ao próprio IP reduz bastante a superfície de ataque.

---

## 🔗 Referências

- [Documentação oficial do Amazon EC2](https://docs.aws.amazon.com/pt_br/AWSEC2/latest/UserGuide/concepts.html)
- [Tipos de instância do Amazon EC2](https://aws.amazon.com/pt/ec2/instance-types/)
- [Definição de preços do Amazon EC2](https://aws.amazon.com/pt/ec2/pricing/)
- [Documentação do GitHub](https://docs.github.com/pt)
- [Sintaxe Markdown no GitHub](https://docs.github.com/pt/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax)
- Comandos úteis da AWS CLI: [`comandos-aws-cli.md`](comandos-aws-cli.md)

---

Feito com 💙 durante o bootcamp da [DIO](https://www.dio.me/).
