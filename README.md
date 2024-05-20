Docker - Aula 46: Balanceamento de Carga com HAProxy e WordPress
Este repositório contém o código da aula 46 do curso "Docker do Zero - Introdução à Administração de Containers" do instrutor Ricardo Ferreira. Nele, você aprenderá como implementar um sistema de balanceamento de carga com HAProxy para dois servidores WordPress.
Pré-requisitos
Docker instalado e configurado em sua máquina.
Familiaridade básica com Docker.
Estrutura do Projeto
O projeto é dividido em três containers principais:
MySQL: Contém o banco de dados para os dois sites WordPress.
WordPress (x2): Dois servidores WordPress idênticos, cada um com um site diferente.
HAProxy: Atua como um balanceador de carga, direcionando o tráfego para um dos servidores WordPress.
Como Executar o Projeto
Criar a rede:
docker network create wordpress-network
Use code with caution.
Bash
Criar o container do MySQL:
docker run -d --name mysql -p 3306:3306 --network wordpress-network -e MYSQL_ROOT_PASSWORD=password mysql:latest
Use code with caution.
Bash
Criar os containers do WordPress:
# Para o primeiro site WordPress
docker run -d --name wordpress1 -p 8081:80 --network wordpress-network --env WORDPRESS_DB_HOST=mysql --env WORDPRESS_DB_USER=root --env WORDPRESS_DB_PASSWORD=password --env WORDPRESS_DB_NAME=wordpress1 wordpress:latest

# Para o segundo site WordPress
docker run -d --name wordpress2 -p 8082:80 --network wordpress-network --env WORDPRESS_DB_HOST=mysql --env WORDPRESS_DB_USER=root --env WORDPRESS_DB_PASSWORD=password --env WORDPRESS_DB_NAME=wordpress2 wordpress:latest
Use code with caution.
Bash
Criar o container do HAProxy:
docker run -d --name haproxy -p 80:80 --network wordpress-network -v haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro haproxy:latest
Use code with caution.
Bash
Criar o arquivo haproxy.cfg:
global
    log 127.0.0.1 local0
    log 127.0.0.1 local1 notice
    chroot /var/lib/haproxy
    stats socket /run/
Use code with caution.

User
ajuste o arquivo com o meu docker-compose.yml:
services:
mysql:
image: mysql:5.5
restart: always
container_name: mysql_wordpress_ha_compose
volumes:
- db-ha-data:/var/lib/mysql
networks:
- backend-web
environment:
MYSQL_ROOT_PASSWORD: root
MYSQL_DATABASE: wordpress
MYSQL_USER: wordpress
MYSQL_PASSWORD: wordpress
wordpress1:
depends_on:
- mysql
networks:
- backend-web
image: wordpress:4.7-php5.6
restart: always
container_name: wordpress1_ha_compose
volumes:
- wp-ha-data:/var/www/html
ports:
- 81:80
environment:
WORDPRESS_DB_HOST: mysql:3306
WORDPRESS_DB_NAME: wordpress
WORDPRESS_DB_USER: wordpress
WORDPRESS_DB_PASSWORD: wordpress
wordpress2:
depends_on:
- mysql
networks:
- backend-web
image: wordpress:4.7-php5.6
restart: always
container_name: wordpress2_ha_compose
volumes:
- wp-ha-data:/var/www/html
ports:
- 82:80
environment:
WORDPRESS_DB_HOST: mysql:3306
WORDPRESS_DB_NAME: wordpress
WORDPRESS_DB_USER: wordpress
WORDPRESS_DB_PASSWORD: wordpress
haproxy:
image: haproxy:1.6
restart: always
networks:
- frontend-web
- backend-web
ports:
- 8080:80
volumes:
- "./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro"
volumes:
db-ha-data:
wp-ha-data:
networks:
frontend-web:
internal: false
backend-web:
internal: true

Model
7,1s
version: '3.7'

