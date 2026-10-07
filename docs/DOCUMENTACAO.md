# Documentação Técnica — POC de Orquestração com Kubernetes

**Projeto:** Web Solutions Ltda. — Prova de conceito Nginx + Apache HTTPD no Kubernetes
**Autor:** Guilherme Oliveira

## 1. Contexto e objetivo

A Web Solutions hospeda suas aplicações diretamente em máquinas virtuais. Esse modelo é
inflexível (alocação fixa de recursos), lento para escalar (processo manual) e propenso a
conflitos de dependências entre aplicações que dividem o mesmo host.

Esta POC demonstra a migração para contêineres orquestrados por Kubernetes, usando dois
serviços web independentes — **Nginx** e **Apache HTTPD** — coexistindo no mesmo cluster,
cada um acessível em sua própria porta:

| Serviço | Porta externa | Porta interna do container |
|---|---|---|
| Nginx | 8080 | 8080 |
| Apache HTTPD | 8081 | 8081 |

## 2. Estrutura do repositório

```
docker/
  nginx/
    Dockerfile
    nginx.conf
    html/index.html
  apache/
    Dockerfile
    html/index.html
k8s/
  00-namespace.yaml
  nginx-deployment.yaml
  nginx-service.yaml
  nginx-hpa.yaml
  apache-deployment.yaml
  apache-service.yaml
  apache-hpa.yaml
docs/
  DOCUMENTACAO.md
  roteiro-video.md
```

## 3. Imagens Docker

### 3.1 Decisões de projeto

- **Imagens base Alpine** (`nginx:1.27-alpine`, `httpd:2.4-alpine`): reduzem drasticamente o
  tamanho final da imagem (dezenas de MB, contra centenas nas variantes Debian) e diminuem a
  superfície de ataque, já que trazem menos pacotes do sistema operacional.
- **Cada imagem escuta a porta exigida pelo desafio já dentro do container** (Nginx em 8080,
  Apache em 8081), em vez de manter a porta padrão (80) e remapear só no Kubernetes. Isso deixa
  o comportamento do container idêntico em qualquer ambiente (`docker run` local, CI, ou dentro
  do pod), sem depender de tradução de porta feita só pelo orquestrador.
- **HEALTHCHECK** embutido em cada Dockerfile: permite depurar o container isoladamente
  (`docker ps` já mostra o status `healthy`/`unhealthy`) antes mesmo de chegar ao cluster.
- **Apenas o conteúdo estático necessário é copiado** para dentro da imagem — nenhuma
  ferramenta de build permanece na imagem final, mantendo-a enxuta.

### 3.2 Build das imagens

```bash
docker build -t web-solutions/nginx:1.0 ./docker/nginx
docker build -t web-solutions/apache:1.0 ./docker/apache
```

### 3.3 Teste local (fora do Kubernetes)

```bash
docker run --rm -p 8080:8080 web-solutions/nginx:1.0
docker run --rm -p 8081:8081 web-solutions/apache:1.0
```

Acessar `http://localhost:8080` e `http://localhost:8081` no navegador.

## 4. Manifestos Kubernetes

### 4.1 Namespace

Todos os recursos vivem no namespace `web-solutions` (`00-namespace.yaml`). Isso isola a POC
do restante do cluster, simplifica a limpeza (`kubectl delete namespace web-solutions`) e
prepara o terreno para aplicar políticas de RBAC/quota por projeto no futuro.

### 4.2 Deployments

Cada serviço tem seu próprio `Deployment`, com **2 réplicas** por padrão. Usar um Deployment
(em vez de criar Pods soltos) garante:

- **Auto-recuperação**: se um Pod falhar ou o node cair, o controller recria o Pod
  automaticamente para manter o número de réplicas desejado.
- **Rolling updates**: uma nova versão da imagem pode ser publicada sem downtime
  (`kubectl set image deployment/nginx-deployment nginx=web-solutions/nginx:1.1 -n web-solutions`).
- **Escalonamento independente**: Nginx e Apache são Deployments separados, então é possível
  escalar um sem afetar o outro (`kubectl scale deployment/apache-deployment --replicas=4 -n web-solutions`).

Cada Deployment define:
- `resources.requests/limits`: evita que um serviço consuma recursos do outro no mesmo node
  (requisito indireto do desafio: "alocação de recursos flexível", resolvendo exatamente o
  problema relatado no modelo antigo de VMs).
- `readinessProbe`/`livenessProbe`: o Kubernetes só envia tráfego para um Pod depois que ele
  responde corretamente (readiness) e reinicia o container automaticamente se ele parar de
  responder (liveness).

### 4.3 Services

Cada serviço web tem um `Service` do tipo `LoadBalancer` dedicado, mapeando a porta externa
exigida (8080/8081) para a porta do container. O `Service` seleciona os Pods pelo label
`app: nginx` ou `app: apache`, desacoplando o endereço de rede estável da identidade (IP
efêmero) de cada Pod individual — é isso que permite que Pods sejam recriados livremente sem
quebrar o acesso externo.

