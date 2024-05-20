## Aula 46: Balanceamento de Carga com HAProxy e WordPress

Este arquivo Markdown descreve a configuração de um ambiente Docker com balanceamento de carga utilizando HAProxy para dois servidores WordPress.

### Arquivos

Este repositório contém os seguintes arquivos:

- **docker-compose.yml**: Define a configuração dos serviços Docker.
- **haproxy.cfg**: Arquivo de configuração do HAProxy.

### docker-compose.yml

```yaml
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
```

**Explicação:**

- **mysql**: Define o serviço do banco de dados MySQL.
    - **image**: Utiliza a imagem `mysql:5.5`.
    - **restart**: Reinicia o container automaticamente em caso de falha.
    - **container_name**: Nome do container.
    - **volumes**: Mapeia o volume `db-ha-data` para o diretório de dados do MySQL.
    - **networks**: Conecta o container à rede `backend-web`.
    - **environment**: Define as variáveis de ambiente do MySQL.
- **wordpress1** e **wordpress2**: Definem os serviços dos servidores WordPress.
    - **depends_on**: Garante que o MySQL esteja iniciado antes dos servidores WordPress.
    - **networks**: Conecta os containers à rede `backend-web`.
    - **image**: Utiliza a imagem `wordpress:4.7-php5.6`.
    - **restart**: Reinicia o container automaticamente em caso de falha.
    - **container_name**: Nome do container.
    - **volumes**: Mapeia o volume `wp-ha-data` para o diretório raiz do WordPress.
    - **ports**: Exporta a porta 80 do container para as portas 81 e 82 do host, respectivamente.
    - **environment**: Define as variáveis de ambiente do WordPress.
- **haproxy**: Define o serviço do balanceador de carga HAProxy.
    - **image**: Utiliza a imagem `haproxy:1.6`.
    - **restart**: Reinicia o container automaticamente em caso de falha.
    - **networks**: Conecta o container às redes `frontend-web` e `backend-web`.
    - **ports**: Exporta a porta 80 do container para a porta 8080 do host.
    - **volumes**: Mapeia o arquivo `haproxy.cfg` para o diretório de configuração do HAProxy.
- **volumes**: Define os volumes compartilhados para os containers.
- **networks**: Define as redes utilizadas pelos containers.

### haproxy.cfg

```
defaults
        log     global
        mode    http
        timeout connect 50000
        timeout client  50000
        timeout server  50000

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
```

**Explicação:**

- **defaults**: Define configurações padrão para o HAProxy.
    - **log global**: Habilita o log global.
    - **mode http**: Define o modo de operação como HTTP.
    - **timeout connect**: Define o tempo limite para a conexão.
    - **timeout client**: Define o tempo limite para o cliente.
    - **timeout server**: Define o tempo limite para o servidor.
- **frontend main**: Define o frontend principal do HAProxy.
    - **bind *:80**: Vincula a porta 80 do host ao frontend.
    - **default_backend wordpress**: Define o backend padrão como `wordpress`.
- **backend wordpress**: Define o backend `wordpress`.
    - **balance roundrobin**: Define o balanceamento de carga como `roundrobin`.
    - **mode http**: Define o modo de operação como HTTP.
    - **server wp1 wordpress1:80**: Define o servidor `wp1` com endereço `wordpress1:80`.
    - **server wp2 wordpress2:80**: Define o servidor `wp2` com endereço `wordpress2:80`.
    - **stats uri /haproxy_stats**: Define a URL para acessar as estatísticas do HAProxy.
    - **stats auth admin:admin**: Define a autenticação para acessar as estatísticas.
    - **stats admin if TRUE**: Habilita a administração do HAProxy.

### Usando o ambiente

1. **Criar o ambiente:**

   ```bash
   docker-compose up -d
   ```

2. **Acessar os servidores WordPress:**

   - **wordpress1**: http://localhost:81
   - **wordpress2**: http://localhost:82

3. **Acessar o HAProxy:**

   - http://localhost:8080

4. **Acessar as estatísticas do HAProxy:**

   - http://localhost:8080/haproxy_stats

### Considerações

- Este ambiente é uma demonstração simples de balanceamento de carga com HAProxy e WordPress.
- Para um ambiente de produção, é recomendado utilizar uma configuração mais robusta, incluindo monitoramento, redundância e segurança.
- É importante entender o funcionamento do HAProxy e do Docker antes de implementar este ambiente em produção.

### Recursos

- [Documentação do HAProxy](https://cbonte.github.io/haproxy-dconv/configuration-1.5/index.html)
- [Documentação do Docker](https://docs.docker.com/)

### Próximos Passos

- Implementar monitoramento e redundância.
- Configurar segurança para o HAProxy e os servidores WordPress.
- Implementar um esquema de persistência de sessão para o HAProxy.

Espero que este lab tenha sido útil para você.
