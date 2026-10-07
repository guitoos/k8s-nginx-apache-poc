# Imagem base oficial, variante Alpine: menor superficie de ataque e
# menor tamanho final de imagem que a imagem "full" baseada em Debian.
FROM httpd:2.4-alpine

# Altera a porta padrao de escuta do Apache (80) para 8081,
# conforme exigido pelo desafio da Web Solutions.
RUN sed -i 's/^Listen 80$/Listen 8081/' /usr/local/apache2/conf/httpd.conf

# Conteudo estatico do site
COPY html/ /usr/local/apache2/htdocs/

EXPOSE 8081

# Usada pelas probes de liveness/readiness do Kubernetes
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s \
    CMD wget -qO- http://127.0.0.1:8081/ || exit 1
