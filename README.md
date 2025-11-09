
# Projeto AWS – CI/CD com Docker, Terraform e ECS Fargate

**Autor:** projetotiteddy  
**Data:** 09/11/2025  
**Região AWS:** `us-east-1`  
**Cluster ECS:** `aws-teste-cluster`  
**Serviço ECS:** `aws-teste-svc`  
**Repositório ECR:** `aws-teste-web`

---

## 📘 Sumário Executivo
Pipeline completo de **Infra como Código e CI/CD**:
- **Etapa 1:** Docker em Linux, pull/run do Nginx e validação com `curl`.
- **Etapa 2:** **Terraform** para provisionar **EC2 com Docker** (rede, SG, AMI Ubuntu, user_data).
- **Etapa 3:** **CI/CD (GitHub Actions)** → build & push no **ECR** e deploy no **ECS Fargate**.
- **Etapa 4:** **Monitoramento** com **CloudWatch Logs** e **alarms** (CPU/Mem/TaskCount).

---

## 🧱 Arquitetura

GitHub Actions (CI/CD)
│
▼
Amazon ECR (imagem)
│
▼
Amazon ECS (Fargate)
│
▼
Container Nginx (porta 80)


---

## 🧩 Componentes
| Camada | Tecnologia | Descrição |
| --- | --- | --- |
| IaC | Terraform | EC2 + SG (HTTP/SSH) + user_data instalando Docker e Nginx |
| Container | Docker | Imagem baseada em Nginx com index.html customizado |
| Registro | Amazon ECR | Armazena imagens versionadas |
| Orquestração | Amazon ECS Fargate | Executa a task sem gerenciar servidores |
| CI/CD | GitHub Actions | Build → Push → Update Service |
| Observabilidade | CloudWatch | Logs e alarmes (CPU, memória, tasks) |

---

## 🚀 Como reproduzir

### 1) Terraform (Etapa 2)

cd terraform-ec2-docker
terraform init
terraform apply -auto-approve

Outputs esperados: `ec2_public_ip` e `sg_id`.

### 2) Imagem Docker (Etapa 1/3)


cd app
docker build -t aws-teste-web:latest .
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com
docker tag aws-teste-web:latest <ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com/aws-teste-web:latest
docker push <ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com/aws-teste-web:latest


### 3) CI/CD (GitHub Actions) (Etapa 3)
Workflow: `.github/workflows/deploy.yml`  
Secrets necessários:


AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_REGION
ECR_REGISTRY
ECR_REPOSITORY
ECS_CLUSTER
ECS_SERVICE
CONTAINER_NAME
TASKDEF_FAMILY
LOG_GROUP
LOG_STREAM_PREFIX

Um `git push` na `main` dispara o pipeline.

### 4) Validar deploy no ECS


aws ecs describe-services --cluster aws-teste-cluster --services aws-teste-svc --region us-east-1
--query "services[0].{running:runningCount,desired:desiredCount,rollout:deployments.rolloutState}"

task_arn=$(aws ecs list-tasks --cluster aws-teste-cluster --service-name aws-teste-svc --query "taskArns[0]" --output text)
eni_id=$(aws ecs describe-tasks --cluster aws-teste-cluster --tasks "$task_arn" --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" --output text)
public_ip=$(aws ec2 describe-network-interfaces --network-interface-ids "$eni_id" --query "NetworkInterfaces[0].Association.PublicIp" --output text)
curl -i "http://$public_ip"

Esperado: `HTTP/1.1 200 OK` + página “Deploy OK via GitHub Actions → ECS Fargate”.

---

## 📊 Monitoramento (Etapa 4)
- **Log group:** `/ecs/aws-teste-web` (prefixo `ecs/<container>/<taskId>`)
- **Alarms criados:**
  - `ECS-CPU-High-aws-teste-svc` (>= 80% por 5 min)
  - `ECS-Memory-High-aws-teste-svc` (>= 80% por 5 min)
  - `ECS-TaskCount-Low-aws-teste-svc` (RunningTaskCount < 1)

---

## ✅ Evidências
- ECR: digest/tag da imagem mais recente  
- ECS: `rolloutState: COMPLETED`, `runningCount=desiredCount=1`  
- `curl -i` retornando **200 OK** da task pública  
- Arquivo **Entrega-Projeto-AWS-DevOps.docx** com prints e comandos  

---

## 🧽 Limpeza (opcional)
- `terraform destroy` em `terraform-ec2-docker`  
- Remover imagens antigas do ECR  
- Apagar serviço/cluster se criar outros ambientes  

---

**Autor:** projetotiteddy – projetotiteddy@gmail.com  
**Data de Entrega:** 09/11/2025
EOF

