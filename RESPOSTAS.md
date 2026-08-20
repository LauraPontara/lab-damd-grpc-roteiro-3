# Respostas — Transparências em Sistemas Distribuídos e gRPC

## Parte A — Transparências em Sistemas Distribuídos

### Tarefa 4.1 (revisão do laboratório anterior)

**Questão 1:** O endereço do servidor (localhost, IP, grupo multicast) está escrito diretamente no código do cliente? Isso favorece ou prejudica a transparência de localização?

**Resposta:** Nas quatro soluções o endereço do servidor está escrito diretamente no código do cliente: `localhost` no TCP (`ClienteTCP.java`), `localhost` no UDP (`cliente_udp.py`), o grupo `230.0.0.1` no Multicast (`cliente_multicast.py`) e a URI `ws://localhost:8918` no WebSocket (`mural_cliente.py`). Isso prejudica a transparência de localização, já que o cliente precisa saber de antemão onde o servidor está. O Multicast é um caso parcialmente diferente: o endereço fixo no código é o de um grupo lógico, não o de uma máquina específica, então ele dá um grau de transparência de localização maior que os outros três, mesmo estando também fixo no código.

**Questão 2:** Para "perguntar uma coisa" ao servidor, o cliente precisa montar uma string de texto manualmente (e o servidor precisa interpretá-la/fazer parsing)? Isso é meio-termo, presença ou ausência de transparência de acesso?

**Resposta:** Sim, em todas as quatro soluções o cliente monta ou recebe uma string de texto livre, sem contrato formal de dados. No TCP e no UDP o cliente envia a mensagem digitada diretamente (`saida.println`, `mensagem.encode("utf-8")`) e o servidor faz parsing manual comparando strings (`mensagem.equalsIgnoreCase("hora")` em `ServidorTCP.java`). No Multicast o cliente só recebe texto puro (`dados.decode("utf-8")`), sem nenhuma estrutura. No WebSocket a troca também é texto livre, mas o próprio protocolo cuida da delimitação de cada mensagem automaticamente, diferente do TCP cru, em que a aplicação precisa decidir onde uma mensagem termina. Ainda assim, o conteúdo continua sendo texto sem contrato formal nos quatro casos, o que caracteriza ausência de transparência de acesso.

**Questão 3:** O que aconteceria com o cliente se o servidor mudasse de máquina amanhã? Alguma dessas quatro soluções sobreviveria a essa mudança sem alterar o código-fonte do cliente?

**Resposta:** Como o endereço está escrito diretamente no código-fonte, o cliente TCP, o cliente UDP e o cliente WebSocket não sobreviveriam a uma mudança de máquina do servidor sem alteração e recompilação do código. O Multicast é a exceção: como o cliente se inscreve em um endereço de grupo, e não no endereço de uma máquina específica, se o servidor mudasse de máquina mas continuasse enviando para o mesmo grupo multicast, o cliente continuaria funcionando sem nenhuma alteração de código. Das quatro soluções, essa é a única que sobrevive a essa mudança.

### Pergunta 1

**Pergunta:** Dentre os 8 tipos de transparência listados, qual você diria que é a mais visível para o programador que está usando um serviço remoto (e não construindo a infraestrutura por trás dele)? Justifique.

**Resposta:** A transparência de acesso é a mais visível para o programador que apenas usa um serviço remoto. É ela que determina se o código escrito parece uma chamada de função comum ou se exige lidar diretamente com sockets, montagem de mensagens e parsing de resposta. Diferente de transparências como replicação ou escala, que ficam escondidas na infraestrutura por trás do serviço, a transparência de acesso aparece diretamente na sintaxe do código que o programador escreve no dia a dia.

### Pergunta 2

**Pergunta:** Transparência total é sempre desejável? Dê um exemplo (pode ser hipotético) de uma situação em que esconder completamente que uma operação é remota atrapalharia mais do que ajudaria.

