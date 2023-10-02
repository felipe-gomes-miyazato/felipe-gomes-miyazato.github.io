---
layout: post
title: Pipelines de dados com monitoramento na AWS 
date: 2023-09-30 #10:45:00-0400
description: um exemplo não exaustivo de implementação de processo de ingestão de dados
tags: serverless dados monitoramento AWS
categories: pt-br
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
        {% include figure.html path="assets/img/Captura de tela 2023-10-01 105928.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Para deixar o exemplo menos exaustivo, utilizar <b>Python Shell script editor</b> para criar o job.
</div>

### Catalogar com crawler

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-10-01 202812.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Configurações.
</div>

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-10-01 205915.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Após rodar o crawler já é possível acessar a catalogação.
</div>

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-10-01 210105.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    E dar uma boa espiada na estrutura dos dados.
</div>

### Desenvolver ETL

Segue um exemplo minimalista de script de ETL em Python, que quebra 50% das vezes para ajudar a explorar as ferramentas de monitoramento na Parte 3.

{% highlight python linenos %}
import json
import boto3
import pandas as pd
import random

# 50% breaks
if random.random() < 0.5:
    raise Exception("Random exception")

# Initialize boto3 client
s3 = boto3.client('s3')

def read_from_s3(bucket, key):
    response = s3.get_object(Bucket=bucket, Key=key)
    content = response['Body'].read().decode('utf-8')
    return json.loads(content)

def write_to_s3(bucket, key, data):
    s3.put_object(Body=data, Bucket=bucket, Key=key)

def process_data(input_bucket_name, output_bucket_name, input_key, output_key):
    # Read JSON from S3
    data = read_from_s3(input_bucket_name, input_key)

    # Serialize the content from key "entries" in a dataframe
    df = pd.DataFrame(data['entries'])

    description = df.describe().to_csv()
    write_to_s3(output_bucket_name, output_key, description)

# Call the function with your bucket name and keys
process_data('felipe-gomes-miyazato-bucket', 'dlakeanalytics', 'public_apis.json', 'describe_public_apis.csv')
{% endhighlight %}

## Parte 3: Monitoramento

### CloudWatch Dashboards

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-10-01 215426.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Usar a ferramenta de dashboards nativa do CoudWatch para monitoria.
</div>

### Monitorar função lambda

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-10-01 215709.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Adicionar widget de tabela de logs, configurada com o grupo de logs da lambda.
</div>

### Monitorar erros nos jobs do Glue

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-10-01 220104.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Adicionar widget de tabela de logs, configurada com o grupo de logs de erros do Glue.
</div>
