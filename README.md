# Projeto: Arquitetura Cloud Segura (AWS)

Atividade proposta para o tema **Segurança Cloud: IAM, Redes e Boas Práticas**.

**Equipe:** Pedro Henrique, Emanuel Gondim, Ricardo Silva e Vitor Marques

---

## 1. Visão Geral do Cenário

Uma empresa está migrando para a AWS uma aplicação web composta por **interface web, API de negócio, banco de dados relacional e armazenamento de arquivos**. 

O objetivo central desta arquitetura é garantir que a aplicação funcione com alta disponibilidade e resiliência, **sem expor desnecessariamente seus componentes internos à internet**, garantindo:
- **Defesa em profundidade** através de múltiplas camadas independentes;
- **Isolamento de recursos** via VPC 3-tier distribuída em 2 Availability Zones;
- **Controle de acesso rigoroso** e aplicação do princípio do menor privilégio;
- **Proteção nas comunicações** com criptografia ponta a ponta (TLS 1.2+ e KMS);
- **Rastreabilidade e auditoria** contínuas de eventos e acessos.

---

## 2. Diagrama da Arquitetura

![Diagrama da Arquitetura](docs/arquitetura-cloud-segura.png)

---

## 3. Estrutura do Repositório e Entregáveis

| Arquivo | Descrição |
|---|---|
| [`arquitetura-cloud-segura.drawio`](arquitetura-cloud-segura.drawio) | **Diagrama editável** da arquitetura no Draw.io (diagrams.net). |
| [`docs/arquitetura-cloud-segura.png`](docs/arquitetura-cloud-segura.png) | Exportação em alta resolução do diagrama da arquitetura. |
| [`docs/justificativa_tecnica.pdf`](docs/justificativa_tecnica.pdf) | **Justificativa técnica completa** com todas as decisões de projeto e segurança. |
| [`docs/fluxos_de_comunicacao.md`](docs/fluxos_de_comunicacao.md) | **Tabelas de fluxos de rede:** conexões permitidas, regras de Security Groups, NACLs e conexões bloqueadas. |

---

## 4. Resumo das Principais Decisões de Arquitetura

1. **Entrada Única e Protegida:** Acesso público via Route 53 e CloudFront com terminação TLS (ACM) e AWS WAF na borda. O Application Load Balancer (ALB) só aceita conexões do CloudFront validadas por um cabeçalho HTTP secreto.
2. **VPC em 3 Camadas (2 AZs):**
   - **Pública:** ALB e NAT Gateways (`0.0.0.0/0 → IGW`).
   - **Privada de Aplicação:** ECS Fargate e VPC Endpoints (sem IP público, saída controlada via NAT).
   - **Privada de Dados:** RDS PostgreSQL Multi-AZ com **zero rota para internet** (isolamento total).
3. **Armazenamento Seguro de Arquivos:** Bucket S3 privado com *Block Public Access*, criptografia SSE-KMS e acesso exclusivo pelo **VPC Gateway Endpoint** via bucket policy.
4. **Acesso Administrativo Sem Bastion:** **AWS Client VPN** com autenticação federada (IAM Identity Center), **MFA obrigatório** e autorização granular por grupos. Nenhuma porta 22 (SSH) ou 3389 (RDP) é aberta.
5. **Menor Privilégio e Gestão de Segredos:** Permissões mínimas no IAM (Task Roles temporárias para serviços) e senhas do banco gerenciadas com rotação automática pelo **AWS Secrets Manager**.
6. **Auditoria e Observabilidade:** Logs centralizados no CloudWatch, rastreamento de chamadas com AWS CloudTrail, telemetria de rede via VPC Flow Logs, detecção de ameaças com Amazon GuardDuty e retenção imutável de logs no S3 com **Object Lock**.