**Resposta:** Não. Um exemplo é o tratamento de falhas: se uma chamada remota for escondida a ponto de parecer idêntica a uma chamada local, o programador pode deixar de prever que essa chamada pode falhar por timeout, queda de conexão ou indisponibilidade do servidor, situações que uma chamada local nunca teria. Se o código tratar a chamada remota exatamente como uma chamada local, sem definir um tempo limite ou tratar exceções de rede, uma falha na rede pode travar a aplicação inteira esperando uma resposta que nunca chega. Esconder completamente a natureza remota da operação, nesse caso, atrapalha mais do que ajuda, porque impede o programador de tomar decisões importantes sobre como lidar com falhas parciais, que são próprias de sistemas distribuídos e não existem em chamadas locais.

### Pergunta 3 (responder depois de concluir as Partes C e D)

**Pergunta:** Comparando o cliente TCP do laboratório anterior com o cliente gRPC que você vai construir agora: qual dos dois exige que você "pense em rede" (sockets, send/receive, parsing de string) e qual permite que você "pense no problema" (chamar uma função e receber um resultado)? A que tipo de transparência isso se relaciona?

**Resposta:** O cliente TCP do laboratório anterior é o que exige "pensar em rede": ele abre o socket manualmente, monta a mensagem como uma linha de texto (`saida.println(linha)`) e depois lê e interpreta a resposta como outra linha de texto (`entrada.readLine()`), sempre lidando diretamente com os detalhes de envio e recebimento de bytes e com o parsing do conteúdo, já que não existe nenhuma estrutura formal de dados. Já o cliente gRPC deixa isso completamente por baixo dos panos: escrever `stub.consultarHorario(pergunta)` ou `stub.acompanharAvisos(inscricao)` "parece" apenas chamar uma função comum e receber um resultado (um objeto `RespostaHorario`, ou uma sequência de objetos `Aviso`, no caso do streaming), sem que o programador precise abrir socket, montar string ou interpretar texto de resposta. Essa diferença se relaciona diretamente com a transparência de acesso, que no TCP está praticamente ausente, já que o programador vê e manipula a natureza remota da chamada o tempo todo, enquanto no gRPC ela é alta, pois a chamada remota fica escondida atrás de uma interface que parece local, exatamente o ponto discutido na Pergunta 1 da Parte A.

## Parte B — Protocol Buffers e o contrato do serviço

### Pergunta 1

**Pergunta:** No laboratório anterior, cada um de vocês definiu o formato das mensagens de forma implícita (comentários e convenção entre quem escreveu o cliente e o servidor). Aqui, o formato está no `central.proto`. Qual a vantagem de ter esse contrato explícito e gerado automaticamente em vez de combinado apenas "de boca"?

**Resposta:** A vantagem principal é que o formato das mensagens deixa de depender de comunicação informal e de disciplina entre quem escreve o cliente e quem escreve o servidor. O contrato é único, fica versionado no repositório, e é a partir dele que são geradas automaticamente as classes de mensagem e o código de serialização nas duas pontas. Isso significa que cliente e servidor não conseguem divergir sobre o formato dos dados sem que o build quebre, por exemplo se um campo for renomeado ou tiver o tipo trocado no `.proto`. Isso elimina o tipo de erro que podia acontecer no laboratório anterior, em que cada lado podia interpretar o protocolo combinado de um jeito ligeiramente diferente.

### Pergunta 2

**Pergunta:** O mesmo arquivo `central.proto` gerou código para Java e para Python. O que isso sugere sobre como equipes que usam linguagens diferentes podem se comunicar em um sistema distribuído real?

**Resposta:** Sugere que equipes podem trabalhar em linguagens totalmente diferentes e ainda assim se comunicar de forma confiável, porque o contrato do serviço é neutro em relação à linguagem de implementação. Cada equipe gera, na sua própria linguagem, o código cliente/servidor a partir do mesmo arquivo `.proto`, sem precisar reimplementar manualmente a lógica de serialização nem combinar detalhes de baixo nível como o formato de bytes usado na comunicação. É isso que viabiliza, na prática, sistemas distribuídos heterogêneos, como um serviço em Java conversando com um serviço em Python, sem que a escolha de linguagem de cada equipe vire um bloqueio de integração.

### Pergunta 3

**Pergunta:** Observe os arquivos gerados (`target/generated-sources/.../CentralAtendimentoGrpc.java` ou `central_pb2_grpc.py`). Sem entender todo o código gerado, você consegue identificar onde ficam definidas as operações `ConsultarHorario` e `AcompanharAvisos`? Cite o nome de pelo menos uma classe ou método gerado que você reconheceu.

