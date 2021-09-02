---
title: "Graceful Shutdown"
date: 2021-09-01T01:50:14-03:00
tags: ["Go", "Kubernetes", "Istio"]
lightgallery: true
resources:
- name: "featured-image"
  src: "featured-image.jpg"
---
**TL;DR:**
Este artigo aborda a importância do Graceful Shutdown nos microsserviços, 
os sinais de desligamento (`SIGTERM`, `SIGINT` e `SIGKILL`) 
e três visões sobre esse desafio: Go, Kubernetes e Istio.
<!--more-->

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

A implementação mais comum de _graceful shutdown_ em Go, 
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

Nos [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) do Kubernetes, 
que representam réplicas de um [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/),
é possível configurar um _hook_ chamado `preStop`, que é invocado antes do sinal `SIGTERM` ser enviado.

Configurando um `sleep` neste _hook_, podemos ter um _graceful shutdown_, como no exemplo abaixo.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
      lifecycle:
        preStop:
          exec:
            command: ["sleep", "15"]
```

O intervalo do `sleep` deve ser o suficiente para que a alteração de endpoint do Kubernetes seja propagada ao 
kube-proxy, Ingress Controller, CoreDNS, etc. 
Para mais detalhes, veja [esse artigo](https://learnk8s.io/graceful-shutdown).

Por padrão, o Kubernetes espera até 30 segundos no processo de desligamento de um Pod 
antes de forçar o encerramento do processo (`SIGKILL`, que não pode ser interceptado).

{{< admonition tip >}}
Recomendo fortemente a leitura [deste artigo](https://learnk8s.io/graceful-shutdown) para maiores detalhes 
sobre o Graceful Shutdown no Kubernetes.
{{< /admonition >}}

{{< admonition warning >}}
A grande desvantagem dessa abordagem é que a imagem Docker precisa ter o comando `sleep`, 
dificultando o uso de imagens [Distroless](https://github.com/GoogleContainerTools/distroless).
{{< /admonition >}}

## Istio

O Istio possui uma configuração chamada `TerminationDrainDuration` em que é possível definir uma pausa antes do desligamento do sidecar.

{{< admonition info >}}
Sidecar é um conceito comum nas implementações de Service Mesh:
é um container que acompanha a aplicação (que também e um container) dentro do Pod do Kubernetes. 
Assim, temos 2 containers dentro do Pod: `app + sidecar`.

O sidecar é um proxy (no caso do Istio, é o [Envoy](https://www.envoyproxy.io/))
que faz a intermediação de todo tráfego do Pod para que tenhamos todas as vantagens do Service Mesh. 
{{< /admonition >}}

Quando o proxy recebe `SIGTERM` ou `SIGINT`, ele começa a drenar as conexões, 
impedindo novas conexões e permitindo que as conexões existentes sejam concluídas.

{{< admonition tip >}}
Lembrando que o `SIGTERM` é enviado após a execução do hook `preStop`
{{< /admonition >}}

A duração desse processo de drenagem é configurável tanto [globalmente](https://istio.io/v1.11/docs/reference/config/istio.mesh.v1alpha1/#ProxyConfig):

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    defaultConfig:
      terminationDrainDuration: 50s
```

quanto por workload (por Pod):

```yaml
annotations:
  proxy.istio.io/config: '{ "terminationDrainDuration": 50s }'
```

**A duração padrão é 5 segundos**.

## Conclusão

A melhor abordagem para habilitar Graceful Shutdown depende do cenário de cada projeto/sistema.

Existem projetos em que: 
- alterar o código dos microsserviços é muito trabalhoso;
- não usa Service Mesh (Istio);
- o uso de imagens Distroless é prioridade;
- não estão no Kubernetes;
- que estão no Kubernetes, usando Istio, e tem agilidade para alterar o código dos microsserviços.
  Nesse caso é possível conciliar mais de uma estratégia para garantir _zero downtime_.

{{< admonition question >}}
Compartilhe nos comentários os desafios e aprendizados do seu projeto! :wink:

- Qual é o cenário do seu projeto? 
- Qual abordagem é utilizada para _graceful shutdown_? 
- Como é a implementação na sua linguagem de programação e _framework_ preferido?

{{< /admonition >}}

## Referências

1. [Termination Signals](https://www.gnu.org/software/libc/manual/html_node/Termination-Signals.html)
2. [Graceful shutdown and zero downtime deployments in Kubernetes](https://learnk8s.io/graceful-shutdown)
3. [signal package](https://pkg.go.dev/os/signal)
4. [Challenges of running Istio distroless images](https://www.solo.io/blog/challenges-of-running-istio-distroless-images/)
5. [Graceful shutdown in Go http server](https://medium.com/honestbee-tw-engineer/gracefully-shutdown-in-go-http-server-5f5e6b83da5a)
