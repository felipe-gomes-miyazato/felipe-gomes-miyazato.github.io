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

## Parte 1: Coletando dados

Requisitos:

- Conta AWS
- Conta kaggle

Conteúdo: demostração de contrução de uma função AWS Lambda que transfere dados do kaggle para um bucket S3.

### Provisionando Lambda

Esta Lambda age apenas como uma proxy, para disponibilizar os dados em um bucket S3. Para provisionar bastam as configurações padrão, neste caso com Python runtime.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/Captura de tela 2023-09-30 114229.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

### Código e configuração da Lambda

{% highlight python linenos %}

import boto3
import kaggle
import os

def lambda_handler(event, context):
    # Get the Kaggle dataset URL and bucket name from the event object
    kaggle_dataset_url = event['kaggle_dataset_url']
    bucket_name = event['bucket_name']

    # Configure Kaggle
    kaggle.api.authenticate()

    # Download dataset from Kaggle
    kaggle.api.dataset_download_files(kaggle_dataset_url, path='/tmp', unzip=True)

    # Get the list of downloaded files
    files = os.listdir('/tmp')

    # Create a session using your user credentials
    session = boto3.Session()

    # Create an S3 client
    s3 = session.client('s3')

    # For each file, upload it to the S3 bucket
    for file in files:
        with open(f'/tmp/{file}', 'rb') as data:
            s3.upload_fileobj(data, bucket_name, file)

    return {
        'statusCode': 200,
        'body': f'Successfully uploaded {len(files)} files to S3 bucket'
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

## Customizing Your Table of Contents
{:data-toc-text="Customizing"}

If you want to learn more about how to customize the table of contents of your sidebar, you can check the [bootstrap-toc](https://afeld.github.io/bootstrap-toc/) documentation. Notice that you can even customize the text of the heading that will be displayed on the sidebar.

### Example of Sub-Heading 2

Jean shorts raw denim Vice normcore, art party High Life PBR skateboard stumptown vinyl kitsch. Four loko meh 8-bit, tousled banh mi tilde forage Schlitz dreamcatcher twee 3 wolf moon. Chambray asymmetrical paleo salvia, sartorial umami four loko master cleanse drinking vinegar brunch. <a href="https://www.pinterest.com">Pinterest</a> DIY authentic Schlitz, hoodie Intelligentsia butcher trust fund brunch shabby chic Kickstarter forage flexitarian. Direct trade <a href="https://en.wikipedia.org/wiki/Cold-pressed_juice">cold-pressed</a> meggings stumptown plaid, pop-up taxidermy. Hoodie XOXO fingerstache scenester Echo Park. Plaid ugh Wes Anderson, freegan pug selvage fanny pack leggings pickled food truck DIY irony Banksy.

### Example of another Sub-Heading 2

Jean shorts raw denim Vice normcore, art party High Life PBR skateboard stumptown vinyl kitsch. Four loko meh 8-bit, tousled banh mi tilde forage Schlitz dreamcatcher twee 3 wolf moon. Chambray asymmetrical paleo salvia, sartorial umami four loko master cleanse drinking vinegar brunch. <a href="https://www.pinterest.com">Pinterest</a> DIY authentic Schlitz, hoodie Intelligentsia butcher trust fund brunch shabby chic Kickstarter forage flexitarian. Direct trade <a href="https://en.wikipedia.org/wiki/Cold-pressed_juice">cold-pressed</a> meggings stumptown plaid, pop-up taxidermy. Hoodie XOXO fingerstache scenester Echo Park. Plaid ugh Wes Anderson, freegan pug selvage fanny pack leggings pickled food truck DIY irony Banksy.
