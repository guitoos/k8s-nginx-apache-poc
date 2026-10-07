# Imagem base oficial, variante Alpine: consideravelmente menor (~40MB)
# e com menos superficie de ataque que a imagem "full" baseada em Debian.
FROM nginx:1.27-alpine

# Remove a configuracao padrao (escuta na porta 80) para evitar conflito
# com a configuracao customizada abaixo.
RUN rm /etc/nginx/conf.d/default.conf

# Configuracao customizada: Nginx passa a escutar na porta 8080,
# conforme exigido pelo desafio da Web Solutions.
COPY nginx.conf /etc/nginx/conf.d/nginx.conf

# Conteudo estatico do site
COPY html/ /usr/share/nginx/html/

EXPOSE 8080

# Usada pelas probes de liveness/readiness do Kubernetes
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s \
    CMD wget -qO- http://127.0.0.1:8080/health || exit 1
