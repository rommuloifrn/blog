+++
date = '2024-11-15T23:33:06-03:00'
draft = true
title = 'Guia de Spring Security'
+++

# Guia de Spring Security

## Introdução

A ideia desse artigo, além de exercitar meus conhecimentos do módulo Security e da arquitetura do Spring, é prover um guia razoavelmente detalhado sobre como começar no SS e o que acontece quando você o adiciona na sua aplicação, seja ela monolítica (MVC) ou uma API RESTful.

Note que ao longo desse artigo eu estou presumindo que você sabe o básico de organização da arquitetura MVC e o básico de Spring Boot, especialmente o conceito de injeção de dependências.

## Índice 

1. A aplicação de exemplo
2. Adicionando o módulo Security
3. Comportamento inicial
4. Configurations e a lista de filtros de segurança
5. Login e a classe UserDetails

## Metodologia

Aqui eu irei tratar de duas arquiteturas de desenvolvimento: MVC (monolítico) e REST. O artigo irá acompanhar tratar desde a inclusão do módulo Security em duas aplicações de arquiteturas diferentes.

- Aplicação exemplo -> Adição do módulo -> Comportamento Inicial (proteção de rotas e como funciona o login por enquanto)
- Login
- Corrente de Filtros de Segurança

## 1. A aplicação de exemplo

Eu recomendo que por motivos didáticos você use sua própria aplicação pra praticar as ferramentas desse texto, mas irei utilizar uma aplicação de exemplo que ficará disponível no link XXXXXXXXXXX

A aplicação que usarei como exemplo aqui será um sistema de cadastro de plantas – O usuário poderá adicionar as plantas que tem e acessá-las. A ideia é que após a implementação do módulo Security, cada usuário só possa ver suas próprias plantas. 

Há 5 endpoints: findAll, get, create, read, update, delete. Todos usam o caminho padrão "http://localhost:8080", sendo que as operações de read, update e delete tem "/{id}" no final, onde {id} é o número da planta no banco de dados.

## 2. Adicionando o módulo Security

Você pode adicionar o módulo por meio do pom.xml ou pela extensão do Spring Boot na sua IDE. Caso não souber como, recomendo que pesquise sobre.

## 3. Comportamento inicial

Após adicionar o Security *e reiniciar* o projeto, todas as rotas ficam bloqueadas e você é automaticamente redirecionado para uma página de login, **que pode ser vista no navegador** caso não consiga ver no seu cliente HTTP. 

**O usuário padrão é "user"**, e a senha, que é re-gerada toda vez que a aplicação é reiniciada, é mostrada no terminal. Teste o acesso na sua aplicação e depois faça login com os dados!

Por padrão, o Security cria um bean que é uma implementação da _entidade do usuário_ **em memória** (isso significa que os dados ficam na memória RAM do servidor, e não serão persistidos).

Neste momento, estarão sendo usados:
- Uma implementação da entidade usuário (User, que implementa UserDetails)
- Uma implementação de *serviço* de usuário (InMemoryUserDetailsManager, que implementa UserDetailsService)

Basicamente, o que você precisa entender é que há uma classe para a *entidade* do usuário e outra para a *camada de serviço* dele (ver arquitetura).

Observe esse trecho da [documentação](https://docs.spring.io/spring-security/reference/servlet/getting-started.html).
```java
@EnableWebSecurity 
@Configuration
public class DefaultSecurityConfig {
    @Bean
    @ConditionalOnMissingBean(UserDetailsService.class)
    InMemoryUserDetailsManager inMemoryUserDetailsManager() { 
        String generatedPassword = // ...;
        return new InMemoryUserDetailsManager(User.withUsername("user")
                .password(generatedPassword).roles("USER").build());
    }

	@Bean
    @ConditionalOnMissingBean(AuthenticationEventPublisher.class)
    // bean "DefaultAuthenticationEventPublisher"
```

Note que o método **inMemoryUserDetailsManager** retorna a instanciação da classe de mesmo nome, que recebe um User como argumento.

Pra quem não tá acostumado com o Spring, é bastante nome. Com o tempo você se habitua.

Mas o importante por agora é que você entenda que, caso não implemente suas classes User e UserService, o Spring Boot irá cuidar da implementação do usuário pra você, usando esses _defaults_ nas camadas de entidade e de serviço.###############


*####isso só acontece porque é o Spring Boot, pelo que li nesse link da doc.*



## 4. Configurations e a lista de filtros de segurança

Pra começar a alterar o comportamento do SS, vamos criar o arquivo SecurityConfigurations.java na pasta /security, dentro da pasta principal do nosso app. 

No meu caso, vai ficar em src/main/java/com/romm/plants/SecurityConfigurations.java. Essa classe vai ser o bean em que vamos definir nossas alterações ao que já vem por padrão no SS.

```java

import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;

@Configuration
@EnableWebSecurity
public class SecurityConfigurations {
}
```

A anotação __@Configuration__ define define beans de configuração. ######
Já a __@EnableWebSecurity__ é a anotação que permite que nós acessemos a cadeia de filtros de segurança, ou SecurityFilterChain. #######

//////imagem

A SecurityFilterChain é uma bean que vai definir os filtros que uma requisição vai passar antes de ser respondida. Há atualmente 12 filtros, e você pode adicionar mais. É o que vou fazer pra expor os 5 endpoints da minha API:##########

```java
@Configuration
@EnableWebSecurity
public class SecurityConfigurations {
	@Bean
	public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception {
		return httpSecurity
		.authorizeHttpRequests(authorize -> authorize
		.requestMatchers(HttpMethod.GET, "/").permitAll()
		.requestMatchers(HttpMethod.POST, "/").permitAll()
		.requestMatchers(HttpMethod.GET, "/{id}").permitAll()
		.requestMatchers(HttpMethod.POST, "/{id}").permitAll()
		.requestMatchers(HttpMethod.DELETE, "/{id}").permitAll()
		)
		.build();
	
	}

}
```

O método securityFilterChain vai ser a bean que adicionará nossos filtros adicionais. Note que ela retorna um objeto que alterado pelo método _authorizeHttpRequests_, que por sua vez é um lambda que usamos para adicionar nossos filtros.

Cada chamada de requestMatchers checa se a requisição acessa uma url específica, nesses casos, os endpoints da nossa aplicação. Note também a chamada final de cada linha de .requestMatchers:
```java
.permitAll()
```
Ela que define quem pode acessar o endpoint, e em breve entenderemos customizar melhor isso.

## 5. Login e a classe UserDetails

Quando recém adicionado, o Security guarda o usuário padrão em memória. Como isso é apenas um recurso de testagem, toda vez que a aplicação for reiniciada, a senha dele será gerada novamente.

Pra mudar essa forma de autenticação, precisamos implementar nossa versão do método que irá fazer a checagem da senha e definir o modo com iremos armazená-la.