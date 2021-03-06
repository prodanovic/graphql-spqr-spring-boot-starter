# graphql-spqr-spring-boot-starter
Spring Boot 2 starter powered by GraphQL SPQR

[![Join the chat at https://gitter.im/leangen/graphql-spqr](https://badges.gitter.im/leangen/graphql-spqr.svg)](https://gitter.im/leangen/graphql-spqr?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/io.leangen.graphql/graphql-spqr-spring-boot-starter/badge.svg)](https://maven-badges.herokuapp.com/maven-central/io.leangen.graphql/graphql-spqr-spring-boot-starter)
[![Javadoc](http://javadoc-badge.appspot.com/io.leangen.graphql/graphql-spqr-spring-boot-starter.svg?label=javadoc)](http://www.javadoc.io/doc/io.leangen.graphql/graphql-spqr-spring-boot-starter)
[![Build Status](https://travis-ci.org/leangen/graphql-spqr.svg?branch=master)](https://travis-ci.org/leangen/graphql-spqr-spring-boot-starter)
[![Hex.pm](https://img.shields.io/hexpm/l/plug.svg?maxAge=2592000)](https://raw.githubusercontent.com/leangen/graphql-spqr-spring-boot-starter/master/LICENSE)
[![Semver](http://img.shields.io/SemVer/2.0.0.png)](http://semver.org/spec/v2.0.0.html)

## A friendly warning

This project is still in early development and, while fairly well tested, should be considered as ALPHA stage as long as the version is 0.0.X.

## Project setup / Dependencies

To use this starter in a typical Spring Boot project, add the following dependencies:

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <!--Will be optional as of 0.0.2-->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
  </dependency>
  <dependency>
    <groupId>io.leangen.graphql</groupId>
    <artifactId>graphiql-spring-boot-starter</artifactId>
    <version>0.0.1</version>
  </dependency>
  <!--Will be implicit as of 0.0.2-->
  <dependency>
    <groupId>io.leangen.graphql</groupId>
    <artifactId>spqr</artifactId>
    <version>0.9.7</version>
  </dependency>
</dependencies>
```

### Note

The plan is that in the 0.0.2 alpha release `spring-boot-starter-websocket` dependency won't be necessary if you're not using autoconfig for GraphQL subscriptions over WebSockets.
Additionally, an option to use server-sent-events instead will be added.

## Defining the operation sources (the beans that get exposed via the API)

All beans in Spring's application context annotated with `@GraphqlApi` are considered to be operation sources (a concept similar to `Controller` beans in Spring MVC).
This annotation can be used in combination with `@Component/@Service/@Repository` or `@Bean` annotations, e.g.

```java
    @Component
    @GraphQLApi
    private class UserService {
        //Query/mutation/subscription methods
        ...
    }
```
or 
```java
    @Bean
    @GraphQLApi
    public userService() {
        return new UserService(...);
    }
```

## Choosing which methods get exposed through the API

To deduce which methods of each operation source class should be exposed as GraphQL queries/mutations/subscriptions, SPQR uses the concept of a `ResolverBuilder` (since each exposed method acts as a resolver function for a GraphQL operation).
To cover the basic approaches `SpqrAutoConfiguration` registers a bean for each of the three built-in `ResolverBuilder` implementations:

* `AnnotatedResolverBuilder` - exposes only the methods annotated by `@GraphQLQuery`, `@GraphQLMutation` or `@GraphQLSubscription`
* `PublicResolverBuilder` - exposes all `public` methods from the operations source class (methods returning `void` are considered mutations)
* `BeanResolverBuilder` - exposes all getters as queries and setters as mutations (getters returning `Publisher<T>` are considered subscriptions)

It is also possible to implement custom resolver builders by implementing the `ResolverBuilder` interface.

Resolver builders can be declared both globally and on the operation source level. If not sticking to the defaults, it is generally safer to explicitly customize on the operation source level, unless the rules are absolutely uniform across all operation sources.
Customizing on both levels simultaniously will work but could prove tricky to control as your API grows.

At the moment SPQR's (v0.9.7) default resolver builder is `AnnotatedResolverBuilder`. This starter follows that convention and will continue to do so if at some point SPQR's default changes.

### Customizing resolver builders globally

To change the default resolver builders globally, implement and register a bean of type `ExtensionProvider<ResolverBuilder>`.
A simplified example of this could be:

```java
    @Bean
    public ExtensionProvider<ResolverBuilder> resolverBuilderExtensionProvider() {
        return (config, defaults) -> {
            List<ResolverBuilder> resolverBuilders = new ArrayList<>();

            //add a custom subtype of PublicResolverBuilder that only exposes a method of it's called "greeting"
            resolverBuilders.add(new PublicResolverBuilder() {
                @Override
                protected boolean isQuery(Method method) {
                    return super.isQuery(method) && method.getName().equals("greeting");
                }
            });
            //add the default builder
            resolverBuilders.add(new AnnotatedResolverBuilder());

            return resolverBuilders;
        };
    }
```
This would add two resolver builders that apply to _all_ operation sources.
The First one exposes all public methods named _greeting_. The second is the inbuilt `AnnotatedResolverBuilder` (that exposes only the explicitly annotated methods).
A quicker way to achieve the same would be:

```java
    @Bean
    public ExtensionProvider<ResolverBuilder> resolverBuilderExtensionProvider() {
        //prepend the custom builder to the provided list of defaults
        return (config, defaults) -> defaults.insert(0, new PublicResolverBuilder() {
                @Override
                protected boolean isQuery(Method method) {
                    return super.isQuery(method) && method.getName().equals("greeting");
                }
            });
    };
```

### Customizing the resolver builders for a specific operation source

To attach a resolver builder to a specific source (bean), use the `@WithResolverBuilder` annotation on it.
This annotation also works both on the beans registered by `@Component/@Service/@Repository` or `@Bean` annotations.

As an example, we can expose the `greeting` query by using:

```java
    @Component
    @GraphQLApi
    @WithResolverBuilder(BeanResolverBuilder.class) //exposes all getters
    private class MyOperationSource {
        public String getGreeting(){
            return "Hello world !"; 
        }
    }
```

or:

```java
    @Bean
    @GraphQLApi
    //No explicit resolver builders declared, so AnnotatedResolverBuilder is used
    public MyOperationSource() {
        @GraphQLQuery(name = "greeting")
        public String getGreeting(){
            return "Hello world !"; 
        }
    }
``` 

It is also entirely possible to use more than one resolver builder on the same operation source e.g.

```java
    @Component
    @GraphQLApi
    @WithResolverBuilder(BeanResolverBuilder.class)
    @WithResolverBuilder(AnnotatedResolverBuilder.class)
    private class MyOperationSource {
        //Exposed by BeanResolverBuilder because it's a getter
        public String getGreeting(){
            return "Hello world !"; 
        }

        //Exposed by AnnotatedResolverBuilder because it's annotated
        @GraphQLQuery
        public String personalGreeting(String name){
            return "Hello " + name + " !"; 
        }
    }
```
This way, both queries are exposed but in different ways. The same would work on a bean registered using the `@Bean` annotation.

### More to follow soon ...