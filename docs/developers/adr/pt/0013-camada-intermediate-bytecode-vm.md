# ADR-0013: Camada Intermediate: representação em bytecode e máquina virtual

* Estado: Proposta
* Data: 2026-09-10
* Autores: RunStack Team
* Substitui: nenhuma
* Substituída por: nenhuma
* Relacionadas: ADR-0005 (AST exigida para transformações estruturais), ADR-0007 (modelo de protecção, quatro camadas), ADR-0011 (encriptação total do código-fonte / estratégia de execução em runtime), ADR-0012 (ponto de entrada da pipeline de produção)

## Contexto

O `Layer::Intermediate` existe como nome desde a ADR-0007 e como directório vazio (`src/Intermediate/.gitkeep`) desde a estrutura inicial do projecto, mas nada foi construído ali. Todos os passes implementados hoje são `Source` (`SymbolRenamePass`, `StringProtectionPass`, `ControlFlowPass`, `MinificationPass`), `Runtime` (`IntegrityVerificationPass`), ou `Packaging` (`EncryptionPass`, e o registo `loader-minification` do `MinificationPass`). A Fase 2 do roadmap nomeia esta lacuna directamente: "Representação em bytecode e uma máquina virtual real baseada em dispatch, substituindo a VM não funcional da implementação anterior," a par de benchmarks publicados.

Esta ADR existe para resolver o desenho da representação e da execução antes de qualquer código ser escrito, da mesma forma que a ADR-0011 resolveu o desenho da encriptação total do código-fonte antes de o `EncryptionPass` existir. Não decide metas de performance nem uma data de lançamento; o roadmap já diz que nada aqui é uma funcionalidade de produto até ter uma implementação e um teste (ADR-0003).

## O que "Intermediate" significa hoje, em concreto

A ADR-0007 define esta camada como actuando sobre "uma representação compilada ou serializada do código, depois da camada source, antes do packaging." Dois desenhos muito diferentes satisfazem essa frase, e esta ADR tem de escolher um antes de qualquer outra coisa ser decidida:

**Opção A: manter `SourceCode -> SourceCode`, auto-contido.** O passe compila o PHP protegido para um stream de instruções próprio, embute esse stream (como array ou literal de string PHP) junto de um pequeno interpretador em ciclo de dispatch, e emite o resultado como um ficheiro `.php` que corre o interpretador contra as instruções embutidas quando é incluído. É a mesma forma que o `EncryptionPass` já usa: um payload mais um loader auto-contido, sem runtime partilhado, sem dependência de Composer acrescentada à aplicação protegida (ADR-0011). O `Layer::Intermediate` classificá-lo-ia da mesma forma que o `Layer::Packaging` classifica o `EncryptionPass`, pelo que opera, não por exigir um novo tipo de `Representation`.

**Opção B: introduzir um tipo de `Representation` genuinamente novo** (por exemplo `BytecodeProgram`, distinto de `SourceCode`), com o `PassRegistry`/`Pipeline` a ganhar a capacidade de encadear um passe cujo tipo de output difere do seu tipo de input. Esta é uma alteração arquitectural real: hoje todos os passes são `SourceCode -> SourceCode` (o contrato de `Pass::process()` nunca variou), e o `Pipeline::run()` nunca teve de verificar que o tipo de output de um passe coincide com o tipo de input esperado pelo passe seguinte.

Esta ADR propõe a **Opção A**, pela mesma razão que a ADR-0011 e a ADR-0012 recusaram introduzir abstracções novas de forma especulativa: o padrão auto-contido já está provado (é o que corre em Enterprise hoje), não exige nenhuma alteração à `Pipeline`, ao `PassRegistry`, ou ao contrato `Pass`, e a generalidade da Opção B ainda não se justifica por uma necessidade concreta. Se algum dia for proposto um segundo passe da camada Intermediate que genuinamente não consiga funcionar assim, a Opção B passa a ser uma decisão para a própria ADR desse passe, não uma decisão especulativa tomada aqui.