services:
  mysql:
    image: mysql:5.5
    restart: always
    container_name: mysql_wordpress_ha_compose
    volumes:
      - db-ha-data:/var/lib/mysql
    networks:
      - backend-web
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress

  wordpress1:
    depends_on:
      - mysql
    networks:
      - backend-web
    image: wordpress:4.7-php5.6
    restart: always
    container_name: wordpress1_ha_compose
    volumes:
      - wp-ha-data:/var/www/html
    ports:
      - "81:80" # Expondo a porta 81 do container na porta 81 da máquina
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress

  wordpress2:
    depends_on:
      - mysql
    networks:
      - backend-web
    image: wordpress:4.7-php5.6
    restart: always
    container_name: wordpress2_ha_compose
    volumes:
      - wp-ha-data:/var/www/html
    ports:
      - "82:80" # Expondo a porta 82 do container na porta 82 da máquina
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress

  haproxy:
    image: haproxy:1.6
    restart: always
    networks:
      - frontend-web
      - backend-web
    ports:
      - "8080:80" # Expondo a porta 8080 do container na porta 8080 da máquina
    volumes:
      - "./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro"
    environment:
      - "HAPROXY_BACKEND_SERVERS=wordpress1:80,wordpress2:80" # Definindo os backends para o HAProxy

volumes:
  db-ha-data:
  wp-ha-data:

networks:
  frontend-web:
    internal: false
  backend-web:
    internal: true
Use code with caution.
Yaml
Explicação das mudanças:
Ports: Os ports foram modificados para serem acessíveis externamente. O wordpress1 está na porta 81, wordpress2 na porta 82 e o haproxy na porta 8080.
HAPROXY_BACKEND_SERVERS: Essa variável de ambiente no haproxy define os servidores backend que ele irá balancear. No exemplo, wordpress1:80 e wordpress2:80 são os servidores backend.
haproxy.cfg: Você precisará criar um arquivo haproxy.cfg no diretório raiz do projeto e configurá-lo para balancear o tráfego entre os servidores wordpress1 e wordpress2.
Observações:
Certifique-se de que os volumes db-ha-data e wp-ha-data sejam criados corretamente.
O arquivo haproxy.cfg é fundamental para o funcionamento do balanceador de carga.
Você pode usar outras versões do Docker e dos serviços, ajustando o arquivo docker-compose.yml de acordo com as necessidades.
Lembre-se de executar docker-compose up -d para iniciar os containers.

User
segue o arquivo haproxy.cfg:
defaults
log global
mode http
timeout connect 50000
timeout client 50000
timeout server 50000
frontend main
bind *:80
default_backend wordpress
backend wordpress
balance roundrobin
mode http
# Verify that service is available
#option httpchk OPTIONS * HTTP/1.1\r\nHOST:localhost
# Insert X-Forwarded-For header
#option forwardfor
server wp1 wordpress1:80 weight 1 minconn 3 maxconn 500 check
server wp2 wordpress2:80 weight 1 minconn 3 maxconn 500 check
#This is the virtual URL to access the stats page
    stats uri /haproxy_stats

    #The user/pass you want to use
    stats auth admin:admin
    
    #This allow you to take down and bring up back end servers.
    stats admin if TRUE
Use code with caution.

Model
7,4s
warning
O arquivo haproxy.cfg que você forneceu está bem configurado para o balanceamento de carga entre os dois servidores WordPress. Aqui está uma análise detalhada de cada seção:
1. defaults:
log global: Define que os logs do HAProxy serão enviados para o servidor global (definido em global).
mode http: Especifica que o HAProxy está operando no modo HTTP.
timeout connect: Define o tempo limite para conexão com o servidor backend (50 segundos).
timeout client: Define o tempo limite para a conexão do cliente (50 segundos).
timeout server: Define o tempo limite para a resposta do servidor backend (50 segundos).
2. frontend main:
bind *:80: Define que o HAProxy deve ouvir na porta 80 de todas as interfaces de rede.
default_backend wordpress: Especifica que o backend padrão para todas as solicitações é wordpress.
3. backend wordpress:
balance roundrobin: Define que o HAProxy deve balancear o tráfego entre os servidores backend usando o método round robin (em rotação).
mode http: Especifica que o backend está operando no modo HTTP.
option httpchk OPTIONS * HTTP/1.1\r\nHOST:localhost: Define uma verificação de saúde para os servidores backend usando um pedido HTTP OPTIONS. Isso garante que os servidores backend estejam ativos antes de enviar tráfego para eles.
option forwardfor: Insere o cabeçalho X-Forwarded-For nas solicitações encaminhadas para os servidores backend, fornecendo ao servidor backend o endereço IP original do cliente.
server wp1 wordpress1:80 weight 1 minconn 3 maxconn 500 check: Define o primeiro servidor backend (WordPress 1) com as seguintes configurações:
wp1: O nome do servidor.
wordpress1:80: O endereço IP e porta do servidor backend.
weight 1: O peso do servidor, que determina a probabilidade de o HAProxy enviar tráfego para ele.
minconn 3: O número mínimo de conexões ativas que devem ser mantidas com o servidor.
maxconn 500: O número máximo de conexões simultâneas permitidas com o servidor.
check: Habilita a verificação de saúde para este servidor.
server wp2 wordpress2:80 weight 1 minconn 3 maxconn 500 check: Define o segundo servidor backend (WordPress 2) com as mesmas configurações do primeiro.
4. stats:
stats uri /haproxy_stats: Define o URI para a página de estatísticas do HAProxy.
stats auth admin:admin: Define o nome de usuário e senha para acesso à página de estatísticas.
stats admin if TRUE: Habilita a página de estatísticas.
Observações:
O arquivo haproxy.cfg está bem configurado para um cenário básico de balanceamento de carga. Você pode personalizar as configurações de acordo com suas necessidades específicas, como ajustar o método de balanceamento, o peso dos servidores, os tempos limite, etc.
A verificação de saúde é uma funcionalidade importante para garantir que o HAProxy não envie tráfego para servidores backend que estão inativos.
A página de estatísticas do HAProxy é útil para monitorar o desempenho do balanceador de carga e dos servidores backend.
Com este arquivo haproxy.cfg e o seu docker-compose.yml, você terá um sistema de balanceamento de carga funcional para seus dois servidores WordPress!

