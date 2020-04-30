SAS Viya REST APIs
===================
SAS Viya REST APIs, ou de maneira simplifcada API do Viya, nada mais é do que uma API do tipo REST, que é disponibilizada pelo SAS para toda a comunidade desenvolvendora, e para os cientistas de dados que já estão acostumados a trabalhar com ferramentas open source.

De maneira resumida, API (Application Programming Interface) é um sistema responsável pela comunicação entre um cliente e um servidor. REST (Representational State Transfer) é toda API que segue os seguintes padrões, usando o protocolo HTTP:

1) Client-server
2) Stateless 
3) Cacheable 
4) Layered System 
5) Code on demand (Opcional)


Trabalhar com este conjunto de APIs pode dar muita vantagem na hora de usar o SAS em algum processo de negócio, já que com apenas algumas requisições o SAS Viya pode ser integrado em algum serviço web.

Para o público desenvolvedor e da ciência de dados, as APIs são a base de diversas bibliotecas capazes de integrar linguagens como Python, Lua, R, e Java, diretamente com o SAS Viya.

É por isso que, se você está interessado em entender como funciona esta integração, te convido a continuar lendo este guia prático de como consumir as APIs do Viya em qualquer serviço web.

Cabe lembrar que é sempre bom consultar a documentação oficial que se encontra no link:
https://developer.sas.com/apis/rest/

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
    * [Referências](#referencias)
# Autenticação e Tokens de Acesso

## Introdução
A etapa de autenticação é o primeiro passo que se deve dar se se pensa em fazer uso das APIs do SAS Viya. Toda vez que se faz uma requisição (GET/POST/PUT/DELETE) ao Viya, é requerida uma chave de acesso para a validação da chamada. Chamamos esta chave de acesso de `access_token`, e ela é gerada apenas uma vez por sessão, podendo ser definido um tempo de duração até ela expirar.

Para gerar um `access_token` é necessário cadastrar um cliente no servidor Viya. É encima deste cadastro que serão gerados os `access_token` que serão usado para validar toda requisição feita por nossa aplicação.

Esta etapa pode parecer um pouco complicada (ela realmente é complicada), por possuir um grande sistema de autenticação e segurança, mas não é este o preço de ter um processo de segurança robusto? Além disso, existem diversas ferramentas que já foram desenvolvidas para facilitar nossa vida durante a autenticação. Deixo aqui o [link](http://sww.sas.com/blogs/wp/gate/37070/sas-viya-3-5-calling-rest-apis-demo-application) do blog escrito pela Mary Kathryn Queen sobre uma fantástica ferrramenta para autenticação (Apenas para SAS Employees).

Este processo será divido em 2 partes: Registro de Clientes, e Obtenção de um Access Token (via terminal e via implementação numa aplicação Python).

É essencial levar em consideração 4 coisas:
1) Inicialmente é necessário possuir acesso de administrador no servidor onde estiver instalado o SAS Viya.
2) Ter o `curl` instalado dentro do servidor SAS Viya.
3) Possuir um usuário administrador no ambiente SAS Viya.
4) Ter a opção `Setting Cross-Origin Resource Sharing (CORS)` habilitada. (Para habilitar esta opção você pode seguir as instruções no [site](https://developer.sas.com/reference/cors/)).

## Registro de Clientes
O cadastro ou registro de clientes é um processo único que faremos dentro do servidor para configurar nome do cliente, senha do cliente, e até o tempo de validade de cada `access_token`.

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
`Oauth` é uma norma de autenticação que protege nossas credenciais durante o envio de informação usando protocolo HTTP. É com o intuito de segurança que o SAS fornece um `Oauth token` permitirá fazer o registro da nossa aplicação de maneira segura. Para obté-lo precisamos fornecer 2 coisas:
1) Um nome que posteriormente será atribuído ao cliente registrado. Se você estiver criando uma aplicação para Open Banking que o nome seja `OpenBankingApp`, mas para fins didáticos vamos usar o nome `app`.
2) O `consul token` obtido no passo anterior.

```
curl -X POST "http://<hostname>/SASLogon/oauth/clients/consul?callback=false&serviceId=<nome-do-seu-aplicativo>" \
      -H "X-Consul-Token: <value-of-consul-token-goes-here>"
```
Nota: Lembre-se também de substituir ``<hostname>`` pelo hostname do seu servidor SAS Viya.

O retorno desta chamada será uma string do tipo JSON, na qual apenas estamos interessados no valor do campo `access_token`, que ainda não é nosso `access_token` final.

Copie e guarde ele para usá-lo no próximo passo.

### Registro de cliente
No passo anterior pedimos para o sistema reservar o nome do nosso cliente. Neste passo faremos outra chamada fornecendo este mesmo nome, e acrescentando a senha do cliente (você pode usar 'Orion123'), e o `Oauth token` obtido no passo anterior.

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
Nota 1: Por padrão, o tempo validade do `access token` é configurado como 43199 segundos (12 horas), mas isto pode ser definido alterando o valor no campo `"access_token_validity"`.

Nota 2: O nome e senha da aplicação são muito importantes na hora de fazer as chamadas das APIs, portanto, é necessário que você nao se esqueça deles.

Se a requisição foi feita com sucesso, receberemos uma mensagem parecida com a seguinte:
```
{"scope":["openid"],"client_id":"<nome-do-aplicativo>","resource_ids":["none"],"authorized_grant_types":["password","refresh_token"],
"access_token_validity":43199,"authorities":["uaa.none"],"lastModified":1521124986406}
```

Pronto! O cliente da aplicação já foi registrado. Agora poderemos solicitar um `access token` toda vez que a aplicação deseje fazer requisições.

## Obtendo um Access Token
Agora que já há um cliente registrado podemos solicitar um `access token` para que a aplicação possa chamar as APIs do Viya.

Serão mostradas 2 formas de fazer isto, primeiro usando o curl desde o terminal, e segundo, implementando uma função Python para que a aplicação possa fazer isto de maneira automática.

### Usando CURL
Aqui precisamos fornecer 4 informações importantes:

1) Usuário Viya
2) Senha do usuário Viya
3) Nome do cliente
4) Senha do cliente

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
Nota: É necessário ter a bibliotéca `requests` instalada no seu ambiente Python 3.

Se o status da chamada for OK (200), será retornada a string `access token`. Este retorno pode ser armazenado numa varíavel para ser usada toda vez que se faça uma chamada nas APIs REST do SAS Viya.

Você pode encontrar mais informações sobre Autenticação e Registro de Clientes nos seguintes links:

## Referências
- https://developer.sas.com/reference/auth/
- https://developer.sas.com/apis/rest/Topics/?shell#authentication-and-access-tokens
- https://blogs.sas.com/content/sgf/2019/01/25/authentication-to-sas-viya/
- https://communities.sas.com/t5/SAS-Communities-Library/Getting-Python-Talking-with-The-SAS-Viya-REST-API-Registering-a/ta-p/557882
- https://communities.sas.com/t5/SAS-Communities-Library/Getting-Python-Talking-with-The-SAS-Viya-REST-API-PART-2/ta-p/557933