**Resposta:** Sim. Em Java, os métodos `consultarHorario` e `acompanharAvisos` aparecem na classe `CentralAtendimentoGrpc.java`, dentro da classe interna `CentralAtendimentoImplBase` (que o servidor estende para implementar o serviço) e também na classe `CentralAtendimentoBlockingStub` (usada pelo cliente para chamar os métodos). Em Python, os mesmos dois métodos aparecem como métodos da classe `CentralAtendimentoServicer`, dentro de `central_pb2_grpc.py`, que é a classe que o servidor implementa sobrescrevendo `ConsultarHorario` e `AcompanharAvisos`, e também na classe `CentralAtendimentoStub`, usada pelo cliente.

## Parte C — RPC unário: ConsultarHorario

### Pergunta 1

**Pergunta:** No cliente, a linha `stub.consultarHorario(pergunta)` (Java) ou `stub.ConsultarHorario(...)` (Python) parece uma chamada de método comum. Cite, em alto nível, pelo menos três coisas que acontecem "por baixo dos panos" entre essa chamada e o `return` da função no servidor.

**Resposta:** Primeiro, o stub serializa os campos de `PerguntaHorario` (o `nome_aluno`) para o formato binário do Protocol Buffers. Depois, esses bytes são enviados pela conexão HTTP/2 já aberta pelo canal (`ManagedChannel` em Java, `grpc.insecure_channel` em Python) até o endereço e a porta do servidor. No servidor, o runtime do gRPC recebe esses bytes, desserializa de volta para um objeto `PerguntaHorario` e só então invoca o método `consultarHorario`/`ConsultarHorario` implementado na aplicação. Depois que o método retorna, o processo se repete de volta: o `RespostaHorario` é serializado, enviado pela mesma conexão, e desserializado do lado do cliente antes de a chamada `stub.consultarHorario(...)` finalmente retornar o valor.

### Pergunta 2

**Pergunta:** Compare esta implementação com o `ClienteTCP` do roteiro anterior. Onde estava, no TCP, o equivalente a "montar a mensagem" e "interpretar a resposta"? Quem faz esse trabalho agora, no gRPC?

**Resposta:** No `ClienteTCP.java`, "montar a mensagem" era simplesmente enviar como texto puro a linha digitada pelo usuário (`saida.println(linha)`), e "interpretar a resposta" era ler de volta outra linha de texto do socket (`entrada.readLine()`) e imprimir, sem nenhuma estrutura, o significado do conteúdo dependia de um acordo informal entre quem escreveu o cliente e quem escreveu o servidor. No gRPC, esse trabalho é feito automaticamente pelo código gerado a partir do `central.proto`. O stub monta a mensagem `PerguntaHorario` a partir de campos nomeados definidos no contrato, cuida da serialização e da desserialização dos dados, e devolve ao cliente um objeto `RespostaHorario` já estruturado, com campos como `getMensagem()` (Java) ou `resposta.mensagem` (Python), sem que o programador escreva nenhuma lógica de parsing manual.

### Pergunta 3

**Pergunta:** O que aconteceria se você chamasse `stub.consultarHorario(pergunta)` com o servidor desligado? Teste e descreva o comportamento observado (em qualquer uma das duas linguagens).

**Resposta:** Testei em Python, com o servidor desligado antes de rodar o cliente. A chamada `stub.ConsultarHorario(...)` lançou a exceção `grpc._channel._InactiveRpcError`, com `status = StatusCode.UNAVAILABLE` e a mensagem `failed to connect to all addresses; last error: UNKNOWN: ipv4:127.0.0.1:50091: Failed to connect to remote host: Connection refused`. Ou seja, mesmo parecendo uma chamada de função comum, o gRPC não deixa passar despercebido o fato de que a chamada é remota: ele expõe o erro de rede através de um código de status próprio (`UNAVAILABLE`), de forma parecida com o `ConnectionRefusedError` que já tínhamos visto no TCP e no UDP do laboratório anterior quando o servidor estava fora do ar.

## Parte D — RPC com streaming de servidor: AcompanharAvisos

### Pergunta 1

