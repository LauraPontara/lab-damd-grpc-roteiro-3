# lab-damd-grpc-roteiro-3
Roteiro de Laboratório - Transparências em Sistemas Distribuídos e gRPC

## Estrutura

* [`ROTEIRO.md`](ROTEIRO.md): enunciado completo da atividade
* [`RESPOSTAS.md`](RESPOSTAS.md): respostas às 12 perguntas (Partes A, B, C e D) e à reflexão da seção 4.1
* [`proto/central.proto`](proto/central.proto): contrato do serviço `CentralAtendimento`
* [`java/grpc-central`](java/grpc-central): implementação em Java (servidor e cliente)
* [`python/grpc_central`](python/grpc_central): implementação em Python (servidor e cliente)

## Evidências de execução

| Parte | Java | Python |
| --- | --- | --- |
| RPC unário (`ConsultarHorario`) | [unario-java.png](evidencias/unario/unario-java.png) | [unario-python.png](evidencias/unario/unario-python.png) |
| RPC streaming (`AcompanharAvisos`) | [streaming-java.png](evidencias/streaming/streaming-java.png) | [streaming-python.png](evidencias/streaming/streaming-python.png) |

## Nota de transparência sobre uso de IA

Este repositório foi desenvolvido com apoio do Claude (Anthropic), conforme permitido pela nota de transparência do próprio roteiro (seção 1). O uso incluiu:

* Diagnóstico e correção de problemas de ambiente: uma falha de build do Maven causada por uma pasta `target` corrompida, e uma incompatibilidade de versão entre o `protobuf` instalado e os stubs Python gerados (`central_pb2.py`), resolvida atualizando `grpcio`/`grpcio-tools`/`protobuf` e regerando os stubs.
* Implementação do RPC de streaming (`AcompanharAvisos`) em Java e Python, a pedido explícito da autora, seguindo estritamente o código apresentado nas seções 7.1 a 7.4 do roteiro.
* Rascunho das respostas do `RESPOSTAS.md` referentes à Pergunta 3 da Parte A e às 3 perguntas da Parte D, revisadas pela autora antes do commit. 

