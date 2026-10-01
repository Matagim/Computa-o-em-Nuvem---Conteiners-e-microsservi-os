# Atividade Prática — Containers e Microsserviços na AWS

Nesta atividade vamos construir, passo a passo, uma aplicação de **dois microsserviços** (`web` e `api`) e fazer ela rodar na AWS usando, em conjunto, todos os serviços estudados:

| Serviço | Papel na atividade | Onde aparece |
|---|---|---|
| **Docker** | Empacotar `web` e `api` em imagens | Parte 1 |
| **Amazon ECR** | Guardar as imagens (tags, lifecycle, scan) | Parte 2 |
| **AWS CloudFormation** | Criar tudo por código (parâmetros, outputs, limpeza) | Partes 2, 3 e 7 |
| **Amazon ECS** | Cluster, task definition, service, desired count, deployment | Partes 3, 4 e 5 |
| **AWS Fargate** | Executar as tasks sem gerenciar servidores | Partes 3, 4 e 5 |
| **Elastic Load Balancing (ALB)** | Porta de entrada única, roteamento por caminho, health checks | Partes 3, 4 e 5 |
| **Amazon EKS** | Mesma aplicação no Kubernetes (opcional/avançado) | Parte 6 |

## Arquitetura final

```text
                       Internet
                          |
                  +-------v--------+
                  |  ALB  (porta 80)|   <- Listener
                  +---+--------+---+
        /  (padrão)   |        |   /api/*  (regra de listener)
              +-------v--+  +--v---------+
              | TG  web  |  |  TG  api   |   <- Target Groups + health checks
              +----+-----+  +----+-------+
                   |             |
        +----------v---+    +----v--------------------+
        | Service web  |    | Service api             |   <- ECS Services
        | task (8080)  |    | task (3000) x N tasks   |   <- Fargate (awsvpc)
        +--------------+    +-------------------------+
                   ^                   ^
                   +----- imagens -----+
                        Amazon ECR
        (tudo acima criado por 2 stacks do CloudFormation)
```

O que a demo prova:

- O ALB distribui as chamadas entre **várias tasks** da `api` (a página mostra qual task respondeu).
- Se uma task cai, o ECS **cria outra sozinho** (desired count).
- Uma nova versão entra por **rolling deployment**, sem derrubar a aplicação.
- O mesmo código roda de forma repetível: apagou tudo, rodou de novo, voltou igual (CloudFormation).

> Todos os comandos foram pensados para o **AWS CloudShell** na região `us-east-1`, em uma conta do AWS Academy Learner Lab (usa a `LabRole`, já existente, porque criar IAM Roles novas costuma ser bloqueado).

---

# Acesso — entrando no ambiente do AWS Academy

Antes do Passo 0, é preciso "ligar" o laboratório. É diferente de uma conta AWS normal: você não usa e-mail/senha próprios, e o acesso expira.

1. **Abra o Learner Lab.** No painel do curso (Vocareum), entre no módulo do laboratório e clique em **Start Lab**. O indicador ao lado começa cinza/vermelho e fica **verde** depois de 1 a 3 minutos — é o AWS "ligando" a conta pra você.
2. **Abra o console.** Com o indicador verde, clique no botão **AWS** (ao lado do Start Lab). Ele abre o AWS Management Console em outra aba, **já autenticado** — não existe usuário/senha pra digitar aqui.
3. **Confira a região.** No canto superior direito do console deve aparecer **N. Virginia** (`us-east-1`), a mesma região usada em todos os comandos deste guia. Na maioria dos cursos da Academy essa região vem travada — se o seletor estiver bloqueado, é esperado.
4. **Abra o CloudShell.** Na barra superior do console, clique no ícone de terminal (`>_`), perto do sino de notificações. Na primeira vez leva cerca de 1 minuto para provisionar. Ele já vem com a **AWS CLI** e o **Docker** pré-instalados e autenticados com as credenciais da sua sessão — não é preciso rodar `aws configure` nem instalar nada; é só o que o Passo 0 confirma com `docker --version` e `aws --version`.

> **Sobre permissões:** essa é uma conta restrita de estudante, não uma conta de administrador. Tentar criar uma IAM Role, um usuário IAM ou mexer em cobrança/orçamento vai dar `AccessDenied` — isso é esperado, não é erro seu. É exatamente por isso que a atividade inteira reaproveita a `LabRole` que a Academy já deixa pronta, em vez de criar roles novas (ver Passo 12 e a tabela de Problemas Comuns).

> **Sobre o tempo de sessão:** o Learner Lab tem um cronômetro (geralmente 4 horas, renovável clicando em Start Lab de novo) e um orçamento de crédito limitado. Quando o tempo acaba ou você clica em **End Lab**, o acesso ao console e ao CloudShell para — por isso, mesmo que a Academy pause os recursos sozinha ao encerrar, **sempre rode a Parte 7 (limpeza) antes de sair**, pra não arriscar consumir orçamento à toa entre uma sessão e outra.

> Se o CloudShell for reaberto numa sessão nova (ou reiniciar sozinho), as variáveis de ambiente do Passo 0 se perdem — é só rodar aquele passo de novo, como o próprio guia já lembra ao longo do texto.

