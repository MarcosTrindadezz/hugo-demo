---
title: "Implantação Azul/Verde na AWS com Terraform: Grupos-alvo ponderados "
date: 2023-09-27T21:29:31-03:00
draft: false
tags: ['terraform', 'wordpress', 'portifolio', 'aws']

cover:
  image: "/banner/illustration-2.png" # image path/url
  alt: "<alt text>" # alt text
  caption: "<text>" # display caption under cover
  relative: false # when using page bundles set this to true
  hidden: false # only hide on current single page
---

## "Implantação Azul/Verde na AWS: Demonstração de Infraestrutura com Terraform"
Confira o código completo aqui no [github](https://github.com/BrendoTrindade/aws-verde-azul.git)

> ### Nesse projeto foram usado as seguintes tecnologias
>
> - AWS
> -  ALB
> -  target Group
> -  Vpc
> -  Subnet
> -  Router Table
> -  IGW (internet gateway)
> -  Availabillity Zone (AZ)
> -  Terraform



### Estrutura básica dos diretórios
```bash

├── terraform # Arquivos do terraform pra deploy da ec2, target group, vpc, subnet, router table, IGW, ALB
│   ├── alb_target.tf
│   ├── ec2_azul_verde.tf
│   ├── igw.tf
│   ├── vpc_subnet.tf
├── terraform.tfstate
└── terraform.tfstate.backup

```
## Terraform


```c++
provider "aws" {
  region = "us-east-1" 
}
```

Neste bloco, estamos configurando a região da AWS onde nossa infraestrutura será criada.

```c++

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  enable_dns_support = true
  enable_dns_hostnames = true
}

```


Aqui, estamos criando uma Virtual Private Cloud (VPC), que é uma rede privada virtual na AWS. Definimos um bloco de endereço IP para a VPC (10.0.0.0/16) e ativamos o suporte a DNS e a capacidade de hospedar nomes de host DNS na VPC. Isso permite que as instâncias dentro da VPC se comuniquem usando nomes de domínio.


```go
resource "aws_subnet" "private_subnet" {
  count = 2
  vpc_id = aws_vpc.main.id
  cidr_block = element(["10.0.0.0/24", "10.0.1.0/24"], count.index)
  availability_zone = element(["us-east-1a", "us-east-1b"], count.index)
  map_public_ip_on_launch = false
}
```

Aqui, estamos criando duas subnets privadas dentro da VPC, uma em cada zona de disponibilidade da região "us-east-1". Cada subnet tem seu próprio bloco de endereço IP e está configurada para não atribuir endereços IP públicos às instâncias que forem lançadas nela. Isso mantém as instâncias isoladas da Internet.



```c++
resource "aws_route_table" "private_route_table" {
  vpc_id = aws_vpc.main.id
}

```


Este bloco cria uma tabela de roteamento para as subnets privadas na VPC. Isso permite que configuremos as rotas para o tráfego dentro da VPC.


```c#
resource "aws_internet_gateway" "internet_gateway" {
  vpc_id = aws_vpc.main.id
}

```


Neste bloco, estamos criando uma Internet Gateway (IGW) e a associando à VPC que criamos anteriormente. A IGW permite que as instâncias na VPC acessem a Internet.

```c++
resource "aws_route" "internet_route" {
  route_table_id         = aws_vpc.main.main_route_table_id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id              = aws_internet_gateway.internet_gateway.id
}
```

Aqui, estamos criando uma rota que encaminha todo o tráfego com destino à "0.0.0.0/0" (ou seja, a Internet) para a Internet Gateway que associamos à VPC. Isso permite que as instâncias na VPC acessem a Internet.


```c++
resource "aws_instance" "blue" {
  count = 2
  ami           = "ami-041feb57c611358bd" 
  instance_type = "t2.micro" 
  key_name      = "AWS" 
  subnet_id     = aws_subnet.private_subnet[count.index].id
}
```


Neste bloco, estamos criando instâncias EC2 no ambiente "azul". Criamos duas instâncias usando uma imagem AMI específica (uma imagem de máquina virtual), um tipo de instância "t2.micro", uma chave SSH chamada "AWS" e as lançamos nas subnets privadas criadas anteriormente.


```c++
resource "aws_instance" "green" {
  count = 2
  ami           = "ami-041feb57c611358bd" 
  instance_type = "t2.micro" 
  key_name      = "AWS" 
  subnet_id     = aws_subnet.private_subnet[count.index].id
}

```


Este bloco é semelhante ao anterior, mas cria instâncias EC2 no ambiente "verde". Também usamos a mesma imagem AMI, tipo de instância e chave SSH.


```c++
resource "aws_lb" "alb" {
  name               = "BG-alb"
  internal           = false
  load_balancer_type = "application"
  subnets            = aws_subnet.private_subnet[*].id

  enable_deletion_protection = false
}
```

Neste bloco, estamos criando um Application Load Balancer (ALB) chamado "BG-alb". O ALB é um serviço que distribui o tráfego entre as instâncias EC2 nos ambientes "azul" e "verde". O ALB é configurado para ser público (não interno) e para rotear o tráfego para as subnets privadas que criamos anteriormente.


```c++
resource "aws_lb_listener" "blue" {
  load_balancer_arn = aws_lb.alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "fixed-response"
    
    fixed_response {
      content_type = "text/plain"
      status_code  = "200"
    }
  }
}

```

Este bloco configura um listener no ALB para o ambiente "azul". O listener escuta na porta 80 e direciona o tráfego para uma resposta fixa que retorna "Blue Environment" com um código de status 200.


```c++
resource "aws_lb_listener" "green" {
  load_balancer_arn = aws_lb.alb.arn
  port              = 81
  protocol          = "HTTP"

  default_action {
    type = "fixed-response"
    
    fixed_response {
      content_type = "text/plain"
      status_code  = "200"
    }
  }
}

```

Este bloco configura um listener no ALB para o ambiente "verde". O listener escuta na porta 81 e direciona o tráfego para uma resposta fixa que retorna "Green Environment" com um código de status 200.


Em resumo, esses arquivos Terraform configuram uma infraestrutura básica na AWS com uma VPC, subnets privadas, instâncias EC2 em dois ambientes e um Application Load Balancer (ALB) para rotear o tráfego entre esses ambientes. Isso é uma simplificação e pode ser personalizado conforme necessário para a sua aplicação.




Agora é só testar, até a Proxima :D.
