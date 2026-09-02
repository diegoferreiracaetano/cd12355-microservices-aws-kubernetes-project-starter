# Guia de Deploy — Coworking Space Service (referência pessoal)

Guia passo a passo com todos os comandos usados para levar a aplicação de analytics do zero até rodando em produção no AWS EKS. Use como consulta sempre que precisar refazer o processo (ex: workspace resetado, novo cluster, etc).

> Todos os comandos AWS/kubectl/docker abaixo devem ser executados no **workspace Udacity** (ou qualquer ambiente com AWS CLI, Docker, kubectl e eksctl instalados). Os arquivos do projeto (Dockerfile, buildspec.yaml, YAMLs) já estão neste repositório.

---

## 0. Toda vez que o workspace reseta

```bash
# Identidade do git
git config --global user.email "diegofcaetano@gmail.com"
git config --global user.name "Diego Caetano"

# Gerar nova chave SSH e adicionar em https://github.com/settings/ssh/new
ssh-keygen -t ed25519 -C "diegofcaetano@gmail.com"
cat ~/.ssh/id_ed25519.pub

# Configurar credenciais AWS (cole as do Cloud Lab)
aws configure

# Clonar o repositório
git clone git@github.com:diegoferreiracaetano/cd12355-microservices-aws-kubernetes-project-starter.git
cd cd12355-microservices-aws-kubernetes-project-starter
```

---

## 1. Criar o cluster EKS

Antes de criar qualquer recurso, confirme que as credenciais AWS estão corretas e com permissão:

```bash
aws sts get-caller-identity
```

Se der erro de credencial, rode `aws configure` e cole as credenciais do Cloud Lab.

```bash
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --nodegroup-name my-nodes \
  --node-type t3.small \
  --nodes 1 --nodes-min 1 --nodes-max 2
```

Isso demora ~15-20 minutos. Depois, aponte o kubectl para o cluster:

```bash
aws eks --region us-east-1 update-kubeconfig --name my-cluster
kubectl config current-context
kubectl get namespace
```

**Ao terminar o projeto, sempre destrua o cluster para não estourar o orçamento:**

```bash
eksctl delete cluster --name my-cluster --region us-east-1
```

---

## 2. Subir o Postgres no cluster

Os manifests já estão em `deployment/postgres/`.

**Antes de aplicar**, confira se `storageClassName: gp2` usado em `pvc.yaml`/`pv.yaml` bate com a storage class default do seu cluster:

```bash
kubectl get storageclass
```

Se o nome for diferente (ex: `gp3`), edite os dois arquivos antes de continuar.

```bash
kubectl apply -f deployment/postgres/pvc.yaml
kubectl apply -f deployment/postgres/pv.yaml
kubectl apply -f deployment/postgres/postgresql-deployment.yaml
kubectl apply -f deployment/postgres/postgresql-service.yaml

kubectl get pods
kubectl get svc
```

### Testar conexão via pod

```bash
kubectl exec -it <NOME_DO_POD_POSTGRES> -- bash
psql -U myuser -d mydatabase
\l
\q
exit
```

### Testar conexão via port-forward (do seu terminal)

```bash
kubectl port-forward service/postgresql-service 5433:5432 &
```

### Popular o banco (rodar uma vez para cada .sql)

```bash
apt update && apt install -y postgresql-client

export DB_PASSWORD=mypassword
PGPASSWORD="$DB_PASSWORD" psql --host 127.0.0.1 -U myuser -d mydatabase -p 5433 < db/1_create_tables.sql
PGPASSWORD="$DB_PASSWORD" psql --host 127.0.0.1 -U myuser -d mydatabase -p 5433 < db/2_seed_users.sql
PGPASSWORD="$DB_PASSWORD" psql --host 127.0.0.1 -U myuser -d mydatabase -p 5433 < db/3_seed_tokens.sql
```

### Conferir as tabelas

```bash
PGPASSWORD="$DB_PASSWORD" psql --host 127.0.0.1 -U myuser -d mydatabase -p 5433
select * from users;
select * from tokens;
\q
```

