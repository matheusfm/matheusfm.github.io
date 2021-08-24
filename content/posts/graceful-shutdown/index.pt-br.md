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

O que acontece se, no momento em que uma réplica está processando requisições,
ela recebe um sinal para ser desligada?
{{< /admonition >}}

Algumas ocasiões em que uma réplica pode receber um sinal de desligamento, são:
1. diminuição de carga;
2. _rolling update_;
3. _rolling restart_.

Normalmente, o processamento dessas requisições seriam interrompidos e os clientes receberiam erros.

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

Graceful Shutdown significa um desligamento inteligente, agradável, sem danos ao nosso sistema.

Para que microsserviços tenham esse desligamento inteligente, eles precisam lidar com os sinais `SIGTERM` e `SIGINT`.

O comportamento padrão da maioria das tecnologias é interromper o processamento do programa, 
de forma que, muitas vezes, é prejudicial para o funcionamento.

Em Go, por exemplo, um sinal síncrono é convertido em `panic` em tempo de execução.

Uma forma simples de lidar com esses sinais, é esperar alguns segundos para que o processamento seja finalizado. 
Mas pode ser necessário fechar conexões com banco de dados, [redis](https://redis.io/) ou um _message broker_, por exemplo.

## Go

{{< gist matheusfm 3e66745244ae7c0c888e51c3eacc59a2 "graceful.go" >}}

## Kubernetes

## Istio

## Referências

1. https://www.gnu.org/software/libc/manual/html_node/Termination-Signals.html
2. https://learnk8s.io/graceful-shutdown
3. https://pkg.go.dev/os/signal
4. https://github.com/ory/graceful
5. https://www.solo.io/blog/challenges-of-running-istio-distroless-images/
6. https://medium.com/honestbee-tw-engineer/gracefully-shutdown-in-go-http-server-5f5e6b83da5a