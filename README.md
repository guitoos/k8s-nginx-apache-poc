# Web Solutions Ltda. — POC Nginx + Apache HTTPD no Kubernetes

Projeto acadêmico do curso de Análise e Desenvolvimento de Sistemas da UniFECAF.

Trabalho: **Orquestração de Containers com Kubernetes: Desafios e Soluções no Mundo Real**

Autor: **Guilherme Oliveira** · [LinkedIn](https://www.linkedin.com/in/guilhermeoss)

## Resumo

Solução de conteinerização e orquestração para a Web Solutions Ltda., colocando dois
servidores web independentes — Nginx e Apache HTTPD — para coexistir em um único cluster
Kubernetes, cada um exposto em sua própria porta:

- **Nginx → http://localhost:8080**
- **Apache HTTPD → http://localhost:8081**

A solução foi construída, implantada e validada de ponta a ponta: imagens Docker próprias,
manifestos Kubernetes aplicados em um cluster real (Docker Desktop + Kubernetes), com os dois
serviços respondendo HTTP 200 nas portas especificadas.

## Estrutura

```
docker/                       Dockerfiles otimizados (imagens Alpine)
  nginx/
  apache/
k8s/                           Manifestos Kubernetes (Deployment + Service + HPA)
docs/
  DOCUMENTACAO.md              Documentação técnica completa do processo e decisões
  apresentacao.html            Fonte do documento de apresentação
  screenshots/                 Prints do Nginx (8080) e Apache (8081) em produção
  trabalho-conceitual-yaml.html Fonte do trabalho conceitual
README.md                      Este arquivo
```

## Entregáveis

1. **Apresentação do projeto** — `docs/apresentacao.html`: problema, solução, arquitetura,
   demonstração com prints reais do Nginx e do Apache em execução, e benefícios entregues.
2. **Trabalho Conceitual sobre YAML** — `docs/trabalho-conceitual-yaml.html`.
3. **Solução Técnica YAML** — pasta `k8s/`: Deployments e Services do Nginx e do Apache, nas
   portas 8080 e 8081, mais HPA para autoscaling independente.
4. **Documentação técnica** — `docs/DOCUMENTACAO.md`: decisões de arquitetura, comandos de
   build/deploy e como a solução se expande para novas aplicações.

## Como rodar a solução

```bash
# 1. Build das imagens
docker build -t web-solutions/nginx:1.0 ./docker/nginx
docker build -t web-solutions/apache:1.0 ./docker/apache

# 2. Deploy no cluster
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/nginx-deployment.yaml -f k8s/nginx-service.yaml
kubectl apply -f k8s/apache-deployment.yaml -f k8s/apache-service.yaml

# opcional: autoscaling (requer metrics-server)
kubectl apply -f k8s/nginx-hpa.yaml -f k8s/apache-hpa.yaml

# 3. Verificar
kubectl get all -n web-solutions
```

Acesso:
- Nginx: http://localhost:8080
- Apache: http://localhost:8081

Detalhes completos de cada decisão de arquitetura estão em `docs/DOCUMENTACAO.md`.