### Fechar as portas abertas (quando terminar)

```bash
ps aux | grep 'kubectl port-forward' | grep -v grep | awk '{print $2}' | xargs -r kill
```

---

## 3. Rodar a aplicação localmente (sem Docker) — validação

```bash
cd analytics
apt update -y
apt install -y build-essential libpq-dev
pip install --upgrade pip setuptools wheel
pip install -r requirements.txt

export DB_USERNAME=myuser
export DB_PASSWORD=mypassword
export DB_HOST=127.0.0.1
export DB_PORT=5433
export DB_NAME=mydatabase
python app.py
```

Em outro terminal (a app sobe na porta 5153):

```bash
curl 127.0.0.1:5153/api/reports/daily_usage
curl 127.0.0.1:5153/api/reports/user_visits
```

---

## 4. Dockerizar a aplicação

```bash
cd analytics
docker build -t test-coworking-analytics .

# Testar localmente usando a rede do host (feche a app local antes)
docker run --network="host" test-coworking-analytics

curl 127.0.0.1:5153/api/reports/daily_usage
```

---

## 5. Criar o repositório no Amazon ECR

```bash
aws ecr create-repository \
  --repository-name coworking \
  --region us-east-1
```

Guarde a `repositoryUri` retornada — é o `<AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/coworking`.

Pra pegar seu Account ID a qualquer momento:

```bash
aws sts get-caller-identity --query Account --output text
```

📸 **Screenshot**: console do ECR mostrando o repositório `coworking` com uma imagem versionada (ex: `1.0.1`).

---

## 6. Criar o projeto no AWS CodeBuild

No console AWS (CodeBuild > Create build project):

1. **Source**: conectar ao GitHub, apontar para este repositório e branch `main`.
2. **Environment**: imagem gerenciada Ubuntu, runtime standard, habilitar **"Privileged"** (obrigatório para rodar `docker build` dentro do CodeBuild).
3. **Environment variables** (em texto simples, exceto se preferir Secrets Manager):
   - `AWS_ACCOUNT_ID` = seu account id
   - `AWS_DEFAULT_REGION` = `us-east-1`
   - `IMAGE_REPO_NAME` = `coworking`
4. **Buildspec**: usar o arquivo `buildspec.yaml` já existente na raiz do repo.
5. **Webhook**: habilitar rebuild automático a cada push (Primary source webhook events → PUSH).

### Dar permissão de ECR para a Role do CodeBuild

O CodeBuild cria uma IAM Role automaticamente (`codebuild-<projeto>-service-role`). Anexe a policy gerenciada `AmazonEC2ContainerRegistryPowerUser` a essa role (IAM > Roles > buscar a role > Add permissions).

### Rodar o build

No console do CodeBuild, clique em **Start Build**. Depois confira o ECR — deve aparecer uma nova imagem.

📸 **Screenshots**: pipeline do CodeBuild mostrando build disparado automaticamente pelo push + resultado no ECR.

---

## 7. Deploy da aplicação no Kubernetes

Ordem de apply importa: Secret e ConfigMap antes do Deployment que os referencia.

```bash
kubectl apply -f deployment/secret.yaml
kubectl apply -f deployment/configmap.yaml
```

Antes de aplicar o deployment da app, edite `deployment/coworking.yaml` e troque a linha `image:` pela URI real da imagem no ECR (com a tag gerada pelo CodeBuild, ex: `1.0.3`):

```yaml
image: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/coworking:1.0.3
```

```bash
kubectl apply -f deployment/coworking.yaml
```

### Verificar

```bash
kubectl get svc
kubectl get pods
kubectl describe svc postgresql-service
kubectl describe deployment coworking
```

Pegue o `EXTERNAL-IP` do serviço `coworking` (do `kubectl get svc`) e teste:

```bash
curl <EXTERNAL-IP>:5153/api/reports/daily_usage
curl <EXTERNAL-IP>:5153/api/reports/user_visits
```