## Decisão proposta

- Um novo passe, provisoriamente chamado `BytecodeCompilationPass`, registado como `bytecode-compilation`, classificado `Layer::Intermediate`.
- Input: a AST já disponível a partir dos passes da camada Source (este passe estende `AbstractAstPass`, a mesma base que todo o passe consumidor de AST já usa, segundo a ADR-0005).
- Output: código-fonte PHP contendo (1) um stream de instruções embutido e uma pool de constantes, serializados numa forma que `var_export()` ou equivalente consiga reconstruir sem `eval()` sobre os próprios dados, e (2) uma função interpretadora pequena e auto-contida, em ciclo de dispatch, que executa esse stream de instruções. Sem API de Runtime partilhada, pelo mesmo raciocínio que a ADR-0011 usou para manter a Camada Runtime fechada até surgir uma segunda necessidade concreta.
- Ordem: este passe corre depois dos passes da camada Source. Não é encadeado com o `EncryptionPass` sobre o mesmo artefacto; ver "Decidido: estratégias terminais alternativas" abaixo.
- Âmbito para uma primeira versão: um conjunto mínimo de instruções suficiente para expressar fluxo de controlo e chamadas a funções PHP comuns, sem tentar explicitamente cobertura total da linguagem PHP num só passe. Qual o subconjunto exacto é a primeira pergunta a responder depois de esta ADR ser aceite, não antes.

## Não-objectivos

- Esta ADR não desenha o conjunto de instruções em si (opcodes, codificação de operandos, formato da pool de constantes). Isso é trabalho de implementação que decorre da aceitação desta ADR, da mesma forma que a ADR-0011 precedeu a implementação real do `EncryptionPass`.
- Esta ADR não se compromete com uma meta de performance. O roadmap já pede benchmarks publicados junto da metodologia; são esses benchmarks que vão mostrar se este desenho é viável, não um número afirmado aqui.
- Esta ADR não decide se `bytecode-compilation` e `encryption` podem correr na mesma edição. Ver questões em aberto.
- Esta ADR não decide que edição(ões) activam este passe, da mesma forma que a ADR-0011 separou a decisão de desenho da decisão de activação, posterior e separada, do `EncryptionPass`.

## Decidido (2026-09-10): `BytecodeCompilationPass` e `EncryptionPass` são estratégias terminais alternativas, não uma pipeline sequencial

A pergunta não era simplesmente "correm na mesma edição" ou "correm em sequência." É uma questão arquitectural distinta de ambas: se duas técnicas de protecção têm de se compor numa pipeline apenas porque ambas calham a ser passes.

O `BytecodeCompilationPass` e o `EncryptionPass` são estratégias de protecção terminais alternativas para um dado artefacto:

- **Encryption**: loader mais payload PHP encriptado, desencriptado e executado em runtime (ADR-0011).
- **Bytecode**: um artefacto compilado com o seu próprio mecanismo de execução, não colocado por cima, nem por baixo, da encriptação.

Uma edição pode disponibilizar as duas estratégias como opções, mas uma única operação de protecção sobre um dado ficheiro selecciona uma estratégia de execução, nunca as duas aplicadas ao mesmo artefacto. Isto é deliberadamente mais fraco do que "compor sempre as duas" ou "nunca permitir as duas numa edição": não assume que os dois mecanismos são tecnicamente compostos, e não colapsa a decisão de produto num simples ou-ou, já que uma edição pode continuar a oferecer as duas, apenas não encadeadas.

> O `BytecodeCompilationPass` e o `EncryptionPass` são estratégias de protecção terminais alternativas para um artefacto. A arquitectura não deve exigir que corram juntas. Uma edição pode disponibilizar as duas estratégias, mas uma operação de protecção selecciona uma estratégia de execução, a menos que uma decisão técnica posterior demonstre uma composição válida e útil.

Isto deixa para uma implementação futura provar se existe uma composição segura e útil, em vez de esta ADR pagar esse custo antecipadamente, assumindo uma.

