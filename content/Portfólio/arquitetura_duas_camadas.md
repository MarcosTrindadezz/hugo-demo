---
title: "Criando uma arquitetura de duas camadas na AWS usando Terraform "
date: 2023-09-27T21:29:31-03:00
draft: false
tags: ['terraform', 'portifolio', 'aws']

cover:
  image: "/banner/arquitura duas camadas.webp" # image path/url
  alt: "<alt text>" # alt text
  caption: "<text>" # display caption under cover
  relative: false # when using page bundles set this to true
  hidden: false # only hide on current single page
---

## "Implantação uma arquitetura de duas camadas na AWS usando Terraform"
Confira o código completo aqui no [github](https://github.com/BrendoTrindade/aws-duas-camadas.git)

> ### Nesse projeto foram usado as seguintes tecnologias
>
> - AWS
> -  NLB
> -  Rds
> -  Mysql
> -  Vpc
> -  Subnet
> -  IGW (internet gateway)
> -  Availabillity Zone (AZ)
> -  Ec2
> -  Terraform



### Estrutura básica dos diretórios
```bash

├── terraform # Arquivos do terraform pra deploy da ec2, target group, vpc, subnet, router table, IGW, ALB
│   ├── elb.tf
│   ├── vpc_sub_ec2.tf
│   ├── variables.tf
├── terraform.tfstate
└── terraform.tfstate.backup

```


o provisionamento de infraestrutura na Amazon Web Services (AWS) usando Terraform, uma ferramenta de infraestrutura como código (IaC).Vamos analisar a estrutura de código Terraform e os diferentes arquivos envolvidos no processo.


### *variables.tf:* 

vpc_east1: Define o bloco CIDR da Virtual Private Cloud (VPC) na região leste da AWS.

```c++

variable "vpc_east1" {
  type = string
  default = "10.0.0.0/16"

}
```



#### subnet1_public e subnet2_public: Essas variáveis representam os blocos CIDR das sub-redes públicas em duas zonas de disponibilidade.

```c++
# variaveis de subnet Publicas
variable "subnet1_public" {
  description = "public subnet 1 cider block"
  type        = string
  default     = "10.0.1.0/24"
}

```
```python
variable "subnet2_public" {
  description = "public subnet 2 cidr block"
  type        = string
  default     = "10.0.2.0/24"
}
```

#### subnet1_private e subnet2_private: Essas variáveis representam os blocos CIDR das sub-redes privadas em duas zonas de disponibilidade.

```c++
# variaveis de subnet Privadas
variable "subnet1_private" {
  description = "private subnet1 cidr block"
  type        = string
  default     = "10.0.3.0/24"
}

variable "subnet2_private" {
  description = "private subnet2 cidr block"
  type        = string
  default     = "10.0.4.0/24"
}

```
#### aws_internet_gateway: Define o nome do Internet Gateway.

```c++
variable "aws_internet_gateway" {
  type = string
  default = "igw"
}

```

### *Arquivo: vpc_sub_ec2.tf:*

Este arquivo é responsável pela configuração da infraestrutura na AWS. Abaixo estão as principais seções e recursos definidos neste arquivo.

Configuração do Provedor

Nesta seção, configuramos o provedor AWS para a região leste dos EUA (us-east-1).

```c++

provider "aws" {
  region = "us-east-1"
}

```
#### Recurso VPC:

Aqui, definimos a VPC na região leste da AWS com os parâmetros especificados.

```c
resource "aws_vpc" "vpc_east1" {
  cidr_block           = var.vpc_east1
  enable_dns_support   = true
  enable_dns_hostnames = true
}

```

#### Internet Gateway: 

Este recurso cria um Internet Gateway associado à VPC e atribui um nome a ele.

```c++
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.vpc_east1.id
  tags = {
    Name = "igw_east1"
  }
}

```

#### Subnets Públicas e Privadas:

Recursos para sub-redes públicas e privadas são definidos em seções separadas, com configurações semelhantes.

Subnet Pública 1: Aqui, criamos uma sub-rede pública chamada "subnet1_public" na zona de disponibilidade "us-east-1a" com a capacidade de mapear endereços IP públicos quando as instâncias forem lançadas.

```c++
resource "aws_subnet" "subnet1_public" {
  tags = {
    Name = "subnet1_public"
  }
  vpc_id            = aws_vpc.vpc_east1.id
  cidr_block        = var.subnet1_public
  availability_zone = "us-east-1a"
  map_public_ip_on_launch = true
}
```

#### Subnet Pública 2: 

segunda sub-rede pública chamada "subnet2_public" na zona de disponibilidade "us-east-1b" com as mesmas configurações de mapeamento de endereços IP públicos.

```c++
resource "aws_subnet" "subnet2_public" {
  tags = {
    Name = "subnet2_public"
  }
  vpc_id            = aws_vpc.vpc_east1.id
  cidr_block        = var.subnet2_public
  availability_zone = "us-east-1b"
  map_public_ip_on_launch = true
}

```

###  Subnets Privadas para instacias RDS

Subnet Privada 1: A seguir criamos uma sub-rede privada chamada "subnet1_private" na zona de disponibilidade "us-east-1a" com a capacidade de não mapear endereços IP públicos quando as instâncias forem lançadas.


```c++
resource "aws_subnet" "subnet1_private" {
  tags = {
    Name = "subnet1_private"
  }
  vpc_id            = aws_vpc.vpc_east1.id
  cidr_block        = var.subnet1_private
  availability_zone = "us-east-1a"
  map_public_ip_on_launch = false
}
```

#### Subnet Privada 2: 

Da mesma forma, aqui criamos uma segunda sub-rede privada chamada "subnet2_private" na zona de disponibilidade "us-east-1b" com configurações semelhantes de não mapear endereços IP públicos.


```c++
resource "aws_subnet" "subnet2_private" {
  tags = {
    Name = "subnet2_private"
  }
  vpc_id            = aws_vpc.vpc_east1.id
  cidr_block        = var.subnet2_private
  availability_zone = "us-east-1b"
  map_public_ip_on_launch = false
}

```

#### Recurso de Instância EC2 Pública:

 Este recurso cria uma instância EC2 na primeira sub-rede pública com as configurações especificadas.

```c++
resource "aws_instance" "EC2-1" {
  ami                         = "ami-0df435f331839b2d6"
  instance_type               = "t2.micro"
  subnet_id                   = aws_subnet.subnet1_public.id
  associate_public_ip_address = true
  count                       = 1
}

```

#### Recurso de Instância EC2 Pública Adicional: 

Este recurso cria uma segunda instância EC2 na segunda sub-rede pública.

#### Crie uma instance EC2 em uma publica subnet2

```c++
resource "aws_instance" "EC2_2" {
  ami                         = "ami-0df435f331839b2d6"
  instance_type               = "t2.micro"
  subnet_id                   = aws_subnet.subnet2_public.id
  associate_public_ip_address = true
  count                       = 1
}

```

#### Recurso do Banco de Dados RDS: 

Este recurso define uma instância do banco de dados RDS MySQL com as configurações especificadas, incluindo alocação de armazenamento, engine, versão do engine, nome do banco de dados, nome de usuário, senha e outros parâmetros.


```c++
resource "aws_db_instance" "instance_rds" {
  allocated_storage    = 10
  engine               = "mysql"
  engine_version       = "5.7"
  instance_class       = "db.t2.micro"
  db_name              = "teste_db"
  username             = "admin"
  password             = "15468105540"
  parameter_group_name = "default.mysql5.7"
  skip_final_snapshot  = true


  db_subnet_group_name = aws_db_subnet_group.my_db_subnet_group.name
  }

  resource "aws_db_subnet_group" "my_db_subnet_group" {
  name       = "my-db-subnet-group"
  description = "My DB subnet group"
  subnet_ids = [aws_subnet.subnet1_private.id, aws_subnet.subnet2_private.id] 
}
```

### *Arquivo elb.tf*

Este arquivo é responsável pela configuração de um Load Balancer (ELB).

#### Recurso AWS ELB: 

```c++
resource "aws_lb" "ALB" {
  name               = "test-lb"
  internal           = false
  load_balancer_type = "network"
  subnets            =  [aws_subnet.subnet1_public.id , aws_subnet.subnet2_public.id]

  enable_deletion_protection = false

  tags = {
    Environment = "production"
  }
}

```

Este recurso cria um Load Balancer AWS com configurações específicas, como nome, tipo e sub-redes nas quais o Load Balancer opera.


exploramos os arquivos e códigos usados para provisionar infraestrutura na AWS usando Terraform. Os arquivos variables.tf, vpc_sub_ec2.tf e elb.tf descrevem as variáveis, os recursos de infraestrutura e as configurações associadas a eles. Isso permite que você automatize a criação e gerenciamento de recursos na AWS de maneira consistente e controlada.

Com Terraform, é possível criar infraestruturas complexas e definir todos os detalhes usando código, tornando o processo de implantação e gerenciamento mais eficiente e escalável.

## Resumo da Arquitetura

Nossa arquitetura de duas camadas conterá os seguintes componentes:

Implantar uma VPC com CIDR 10.0.0.0/16.

Dentro da VPC teremos 2 sub-redes públicas com CIDR 10.0.1.0/24 e 10.0.2.0/24. Cada sub-rede pública estará em uma zona de disponibilidade diferente para alta disponibilidade.

Crie 2 sub-redes privadas com CIDR '10.0.3.0/24' e '10.0.4.0/24' e cada uma estará em uma zona de disponibilidade diferente.

Instância RDS MySQL (micro).

Um balanceador de carga que direcionará o tráfego para as sub-redes públicas.

Implante 1 instância EC2 t2.micro em cada sub-rede pública.

IGW anexado a VPC na região east-1.