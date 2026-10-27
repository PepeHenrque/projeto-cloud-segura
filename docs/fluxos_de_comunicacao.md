# Fluxos de Comunicação — Arquitetura Cloud Segura (AWS)

- **Tema:** Segurança Cloud: IAM, Redes e Boas Práticas
- **Equipe:** Pedro Henrique, Emanuel Gondim, Ricardo Silva e Vitor Marques

Este documento descreve detalhadamente todos os fluxos de rede da arquitetura, especificando as conexões permitidas (origem, destino, protocolo, porta e finalidade), regras de Security Groups, regras de Network ACLs e conexões bloqueadas explicitamente.

---

## 1. Conexões Permitidas

A tabela abaixo atende diretamente ao requisito de mapeamento de fluxos da atividade:

| # | Origem | Destino | Protocolo | Porta | Finalidade |
|---|---|---|---|---|---|
| **1** | Clientes (Internet) | Amazon Route 53 | UDP/TCP (DNS) | 53 | Resolução de nomes do domínio da aplicação |
| **2** | Clientes (Internet) | Amazon CloudFront (+ AWS WAF) | HTTPS (TCP) | 443 | Acesso público seguro à interface web e API |
| **3** | Amazon CloudFront | Application Load Balancer (sg-alb) | HTTPS (TCP) | 443 | Encaminhamento de requisições à origem na VPC via Internet Gateway |
| **4** | Application Load Balancer (sg-alb) | ECS Fargate (sg-app) | HTTP (TCP) | 8080 | Distribuição de requisições para as tarefas da aplicação |
| **5** | ECS Fargate (sg-app) | Amazon RDS PostgreSQL (sg-db) | PostgreSQL / TLS (TCP) | 5432 | Leitura e gravação de dados relacionais transacionais |
| **6** | ECS Fargate (sg-app) | Amazon S3 (via Gateway Endpoint) | HTTPS (TCP) | 443 | Gravação e leitura de arquivos/uploads de usuários de forma privada |
| **7** | ECS Fargate (sg-app) | Interface Endpoints (sg-vpce) | HTTPS (TCP) | 443 | Comunicação privada com Secrets Manager, ECR, CloudWatch Logs e KMS |
| **8a** | Equipe Administrativa | AWS Client VPN | OpenVPN / TLS (UDP/TCP) | 443 | Estabelecimento de túnel administrativo seguro autenticado com MFA |
| **8b** | Client VPN (sg-vpn) | Amazon RDS (sg-db) | PostgreSQL / TLS (TCP) | 5432 | Manutenção administrativa e migrações de banco de dados |
| **8c** | Client VPN (sg-vpn) | ECS Fargate (sg-app) | HTTP (TCP) | 8080 | Acesso a endpoints internos de diagnóstico, métricas e health checks |
| **9** | Serviços AWS (ALB, WAF, VPC, VPN, CloudTrail) | CloudWatch Logs / S3 de Logs | HTTPS (Interno AWS) | 443 | Registro contínuo de logs de auditoria e segurança |
| **10** | ECS Fargate (sg-app) | Internet (via NAT Gateway → IGW) | HTTPS (TCP) | 443 | Conexões de saída para APIs externas e atualizações (sem portas de entrada) |
| **—** | RDS Primário (AZ A) | RDS Standby (AZ B) | Replicação Síncrona AWS | — | Alta disponibilidade e failover automático Multi-AZ |

---

## 2. Regras dos Security Groups (Stateful)

Os Security Groups são configurados referenciando **IDs de outros Security Groups** (e não faixas de IP voláteis), garantindo que as regras permaneçam corretas mesmo com auto-scaling:

| Security Group | Associado a | Regras de Entrada (Ingress) | Regras de Saída (Egress) |
|---|---|---|---|
| **sg-alb** | Application Load Balancer | TCP 443 a partir do *prefix list* gerenciado `com.amazonaws.global.cloudfront.origin-facing` | TCP 8080 direcionado exclusivamente para `sg-app` |
| **sg-app** | Tarefas ECS Fargate | TCP 8080 originado de `sg-alb`<br>TCP 8080 originado de `sg-vpn` | TCP 5432 → `sg-db`<br>TCP 443 → *prefix list* do S3<br>TCP 443 → `sg-vpce`<br>TCP 443 → `0.0.0.0/0` (via NAT) |
| **sg-db** | RDS PostgreSQL | TCP 5432 originado de `sg-app`<br>TCP 5432 originado de `sg-vpn` | Nenhuma regra adicional necessária |
| **sg-vpce** | VPC Interface Endpoints | TCP 443 originado de `sg-app` | — |
| **sg-vpn** | ENIs do Client VPN | Tráfego dos clientes VPN autenticados com MFA | TCP 5432 → `sg-db`<br>TCP 8080 → `sg-app` |

---

## 3. Network Access Control Lists (NACLs - Stateless)

Como segunda camada de defesa na borda das sub-redes:

| Camada de Sub-rede | Regras de Entrada Permitidas | Regras de Bloqueio Explícito |
|---|---|---|
| **Pública** | TCP 443 vindo de `0.0.0.0/0`<br>Portas efêmeras 1024–65535 (tráfego de retorno) | TCP 22 (SSH) e TCP 3389 (RDP) de qualquer origem |
| **Privada (Aplicação)** | TCP 8080 vindo das sub-redes públicas<br>TCP 8080 vindo da faixa Client VPN (`10.100.0.0/22`)<br>TCP 443 da própria VPC<br>Portas efêmeras de retorno | Bloqueio de todo o restante |
| **Privada (Dados)** | TCP 5432 vindo das sub-redes de aplicação<br>TCP 5432 vindo da faixa Client VPN (`10.100.0.0/22`)<br>Portas efêmeras de retorno | **Bloqueio total** de qualquer outro tráfego, inclusive de sub-redes públicas |

---

## 4. Conexões Bloqueadas

| Origem | Destino | Mecanismo de Bloqueio |
|---|---|---|
| **Internet pública** | Application Load Balancer diretamente | `sg-alb` aceita apenas o prefix list do CloudFront, e o ALB exige cabeçalho HTTP secreto |
| **Internet pública** | Tarefas ECS Fargate ou RDS | Alocadas em sub-redes privadas sem IP público e sem rota no IGW |
| **Internet pública** | Bucket S3 de uploads | *Block Public Access* ativo e bucket policy com negação explícita fora do `aws:SourceVpce` |
| **Sub-redes Públicas** | Banco de dados (RDS) | `sg-db` não referencia `sg-alb`; NACL de dados rejeita sub-redes públicas |
| **Sub-redes de Dados** | Internet | Tabela de rotas sem rota default `0.0.0.0/0` |
| **Qualquer origem** | Portas SSH (22) e RDP (3389) | Negadas em todos os SGs e NACLs (sem servidores bastion) |
| **Usuários sem VPN/MFA** | Recursos internos | Não há caminho de rede alternativo para a VPC |
