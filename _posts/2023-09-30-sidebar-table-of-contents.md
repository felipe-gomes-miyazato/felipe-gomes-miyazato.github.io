---
layout: post
title: Pipelines de dados com monitoramento na AWS 
date: 2023-09-30 #10:45:00-0400
description: um exemplo não exaustivo de implementação de processo de ingestão de dados
tags: serverless dados monitoramento AWS
categories: pt-br
giscus_comments: true
related_posts: false
toc:
  sidebar: left
---
Este post mostra como construir e viabilizar um bom gerenciamento de pipeline de dados utilizando serviços serverless da AWS.

Requisitos:

- Conta AWS

## Parte 1: Coletando dados

Demostração de contrução de uma função AWS Lambda que transfere dados de uma API pública para um bucket S3.

### Provisionando Lambda

Esta Lambda age apenas como uma proxy, para disponibilizar os dados em um bucket S3. Para provisionar bastam as configurações padrão, neste caso com Python runtime.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-10-01 200328.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

### Código e configuração da Lambda

{% highlight python linenos %}
import os
import boto3
import requests
from botocore.exceptions import NoCredentialsError

def lambda_handler(event, context):
    # URL da API pública "Public APIs"
    url = "https://api.publicapis.org/entries"

    # Fazer a solicitação GET
    response = requests.get(url)

    # Verificar se a solicitação foi bem-sucedida
    if response.status_code == 200:
        # Salvar os dados em um arquivo JSON
        with open("/tmp/public_apis.json", "w") as f:
            f.write(response.text)
        
        # Obter o nome do bucket do objeto do evento
        bucket_name = event['bucket_name']

        # Criar um cliente S3
        s3 = boto3.client('s3')

        # Fazer upload do arquivo para S3
        try:
            s3.upload_file("/tmp/public_apis.json", bucket_name, "public_apis.json")
            return {
                'statusCode': 200,
                'body': 'Conjunto de dados carregado com sucesso no bucket S3'
            }
        except FileNotFoundError:
            return {
                'statusCode': 404,
                'body': 'O arquivo não foi encontrado'
            }
        except NoCredentialsError:
            return {
                'statusCode': 401,
                'body': 'Credenciais não disponíveis'
            }
    else:
        return {
            'statusCode': response.status_code,
            'body': 'Solicitação falhou'
        }
{% endhighlight %}

### Evento gatilho

Configurar o evento de teste com o JSON:

{% highlight json linenos %}

{
  "bucket_name": "seu-nome-bucket"
}

{% endhighlight %}

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-09-30 143901.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Para usar o pacote requests, há uma <a href="https://github.com/keithrozario/Klayers/tree/master">layer pública disponível</a>.
</div>

### Permissionamento da Lambda

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-10-01 201151.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Para autorizar a lambda a inputar objetos no bucket é necessário acessar a configuração da role.
</div>

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-10-01 201325.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Em seguida, acessar a configuração <mark>Add permissions</mark> > <mark>Attach policies</mark>.
</div>

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-10-01 201511.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Neste case, será adicionada a política <b>AmazonS3FullAccess</b>.
</div>

### Criar um bucket no Amazon S3

<https://s3.console.aws.amazon.com/s3/get-started>

### Configurar o bucket

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-09-30 212902.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Para este case, as configurações padrão funcionam.
</div>

### Permissionamento do bucket

Para configurar permissões de acesso para que um bucket do Amazon S3 seja público (somente leitura), pode-se seguir estas etapas:

1. Abrir o console do Amazon S3.
2. No painel de navegação do Buckets, escolher o nome do seu bucket.
3. Escolher a guia "Permissões".
4. Retirar o bloqueio de acesso público
5. Em "Gerenciamento de acesso ao bucket", escolher "Editar".
6. Em "Política de bucket", inserir a seguinte política, substituindo `'your_bucket_name'`:

    {% highlight json linenos %}
    {
      "Version":"2012-10-17",
      "Statement":[
        {
          "Sid":"PublicReadGetObject",
          "Effect":"Allow",
          "Principal": "*",
          "Action":["s3:GetObject"],
          "Resource":["arn:aws:s3:::your_bucket_name/*"]
        }
      ]
    }{% endhighlight %}

Essa política permite que qualquer pessoa leia os objetos no bucket.

## Parte 2: Pipeline de Dados

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-10-01 111047.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Para esta parte é necessário configurar em <mark>Set up roles and users</mark>.
</div>

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-10-01 111330.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Marcar <b>Read and write</b> em <mark><b>Data access permissions</b></mark> e as outras configurações deixar no padrão.
</div>

### Criar bucket de dados processados

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-10-01 105420.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

### Criar um job no AWS Glue

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-09-30 212902.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Para deixar o exemplo menos exaustivo, utilizar <b>Python Shell script editor</b> para criar o job.
</div>

### Catalogar com crawler

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-10-01 120531.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

## Parte 3: Monitoramento
