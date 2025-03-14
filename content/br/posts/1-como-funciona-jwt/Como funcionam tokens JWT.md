+++
date = '2025-02-15T23:33:06-03:00'
draft = false
title = 'A lógica da segurança por trás dos tokens JWT'
+++


# A lógica da segurança por trás dos tokens JWT

Eu, mesmo sabendo fazer autenticação RESTful no Spring por bastante tempo, ainda confundia minha cabeça pra entender como o JWT é seguro. Como assim passar informações em um token que pode ser facilmente ser decodificado? Esse é um apanhado geral do tema que busca explicar isso.

## 1: Tokens web

No contexto de APIs web, tokens são "chaves" que utilizamos para o servidor verificar quem somos. Geralmente recebemos um token no login, e temos de guardar a informação no front-end para utilizá-la junto das nossas próximas requisições para acessar recursos restritos [^1]

![](/posts/1-como-funciona-jwt/imagens/quero-logar.png)

Vamos usar como exemplo a TokenAuthentication, um módulo do DRF [^2]

```python
from rest_framework.authtoken.models import Token

token = Token.objects.create(user=...) # cria token
print(token.key)
```

Esse módulo provê o model Token, e cada usuário pode estar ligado a um token apenas. A ideia é que o dev, no login, use a função mostrada acima para criar um token que vai ficar salvo no banco de dados, e em seguida retorne ele para o usuário. 

Aqui está um pedaço da minha View de login, onde além de retornar o token, eu retorno dados do usuário junto:

```python
if senha_correta(usuario, request.data['password']):
            token = Token.objects.create(user=usuario) # cria o token
    
            usuario_serializado = UsuarioSerializer(
                usuario,
                context = {"request":request}
            )        
            
            r = Response( # monta request
                data={
                    "token":str(token), # token
                    "usuario": usuario_serializado.data # dados do usuário
                }
            )
            r["Access-Control-Allow-Origin"] = "http://localhost:5173"
            return r
```

Uma vez com o token recebido, o front-end deverá salvá-lo e envia junto de toda requisição que necessitar de autorização.

![](/posts/1-como-funciona-jwt/imagens/cabecao.png)

Caso o usuário não inclua o token numa requisição restrita, o servidor provavelmente responderá com o código 401 UNAUTHORIZED.

![](/posts/1-como-funciona-jwt/imagens/agora_sim.png)

Em cada requisição, o servidor vai checar a qual usuário percente aquele token (com base no **banco de dados**, guarde essa informação) e assim devolver os dados correspondentes.

[^1]: Recursos restritos = qualquer página que não possa ser acessada por qualquer pessoa. Tome como exemplo a edição do seu perfil no Twitter.
[^2]: Django Rest Framework.

## 2: Tokens JWT

O JWT é um [padrão aberto](https://datatracker.ietf.org/doc/html/rfc7519) que define uma forma específica de montar tokens. Diferente do nosso token simplório do módulo do DRF, _o JWT pode carregar diversas informações dentro dele_. 

Um JWT tem três partes:
- Cabeçalho
- Conteúdo
- Assinatura

![](/posts/1-como-funciona-jwt/imagens/jwt.png)

Esse tipo de token tem a vantagem de carregar consigo tudo que o servidor precisa pra identificar um usuário. Isso mesmo, dentro dele. A autenticidade dele não depende de nada no banco de dados. 

Essa é uma característica definida pelo padrão REST, chamada _stateless_ (sem estado). Ou seja, o servidor não guarda informações de "sessão" de nenhum usuário, os dados necessários ficam nos tokens. Recebeu uma nova requisição? Olha no token de quem que é.

![](/posts/1-como-funciona-jwt/imagens/verificando.png)

Essa característica é relevante por vários motivos: 

- Simplifica a lógica no servidor, pois não é necessário armazenar nada sobre a sessão atual do cliente.
- Ajuda o _escalonamento_ do aplicativo: Você pode ter vários servidores diferentes rodando a sua API, e qualquer um deles consegue checar a veracidade dos dados já que não há sessões em cada um deles (normalmente sessões são implentadas por servidor).
- Simplifica os processos de cache, pois normalmente seria necessário checar dados de sessão para decidir o que cachear ou não.

### A importância das assinaturas

Algo que me incomodou ao entender como isso tudo funcionava, era o fato de que o servidor depende somente das informações contidas no token pra identificar o usuário. Até aí beleza, mas lembra que eu te disse que os dados não necessariamente são criptografados? Você mesmo consegue ir num site de decodificação de JWT e olhar o conteúdo de um token desse tipo.

![](/posts/1-como-funciona-jwt/imagens/decodificando.png)

Na verdade, você pode inclusive modificar o conteúdo do token e mandar para o servidor de volta... Isso, na minha cabeça, significava uma falha de segurança grave. Mas aí que entram as assinaturas.

A tal da assinatura em um token JWT é gerada a partir do conteúdo original do token e de uma **chave secreta** que só o servidor sabe.

![](/posts/1-como-funciona-jwt/imagens/servidor_escrevendo.png)

Isso significa que a toda requisição, o servidor checa se os dados batem com a assinatura e a chave secreta.

![](/posts/1-como-funciona-jwt/imagens/checagem.png)

E por isso tokens JWT são seguros.

## Considerações finais

Eu simplifiquei o possível tentando não perder muito conteúdo. Por exemplo, ainda há outra forma de assinatura de JWT que utiliza uma chave pública e outra privada, mas preferi deixar de fora pois entendendo o que eu mostrei você já tá massinha.

## Referências

JWT.io - Introdução
https://jwt.io/introduction

RestFulAPI - Statelessness
https://restfulapi.net/statelessness/