# Criação de uma Função Lambda + Python + DynamoDB

Para a criação da nossa função Lambda primeiro temos que realizar a criação de:
  * DynamoDB
  * Bucket s3

Nota:
  * Os passos a seguir consideram que o usuário tenha e esteja logado em uma conta AWS
  * Os links estão passando como paramentro a região "region=sa-east-1" caso necessário altere para região


# Então primeiro vamos criar e configurar o nosso Bucket S3.

Acesse o link: https://s3.console.aws.amazon.com/s3/buckets?region=sa-east-1

### Criar Bucket e definir o uso público
```bash
  1 - Aperte no botão "Criar Bucket"

  2 - Preencha um nome para o bucket
  
  3 - Logo abaixo desmarque a opção "Bloquear todo o acesso público"
  
  4 - E agora, um pouco abaixo, marque a opção "Reconheço que as configurações atuais 
podem fazer com que este bucket e os objetos dentro dele se tornem públicos."

  5 - E finalize a criação apertando em "Criar Bucket"
```

### Adicione o arquivo csv no bucket
```bash
  1 - Acesse o Bucket criado

  2 - Na aba "Objetos" aperte o botão "Carregar"

  3 - Agora aperte na opção de "Adicionar arquivos" e selecione o arquivo
.csv que será adicionado

  4 - Aperte no botão "Carregar"

  5- Em seguida, aperte no botão "Fechar"
```

Exemplo de arquivo .csv (cpf, cnpj, data):
```bash
  123.456.789-10,82.294.958/0001-10,17-03-2023
  987.654.321-00,61.713.523/0001-11,31-12-2022
  456.789.123-00,88.041.182/0001-80,15-06-2022
```


### Capture o ARN do Bucket criado
```bash  
  2 - Selecione a aba "Propriedades"

  3 - Copie o ARN do bucket:
    Formato: arn:aws:s3:::${BucketName}/${KeyName}
```

### Criação do nosso json de permissões
```bash
  1 - Em OUTRA ABA, acesse o link: https://awspolicygen.s3.amazonaws.com/policygen.html 

  2 - 1º campo (Select Type of Policy): selecione a opção "S3 Bucket Policy"

  3 - 3º campo (Principal): Digite "*"

  4 - 5º campo (Actions): Selcione o checkbutton ao lado "All Actions ('*')"

  5 - 6º campo (Amazon Resource Name (ARN)): escreva a ARN no bucket (que 
  copiamos anteriormente)

  6 - Agora aperte no botão "Add Statement"

  7 - No fim da página aperte no botão "Generate Policy"

  8 - Copie o json que aparecerá na tela e ja pode fechar a aba
```

### Editar as permissões do nosso Bucket
```bash
  1 - Selecione a aba de "Permissões"
  
  2 - Desça até achar as opções sobre "Política do bucket" e aperte no botão "Editar"

  3 - Cole o json contendo os parametros da politica de permissões que foi criada anteriormente 
  
  3- Aperte o botão "Salvar alterações"
```
# Agora vamos criar uma tabela no nosso DynamoDB.

Acesse o link: https://sa-east-1.console.aws.amazon.com/dynamodbv2/home?region=sa-east-1#tables

### Criar nossa tabela
```bash
  1 - Aperte no botão "Criar tabela"

  2 - Preencha um nome para o a tabela
  
  3 - No 2º campo (Chave de partição): Digite "key"
  
  4 - E finalize a criação apertando em "Criar tabela"
```
# Agora vamos criar nossa função Lambda.

Acesse o link: https://sa-east-1.console.aws.amazon.com/lambda/home?region=sa-east-1#/functions

### Criar a função
```bash
  1 - Aperte no botão "Criar função"

  2 - Selecione o card "Criar do zero"
  
  3 - No 1º campo (Nome da função): Digite o nome da função
  
  4 - No 2º campo (Tempo de execução): Selecione a linguagem que deseja usar
(neste exemplo usamos a lingugem Python 3.9)
  
  4 - E finalize a criação apertando em "Criar função"
```

# Antes de adicionarmos nossa função devemos adicionar permissões do DynamoBO

Acesse, em outra aba, o link: https://us-east-1.console.aws.amazon.com/iamv2/home?region=sa-east-1#/roleshttps://us-east-1.console.aws.amazon.com/iamv2/home?region=sa-east-1#/roles

```bash
  1 - No campo de pesquisar procure a função criada e clique nela

  2 - Na parte de "Políticas de permissões" clique na primeira  

  3 - Agora aperte no botão "Editar Política"

  4 - Aperte em "JSON" 

  5 - Substitura parra esse JSON: 

  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "logs:CreateLogGroup",
            "Resource": "arn:aws:logs:sa-east-1:186538110892:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": [
                "arn:aws:logs:sa-east-1:186538110892:log-group:/aws/lambda/upload_csv_s3:*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": "dynamodb:PutItem",
            "Resource": "arn:aws:dynamodb:sa-east-1:186538110892:table/dados-csv"
        }
    ]
  }

  6 - Finalize apertando em "Revisar Política" 
```

# Agora podemos voltar a nossa configuração da nossa função Lambda

Vamos configurar a nossa função, na aba de código adicione o seguinte script:
```bash
import json
import boto3
import uuid

import re
from datetime import datetime

s3_cient = boto3.client('s3')
dynamo_db = boto3.resource('dynamodb')
table = dynamo_db.Table('<nome_tabela>')  # DynamoDB table name

def lambda_handler(event, context):
    # TODO implement
    bucket_name = event["bucket_name"]
    s3_file_name = event["object_key"]
    resp = s3_cient.get_object(Bucket=bucket_name, Key=s3_file_name)
    
    data = resp['Body'].read().decode('utf-8')

    employees = data.split("\n")
    for emp in employees:
        emp = emp.split(",")
        key = str(uuid.uuid4())
        # Exception handling done for skipping errors. When we are dealing with CSV file content, there may be a chance to get last row as empty row. So I have used try-catch for avoiding the runtime issues.
        try:                
            table.put_item(
                Item = {
                    "key": key,
                    "cpf": re.sub(r'\D', '', emp[0]),
                    "cnpj":  re.sub(r'\D', '', emp[1]),
                    "data":  str(datetime.strptime(emp[2].strip(), '%d-%m-%Y').strftime('%Y-%m-%d'))
                })
        except:
            continue
    return {
        'statusCode': 200,
        'body': json.dumps('Operação realizada com sucesso!')
    }
```

Nota importante: Na linha 10 coloque no parametro o nome da tabela criada
  * table = dynamo_db.Table('<nome_tabela>')
  *Lembre-se de apertar no botão "Deploy" antes de iniciar os testes


Agora vamos inciar os testes:
```bash
  1 - Aperte no botão "Test"

  2 - Crie um novo modelo de teste com o nome que desejar

  3 - No campo de "json do evento" adicione este modelo:
  {
    "bucket_name": "projeto-jloiola6",
    "object_key": "file.csv"
  }

  4 - Agora só rodar o teste criado e pronto.
```


Após rodar o teste ele, e tudo estiver configurado corretamente, será retornado:
```bash
{
  "statusCode": 200,
  "body": "\"Opera\\u00e7\\u00e3o realizada com sucesso!\""
}
```

Isso significa que ao explorar os itens na nossa tabela criada os campos foram tratados adicionados.
