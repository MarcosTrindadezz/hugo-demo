---
title: "Dockerize a Java Web Application using docker-compose"
date: 2024-12-27T21:29:31-03:00
draft: false
tags: ['terraform', 'portifolio', 'aws', 'Java', 'Docker'  ]

cover:
  image: "/banner/aws-docker-architecture-complete.png" # image path/url
  alt: "<alt text>" # alt text
  caption: "<text>" # display caption under cover
  relative: false # when using page bundles set this to true
  hidden: false # only hide on current single page
---


# Implementação de uma Aplicação Java Multi-Container com Docker

Este artigo documenta a implementação de uma aplicação Java de login usando Docker Compose, com uma arquitetura multi-container incluindo Nginx, Tomcat e MySQL.

## Visão Geral do Projeto

### Objetivos
- Implementar uma aplicação Java de login
- Utilizar arquitetura multi-container com Docker Compose
- Configurar proxy reverso com Nginx
- Implementar persistência de dados com MySQL
- Automatizar o deployment com Terraform na AWS

### Arquitetura
- **Frontend/Proxy**: Nginx na porta 80
- **Aplicação**: Tomcat na porta 8080
- **Banco de Dados**: MySQL na porta 3306

## Implementação

### 1. Estrutura do Projeto
```
Projeto-docker/
├── java-login-app/     # Aplicação Java
├── mysql/              # Configurações MySQL
│   ├── init.sql       # Script de inicialização
├── nginx/              # Configurações Nginx
│   ├── nginx.conf     # Configuração do proxy
├── tomcat/             # Configurações Tomcat
│   ├── Dockerfile     # Customização Tomcat
│   ├── tomcat-users.xml
│   └── context.xml
└── docker-compose.yml
```

### 2. Configuração dos Containers

#### MySQL Container
```yaml
mysql:
  image: mysql:8.0
  environment:
    MYSQL_DATABASE: logindb
    MYSQL_USER: loginuser
    MYSQL_PASSWORD: loginpass
    MYSQL_ROOT_PASSWORD: rootpass
  ports:
    - "3306:3306"
```

#### Tomcat Container
```dockerfile
FROM tomcat:9.0.83-jdk8-corretto

# Install additional packages
RUN yum update -y && \
    yum install -y mysql telnet && \
    yum clean all

# Remover aplicações padrão
RUN rm -rf /usr/local/tomcat/webapps/*

# Criar diretório ROOT
RUN mkdir -p /usr/local/tomcat/webapps/ROOT

# Copiar o arquivo WAR
COPY login.war /usr/local/tomcat/webapps/ROOT.war

# Extrair o WAR manualmente
RUN cd /usr/local/tomcat/webapps/ROOT && jar xf ../ROOT.war && rm ../ROOT.war

# Configurar permissões
RUN chmod +x /usr/local/tomcat/bin/*.sh

# Copy configuration files
COPY tomcat-users.xml /usr/local/tomcat/conf/tomcat-users.xml
COPY context.xml /usr/local/tomcat/webapps/manager/META-INF/context.xml
COPY context.xml /usr/local/tomcat/webapps/host-manager/META-INF/context.xml
```

#### Nginx Container
```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://tomcat:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 3. Schema do Banco de Dados
```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) UNIQUE NOT NULL,
    password VARCHAR(100) NOT NULL,
    email VARCHAR(100)
);

CREATE TABLE Employee (
    id INT PRIMARY KEY AUTO_INCREMENT,
    firstName VARCHAR(50),
    lastName VARCHAR(50),
    email VARCHAR(100)
);
```

### 4. Deployment na AWS

#### Infraestrutura (Terraform)
- VPC com subnets públicas e privadas
- Security Groups para tráfego web
- Instância EC2 t2.micro
- Elastic IP para acesso público

#### Configuração da Instância EC2
```hcl
resource "aws_instance" "app_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  vpc_security_group_ids = [aws_security_group.allow_web.id]
  subnet_id              = aws_subnet.public.id
  
  tags = {
    Name = "java-login-app"
  }
}
```

### 5. Validação do Ambiente

#### Verificações de Saúde
1. Status dos Containers
```bash
docker-compose ps
```

2. Logs da Aplicação
```bash
docker-compose logs tomcat
```

3. Conectividade do Banco
```bash
docker exec tomcat mysql -h mysql -u loginuser -p
```

#### Testes Funcionais
1. Acesso à aplicação via Nginx (porta 80)
2. Registro de novo usuário
3. Login com credenciais
4. Persistência dos dados no MySQL

### 6. Comandos Úteis

#### Deployment
```bash
# Iniciar os containers
docker-compose up -d

# Reconstruir containers
docker-compose build
docker-compose up -d

# Verificar logs
docker-compose logs -f

# Parar containers
docker-compose down
```

#### Debugging
```bash
# Acessar container Tomcat
docker exec -it java-login-app-tomcat-1 bash

# Verificar logs Tomcat
docker-compose logs tomcat | grep -i error

# Testar conexão MySQL
docker exec java-login-app-tomcat-1 mysql -h mysql -uloginuser -ploginpass -e "USE logindb; SHOW TABLES;"
```

## Resultados e Conclusões

### Objetivos Alcançados
- ✅ Deployment multi-container funcionando
- ✅ Proxy reverso Nginx configurado
- ✅ Aplicação Java rodando no Tomcat
- ✅ Banco de dados MySQL configurado e acessível
- ✅ Comunicação entre containers estabelecida

### Lições Aprendidas
1. Importância da ordem de inicialização dos containers
2. Necessidade de wait-for-it scripts para dependências
3. Configuração correta de networking entre containers
4. Gestão de logs e debugging em ambiente containerizado

### Melhorias Futuras
1. Implementar CI/CD pipeline
2. Adicionar monitoramento com Prometheus/Grafana
3. Implementar backup automatizado do banco de dados
4. Adicionar HTTPS com Let's Encrypt
5. Implementar cache com Redis

## Referências
- Docker Documentation
- Nginx Documentation
- Tomcat Documentation
- MySQL Documentation
- AWS Documentation
- Terraform Documentation
