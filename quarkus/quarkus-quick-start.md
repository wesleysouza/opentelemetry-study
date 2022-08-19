# OpenTelemetry

## OpenTracing

Objetivo: servir de padrão para o rastreamento distribuído.

- Especificação: o que é um rastreador, rastro, trecho (span);
- Convenções semânticas: quais atributos usar em quais casos;
	- Quais atributos eu posso incluir em um método HTTP
	- Qual a rota;
	- Comunicação com o banco e etc.
- API de instrumentação em diversas linguagens;
	- Sem diferença semântica entre as APIs de diversas linguagens.
	- OpentracingConfig;
- Bibliotecas de instrumentação.

## OpenCensus

Objetivo: ser "a" biblioteca para captura de telemetria:

- SDKs para diversas linguagens: API e rastreador fazem parte da mesma biblioteca;
- Conduíte de dados de telemetria: opencensus-service;
- Suporte além de rastreamento: métricas disponível, suporte e logs viria em seguida;
- Bibliotecas de instrumentação.

## Opentelemetry = OpenTracing + OpenCensus

- Especificação;
- Convenções semânticas;
- API de instrumentação para diversas linguagens;
- API e SDK em módulos separados;
- Bibliotecas de instrumentação;
- Conduíte para processamento de dados de telemetria: **opentelemetry-collector**

## Padrões e Especificações

### Especificação da API

- Como a API de instrumentação deve ser independente do SDK usado;

### Especificação do SDK

- Como os SDKs devem se comportar, independente da linguagem e implementação;

### Especificação de dados

- Interface description languagem (IDL), especificando como os dados devem ser formagtados e transmitidos ou recebidos;
- OTLP: OpenTelemetry Line Protocol, um conjunto de arquivos proto, denfinindo tanto o transporte quanto o conteúdo da mensagem.
	- gPRC (Linguagem de definição de interfaces)
	- O arquivo definine quais são serviços e os endpoints que precisam ser implementados para que seja compátiveu com o padrão.
		- É especificado também quais são as mensagens que são trocadas por essas interfaces, o que que o cliente precisa enviar para que o servidor entenda.

## Instrumentação 

### Instrumentação Manual

Utilizando diretamente a API de instrumentação;

### Bibliotecas de instrumentação

Utilizando bibliotecas que integram com frameworks populares como Spring, Sevlets e etc;

### Instrumentação automática

Geralmente por meios de manipulação de bytecode

## Intrumentando uma aplicação Java que utiliza o framework Quarkus

### Instalando o Quarkus

```bash
curl -Ls https://sh.jbang.dev | bash -s - trust add https://repo1.maven.org/maven2/io/quarkus/quarkus-cli/
curl -Ls https://sh.jbang.dev | bash -s - app install --fresh --force quarkus@quarkusio
```

### Criando a aplicação

```bash
quarkus create nome-da-sua-aplicacao && cd nome-da-sua-aplicacao
#Adicione as dependências necessárias
quarkus extension add 'opentelemetry-otlp-exporter'
```

Escreva no arquivo src/main/java/GreetingsResource.java o conteúdo abaixo:

```java
package org.acme;

import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;
import org.jboss.logging.Logger;

@Path("/hello")
public class GreetingResource {

    //private static final Logger LOG = Logger.getLogger(TracedResource.class);

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String hello() {
        //LOG.info("AT-AT");
        return "AT-AT";
    }
}
```

#### Criando a configuração

Adicione o conteúdo abaixo no arquivo **src/main/resources/application.properties**: 

```bash
quarkus.application.name=at-at-quarkus 
quarkus.opentelemetry.enabled=true 
quarkus.opentelemetry.tracer.exporter.otlp.endpoint=http://localhost:4317 
quarkus.opentelemetry.tracer.exporter.otlp.headers=Authorization=Bearer my_secret 
quarkus.log.console.format=%d{HH:mm:ss} %-5p traceId=%X{traceId}, parentId=%X{parentId}, spanId=%X{spanId}, sampled=%X{sampled} [%c{2.}] (%t) %s%e%n
```

Explicação sobre o conteúdo:
- Linha 1: amarrar o nome da aplicação ao span, caso não for configurado o padrão será o id do artefato;
- Linha 2: habilitando o openTelemetry;
- Linha 3: endpoint gRPC;
- Linha 4: headers opicionais gRPC;
- Linha 5: adição da informação e tracing na mensagem de log

Agora vamos precisar configurar e iniciar o OpenTelemetry Collector para receber, processar e exportar dados de telemetria para o Jaeger que exibirá os rastreamentos capturados. Para configutar crie o arquivo **otel-collector-config.yaml** com o conteúdo abaixo:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: otel-collector:4317
  otlp/2:
    protocols:
      grpc:
        endpoint: otel-collector:55680

exporters:
  jaeger:
    endpoint: jaeger-all-in-one:14250
    tls: 
      insecure: true

processors:
  batch:

extensions:
  health_check:

service:
  extensions: [health_check]
  pipelines:
    traces:
      receivers: [otlp,otlp/2]
      processors: [batch]
      exporters: [jaeger]
```

### Iniciando os serviços e executando a aplicação 

Inicie o OpenTelemetry Collector e Jaeger com o arquivo **docker-compose.yml** rodando o comando **docker-compose up -d**. Veja o conteúdo desse arquivo abaixo:

```yaml
version: "2"
services:

  # Jaeger
  jaeger-all-in-one:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "14268"
      - "14250"
  # Collector
  otel-collector:
    image: otel/opentelemetry-collector:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - $PWD/otel-config/otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "13133:13133" # Health_check extension
      - "4317:4317"   # OTLP gRPC receiver
      - "55680:55680" # OTLP gRPC receiver alternative port
    depends_on:
      - jaeger-all-in-one
```

Pronto, agora podemos executar a aplicação:

```bash
./mvnw clean quarkus:dev
#ou
quarkus dev
```

```bash

```

## Referências

[Quartus Get Started](https://quarkus.io/get-started/)

