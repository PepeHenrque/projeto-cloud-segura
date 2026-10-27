# Arquitetura Cloud Segura (AWS)

Proposta de arquitetura em nuvem para a migração de uma aplicação web, com foco em **isolamento de recursos, controle de acesso, proteção das comunicações e monitoramento**.

**Tema:** Segurança Cloud: IAM, Redes e Boas Práticas
**Ferramenta:** Draw.io (diagrams.net)
**Equipe:** Pedro Henrique, Emanuel Gondim, Ricardo Silva e Vitor Marques

---

## 1. Cenário

Uma empresa está migrando para a AWS uma aplicação web composta por interface web, API de negócio, banco de dados relacional e armazenamento de arquivos enviados pelos usuários.

O objetivo é que a aplicação funcione com alta disponibilidade e resiliência **sem expor seus componentes internos à internet**, garantindo:

- **Defesa em profundidade**, com várias camadas independentes;
- **Isolamento de recursos** em uma VPC de 3 camadas distribuída em 2 Availability Zones;
- **Controle de acesso rigoroso** e princípio do menor privilégio;
- **Comunicações protegidas** com criptografia (TLS 1.2+ e KMS);
- **Rastreabilidade e auditoria** contínuas de eventos e acessos.

## 2. Diagrama da arquitetura

![Diagrama da arquitetura](docs/arquitetura-cloud-segura.png)

O arquivo editável está em [`arquitetura-cloud-segura.drawio`](arquitetura-cloud-segura.drawio).

## 3. Caminho de uma requisição

```
Cliente → Route 53 → CloudFront + WAF → ALB → ECS Fargate → RDS PostgreSQL
                                                    ├──→ S3 (Gateway Endpoint)
                                                    └──→ Secrets Manager / KMS / ECR / Logs (Interface Endpoints)

Administrador → Client VPN (IAM Identity Center + MFA) → ECS Fargate (8080) / RDS (5432)
```

## 4. Decisões de arquitetura

| Tema | Decisão |
|---|---|
| **Entrada única e protegida** | Acesso público via Route 53 e CloudFront, com TLS (ACM) e AWS WAF na borda. O ALB só aceita conexões do CloudFront e exige um cabeçalho HTTP secreto |
| **VPC em 3 camadas (2 AZs)** | **Pública:** ALB e NAT Gateways. **Privada de aplicação:** ECS Fargate e VPC Endpoints, sem IP público. **Privada de dados:** RDS PostgreSQL Multi-AZ, sem rota para a internet |
| **Armazenamento de arquivos** | Bucket S3 privado com Block Public Access, criptografia SSE-KMS e acesso exclusivo pelo VPC Gateway Endpoint (bucket policy) |
| **Acesso administrativo sem bastion** | AWS Client VPN com autenticação federada (IAM Identity Center), MFA obrigatório e autorização por grupo. Nenhuma porta 22 ou 3389 é aberta |
| **Menor privilégio e segredos** | Task Roles temporárias para os serviços e senha do banco no Secrets Manager, com rotação automática |
| **Auditoria e observabilidade** | CloudTrail, VPC Flow Logs, CloudWatch, GuardDuty, Config e Security Hub, com logs imutáveis no S3 (Object Lock) |

## 5. Requisitos da atividade e como são atendidos

| # | Requisito | Como a arquitetura atende |
|---|---|---|
| 1 | Acesso pela internet com HTTPS | CloudFront com certificado do ACM (TLS 1.2+) |
| 2 | Proteção e distribuição do tráfego | AWS WAF no CloudFront e Application Load Balancer |
| 3 | API processa requisições e acessa recursos | ECS Fargate em sub-rede privada, com Task Role de menor privilégio |
| 4 | Banco sem acesso direto pela internet | RDS em sub-rede de dados, sem IP público e sem rota para a internet |
| 5 | Arquivos em serviço apropriado e controlado | S3 privado, SSE-KMS, acesso só pelo Gateway Endpoint |
| 6 | Acesso administrativo seguro | Client VPN com MFA, sem bastion e sem SSH/RDP |
| 7 | Comunicações limitadas ao necessário | Security Groups encadeados por ID, NACLs e deny implícito |
| 8 | Registro e análise de eventos | CloudTrail, Flow Logs, CloudWatch, GuardDuty, Config e Security Hub |
| 9 | Menor privilégio | IAM Roles temporárias, autorização por grupo na VPN e segredos no Secrets Manager |

## 6. Estrutura do repositório

```
projeto-cloud-segura/
├── README.md
├── arquitetura-cloud-segura.drawio      # Diagrama editável (Draw.io)
└── docs/
    ├── arquitetura-cloud-segura.png     # Exportação do diagrama
    ├── justificativa_tecnica.pdf        # Justificativa técnica completa
    ├── fluxos_de_comunicacao.md         # Fluxos, Security Groups, NACLs, bloqueios e monitoramento
    └── slides_apresentacao.pdf          # Slides da apresentação (9 slides)
```

| Arquivo | Descrição |
|---|---|
| [`arquitetura-cloud-segura.drawio`](arquitetura-cloud-segura.drawio) | Diagrama editável da arquitetura |
| [`docs/arquitetura-cloud-segura.png`](docs/arquitetura-cloud-segura.png) | Exportação em alta resolução do diagrama |
| [`docs/justificativa_tecnica.pdf`](docs/justificativa_tecnica.pdf) | Decisões de projeto e de segurança, componente por componente |
| [`docs/fluxos_de_comunicacao.md`](docs/fluxos_de_comunicacao.md) | Conexões permitidas (origem, destino, protocolo, porta e finalidade), Security Groups, NACLs, conexões bloqueadas e plano de monitoramento |
| [`docs/slides_apresentacao.pdf`](docs/slides_apresentacao.pdf) | Slides da apresentação da equipe |

## 7. Como editar o diagrama

1. Abra o repositório no **VS Code**.
2. Instale a extensão **Draw.io Integration** (Henning Dieterichs).
3. Abra o arquivo `arquitetura-cloud-segura.drawio` e edite direto no editor.
4. Exporte o PNG para `docs/arquitetura-cloud-segura.png` sempre que o diagrama mudar, para os dois ficarem iguais.

## 8. Equipe

| Integrantes |
|---|
| Pedro Henrique |
| Emanuel Gondim |
| Ricardo Silva |
| Vitor Marques |