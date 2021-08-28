---
title: "Graceful Shutdown"
date: 2021-06-02T00:49:55-03:00
draft: true
tags: ["Go", "Kubernetes", "Istio"]
---

## Introdução

A escalabilidade horizontal é uma das principais vantagens das arquiteturas de softwares modernas 
para que os sistemas continuem responsivos sob variações de carga. 

Ou seja, réplicas dos microsserviços são escaladas quando a carga aumenta, e desligadas quando a carga diminui.

Para que essa elasticidade (aumento e diminuição de réplicas) seja natural e transparente, 
é importante que o tempo de inicialização dos microsserviços seja baixo.
Quanto mais rápido um serviço inicializar, 
mais rápido estará disponível para atender requisições 
e dividir a carga com as outras réplicas.

{{< admonition question >}}
E sobre o desligamento dos microsserviços?

O que acontece se uma réplica recebe um sinal para ser desligada no momento em que está processando requisições?
{{< /admonition >}}

Algumas ocasiões em que uma réplica pode receber um sinal de desligamento, são:
1. diminuição de carga;
2. _rolling update_;
3. _rolling restart_.

Normalmente, ao receber um sinal, o processamento dessas requisições seriam interrompidos e os clientes receberiam erros.

A não ser que esse serviço tenha um processo de desligamento mais inteligente: _graceful shutdown_.

## Entendendo os sinais

Os sinais são usados principalmente em sistemas do tipo Unix, e são enviados pelo Kernel ou de algum outro programa.

Os principais sinais de desligamento/encerramento de um programa, são `SIGTERM`, `SIGINT` e `SIGKILL`.

`SIGTERM` é um sinal genérico usado para causar o encerramento do programa. É o sinal gerado pelo comando `kill`.

{{< admonition info >}}
O Kubernetes envia o sinal `SIGTERM` para matar um Pod.
{{< /admonition >}}

O sinal `SIGINT` é enviado quando o usuário digita `CTRL-c`.

`SIGKILL` é usado para causar o encerramento imediato do programa. 
Não pode ser interceptado ou ignorado e, portanto, é sempre fatal.

## Graceful shutdown

Graceful Shutdown significa um desligamento inteligente, agradável, sem danos ao sistema.

Para que microsserviços tenham esse desligamento inteligente, eles precisam lidar com os sinais `SIGTERM` e `SIGINT` listados acima.

O comportamento padrão da maioria das tecnologias é interromper o processamento do programa, 
de forma que, muitas vezes, é prejudicial para o funcionamento.

Em Go, por exemplo, um sinal síncrono é convertido em `panic` em tempo de execução.

Uma forma simples de lidar com esses sinais, é esperar alguns segundos para que o processamento seja finalizado. 
Mas pode ser necessário fechar conexões com banco de dados, [redis](https://redis.io/) ou um _message broker_, por exemplo.

O Graceful Shutdown pode ser implementado diretamente no código do serviço.
Porém, o [Kubernetes](https://kubernetes.io/) e [Istio](https://istio.io/) possuem configurações que podem ajudar nessa tarefa.

## Go

A implementação mais comum de graceful shutdown em Go, 
é usando [Goroutines](https://gobyexample.com/goroutines) e [Channels](https://gobyexample.com/channels), como o exemplo abaixo.

{{< gist matheusfm 3e66745244ae7c0c888e51c3eacc59a2 "stdlib.go" >}}

Dessa forma, o servidor HTTP é inicializado numa nova _goroutine_, enquanto a principal espera por um sinal no _channel_ `quit`.

Assim que um sinal é recebido, o servidor é desligado com um _timeout_ de 5 segundos.
Ou seja, se em 5 segundos ainda existir alguma conexão ativa, a função `Shutdown()` retorna um erro.

{{< admonition note >}}
A função `Shutdown()` foi introduzida no [Go1.8](https://golang.org/doc/go1.8#http_shutdown)
{{< /admonition >}}

Os principais frameworks web de Go sugerem implementações nesse padrão, com Goroutines e Channels:
- [Gin](https://github.com/gin-gonic/gin#graceful-shutdown-or-restart)
- [gorilla/mux](https://github.com/gorilla/mux#graceful-shutdown)
- [echo](https://echo.labstack.com/cookbook/graceful-shutdown/)

Pra quem prefere usar bibliotecas de terceiros criadas exclusivamente para isso, 
eu recomendaria a [ory/graceful](https://github.com/ory/graceful). 
Foi a que mais me chamou atenção pela simplicidade e possibilidade de customizar a função de desligamento.

{{< admonition tip >}}
Uma adaptação do exemplo acima utilizando a biblioteca [ory/graceful](https://github.com/ory/graceful), 
está disponível no meu [GitHub](https://github.com/matheusfm/go-graceful/blob/master/ory.go).
{{< /admonition >}}

## Kubernetes

## Istio

## Referências

1. [Termination Signals](https://www.gnu.org/software/libc/manual/html_node/Termination-Signals.html)
2. [Graceful shutdown and zero downtime deployments in Kubernetes](https://learnk8s.io/graceful-shutdown)
3. [signal package](https://pkg.go.dev/os/signal)
4. [ory/graceful](https://github.com/ory/graceful)
5. [Challenges of running Istio distroless images](https://www.solo.io/blog/challenges-of-running-istio-distroless-images/)
6. [Graceful shutdown in Go http server](https://medium.com/honestbee-tw-engineer/gracefully-shutdown-in-go-http-server-5f5e6b83da5a)