User
reescreva o readme.md de uma forma bonita, referenciando e comentando os dois arquivos docker-compose.ym e o haproxy.cfg

Model
7,9s
Docker - Aula 46: Balanceamento de Carga com HAProxy e WordPress
Este repositório demonstra a implementação de um sistema de balanceamento de carga com HAProxy para dois servidores WordPress, utilizando Docker Compose. O código foi desenvolvido como parte da aula 46 do curso "Docker do Zero - Introdução à Administração de Containers" do instrutor Ricardo Ferreira.
Pré-requisitos
Docker instalado e configurado em sua máquina.
Familiaridade básica com Docker e Docker Compose.
Estrutura do Projeto
O projeto é composto por quatro containers principais:
MySQL: Contém o banco de dados para os dois sites WordPress.
WordPress (x2): Dois servidores WordPress idênticos, cada um com um site diferente.
HAProxy: Atua como um balanceador de carga, direcionando o tráfego para um dos servidores WordPress.
Arquivos do Projeto
docker-compose.yml: Define a configuração dos containers do Docker, incluindo:
mysql: Configura o container do banco de dados MySQL. O volume db-ha-data garante persistência dos dados do banco.
wordpress1: Configura o primeiro servidor WordPress, expondo a porta 81 do container na porta 81 da máquina.
wordpress2: Configura o segundo servidor WordPress, expondo a porta 82 do container na porta 82 da máquina.
haproxy: Configura o container do HAProxy, expondo a porta 8080 do container na porta 8080 da máquina. O volume ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro monta o arquivo de configuração do HAProxy dentro do container. A variável de ambiente HAPROXY_BACKEND_SERVERS define os servidores backend que o HAProxy deve balancear.
haproxy.cfg: Define a configuração do HAProxy, incluindo:
defaults: Define configurações padrão para todas as solicitações.
frontend main: Define o frontend do HAProxy, que escuta na porta 80 de todas as interfaces de rede.
backend wordpress: Define o backend do HAProxy, que balanceia o tráfego entre os servidores WordPress (wordpress1 e wordpress2). A seção inclui a configuração de balanceamento roundrobin, verificação de saúde (check) para os servidores backend e a definição da página de estatísticas do HAProxy.
Como Executar o Projeto
Criar as redes:
docker network create backend-web
docker network create frontend-web
Use code with caution.
Bash
Criar os volumes:
docker volume create db-ha-data
docker volume create wp-ha-data
Use code with caution.
Bash
Iniciar os containers:
docker-compose up -d
Use code with caution.
Bash
Acesso aos Servidores
WordPress 1: Acesse http://localhost:81
WordPress 2: Acesse http://localhost:82
HAProxy: Acesse http://localhost:8080
Observações
Certifique-se de que o arquivo haproxy.cfg está presente no diretório raiz do projeto.
Você pode personalizar as configurações do docker-compose.yml e do haproxy.cfg de acordo com suas necessidades.
A página de estatísticas do HAProxy pode ser acessada via http://localhost:8080/haproxy_stats com o usuário e senha admin:admin.
Para parar os containers, execute docker-compose down.
Este projeto demonstra a implementação de um sistema de balanceamento de carga básico com HAProxy e Docker Compose. Você pode expandi-lo para incluir outras funcionalidades, como autenticação, SSL, monitoramento e logging mais avançados.