Fontes: [Docker no AWS CloudShell](https://aws.amazon.com/about-aws/whats-new/2024/01/aws-cloudshell-docker-13-regions/) · [Começando com o AWS CloudShell](https://docs.aws.amazon.com/cloudshell/latest/userguide/getting-started)

---

# Passo 0 — Preparar o ambiente

Definimos variáveis que serão usadas o tempo todo. **Se o CloudShell reiniciar, rode este passo de novo.**

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export REGISTRY=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
export AWS_DEFAULT_REGION=$AWS_REGION

docker --version
aws --version
echo "Conta: $ACCOUNT_ID | Registry: $REGISTRY"
```

Devem aparecer as versões do Docker e da AWS CLI e o ID da sua conta (12 dígitos).

---

# Parte 1 — Docker: empacotar os dois microsserviços

Conceitos desta parte: **imagem vs container**, **Dockerfile**, **layers**, **portas**, **volumes**, **build/run**.

---

## Passo 1 — Criar o microsserviço `api`

A `api` responde em `/api/info` dizendo qual container respondeu (o `hostname` muda a cada container/task) e qual a versão. Ela também tem `/api/health`, que o ALB usará depois.

```bash
mkdir -p ~/cn2026/api ~/cn2026/web
cd ~/cn2026/api

cat <<'EOF' > package.json
{
  "name": "cn2026-api",
  "version": "1.0.0",
  "main": "server.js",
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF

cat <<'EOF' > server.js
const express = require('express');
const os = require('os');
const app = express();

const VERSAO = 'v1';
const inicio = Date.now();
let contador = 0;

app.get('/api/health', (req, res) => res.json({ status: 'ok' }));

app.get('/api/info', (req, res) => {
  contador++;
  res.json({
    servico: 'api',
    hostname: os.hostname(),
    versao: VERSAO,
    requisicoes: contador,
    uptime_s: Math.round((Date.now() - inicio) / 1000)
  });
});

app.listen(3000, () => console.log('API ativa na porta 3000'));
EOF

cat <<'EOF' > Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json .
RUN npm install --omit=dev
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
EOF
```

`ls -la` deve mostrar `package.json`, `server.js` e `Dockerfile`.

> Repare na ordem do Dockerfile: o `package.json` é copiado **antes** do resto do código. Assim a camada do `npm install` fica em cache enquanto as dependências não mudarem.

---

## Passo 2 — Criar o microsserviço `web`

A `web` entrega uma página HTML que, a cada segundo, chama `/api/info` (o ALB encaminha para a `api`) e mostra quantas respostas cada task deu.

```bash
cd ~/cn2026/web

cat <<'EOF' > package.json
{
  "name": "cn2026-web",
  "version": "1.0.0",
  "main": "server.js",
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF

cat <<'EOF' > server.js
const express = require('express');
const os = require('os');
const fs = require('fs');
const path = require('path');
const app = express();

const pagina = fs.readFileSync(path.join(__dirname, 'index.html'), 'utf8');

app.get('/health', (req, res) => res.send('ok'));
app.get('/', (req, res) => res.send(pagina.replace('__HOST__', os.hostname())));

app.listen(8080, () => console.log('Web ativo na porta 8080'));
EOF

cat <<'EOF' > index.html
<!doctype html>
<html lang="pt-BR">
<head>
  <meta charset="utf-8">
  <title>CN 2026 - Microsservicos</title>
  <style>
    body { font-family: sans-serif; max-width: 680px; margin: 40px auto; padding: 0 16px; }
    .caixa { border: 1px solid #ccc; border-radius: 8px; padding: 12px 16px; margin: 12px 0; }
    td, th { padding: 4px 12px; text-align: left; }
  </style>
</head>
<body>
  <h1>Microsservicos na AWS</h1>
  <div class="caixa">Servico <b>web</b> respondendo da task: <b>__HOST__</b></div>
  <div class="caixa">
    Ultima resposta da <b>api</b>: task <b id="host">-</b> | versao <b id="versao">-</b>
  </div>
  <h3>Respostas de cada task da api (distribuidas pelo ALB)</h3>
  <table id="tabela"></table>
  <script>
    var contagem = {};
    var cabecalho = '<tr><th>Task (hostname)</th><th>Versao</th><th>Respostas</th></tr>';
    function atualizar() {
      fetch('/api/info')
        .then(function (r) { return r.json(); })
        .then(function (d) {
          document.getElementById('host').textContent = d.hostname;
          document.getElementById('versao').textContent = d.versao;
          var chave = d.hostname + '|' + d.versao;
          contagem[chave] = (contagem[chave] || 0) + 1;
          var html = cabecalho;
          Object.keys(contagem).forEach(function (k) {
            var p = k.split('|');
            html += '<tr><td>' + p[0] + '</td><td>' + p[1] + '</td><td>' + contagem[k] + '</td></tr>';
          });
          document.getElementById('tabela').innerHTML = html;
        })
        .catch(function () {
          document.getElementById('host').textContent = 'erro ao chamar a api';
        });
    }
    atualizar();
    setInterval(atualizar, 1000);
  </script>
</body>
</html>
EOF

cat <<'EOF' > Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json .
RUN npm install --omit=dev
COPY . .
EXPOSE 8080
CMD ["node", "server.js"]
EOF
```

`ls -la` deve mostrar quatro arquivos: `package.json`, `server.js`, `index.html` e `Dockerfile`.

---

## Passo 3 — Gerar as imagens (build) e enxergar as layers

```bash
cd ~/cn2026
docker build -t cn2026-api:v1 ./api
docker build -t cn2026-web:v1 ./web
```

O log de cada build deve terminar com `naming to docker.io/library/cn2026-...:v1`, sem linhas `ERROR`.

Agora veja as **layers** (cada instrução do Dockerfile vira uma camada) e o tamanho:

```bash
docker history cn2026-api:v1
docker images | grep cn2026
```

Rode o build da `api` de novo e observe que ele termina em segundos, com `CACHED` nas camadas:

```bash
docker build -t cn2026-api:v1 ./api
```

---

## Passo 4 — Rodar (run), mapear portas e entender imagem vs container

**Imagem** é o molde (somente leitura). **Container** é uma instância em execução dessa imagem. Vamos criar **dois containers da mesma imagem**, em portas diferentes do host:

```bash
docker run -d --name api-1 -p 3000:3000 cn2026-api:v1
docker run -d --name api-2 -p 9000:3000 cn2026-api:v1

curl -s localhost:3000/api/info; echo
curl -s localhost:9000/api/info; echo
```

Cada resposta deve trazer um `hostname` diferente (o ID de cada container), mesmo sendo a mesma imagem. O formato `-p 9000:3000` é `PORTA_DO_HOST:PORTA_DO_CONTAINER`.

Inspecione os containers:

```bash
docker ps
docker logs api-1
docker exec api-1 ls /app
```

`docker ps` mostra os dois com status `Up` e as portas mapeadas; os logs mostram `API ativa na porta 3000`.

---

## Passo 5 — Volumes: dados que sobrevivem ao container

O sistema de arquivos de um container some junto com ele. Um **volume** guarda os dados fora do ciclo de vida do container.

```bash
docker volume create dados-cn

docker run --rm -v dados-cn:/dados alpine sh -c 'echo "gravado pelo container 1" > /dados/nota.txt'
docker run --rm -v dados-cn:/dados alpine cat /dados/nota.txt
```

O segundo comando deve imprimir `gravado pelo container 1`, mesmo o primeiro container já não existindo mais.

---

## Passo 6 — Limpar o ambiente local do Docker

```bash
docker rm -f api-1 api-2
docker volume rm dados-cn
docker ps -a
```

`docker ps -a` não deve listar nenhum container `api-*`. As imagens `cn2026-api:v1` e `cn2026-web:v1` continuam disponíveis para a próxima parte.

---

# Parte 2 — Amazon ECR e CloudFormation: criar o registry por código e publicar as imagens

Conceitos desta parte: **repositories, tags, push/pull, lifecycle policy, scan** e o primeiro contato com **CloudFormation** (template, parâmetro, output).

---

## Passo 7 — Escrever o template CloudFormation dos repositórios

O template cria dois repositórios (`cn2026-web` e `cn2026-api`) com **scan ao enviar** e uma **lifecycle policy** que mantém só as 5 imagens mais recentes. `EmptyOnDelete` permite apagar o repositório mesmo com imagens dentro (útil na limpeza).

```bash
cat <<'EOF' > ~/cn2026/ecr.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: CN 2026 - Repositorios ECR para os microsservicos web e api

Parameters:
  Projeto:
    Type: String
    Default: cn2026
    Description: Prefixo usado nos nomes dos repositorios

Resources:
  RepoWeb:
    Type: AWS::ECR::Repository
    Properties:
      RepositoryName: !Sub '${Projeto}-web'
      EmptyOnDelete: true
      ImageScanningConfiguration:
        ScanOnPush: true
      LifecyclePolicy:
        LifecyclePolicyText: |
          {
            "rules": [
              {
                "rulePriority": 1,
                "description": "Manter apenas as 5 imagens mais recentes",
                "selection": {
                  "tagStatus": "any",
                  "countType": "imageCountMoreThan",
                  "countNumber": 5
                },
                "action": { "type": "expire" }
              }
            ]
          }

  RepoApi:
    Type: AWS::ECR::Repository
    Properties:
      RepositoryName: !Sub '${Projeto}-api'
      EmptyOnDelete: true
      ImageScanningConfiguration:
        ScanOnPush: true
      LifecyclePolicy:
        LifecyclePolicyText: |
          {
            "rules": [
              {
                "rulePriority": 1,
                "description": "Manter apenas as 5 imagens mais recentes",
                "selection": {
                  "tagStatus": "any",
                  "countType": "imageCountMoreThan",
                  "countNumber": 5
                },
                "action": { "type": "expire" }
              }
            ]
          }

Outputs:
  UriRepoWeb:
    Description: URI do repositorio da web
    Value: !GetAtt RepoWeb.RepositoryUri
  UriRepoApi:
    Description: URI do repositorio da api
    Value: !GetAtt RepoApi.RepositoryUri
EOF
```

---

## Passo 8 — Criar a stack de ECR

```bash
aws cloudformation deploy \
  --region us-east-1 \
  --stack-name cn2026-ecr \
  --template-file ~/cn2026/ecr.yaml \
  --no-fail-on-empty-changeset

aws cloudformation describe-stacks \
  --stack-name cn2026-ecr \
  --query "Stacks[0].Outputs[].[OutputKey,OutputValue]" --output table
```

A saída deve terminar com `Successfully created/updated stack - cn2026-ecr`, e a tabela deve mostrar as duas URIs (`SEU_ID.dkr.ecr.us-east-1.amazonaws.com/cn2026-web` e `.../cn2026-api`).

> Pelo console: CloudFormation → Stacks → `cn2026-ecr` → abas **Resources**, **Outputs** e **Events**.

---

## Passo 9 — Autenticar, marcar (tag) e enviar (push) as imagens

A imagem local precisa ter, no nome, o endereço do repositório. É a **tag** que faz essa ligação.

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin $REGISTRY

docker tag cn2026-api:v1 $REGISTRY/cn2026-api:v1
docker tag cn2026-web:v1 $REGISTRY/cn2026-web:v1

docker push $REGISTRY/cn2026-api:v1
docker push $REGISTRY/cn2026-web:v1
```

Deve aparecer `Login Succeeded` e, ao final de cada push, a linha `v1: digest: sha256:...`.

---

## Passo 10 — Conferir o que está no ECR

```bash
aws ecr describe-images --repository-name cn2026-api \
  --query "imageDetails[].{Tags:imageTags,TamanhoMB:imageSizeInBytes,Enviada:imagePushedAt}" --output table

aws ecr get-lifecycle-policy --repository-name cn2026-api \
  --query "lifecyclePolicyText" --output text
```

A tabela deve mostrar a tag `v1`; o segundo comando deve mostrar a regra das 5 imagens mais recentes.

> **Usamos tags de versão (`v1`, `v2`), não `latest`.** Com `latest` o ECS não sabe que a imagem mudou e a task não é atualizada sozinha (era o problema do Passo 16 da atividade anterior). Com uma tag nova, mudar o parâmetro da stack já dispara um deployment.

---

# Parte 3 — CloudFormation: ECS + Fargate + ALB em uma única stack

A segunda stack cria toda a infraestrutura de execução: cluster, task definitions, services, load balancer, target groups, listener, regra de roteamento, security groups e logs.

---

## Passo 11 — Descobrir a VPC e as subnets padrão

O ALB precisa de **pelo menos duas subnets em zonas de disponibilidade diferentes**. A VPC padrão já tem isso.

```bash
export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" --output text)

export SUBNETS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=default-for-az,Values=true" \
  --query "Subnets[].SubnetId" --output text | tr '\t' ',')

echo "VPC: $VPC_ID"
echo "Subnets: $SUBNETS"
```

Devem aparecer o ID da VPC (`vpc-...`) e uma lista de subnets separadas por vírgula (`subnet-a,subnet-b,...`).

---

## Passo 12 — Escrever o template ECS + Fargate + ALB

Leia o template com calma: ele é o "mapa" da apresentação. Pontos de atenção:

- `Parameters`: o mesmo template serve para qualquer número de tasks, versão e tamanho.
- `SgTasks` só aceita tráfego **vindo do `SgAlb`** (as tasks não ficam expostas à internet).
- `TargetType: ip`: obrigatório para Fargate (modo de rede `awsvpc`, cada task tem seu IP).
- `ListenerRule` com `/api/*` manda para a `api`; todo o resto vai para a `web`.
- `ExecutionRoleArn`/`TaskRoleArn` usam a `LabRole` (no Learner Lab não dá para criar roles novas).
- `DeploymentCircuitBreaker`: se o deployment novo não estabilizar, o ECS volta para a versão anterior.
- `TaskSize` + `Mappings`: no Fargate, CPU e memória só aceitam combinações fixas (não é qualquer CPU com qualquer memória). Em vez de dois parâmetros soltos que permitiriam escolher uma combinação inválida, um único parâmetro (`small`/`medium`/`large`) é traduzido para o par certo com `!FindInMap`.

```bash
cat <<'EOF' > ~/cn2026/app.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: CN 2026 - ECS + Fargate + ALB (microsservicos web e api)

Parameters:
  Projeto:
    Type: String
    Default: cn2026
    Description: Prefixo usado nos nomes dos recursos (deve ser o mesmo da stack de ECR)
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: VPC onde o ALB e as tasks serao criados
  Subnets:
    Type: List<AWS::EC2::Subnet::Id>
    Description: Subnets (minimo 2, em AZs diferentes)
  TagWeb:
    Type: String
    Default: v1
    Description: Tag da imagem da web no ECR
  TagApi:
    Type: String
    Default: v1
    Description: Tag da imagem da api no ECR
  QuantidadeTasks:
    Type: Number
    Default: 2
    MinValue: 0
    MaxValue: 6
    Description: Desired count de cada service
  TaskSize:
    Type: String
    Default: small
    AllowedValues: ['small', 'medium', 'large']
    Description: Tamanho da task Fargate (small = 0.25 vCPU/512MB, medium = 0.5 vCPU/1GB, large = 1 vCPU/2GB)

Mappings:
  TaskSizes:
    small:
      Cpu: '256'
      Memory: '512'
    medium:
      Cpu: '512'
      Memory: '1024'
    large:
      Cpu: '1024'
      Memory: '2048'

Resources:
  # ---------- Rede / seguranca ----------
  SgAlb:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ALB - libera HTTP da internet
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  SgTasks:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Tasks - aceitam trafego somente do ALB
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 8080
          ToPort: 8080
          SourceSecurityGroupId: !Ref SgAlb
        - IpProtocol: tcp
          FromPort: 3000
          ToPort: 3000
          SourceSecurityGroupId: !Ref SgAlb

  # ---------- ECS ----------
  Cluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: !Sub '${Projeto}-cluster'

  LogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub '/ecs/${Projeto}'
      RetentionInDays: 3

  TaskWeb:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: !Sub '${Projeto}-web'
      Cpu: !FindInMap [TaskSizes, !Ref TaskSize, Cpu]
      Memory: !FindInMap [TaskSizes, !Ref TaskSize, Memory]
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - FARGATE
      ExecutionRoleArn: !Sub 'arn:aws:iam::${AWS::AccountId}:role/LabRole'
      TaskRoleArn: !Sub 'arn:aws:iam::${AWS::AccountId}:role/LabRole'
      ContainerDefinitions:
        - Name: web
          Image: !Sub '${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/${Projeto}-web:${TagWeb}'
          Essential: true
          PortMappings:
            - ContainerPort: 8080
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref LogGroup
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: web

  TaskApi:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: !Sub '${Projeto}-api'
      Cpu: !FindInMap [TaskSizes, !Ref TaskSize, Cpu]
      Memory: !FindInMap [TaskSizes, !Ref TaskSize, Memory]
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - FARGATE
      ExecutionRoleArn: !Sub 'arn:aws:iam::${AWS::AccountId}:role/LabRole'
      TaskRoleArn: !Sub 'arn:aws:iam::${AWS::AccountId}:role/LabRole'
      ContainerDefinitions:
        - Name: api
          Image: !Sub '${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/${Projeto}-api:${TagApi}'
          Essential: true
          PortMappings:
            - ContainerPort: 3000
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref LogGroup
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: api

  # ---------- Load Balancer ----------
  Alb:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub '${Projeto}-alb'
      Type: application
      Scheme: internet-facing
      Subnets: !Ref Subnets
      SecurityGroups:
        - !Ref SgAlb

  TgWeb:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: !Sub '${Projeto}-tg-web'
      VpcId: !Ref VpcId
      TargetType: ip
      Protocol: HTTP
      Port: 8080
      HealthCheckPath: /health
      HealthCheckIntervalSeconds: 10
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 3
      TargetGroupAttributes:
        - Key: deregistration_delay.timeout_seconds
          Value: '10'

  TgApi:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: !Sub '${Projeto}-tg-api'
      VpcId: !Ref VpcId
      TargetType: ip
      Protocol: HTTP
      Port: 3000
      HealthCheckPath: /api/health
      HealthCheckIntervalSeconds: 10
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 3
      TargetGroupAttributes:
        - Key: deregistration_delay.timeout_seconds
          Value: '10'

  Listener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref Alb
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TgWeb

  RegraApi:
    Type: AWS::ElasticLoadBalancingV2::ListenerRule
    Properties:
      ListenerArn: !Ref Listener
      Priority: 10
      Conditions:
        - Field: path-pattern
          Values:
            - /api/*
      Actions:
        - Type: forward
          TargetGroupArn: !Ref TgApi

  # ---------- Services (Fargate) ----------
  ServicoWeb:
    Type: AWS::ECS::Service
    DependsOn: Listener
    Properties:
      ServiceName: web
      Cluster: !Ref Cluster
      LaunchType: FARGATE
      DesiredCount: !Ref QuantidadeTasks
      TaskDefinition: !Ref TaskWeb
      HealthCheckGracePeriodSeconds: 30
      DeploymentConfiguration:
        MinimumHealthyPercent: 100
        MaximumPercent: 200
        DeploymentCircuitBreaker:
          Enable: true
          Rollback: true
      NetworkConfiguration:
        AwsvpcConfiguration:
          AssignPublicIp: ENABLED
          Subnets: !Ref Subnets
          SecurityGroups:
            - !Ref SgTasks
      LoadBalancers:
        - ContainerName: web
          ContainerPort: 8080
          TargetGroupArn: !Ref TgWeb

  ServicoApi:
    Type: AWS::ECS::Service
    DependsOn: RegraApi
    Properties:
      ServiceName: api
      Cluster: !Ref Cluster
      LaunchType: FARGATE
      DesiredCount: !Ref QuantidadeTasks
      TaskDefinition: !Ref TaskApi
      HealthCheckGracePeriodSeconds: 30
      DeploymentConfiguration:
        MinimumHealthyPercent: 100
        MaximumPercent: 200
        DeploymentCircuitBreaker:
          Enable: true
          Rollback: true
      NetworkConfiguration:
        AwsvpcConfiguration:
          AssignPublicIp: ENABLED
          Subnets: !Ref Subnets
          SecurityGroups:
            - !Ref SgTasks
      LoadBalancers:
        - ContainerName: api
          ContainerPort: 3000
          TargetGroupArn: !Ref TgApi

Outputs:
  UrlAplicacao:
    Description: Endereco publico da aplicacao (via ALB)
    Value: !Sub 'http://${Alb.DNSName}'
  NomeCluster:
    Description: Nome do cluster ECS
    Value: !Ref Cluster
  NomeAlb:
    Description: Nome do Application Load Balancer
    Value: !Sub '${Projeto}-alb'
EOF
```

> Por que `AssignPublicIp: ENABLED`? Na VPC padrão as subnets são públicas e não há NAT Gateway. A task precisa de um IP público **apenas para sair à internet/ECR** e puxar a imagem; a entrada continua bloqueada pelo `SgTasks`.

---

## Passo 13 — Validar o template

```bash
aws cloudformation validate-template \
  --template-body file://$HOME/cn2026/app.yaml \
  --query "Parameters[].ParameterKey" --output text
```

Deve listar os parâmetros: `Projeto VpcId Subnets TagWeb TagApi QuantidadeTasks TaskSize`. Se houver erro de sintaxe, ele aparece aqui, antes de criar qualquer recurso.

---

## Passo 14 — Criar a stack (deploy)

Para repetir a mesma chamada várias vezes (escalar, atualizar versão, mudar tamanho), guardamos o comando em uma função. Os valores variáveis vêm de variáveis de ambiente.

```bash
export TAG_WEB=v1 TAG_API=v1 QTD=2 TASK_SIZE=small
export CLUSTER=cn2026-cluster

deploy_app() {
  aws cloudformation deploy \
    --region us-east-1 \
    --stack-name cn2026-app \
    --template-file ~/cn2026/app.yaml \
    --no-fail-on-empty-changeset \
    --parameter-overrides \
      VpcId=$VPC_ID Subnets=$SUBNETS \
      TagWeb=$TAG_WEB TagApi=$TAG_API \
      QuantidadeTasks=$QTD TaskSize=$TASK_SIZE
}

deploy_app
```

Leva de 3 a 6 minutos (o CloudFormation espera o ECS estabilizar). Enquanto isso, em **outra aba do CloudShell**, acompanhe os eventos:

```bash
aws cloudformation describe-stack-events --stack-name cn2026-app \
  --query "StackEvents[:8].[ResourceStatus,LogicalResourceId]" --output table
```

Ao final deve aparecer `Successfully created/updated stack - cn2026-app`.

---

## Passo 15 — Ler os outputs e testar

```bash
export APP_URL=$(aws cloudformation describe-stacks --stack-name cn2026-app \
  --query "Stacks[0].Outputs[?OutputKey=='UrlAplicacao'].OutputValue" --output text)
echo $APP_URL > ~/cn2026/url.txt

curl -s $APP_URL/health; echo
curl -s $APP_URL/api/info; echo
```

O primeiro `curl` deve responder `ok` (serviço `web`); o segundo deve devolver um JSON com `"servico":"api"`, `"hostname"` e `"versao":"v1"`. Isso já mostra o **roteamento por caminho** do ALB: a mesma URL, dois serviços diferentes.

---

## Passo 16 — Testar pelo navegador

Abra o valor de `$APP_URL` no navegador (`echo $APP_URL`).

A página mostra a task da `web`, a última resposta da `api` e uma tabela contando as respostas de **cada task da api**. Com 2 tasks, aparecem duas linhas com contagens crescendo em paralelo, essa é a distribuição do ALB.

---

# Parte 4 — Explorar o que o CloudFormation criou

Agora "abrimos o capô": cada comando mostra um conceito da apresentação funcionando na AWS.

---

## Passo 17 — ECS: cluster, services e desired count

```bash
aws ecs describe-services --cluster $CLUSTER --services web api \
  --query "services[].{Servico:serviceName,Desejado:desiredCount,Rodando:runningCount,Pendente:pendingCount,Launch:launchType}" \
  --output table
```

Devem aparecer os services `web` e `api`, `Desejado = 2`, `Rodando = 2` e `Launch = FARGATE`.

> Console: ECS → Clusters → `cn2026-cluster` → aba **Services** → clique em `api` → abas **Tasks**, **Deployments**, **Networking**, **Logs**.

---

## Passo 18 — Task definition: a "receita" da task

```bash
aws ecs describe-task-definition --task-definition cn2026-api \
  --query "taskDefinition.{Familia:family,Revisao:revision,Rede:networkMode,Cpu:cpu,Memoria:memory,Imagem:containerDefinitions[0].image,Porta:containerDefinitions[0].portMappings[0].containerPort}" \
  --output table
```

Deve mostrar `Rede = awsvpc`, `Cpu = 256`, `Memoria = 512`, a imagem do ECR com a tag `v1` e a porta `3000`. Guarde o número da **Revisão**: ele vai mudar nos próximos passos.

---

## Passo 19 — Fargate: as tasks em execução

```bash
TASKS=$(aws ecs list-tasks --cluster $CLUSTER --service-name api --query "taskArns" --output text)

aws ecs describe-tasks --cluster $CLUSTER --tasks $TASKS \
  --query "tasks[].{Task:taskArn,Cpu:cpu,Memoria:memory,Launch:launchType,AZ:availabilityZone,Plataforma:platformVersion,Estado:lastStatus}" \
  --output table
```

Cada task tem sua própria CPU/memória (o tamanho da **task**, não de uma máquina), `Launch = FARGATE` e costuma estar em uma **zona de disponibilidade** diferente. Não existe nenhuma instância EC2 sua para gerenciar: confira em EC2 → Instances, sem instâncias novas.

---

## Passo 20 — ALB: listener, regras e target groups

```bash
export ALB_ARN=$(aws elbv2 describe-load-balancers --names cn2026-alb \
  --query "LoadBalancers[0].LoadBalancerArn" --output text)

export LISTENER_ARN=$(aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN \
  --query "Listeners[0].ListenerArn" --output text)

aws elbv2 describe-rules --listener-arn $LISTENER_ARN \
  --query "Rules[].{Prioridade:Priority,Condicao:Conditions[0].Values[0],Destino:Actions[0].TargetGroupArn}" \
  --output table
```

Devem aparecer duas regras: prioridade `10` com condição `/api/*` (destino: target group da `api`) e a regra `default` (destino: target group da `web`).

Agora a **saúde dos alvos** (health checks): cada IP é uma task.

```bash
export TG_API=$(aws elbv2 describe-target-groups --names cn2026-tg-api \
  --query "TargetGroups[0].TargetGroupArn" --output text)

aws elbv2 describe-target-health --target-group-arn $TG_API \
  --query "TargetHealthDescriptions[].{IP:Target.Id,Porta:Target.Port,Estado:TargetHealth.State}" \
  --output table
```

Devem aparecer 2 alvos com `Estado = healthy`.

---

## Passo 21 — Rede: quem pode falar com quem

```bash
aws ec2 describe-security-groups \
  --filters "Name=tag:aws:cloudformation:stack-name,Values=cn2026-app" \
  --query "SecurityGroups[].{Descricao:Description,Regras:IpPermissions[].[FromPort,UserIdGroupPairs[0].GroupId,IpRanges[0].CidrIp]}" \
  --output json
```

O security group do **ALB** aceita a porta 80 de `0.0.0.0/0`. O das **tasks** aceita as portas `8080` e `3000` **somente a partir do security group do ALB** (nenhum CIDR). Ou seja, a única entrada pública é o load balancer.

---

## Passo 22 — Logs das tasks (CloudWatch Logs)

```bash
aws logs tail /ecs/cn2026 --since 10m --format short
```

Devem aparecer as linhas `API ativa na porta 3000` e `Web ativo na porta 8080`, uma para cada task iniciada.

---

# Parte 5 — Operar: distribuir, escalar, atualizar e se recuperar

---

## Passo 23 — Ver o balanceamento de carga

```bash
for i in $(seq 1 12); do curl -s $APP_URL/api/info | grep -o '"hostname":"[^"]*"'; done | sort | uniq -c
```

Devem aparecer 2 hostnames diferentes, cada um com cerca de metade das respostas.

---

## Passo 24 — Escalar: mudar o desired count

Só mudamos um parâmetro e reaplicamos a stack. O ECS cria as tasks que faltam e o ALB as inclui no target group quando ficam `healthy`.

```bash
export QTD=4
deploy_app

aws ecs wait services-stable --cluster $CLUSTER --services web api
aws elbv2 describe-target-health --target-group-arn $TG_API \
  --query "TargetHealthDescriptions[].TargetHealth.State" --output text
```

Deve aparecer `healthy healthy healthy healthy`. Repita o Passo 23: agora são **4 hostnames**. Na página do navegador, a tabela ganha novas linhas sozinha.

---

## Passo 25 — Nova versão: rolling deployment com tag nova

Vamos publicar a versão `v2` da `api` (só muda o texto da versão) e mandar o ECS trocar as tasks **sem derrubar a aplicação**.

**Aba 2 do CloudShell** (deixe rodando para ver a troca acontecer):

```bash
APP_URL=$(cat ~/cn2026/url.txt)
while true; do curl -s $APP_URL/api/info | grep -o '"versao":"[^"]*"'; sleep 1; done
```

**Aba 1**, gere e envie a nova imagem e reaplique a stack:

```bash
cd ~/cn2026/api
sed -i "s/'v1'/'v2'/" server.js
docker build -t cn2026-api:v2 .
docker tag cn2026-api:v2 $REGISTRY/cn2026-api:v2
docker push $REGISTRY/cn2026-api:v2

cd ~/cn2026
export TAG_API=v2
deploy_app
```

Na aba 2, as respostas começam só com `"versao":"v1"`, passam por um período **misturado** (`v1` e `v2`) e terminam só com `"versao":"v2"`, sem nenhuma resposta de erro no meio. Isso é o `MinimumHealthyPercent: 100` e `MaximumPercent: 200`: as tasks novas sobem **antes** de as antigas saírem.

Confira que foi criada uma **nova revisão** da task definition e como está o deployment:

```bash
aws ecs describe-task-definition --task-definition cn2026-api --query "taskDefinition.revision"

aws ecs describe-services --cluster $CLUSTER --services api \
  --query "services[0].deployments[].{Estado:rolloutState,Status:status,Desejado:desiredCount,Rodando:runningCount}" \
  --output table
```

Pare o loop da aba 2 com `Ctrl+C`. A revisão deve ter aumentado em 1 e o deployment deve estar `COMPLETED`.

> Também confira o ECR (`aws ecr describe-images --repository-name cn2026-api`): agora há duas imagens, `v1` e `v2`. A lifecycle policy só começa a apagar a partir da sexta.

---

## Passo 26 — Redimensionar as tasks (task sizing do Fargate)

Cpu e memória são propriedades da **task definition**; mudar gera nova revisão e novo deployment.

```bash
export TASK_SIZE=medium
deploy_app

TASKS=$(aws ecs list-tasks --cluster $CLUSTER --service-name api --query "taskArns" --output text)
aws ecs describe-tasks --cluster $CLUSTER --tasks $TASKS \
  --query "tasks[].{Cpu:cpu,Memoria:memory}" --output table
```

Todas as tasks devem mostrar `Cpu = 512` e `Memoria = 1024` (o par que o `Mappings` associa a `medium`). No Fargate você paga pelo que a task **reserva** (vCPU e GB por segundo), então dimensione com cuidado. Volte ao tamanho mínimo:

```bash
export TASK_SIZE=small
deploy_app
```

---

## Passo 27 — Autocorreção: derrubar uma task de propósito

```bash
TASK_ARN=$(aws ecs list-tasks --cluster $CLUSTER --service-name api --query "taskArns[0]" --output text)
aws ecs stop-task --cluster $CLUSTER --task $TASK_ARN --reason "teste de autocorrecao" > /dev/null

for i in 1 2 3 4 5 6; do
  aws ecs describe-services --cluster $CLUSTER --services api \
    --query "services[0].[desiredCount,runningCount,pendingCount]" --output text
  sleep 10
done
```

O `runningCount` cai (ex.: `4 3 1`) e volta para `4 4 0` em cerca de um minuto, sem você fazer nada: o **service** compara o desired count com o que está rodando e cria a task que faltou. Durante todo esse tempo o `curl $APP_URL/api/info` continua respondendo, atendido pelas tasks restantes.

---

# Parte 6 — Amazon EKS (opcional / avançado)

O EKS resolve o mesmo problema (orquestrar containers) com **Kubernetes**. No Learner Lab a criação pode ser limitada ou bloqueada por permissões. Se não funcionar, apresente só a tabela de equivalência e o manifesto.

| ECS | Kubernetes (EKS) |
|---|---|
| Cluster | Cluster (control plane gerenciado pela AWS + nodes) |
| Task definition | Pod template (dentro do Deployment) |
| Task | Pod |
| Service (desired count) | Deployment (`replicas`) |
| Target group + ALB | Service (`type: LoadBalancer`) / Ingress |
| Fargate | Fargate profile ou managed node group (EC2) |

> **Custo:** o control plane do EKS é cobrado por hora, por cluster, mesmo sem uso. Crie só para a demonstração e apague logo depois.

---

## Passo 28 — Criar o cluster pelo console

1. EKS → **Create cluster** → configuração **Custom**.
2. Name: `cn2026-eks`. Em **Cluster IAM role**, selecione `LabRole`.
3. Networking: VPC padrão e as subnets padrão. Endpoint access: Public.
4. Crie e aguarde o status **Active** (10 a 15 minutos).
5. Aba **Compute** → **Add node group** → Node IAM role: `LabRole`; instância `t3.medium`; desired size `2`.

---

## Passo 29 — Publicar a `api` no Kubernetes

```bash
aws eks update-kubeconfig --region us-east-1 --name cn2026-eks
kubectl get nodes

cat <<'EOF' > ~/cn2026/api-k8s.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: ACCOUNT_ID_AQUI.dkr.ecr.us-east-1.amazonaws.com/cn2026-api:v2
          ports:
            - containerPort: 3000
          readinessProbe:
            httpGet:
              path: /api/health
              port: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 3000
EOF

sed -i "s/ACCOUNT_ID_AQUI/$ACCOUNT_ID/" ~/cn2026/api-k8s.yaml
kubectl apply -f ~/cn2026/api-k8s.yaml
kubectl get pods,svc
```

`kubectl get nodes` deve listar 2 nodes `Ready`. Depois de alguns instantes, `kubectl get pods` mostra 2 pods `Running` e `kubectl get svc api` mostra um endereço em `EXTERNAL-IP`. Teste com `curl http://<EXTERNAL-IP>/api/info`.

Compare com o ECS: o `replicas: 2` é o desired count, o `readinessProbe` é o health check e o Service `LoadBalancer` faz o papel de ALB + target group.

> Se o `kubectl` não estiver disponível no CloudShell, instale-o seguindo a documentação oficial do Amazon EKS.

---

# Parte 7 — Encerrar e limpar tudo

Para não gastar crédito do laboratório, remova os recursos **na ordem inversa da criação**.

---

## Passo 30 — Apagar o que foi criado no EKS (se você fez a Parte 6)

```bash
kubectl delete -f ~/cn2026/api-k8s.yaml
```

Depois, no console: EKS → `cn2026-eks` → **Compute** → apague o node group; quando ele sumir, apague o cluster. O `kubectl delete` vem primeiro para remover o load balancer criado pelo Service.

---

## Passo 31 — Apagar a stack da aplicação

```bash
aws cloudformation delete-stack --stack-name cn2026-app
aws cloudformation wait stack-delete-complete --stack-name cn2026-app
```

Uma única chamada remove services, tasks, target groups, listener, ALB, cluster, security groups e log group.

---

## Passo 32 — Apagar a stack do ECR

Como usamos `EmptyOnDelete`, os repositórios são apagados mesmo com imagens dentro.

```bash
aws cloudformation delete-stack --stack-name cn2026-ecr
aws cloudformation wait stack-delete-complete --stack-name cn2026-ecr
```

---

## Passo 33 — Verificar a limpeza e limpar o Docker local

```bash
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query "StackSummaries[?starts_with(StackName,'cn2026')].StackName" --output text

aws ecr describe-repositories --query "repositories[?starts_with(repositoryName,'cn2026')].repositoryName" --output text

docker system prune -af
```

Os dois primeiros comandos não devem retornar nada.

---

# Problemas comuns

| Sintoma | Causa provável | O que fazer |
|---|---|---|
| Stack fica muito tempo em `CREATE_IN_PROGRESS` no `ServicoApi`/`ServicoWeb` e depois faz rollback | A task não consegue puxar a imagem (tag inexistente ou não enviada) | Confirme o Passo 10 (`describe-images`) e o valor de `TAG_WEB`/`TAG_API`. Veja os motivos em ECS → Service → aba **Events** |
| `exec format error` nos logs da task | Imagem gerada em arquitetura diferente (ex.: Mac com chip ARM) | Gere a imagem no CloudShell (x86_64), como nesta atividade |
| ALB responde `503` logo após o deploy | Targets ainda `initial` (health check) | Aguarde 30 a 60 segundos e confira o Passo 20 |
| ALB responde `502`/`504` | Security group das tasks não libera a porta a partir do ALB, ou a app não escuta na porta esperada | Confira o Passo 21 e as portas `8080`/`3000` |
| `AccessDenied` ao criar recurso com IAM | Tentativa de criar role nova no Learner Lab | Use a `LabRole`, como no template |
| `cannot pull ... AccessDeniedException` no ECR | `ExecutionRoleArn` sem permissão de ECR | Confirme que o template usa `LabRole` |
| `Subnets` com erro no deploy | Subnets em uma única AZ ou lista com espaços | Refaça o Passo 11 (uma vírgula entre os IDs, sem espaços) |
| Variáveis vazias (`$REGISTRY`, `$CLUSTER`...) | O CloudShell reiniciou | Rode o Passo 0, o Passo 11 e o começo do Passo 14 (variáveis e função `deploy_app`) |