`type: LoadBalancer` foi escolhido porque, em ambientes de desenvolvimento locais como Docker
Desktop e Kind (com `cloud-provider=kind` ou similar), ele expõe a porta diretamente em
`localhost`, batendo exatamente com o requisito "Nginx na 8080 / Apache na 8081" sem exigir a
faixa 30000-32767 do NodePort. Em um cluster gerenciado (EKS/GKE/AKS) o mesmo manifesto
provisionaria automaticamente um Load Balancer real do provedor de nuvem — nenhuma alteração de
YAML seria necessária, apenas trocar de ambiente. Alternativas documentadas:

- **NodePort**: funciona em qualquer cluster, mas expõe em uma porta alta (ex.: 30080), exigindo
  redirecionamento externo (proxy/iptables) para bater com 8080/8081 exatamente.
- **Ingress**: recomendado quando o número de aplicações crescer — permite rotear por
  hostname/path (`nginx.websolutions.com`, `apache.websolutions.com`) usando um único IP
  externo, em vez de um Service por porta.
- **kubectl port-forward**: útil apenas para debug pontual, não para expor a aplicação de forma
  permanente.

### 4.4 Deploy

```bash
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/nginx-deployment.yaml -f k8s/nginx-service.yaml
kubectl apply -f k8s/apache-deployment.yaml -f k8s/apache-service.yaml

# opcional — autoscaling (requer metrics-server no cluster)
kubectl apply -f k8s/nginx-hpa.yaml -f k8s/apache-hpa.yaml
```

Verificação:

```bash
kubectl get all -n web-solutions
curl http://localhost:8080
curl http://localhost:8081
```

> Se as imagens não estiverem em um registry remoto, um cluster local (Docker Desktop
> Kubernetes ou Kind) usa diretamente as imagens do Docker local. No Kind é necessário
> `kind load docker-image web-solutions/nginx:1.0` antes do `kubectl apply`.

## 5. Persistência de dados

O desafio explicitamente marca este item como não obrigatório para o cenário atual (páginas
estáticas). Caso uma aplicação futura precise persistir dados (ex.: logs, uploads, conteúdo
editável), o padrão a seguir seria:

1. Criar um `PersistentVolumeClaim` (PVC) solicitando armazenamento de uma `StorageClass`
   disponível no cluster.
2. Montar o PVC como `volumeMount` no Deployment (ex.: em `/usr/share/nginx/html` para
   servir conteúdo dinâmico, ou em `/var/log` para persistir logs entre reinicializações de
   Pod).

Isso não foi implementado aqui porque o conteúdo é estático e já embutido na imagem — adicionar
um volume sem necessidade real adicionaria complexidade sem benefício (indo contra o princípio
de manter a solução simples e replicável).

## 6. Escalabilidade

Escalonamento manual imediato:

```bash
kubectl scale deployment/nginx-deployment --replicas=4 -n web-solutions
kubectl scale deployment/apache-deployment --replicas=1 -n web-solutions
```

Escalonamento automático (bônus, arquivos `*-hpa.yaml`): um `HorizontalPodAutoscaler` por
serviço ajusta o número de réplicas entre 2 e 6 com base no uso de CPU (limite de 70%),
mantendo os dois serviços independentes entre si — exatamente o requisito de "escalar ambos os
serviços independentemente" citado no desafio.

## 7. Como expandir a solução para outras aplicações

O padrão usado aqui é replicável para qualquer nova aplicação da Web Solutions:

1. Criar uma imagem Docker dedicada, otimizada (base Alpine/slim, apenas o necessário copiado).
2. Criar um `Deployment` com requests/limits de recursos e probes de saúde.
3. Criar um `Service` dedicado (ou, à medida que o número de aplicações crescer, migrar todos
   os serviços para trás de um único `Ingress Controller`, evitando um Load Balancer por
   aplicação e centralizando TLS/roteamento por domínio).
4. Manter cada aplicação em seu próprio namespace ou em namespaces agrupados por
   time/domínio, com `ResourceQuota` para evitar que uma aplicação consuma recursos de outra —
   resolvendo em definitivo o problema original de "alocação de recursos inflexível e conflitos
   de ambiente" relatado pela Web Solutions.

## 8. Resumo dos benefícios entregues

| Problema relatado pela Web Solutions | Como esta solução resolve |
|---|---|
| Alocação de recursos inflexível | `resources.requests/limits` por serviço, no mesmo cluster |
| Escalabilidade manual e lenta | `kubectl scale` imediato ou HPA automático |
| Conflitos de dependência/ambiente | Cada serviço isolado em sua própria imagem/container |
| Tempo de resposta para implantar/atualizar | Rolling update declarativo via `kubectl apply` |