📸 **Screenshots exigidos pela rubrica**: `kubectl get svc`, `kubectl get pods`, `kubectl describe svc postgresql-service`, `kubectl describe deployment coworking`.

---

## 8. Habilitar CloudWatch Container Insights

Forma mais simples (addon gerenciado do EKS):

```bash
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name amazon-cloudwatch-observability \
  --region us-east-1
```

Alternativa manual (Fluent Bit quickstart), caso o addon não esteja disponível na conta:

```bash
ClusterName=my-cluster
RegionName=us-east-1
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
[[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off' || FluentBitReadFromTail='On'
[[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'

curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml | \
sed 's/{{cluster_name}}/'${ClusterName}'/;s/{{region_name}}/'${RegionName}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' | \
kubectl apply -f -
```

Depois, no console AWS: **CloudWatch > Container Insights > Performance monitoring**, selecione o cluster `my-cluster` e veja os logs/métricas dos pods.

📸 **Screenshot**: logs do Container Insights mostrando a aplicação `coworking` rodando sem erros (o job de background loga o status a cada 30s).

---

## 9. Diagnóstico rápido de problemas

```bash
# Pod não fica pronto (READY 0/1)
kubectl get pods
kubectl describe pod <NOME_DO_POD>
kubectl logs <NOME_DO_POD>

# Conferir env vars ativas de um deployment
kubectl describe deployment <NOME_DO_DEPLOYMENT>

# Depois de mudar um YAML, reaplique e recrie o pod se necessário
kubectl apply -f deployment/coworking.yaml
kubectl delete pod <NOME_DO_POD_ANTIGO>
```

Erros comuns:
- **Readiness probe failed / timeout**: geralmente a app não consegue conectar no Postgres — confira `DB_HOST`/`DB_PORT`/`DB_USERNAME`/`DB_PASSWORD` no ConfigMap/Secret.
- **ImagePullBackOff**: a URI da imagem em `deployment/coworking.yaml` está errada ou a tag não existe no ECR ainda (rode o CodeBuild antes).
- **CrashLoopBackOff**: veja `kubectl logs` — normalmente é uma env var faltando (`DB_USERNAME`, etc.).

---

## 10. Processo de release de uma nova versão (resumo do fluxo)

1. Altere o código em `analytics/`.
2. `git add`, `git commit`, `git push` para o `main`.
3. O webhook dispara o CodeBuild automaticamente → builda, tageia (`1.0.<build_number>`) e faz push pro ECR.
4. Pegue a nova tag no console do ECR (ou no log do CodeBuild).
5. Atualize `image:` em `deployment/coworking.yaml` com a nova tag.
6. `kubectl apply -f deployment/coworking.yaml` (ou `kubectl set image deployment/coworking coworking=<nova_uri>`).
7. Kubernetes faz o rolling update sozinho, respeitando o readiness probe (zero downtime).

---

## 11. Checklist final de screenshots para submissão

- [ ] Dockerfile (arquivo, não screenshot)
- [ ] AWS CodeBuild — pipeline disparado automaticamente pelo push
- [ ] AWS ECR — repositório `coworking` com imagem versionada
- [ ] `kubectl get svc`
- [ ] `kubectl get pods`
- [ ] `kubectl describe svc postgresql-service`
- [ ] `kubectl describe deployment coworking`
- [ ] AWS CloudWatch Container Insights — logs da aplicação
- [ ] Todos os YAMLs de `deployment/` (arquivos, já estão no repo)
- [ ] README.md (já está no repo)

---

## 12. Limpeza final (evitar gastar o orçamento AWS)

```bash
kubectl port-forward ... # matar qualquer port-forward aberto (ver seção 2)
eksctl delete cluster --name my-cluster --region us-east-1
```

Confira também no console se sobrou algum Load Balancer órfão (o `Service` do tipo `LoadBalancer` cria um ELB que não é destruído automaticamente pelo `eksctl delete cluster` em alguns casos) — delete manualmente em EC2 > Load Balancers se necessário.