## Decidido (2026-09-10): falhar de forma fechada (fail closed) em construções não suportadas, restrito aos passes que exigem cobertura semântica total

Para construções PHP que o conjunto de instruções não consiga representar (reflection, `__call`/`__get`/`__set`, `eval`, callbacks dinâmicos, nomes de classe ou método construídos dinamicamente, acesso dinâmico a propriedades, e semelhantes), o `BytecodeCompilationPass` recusa processar o ficheiro, em vez de recair silenciosamente em embutir o PHP original para as partes que não consegue cobrir.

A regra, enunciada de forma geral:

> Um passe de protecção cuja garantia depende de cobertura semântica total tem de produzir um resultado completamente coberto pela sua semântica suportada. Se encontrar uma construção que não consiga transformar ou representar correctamente, tem de falhar explicitamente, em vez de preservar silenciosamente o código original.

Um fallback silencioso produziria um artefacto híbrido, parte transformado, parte original, que pode muito bem continuar a executar correctamente, mas não dá nenhuma resposta honesta a "quanto deste artefacto está realmente protegido," e um fragmento pequeno e aparentemente sem importância pode determinar o comportamento de todo o programa. Falhar de forma fechada não é grátis: significa que algum código real de clientes vai simplesmente ser rejeitado por este passe, pelo menos numa versão inicial, e essa rejeição é deliberada, não um defeito a contornar silenciosamente degradando a protecção.

Esta regra pertence especificamente aos passes cuja garantia exige cobertura total, sendo o `BytecodeCompilationPass` o primeiro, não a todos os passes da pipeline como política geral. Um passe que consiga preservar em segurança uma construção que não compreenda em profundidade (um hipotético refinamento futuro de minificação, por exemplo) não tem uma razão comparável para recusar; a obrigação de falhar de forma fechada decorre daquilo que a garantia de um passe específico realmente afirma, não do facto de ser um passe em geral.

A cobertura pode crescer com o tempo (construções suportadas são transformadas, as não suportadas falham, e a fronteira entre as duas move-se à medida que o conjunto de instruções cresce), mas o princípio de falhar de forma fechada em si não muda à medida que essa fronteira se move.

## Questões em aberto

1. **Âmbito do conjunto de instruções.** Um conjunto mínimo cobrindo aritmética, fluxo de controlo e chamadas a funções/métodos é o ponto de partida óbvio. Dada a decisão de falhar de forma fechada acima, o âmbito aqui é realmente uma questão de produto/lançamento (quanto código real de clientes este passe consegue aceitar numa primeira versão) e não uma questão de segurança de desenho; não precisa de ser resolvida antes de a implementação começar, ao contrário das duas questões anteriores.
2. **Metodologia de benchmark.** O roadmap pede benchmarks "publicados junto da metodologia," o que implica que a própria metodologia precisa de revisão antes de os números serem publicados, não apenas os números. Esta ADR não propõe uma.
3. **Contribuição para o modelo de ameaça.** A ADR-0009 já declara a fronteira do RunStack Ember (aumenta o custo contra atacantes casuais/semi-automatizados, não contra um engenheiro reverso dedicado). Se a interpretação de bytecode aumenta materialmente esse custo em relação aos passes já existentes da camada Source, ou se sobretudo acrescenta sobrecarga de runtime sem um ganho de protecção proporcional, deve ser declarado quando existir uma primeira implementação para avaliar, não assumido aqui.

## Estado

Proposta. As duas questões arquitecturais que esta ADR mais precisava de resolver antes de existir qualquer código, a composição de estratégias de execução e a política de falha de cobertura, estão decididas acima. O âmbito do conjunto de instruções, a metodologia de benchmark, e a contribuição para o modelo de ameaça permanecem em aberto, e segundo o próprio processo deste projecto (ADR-0003), nada disto é anunciado como funcionalidade de produto até existir uma implementação e um teste para isso.
