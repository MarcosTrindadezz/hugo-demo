---
title: "Criando um Cluster EKS na AWS usando Pulumi "
date: 2023-09-27T21:29:31-03:00
draft: false
tags: ['Pulumi', 'portifolio', 'aws']

cover:
  image: "/banner/Criando um Cluster EKS na AWS usando Pulumi.png" # image path/url
  alt: "<alt text>" # alt text
  caption: "<text>" # display caption under cover
  relative: false # when using page bundles set this to true
  hidden: false # only hide on current single page
---

## "Criando um Cluster EKS na AWS usando Pulumi e com deployament de Aplicação"


> ### Nesse projeto foram usado as seguintes tecnologias
>
> - AWS
> -  NLB
> -  Pulumi
> -  Vpc
> -  Subnet
> -  IGW (internet gateway)
> -  Availabillity Zone (AZ)
> -  AWS CLI (Conta na AWS com credenciais configuradas localmente)
> -  Ec2
> -  Cluster EKS
> -  Docker
> -  Image 
> -  Container

---



### *1. Definição da VPC e Cluster EKS* 

A seguir o código para definir a Virtual Private Cloud (VPC) e o cluster EKS. 

```c++

// Importando os pacotes necessários do Pulumi e AWS
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
import * as eks from "@pulumi/eks";
import * as awsx from "@pulumi/awsx";
import * as k8s from "@pulumi/kubernetes";

// Criando a Virtual Private Cloud (VPC) usando awsx.ec2.Vpc
const vpc = new awsx.ec2.Vpc("vpc-pulumi", {
    numberOfAvailabilityZones: 2, // Definindo o número de zonas de disponibilidade
    tags: { Name: "pulumi-vpc" }, // Adicionando tags para identificação
});

// Criando um cluster Amazon Elastic Kubernetes Service (EKS) usando eks.Cluster
const cluster = new eks.Cluster("cluster-pulumi", {
    vpcId: vpc.vpcId, // Especificando o ID da VPC
    subnetIds: vpc.publicSubnetIds, // Especificando os IDs das sub-redes públicas
    tags: { Name: "pulumi-cluster" }, // Adicionando tags para identificação
    desiredCapacity: 2, // Definindo a capacidade desejada do cluster
    minSize: 2, // Definindo o tamanho mínimo do cluster
    maxSize: 2, // Definindo o tamanho máximo do cluster
    storageClasses: "gp2", // Definindo as classes de armazenamento
    instanceType: "t2.micro", // Especificando o tipo de instância
});

// Exportando algumas informações úteis do cluster
export const clusterName = cluster.core.tags;
export const clusterEndpoint = cluster.core.endpoint;
export const kubeconfig = cluster.kubeconfig;

// Definindo o nome e as labels do aplicativo
const name = "fcapp";
const appLabels = { appClass: name };

// Criando um Deployment no Kubernetes para o aplicativo
const deployment = new k8s.apps.v1.Deployment(name, {
    metadata: {
        labels: appLabels, // Adicionando labels ao Deployment
    },
    spec: {
        selector: { matchLabels: appLabels }, // Especificando o seletor de labels
        replicas: 1, // Definindo o número de réplicas
        template: {
            metadata: {
                labels: appLabels, 
            },
            spec: {
                containers: [
                    {
                        name: name,
                        image: "brendofut/eks-brendo-eks", // Especificando a imagem do contêiner
                        ports: [{ containerPort: 8080 }], // Expondo a porta 8080
                    },
                ],
            },
        },
    },
});

// Criando um Service no Kubernetes para expor o aplicativo
const service = new k8s.core.v1.Service(name, {
    metadata: {
        labels: appLabels, // Adicionando labels ao Service
    },
    spec: {
        type: "LoadBalancer", // Definindo o tipo de Service
        ports: [{ port: 80, targetPort: 8080 }], // Mapeando a porta 80 para a porta 8080
        selector: appLabels, // Especificando o seletor de labels
    },
});

// Exportando o hostname do serviço para acessar o aplicativo
export const serviceHostName = service.status.loadBalancer.ingress[0].hostname;

```

#### Este código implanta um aplicativo de exemplo com um contêiner que expõe a porta 8080 e um serviço Kubernetes para acessar o aplicativo.


### *2. Construção e Implantação*

#### *Após salvar o código, execute os seguintes comandos na linha de comando:*


```c++

pulumi up

```

### 3. Trabalhando com Dockerfile

Este Dockerfile é utilizado para construir uma imagem Docker baseada na imagem oficial do Node.js versão 14. Ele é usado para criar um ambiente de execução para uma aplicação Node.js.

## Dockerfile:

1. **FROM node:14**:
   - Esta instrução define a imagem base como a imagem oficial do Node.js na versão 14. Essa imagem será utilizada como base para construir a imagem personalizada.

2. **WORKDIR /usr/src/app**:
   - Define o diretório de trabalho dentro do contêiner como `/usr/src/app`, onde os arquivos da aplicação serão copiados e onde os comandos subsequentes serão executados.

3. **COPY package*.json ./**:
   - Copia os arquivos `package.json` e `package-lock.json` (se existirem) do diretório local para o diretório de trabalho no contêiner. Isso permite que as dependências sejam instaladas separadamente, evitando a reinstalação sempre que houver uma alteração nos arquivos do projeto.

4. **RUN npm install**:
   - Executa o comando `npm install` para instalar as dependências da aplicação. Isso é feito após copiar os arquivos `package.json` para garantir que as dependências sejam instaladas apenas quando necessário e que o cache do Docker seja aproveitado sempre que possível.

5. **COPY . .**:
   - Copia todos os arquivos e diretórios do diretório local para o diretório de trabalho no contêiner. Isso inclui todos os arquivos necessários para a execução da aplicação, como arquivos de código-fonte, arquivos de configuração, etc.

6. **EXPOSE 8080**:
   - Expõe a porta 8080 do contêiner, permitindo que a aplicação seja acessível a partir de outras partes da rede, se necessário.

7. **CMD ["node", "index.js"]**:
   - Define o comando padrão a ser executado quando o contêiner for iniciado. Neste caso, o comando `node index.js` é usado para iniciar a aplicação Node.js.

## Utilização:

### Image oficial Node.js 14
```c++
FROM node:14
```
```c++
WORKDIR /usr/src/app
```
```c++
COPY package*.json ./
```

### Dependencies
```c++
RUN npm install
```

```c++
COPY . .
```

### Port 8080

```c++
EXPOSE 8080
```

```c++
CMD ["node", "index.js"]
```

### Comandos utilizados: 

```c++


docker build -t eks-brendo-eks .  

docker run -p 8080:8080 eks-brendo-eks

```

### Criando arquivo Index

```c++


const http = require('http');

const hostname = 'localhost';
const port = 8080;

const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/plain');
  res.end('Hello Brendo, estou usando cluster EKS\n');
});

server.listen(port, hostname, () => {
  console.log(`Server running at http://${hostname}:${port}/`);
});



```

Em resumo, exploramos como criar um cluster EKS na AWS usando o Pulumi. Começamos definindo a infraestrutura como código, configurando o cluster EKS e construindo uma imagem Docker para uma aplicação Node.js. Em seguida, implantamos essa aplicação no cluster EKS e validamos seu funcionamento. Ao seguir este guia, os desenvolvedores podem aprender a integrar efetivamente o Pulumi com o EKS para criar e gerenciar clusters Kubernetes na AWS.