**Pergunta:** No laboratório anterior, o Multicast usava um endereço de grupo (230.0.0.1) para alcançar vários clientes com um único envio; aqui, o streaming gRPC é um servidor conversando com um cliente por vez, só que ao longo de uma conexão só. Se você quisesse que vários clientes gRPC recebessem os mesmos avisos ao mesmo tempo, o que precisaria mudar na implementação do servidor?

**Resposta:** Na implementação atual, cada chamada a `AcompanharAvisos` gera sua própria sequência de avisos de forma independente, dentro do próprio método (o `for` em Java e Python roda uma vez por cliente conectado), então se dois clientes se conectarem em momentos diferentes, cada um recebe sua própria contagem de 1 a 5 avisos a partir do instante em que se inscreveu, sem nenhuma coordenação entre eles. Para que vários clientes recebessem exatamente os mesmos avisos ao mesmo tempo, o servidor precisaria manter uma lista central de `StreamObserver` (Java) ou de contextos de streaming ativos (Python), guardando uma referência a cada cliente inscrito no momento em que ele chama `AcompanharAvisos`, e um único processo em segundo plano (uma thread ou tarefa periódica) geraria os avisos e os enviaria (`onNext`, no caso do Java) para todos os observadores registrados de uma vez, em vez de cada chamada gerar sua própria sequência isolada. Ou seja, seria necessário sair do modelo de request e resposta isolado por chamada e introduzir um estado compartilhado do lado do servidor, algo parecido com o registro de clientes que o WebSocket do laboratório anterior já fazia para o mural de avisos.

### Pergunta 2

**Pergunta:** Compare o método de streaming em Java (StreamObserver, chamando onNext() repetidamente) com o de Python (uma função geradora usando yield). Os dois alcançam o mesmo resultado, qual das duas abordagens você achou mais natural de entender? Justifique.

**Resposta:** Achei a abordagem em Python, com `yield`, mais natural de entender. O método `AcompanharAvisos` em Python se lê quase como uma descrição direta do comportamento desejado: "para cada número de 1 a 5, produza um aviso e espere dois segundos", sem nenhuma referência explícita ao mecanismo de envio pela rede, a função apenas produz valores um de cada vez e o framework gRPC cuida de transformar cada valor produzido em uma mensagem enviada ao cliente. Já em Java, o `StreamObserver` exige pensar em termos do próprio mecanismo de envio: é preciso chamar `observador.onNext(aviso)` explicitamente a cada iteração e `observador.onCompleted()` ao final, e o tratamento de erro depende de capturar exceções e repassá-las para `observador.onError(e)`. O resultado final é o mesmo dos dois lados, mas a versão em Python separa melhor a lógica de negócio (o que gerar) do mecanismo de streaming (como entregar), enquanto em Java os dois ficam misturados no mesmo método.

### Pergunta 3

**Pergunta:** No método acompanharAvisos/AcompanharAvisos, o que aconteceria se o cliente fechasse a conexão (por exemplo, fechando o terminal) no meio do envio dos 5 avisos? Pesquise ou teste o comportamento e descreva o que observou.

**Resposta:** Testei isso em Python, com um servidor e um cliente isolados (mesma lógica do `AcompanharAvisos`, com prints extras a cada etapa do laço), matando o processo do cliente à força (equivalente a fechar o terminal) cerca de 3 segundos após a inscrição, ou seja, logo depois de ele já ter recebido os avisos #1 e #2. No terminal do servidor, o laço seguiu normalmente até imprimir "preparando aviso #3", mas nunca chegou a imprimir a confirmação de envio desse aviso: a tentativa de entregar o terceiro item ao stream já fechado interrompe a execução do gerador naquele ponto, sem nenhuma mensagem de erro visível e sem lançar exceção não tratada até o topo do programa. O mais importante é que o processo do servidor continuou de pé depois disso, pronto para atender novas chamadas de outros clientes, sem travar nem encerrar. Isso confirma que o runtime do gRPC absorve o cancelamento da conexão remota e isola essa falha na chamada específica que estava em andamento, uma diferença importante em relação ao TCP bruto do laboratório anterior, onde uma conexão perdida sem tratamento adequado podia deixar a thread do servidor bloqueada esperando por dados que nunca chegariam.
