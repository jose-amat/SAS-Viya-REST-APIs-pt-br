SAS Viya REST APIs
===================

# Sumário
* [Autenticação e Tokens de Acesso](#autenticação-e-tokens-de-acesso)
    * [Introdução](#introdução)
    * [Registro de Clientes](#registro-de-clientes)
        * [Consul Token](#consul-token)
        * [Oauth Token](#oauth-token)
        * [Registro de Cliente](#registro-de-cliente)
    * [Obtendo um Access Token](#obtendo-um-access-token)
        * [Usando Curl](#usando-curl)
        * [Usando Python](#usando-python)

# Autenticação e Tokens de Acesso

## Introdução
A etapa de autenticação e obtenção dos access tokens é o primeiro passo para o uso das APIs REST do SAS Viya. Sem este processo não é possível fazer chamadas para as APIs do SAS. É por este motivo que será mostrado como fazer todo o processo dentro do nosso servidor e posteriormente fazer as requisições usando os access tokens dentro da nossa aplicação Python.

Esta etapa será divida em 2 partes: Registro de Clientes, e Obtenção de um Access Token (via terminal e via implementação numa aplicação Python).

É essencial levar em consideração 4 coisas:
1) Inicialmente, é necessário que você tenha acesso de administrador no servidor onde estiver instalado o SAS Viya.
2) Ter o `curl` instalado dentro do servidor SAS Viya.
3) Você precisa ter um usuário administrador no seu ambiente SAS Viya.
4) Ter a opção `Setting Cross-Origin Resource Sharing (CORS)` habilitada. (Para habilitar esta opção você pode seguir as instruções do site https://developer.sas.com/reference/cors/)

## Registro de Clientes

### Consul Token
Acesse no seu servidor e certifique-se de fazer login com um usuário do grupo sas, ou simplesmente acesse como sudo:

```
sudo su
```
Dirija-se até a pasta onde está localizado o arquivo contendo o `consul token`

```
cd /opt/sas/viya/config/etc/SASSecurityCertificateFramework/tokens/consul/default
```
Copie o `consul token` e guarde ele para usá-lo mais tarde
```
cat client.token
```

### Oauth token
O `Oauth token` é o token de autenticação que nos permitirá fazer o registro da nossa aplicação. Para obté-lo precisamos fornecer 2 coisas:
1) O nome da nossa aplicação. Este nome será separado pelo sistema para posteriormente registrá-lo como cliente. Se você estiver criando uma aplicação para Open Banking faz sentido você nomeá-la como `OpenBankingApp`, mas para fins didáticos você pode usar o nome `app`.
2) O `consul token` obtido no passo anterior.

```
curl -X POST "http://<hostname>/SASLogon/oauth/clients/consul?callback=false&serviceId=<nome-do-seu-aplicativo>" \
      -H "X-Consul-Token: <value-of-consul-token-goes-here>"
```
Nota: Lembre-se também de substituir ``<hostname>`` pelo hostname do seu servidor SAS Viya.

O retorno desta chamada será uma string do tipo JSON, na qual apenas estamos interessados no valor do campo `access_token`, que apesar de ter este nome, ele corresponde ao próprio `Oauth token` e não ao access token que estamos procurando para fazer chamadas dentro da aplicação. 

Copie e guarde ele para usá-lo no próximo passo.

### Registro de cliente
No passo anterior pedimos ao sistema reservar um nome para o cliente, que será usado por nosso aplicativo. Neste passo faremos outra chamada fornecendo o nome do aplicativo, senha do aplicativo (você pode usar 'Orion123'), e o `Oauth token` obtido no passo anterior.

```
curl -X POST "http://<hostname>/SASLogon/oauth/clients" \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer <oauth-token>" \
      -d '{
      "client_id": "<nome-do-aplicativo>",
      "client_secret": "<senha-do-aplicativo>",
      "scope": ["openid"],
      "authorized_grant_types": ["password"],
      "access_token_validity": 43199
      }'
```
Nota 1: Por padrão, o tempo de duração do `access token` é configurado como 43199 segundos (12 horas), mas isto pode ser definido alterando o valor no campo `"access_token_validity"`.

Nota 2: O nome e senha da aplicação são muito importantes na hora de fazer as chamadas das APIs, portanto, é importante que você nao se esqueça deles.

Se a requisição foi feita com sucesso, receberemos uma mensagem parecida com a seguinte:
```
{"scope":["openid"],"client_id":"<nome-do-aplicativo>","resource_ids":["none"],"authorized_grant_types":["password","refresh_token"],
"access_token_validity":43199,"authorities":["uaa.none"],"lastModified":1521124986406}
```

Pronto! O cliente da aplicação já foi registrado. Agora poderemos solicitar um `access token` toda vez que a aplicação deseje fazer requisições.

## Obtendo um Access Token
Agora que já há um cliente registrado podemos solicitar um `access token` para que a aplicação possa chamar as APIs REST do SAS Viya.

Serão mostradas 2 formas de fazer isto, primeiro usando o curl desde o terminal, e segundo, implementando uma função Python para que a aplicação possa fazer isto de maneira automática.

### Usando CURL
Aqui precisamos fornecer 4 informações importantes: usuário e senha do seu ambiente Viya, e nome e senha do aplicativo registrado.

```
curl -X POST "http://<hostname>/SASLogon/oauth/token" \
      -H "Content-Type: application/x-www-form-urlencoded" \
      -d "grant_type=password&username=<usuario-viya>&password=<senha-do-usuario>" \
      -u "<nome-do-aplicativo>:<senha-do-aplicativo>"
```

Esta chamada irá retornar uma string do tipo JSON, na qual apenas estamos interessados no valor do campo `'access_token'`, que corresponde ao `access token` que estavamos procurando.

### Usando Python
Já vimos como registrar um cliente e como obter um `access token`, porém seria muito mais interessante poder solicitar os tokens de maneira automática dentro da nossa aplicação.
Neste caso usaremos uma aplicação feita em Python.

Na função `obtainAccessTokenPythonUser()` definida embaixo, precisamos passar 4 parámetros, que são aqueles mesmos 4 parámetros que usamos para obter o token usando o curl: usuário e senha do nosso ambiente Viya, e nome e senha do aplicativo registrado.

```
import requests

appName = '<nome-do-aplicativo>'
appPass = '<senha-do-aplicativo>'
username = '<usuario-viya>'
password = '<senha-do-usuario>'
host = '<hostname>'

def obtainAccessTokenPythonUser(appName, appPass, username, password):
    url = "https://"+ host +"/SASLogon/oauth/token"

    headers = {"Content-Type":"application/x-www-form-urlencoded"}

    data = "grant_type=password&username=" + str(username)+ "&password=" + str(password)

    auth = (str(appName), str(appPass)) 

    r = requests.post(url,headers=headers, data=data, timeout=30, auth = auth, verify=False)
    print(r)
    print(r.json())
    if r.status_code == 200:
      return r.json()['access_token']
```
Nota: É necessário ter a bibliotéca `requests` instalada no seu ambiente Python.

Se o status da chamada for OK (200), será retornada a string `access token`. Este retorno pode ser armazenado numa varíavel para ser usada toda vez que se faça uma chamada nas APIs REST do SAS Viya.

Você pode encontrar mais informações sobre Autenticação e Registro de Clientes nos seguintes links:

- https://developer.sas.com/reference/auth/
- https://developer.sas.com/apis/rest/Topics/?shell#authentication-and-access-tokens
- https://blogs.sas.com/content/sgf/2019/01/25/authentication-to-sas-viya/
- https://communities.sas.com/t5/SAS-Communities-Library/Getting-Python-Talking-with-The-SAS-Viya-REST-API-Registering-a/ta-p/557882
- https://communities.sas.com/t5/SAS-Communities-Library/Getting-Python-Talking-with-The-SAS-Viya-REST-API-PART-2/ta-p/557933
