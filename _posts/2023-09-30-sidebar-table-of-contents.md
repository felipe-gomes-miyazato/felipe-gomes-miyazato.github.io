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
- Conta kaggle

## Parte 1: Coletando dados

Demostração de contrução de uma função AWS Lambda que transfere dados do kaggle para um bucket S3.

### Provisionando Lambda

Esta Lambda age apenas como uma proxy, para disponibilizar os dados em um bucket S3. Para provisionar bastam as configurações padrão, neste caso com Python runtime.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-09-30 114229.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

### Código e configuração da Lambda

{% highlight python linenos %}

import os
import requests
import boto3
from botocore.exceptions import NoCredentialsError

def lambda_handler(event, context):
    # Get the Kaggle dataset URL and bucket name from the event object
    kaggle_dataset_url = event['kaggle_dataset_url']
    bucket_name = event['bucket_name']

    # Get the Kaggle username and key from environment variables
    kaggle_info = {
        'username': os.environ.get('username'),
        'key': os.environ.get('key')
    }

    # Make the GET request
    response = requests.get(kaggle_dataset_url, auth=(kaggle_info['username'], kaggle_info['key']))

    # Check if the request was successful
    if response.status_code == 200:
        # Write the dataset to a file
        with open('/tmp/dataset.zip', 'wb') as f:
            f.write(response.content)
        
        # Create an S3 client
        s3 = boto3.client('s3')

        # Specify the file name
        file_name = 'dataset.zip'

        # Upload the file to S3
        try:
            s3.upload_file('/tmp/' + file_name, bucket_name, file_name)
            return {
                'statusCode': 200,
                'body': 'Successfully uploaded dataset to S3 bucket'
            }
        except FileNotFoundError:
            return {
                'statusCode': 404,
                'body': 'The file was not found'
            }
        except NoCredentialsError:
            return {
                'statusCode': 401,
                'body': 'Credentials not available'
            }
    else:
        return {
            'statusCode': response.status_code,
            'body': 'Request to Kaggle API failed'
        }

{% endhighlight %}

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-09-30 122908.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Necessário o preenchimento das variáveis de ambiente da lambda com o conteúdo de <code>kaggle.json</code>. Instruções detalhadas disponíveis na documentação da API do kaggle.
</div>

### Evento gatilho

Configure the test JSON event like:

{% highlight json linenos %}

{
  "kaggle_dataset_url": "<https://www.kaggle.com/datasets/aliceadativa/queimadas-brasil-2020/download?datasetVersionNumber=2>",
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
        {% include figure.html path="assets/img/Captura de tela 2023-09-30 144938.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Para autorizar a lambda a inputar objetos no bucket é necessário acessar a configuração da role.
</div>

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-09-30 145253.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Em seguida, acessar a configuração <mark>Add permissions</mark> > <mark>Attach policies</mark>.
</div>

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-09-30 201454.png" class="img-fluid rounded z-depth-1" %}
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
    }
    {% endhighlight %}

7. Escolher "Salvar".

Essa política permite que qualquer pessoa leia os objetos no bucket.

## Parte 2: Pipeline de Dados

Jean shorts raw denim Vice normcore, art party High Life PBR skateboard stumptown vinyl kitsch. Four loko meh 8-bit, tousled banh mi tilde forage Schlitz dreamcatcher twee 3 wolf moon. Chambray asymmetrical paleo salvia, sartorial umami four loko master cleanse drinking vinegar brunch. <a href="https://www.pinterest.com">Pinterest</a> DIY authentic Schlitz, hoodie Intelligentsia butcher trust fund brunch shabby chic Kickstarter forage flexitarian. Direct trade <a href="https://en.wikipedia.org/wiki/Cold-pressed_juice">cold-pressed</a> meggings stumptown plaid, pop-up taxidermy. Hoodie XOXO fingerstache scenester Echo Park. Plaid ugh Wes Anderson, freegan pug selvage fanny pack leggings pickled food truck DIY irony Banksy.

## Parte 3: Monitoramento
