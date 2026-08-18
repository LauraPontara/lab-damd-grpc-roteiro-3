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

**Resposta:** *(a preencher)*

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

*(a preencher)*

### Pergunta 2

*(a preencher)*

### Pergunta 3

*(a preencher)*

## Parte D — RPC com streaming de servidor: AcompanharAvisos

### Pergunta 1

*(a preencher)*

### Pergunta 2

*(a preencher)*

### Pergunta 3

*(a preencher)*
