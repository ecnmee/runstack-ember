# ADR-0012: Ponto de entrada da pipeline de produção

* Estado: Aceite
* Data: 2026-08-22
* Decisores: RunStack Ember
* Âmbito: Pipeline, CLI
* Relacionadas: ADR-0006 (as edições são configuração, não código), ADR-0007 (modelo de protecção), ADR-0011 (encriptação total do código-fonte)

## Contexto

`Edition`, `PassRegistry` e `Pipeline` estão implementadas, testadas unitariamente e exercitadas de ponta a ponta para duas edições (`FreeEditionIntegrationTest`, `EnterpriseEditionIntegrationTest`). Cada um desses testes, no entanto, constrói o seu próprio `PassRegistry` à mão, registando cada fábrica de passe inline. Nenhum ficheiro deste código faz isso uma única vez, para uso real, fora de um teste.

`src/CLI/` existe como um directório vazio e reservado (`.gitkeep`, apontando para a ADR-0007). Ainda não foi construído nada ali.

Entretanto, `tools/diagnose/obfuscate-file.php` já faz algo parecido com um ponto de entrada de produção: recebe um caminho de ficheiro, aplica uma sequência de passes e escreve o output. É o modelo errado para copiar. Constrói `SymbolRenamePass`, `StringProtectionPass`, `ControlFlowPass`, etc., directamente, numa ordem fixa no próprio script, contornando completamente `Edition` e `PassRegistry`. Isso é correcto para uma ferramenta de diagnóstico cujo propósito inteiro é permitir que um developer experimente combinações arbitrárias de passes. É exactamente o padrão que um ponto de entrada de produção não pode repetir, porque deita fora a única garantia que `Edition`/`PassRegistry` existem para fornecer: que o conjunto de passes que protege o código de um cliente é uma configuração nomeada, declarada e testável, e não uma lista ad hoc montada no local da chamada.

Esta ADR define o contrato entre a selecção de edição e um artefacto PHP protegido em disco, e declara que parte desse contrato é específica da CLI versus comportamento da biblioteca core, antes de qualquer parte disso ser implementada.

## Decisão

### Resumo da decisão

Não é criada nenhuma nova fachada (facade) para o ponto de entrada nesta fase. O ponto de entrada usa directamente as abstracções já existentes, `Edition`, `PassRegistry` e `Pipeline`, para transformar um pedido externo numa execução. A composição não pode ser duplicada entre a CLI e qualquer API futura, nem re-codificada à mão em nenhum sítio fora de um ficheiro de edição.

`Pipeline` mantém-se responsável pela execução. `PassRegistry` mantém-se responsável por resolver os passes. `Edition` mantém-se responsável por declarar a composição. O ponto de entrada apenas traduz entre a interface externa e estas abstracções.

Uma fachada só se justificará quando surgir uma responsabilidade real que não pertença naturalmente ao ponto de entrada, à `Edition`, ao `PassRegistry` ou à `Pipeline`. Não será criada apenas para encapsular chamadas que já se compõem de forma limpa.

### A biblioteca core ganha uma peça nova: uma fábrica de `PassRegistry` já populada

Todos os testes que hoje constroem um `PassRegistry` funcional registam à mão o mesmo conjunto de fábricas:

```php
$registry->register('symbol-rename', fn () => new SymbolRenamePass());
$registry->register('string-protection', fn () => new StringProtectionPass());
$registry->register('control-flow', fn () => new ControlFlowPass());
$registry->register('minification', fn () => new MinificationPass());
$registry->register('encryption', fn () => new EncryptionPass());
$registry->register('loader-minification', fn () => new MinificationPass('loader-minification'));
$registry->register('integrity-verification', fn () => new IntegrityVerificationPass());
```

Esta lista tem uma única forma canónica e correcta. Deve existir uma vez, como um construtor nomeado no próprio `PassRegistry`:

```php
PassRegistry::withDefaults(): self
```

O `withDefaults()` é o único sítio que conhece todos os nomes de passes actualmente distribuídos e qual a classe que implementa cada um. Adicionar um novo passe ao produto significa acrescentar uma linha aqui, não actualizar cada consumidor que constrói um registo.

Isto mantém-se dentro do namespace `Pipeline` já existente. Não é um novo conceito arquitectural; é um segundo construtor nomeado numa classe que já tem um (`Edition::fromFile`), seguindo a mesma convenção.

### Transformar um nome de edição e código-fonte PHP em código-fonte PHP protegido

O ponto de entrada é responsável por traduzir um pedido externo numa execução da pipeline. A construção da pipeline usa `Edition` e `PassRegistry`; a execução mantém-se responsabilidade da `Pipeline`. Em concreto, hoje, essa tradução tem este aspecto:

```php
$edition = Edition::fromFile($editionName, $editionsDir . "/{$editionName}.php");
$pipeline = PassRegistry::withDefaults()->buildPipeline($edition);
$result = $pipeline->run(new PipelineContext(new SourceCode($sourceText)));
```

Nenhuma classe de fachada envolve isto. `Edition`, `PassRegistry` e `Pipeline` já se compõem de forma limpa; adicionar uma quarta classe cujo único trabalho é chamar as outras três em ordem seria estrutura adicionada porque um diagrama sugeriu uma caixa, não porque as peças existentes falham a compor-se. Isto segue a mesma contenção já aplicada à Camada Runtime na ADR-0011: construir a abstracção quando surgir uma segunda necessidade concreta para ela, não de forma preventiva.

Esta composição é o que "o core" significa no resto desta ADR. É código de biblioteca simples, utilizável a partir de um script de CLI, de um futuro endpoint HTTP, de um teste, ou de uma REPL, sem depender de como é invocado.

**Quando uma fachada se justificaria.** Não para encapsular as três chamadas acima; isso seria encapsulamento pelo encapsulamento. Uma fachada só ganha o seu lugar quando surge uma responsabilidade real que não pertence naturalmente ao ponto de entrada, à `Edition`, ao `PassRegistry` ou à `Pipeline` individualmente. Nada identificado até agora atinge esse limiar.

### O anti-padrão que esta ADR existe para excluir

A separação já construída neste código atribui cada preocupação a exactamente um sítio:

```text
Edition        declara O QUÊ corre (uma lista nomeada e ordenada de nomes de passes)
PassRegistry   resolve QUAL IMPLEMENTAÇÃO um nome mapeia
Pipeline       executa COMO os passes correm (em ordem, registando o que correu)
Pass           transforma QUALQUER REPRESENTAÇÃO que tenha recebido
```

`tools/diagnose/obfuscate-file.php`, escrito mais cedo neste mesmo projecto, é um exemplo real do que acontece quando um ponto de entrada salta esta separação: constrói `new SymbolRenamePass()`, `new StringProtectionPass()`, `new ControlFlowPass()`, etc., directamente, numa ordem fixa no script, e tem a sua própria opinião sobre que passes vão juntos (a lógica da flag `--lean` vive inteiramente dentro desse script). Isso é legítimo para uma ferramenta de diagnóstico cujo propósito inteiro é experimentar combinações arbitrárias. É exactamente aquilo que o ponto de entrada de produção não pode fazer: conhecer nomes de passes além da única string que identifica que edição foi pedida, construir directamente qualquer `Pass`, ou codificar regras específicas de edição (que passes vão juntos, em que ordem) em qualquer sítio fora de um ficheiro de edição.

O trabalho do ponto de entrada de produção não é "esconder a composição atrás de uma fachada." É: consumir `Edition` e `PassRegistry` como são, e nunca reconstruir, à mão, as decisões que essas duas já codificam.

### O ponto de entrada não é uma nova camada arquitectural

A CLI é um adaptador. Uma futura API HTTP, se algum dia for construída, é outro adaptador. Ambos têm de convergir para o mesmo mecanismo de composição, em vez de cada um carregar a sua própria cópia dele:

```text
                 CLI              API HTTP (futura)
                  \                    /
                   \                  /
                 Tradução do ponto de entrada
                          |
                   Selecção de edição
                          |
                   Resolução de passes (PassRegistry)
                          |
                          v
                      Pipeline
                          |
                          v
                        Passes
```

Se algum dia for adicionada uma API HTTP, o risco que esta ADR quer excluir antecipadamente é `CLI -> lógica própria` e `API -> lógica própria` divergirem ao longo do tempo em duas reimplementações ligeiramente diferentes, e ambas ligeiramente erradas, das mesmas três linhas. Os dois adaptadores chamam a mesma composição; só o parsing de argumentos e a formatação da resposta diferem entre eles.

Isto não exige uma classe de fachada para ser verdade. Exige apenas que a composição seja escrita uma vez e que ambos os adaptadores a chamem, onde quer que esse "uma vez" acabe por viver.

### `Layer` é classificação, não um agendador

`EncryptionPass` é `Packaging`, `IntegrityVerificationPass` é `Runtime`, `SymbolRenamePass`/`StringProtectionPass`/`ControlFlowPass`/`MinificationPass` são `Source` (com o segundo registo do `MinificationPass`, `loader-minification`, deixado como questão em aberto pela correcção de classificação, não resolvida aqui). Cada uma destas classificações está agora correcta em relação ao texto da ADR-0007.

Nada disto significa que o ponto de entrada corra os passes pela ordem `Source -> Intermediate -> Runtime -> Packaging`. Não corre, e esta ADR não propõe fazê-lo assim. A `Pipeline` executa os passes na ordem que uma edição declara, por nome; essa ordem é o que o `EnterpriseEditionIntegrationTest` e o `EncryptionPassTest::test_it_locks_in_the_documented_encrypted_pipeline_order` garantem, e não corresponde a uma ordenação ingénua por camada (`minification` corre duas vezes, uma antes do `encryption` e outra depois, ambas as instâncias classificadas como `Source`; `encryption`, classificado como `Packaging`, corre antes da segunda `minification`, não depois de todos os passes da camada `Source` num sentido global). `layer()` é classificação arquitectural: responde a "que tipo de coisa é este passe", não a "quando é que corre". Confundir as duas coisas significaria derivar a ordem de execução a partir de `layer()` e obtê-la errada, já que a ordem real já codifica restrições (ver o addendum da ADR-0011) que uma ordenação simples em quatro grupos não consegue expressar.

### A CLI é I/O e apresentação de erros em torno dessa composição, nada mais

`src/CLI/` (ou um script `bin/ember`, dependendo de como os pacotes PHP normalmente expõem executáveis; ver a questão em aberto abaixo) é responsável exactamente por:

1. Fazer parsing dos argumentos: caminho do ficheiro de entrada, nome da edição, caminho do ficheiro de saída.
2. Ler o ficheiro de entrada para uma string.
3. Correr a composição de três linhas acima.
4. Escrever a string `SourceCode->code` resultante no caminho de saída.
5. Capturar toda a exception que a composição possa lançar e transformá-la numa mensagem clara e num código de saída diferente de zero, em vez de um stack trace em bruto.

Não constrói directamente nenhum `Pass`. Não conhece nomes de passes além do que precisa para passar uma string com o nome da edição. Se uma alteração futura significar que a CLI precisa de saber mais do que isso para fazer o seu trabalho, isso é um sinal de que o contrato desta ADR está errado nalgum ponto, não uma razão para deixar a CLI ir além dele.

### Tratamento de erros: quatro categorias, não um único wrapper

A composição pode falhar por razões que não são todas o mesmo tipo de falha, e colapsá-las num único tipo de exception esconderia essa diferença de todos os consumidores:

| Categoria | Exemplo | Tipo |
|---|---|---|
| Erro de utilização/validação da CLI | argumento em falta, ficheiro de entrada não existe, caminho de saída igual ao de entrada | Tratado inteiramente dentro da CLI, antes de a composição correr. Não é sequer uma exception da biblioteca. |
| PHP de entrada malformado | o `nikic/php-parser` não consegue analisar (parse) o ficheiro | `RunStack\Ember\Source\ParseException`, lançada por `AbstractAstPass::process()` quando captura `\PhpParser\Error` internamente. Ver "Fronteira de dependência" abaixo: a CLI nunca vê `\PhpParser\Error` directamente, e não precisa. |
| Falha na composição de protecção | nome de passe desconhecido, passe duplicado numa edição, pipeline vazia, ficheiro de edição malformado, falha de encriptação/compressão | Envolvida como `RunStack\Ember\Pipeline\ProtectionException`, preservando a exception original via `getPrevious()`. |
| Falha inesperada | qualquer coisa fora das três categorias acima | Deixada por capturar pelos handlers específicos da CLI. Propaga-se como aquilo que realmente é, com a sua classe real e stack trace intactos, porque isso é um sinal de bug, não um modo de falha normal da ferramenta, e escondê-la sob `ProtectionException` tornaria defeitos reais mais difíceis de encontrar, não mais fáceis. |

**Fronteira de dependência: o `AbstractAstPass` traduz a exception do parser externo, para que a CLI nunca dependa dele.** `\PhpParser\Error extends \RuntimeException`. Se o `AbstractAstPass` não a traduzisse, essa herança importaria para a ordem de captura em cada ponto de chamada consumidor: um bloco `catch (\RuntimeException)` colocado antes de um bloco `catch (\PhpParser\Error)` absorveria silenciosamente erros de parsing como se fossem falhas de composição de protecção. Em vez de empurrar essa exigência de ordenação para cada consumidor, o `AbstractAstPass::process()` captura `\PhpParser\Error` uma única vez, onde já toca directamente no parser, e lança `RunStack\Ember\Source\ParseException` (que estende `\InvalidArgumentException`, uma hierarquia sem relação com `\RuntimeException`, pelo que não existe qualquer risco de ordem de captura a jusante). A CLI captura `ParseException`, não `\PhpParser\Error`; não precisa de saber qual a biblioteca de parsing que este pacote usa internamente.

Isto também significa que os outros dois pontos de `\InvalidArgumentException` do `AbstractAstPass::process()` -- tipo de payload errado a chegar a um passe, e um parse que tem sucesso mas produz uma AST vazia -- ficam deliberadamente inalterados. O primeiro indica um bug em como a pipeline foi composta, não uma propriedade do PHP de entrada; o segundo é uma questão real e separada, em aberto, que esta decisão não resolve (um ficheiro vazio mas sintacticamente válido é uma falha de parsing ou outra coisa) e fica como `\InvalidArgumentException`, não promovido a `ParseException`, até essa questão ser decidida por si só.

O `ProtectionException` só envolve os tipos de exception que a composição já é conhecida por lançar por razões de domínio: `OutOfBoundsException` e `LogicException` do `PassRegistry`, `RuntimeException` de `Edition::fromFile`, `Pipeline::run`, e `EncryptionPass`. Não envolve `ParseException` (uma categoria diferente, acima, com o seu próprio bloco de captura) nem o caso de `\InvalidArgumentException` indicador de bug descrito no parágrafo anterior, que pertence à categoria de "falha inesperada", não capturado, visível.

### Output: um nome derivado e previsível por defeito, nunca silenciosamente no próprio lugar

```text
ember protect input.php --edition=enterprise
```

escreve em `input.ember.php`, no mesmo directório, derivado inserindo `.ember` antes da extensão original. Isto nunca exige `--output` para o caso comum, e nunca sobrescreve `input.php`.

```text
ember protect input.php --edition=enterprise --output=protected.php
```

escreve, em vez disso, no caminho indicado. Seja qual for o caminho usado, os caminhos de entrada e de saída são comparados resolvidos (não como string literal) antes de qualquer coisa correr; se resolverem para o mesmo ficheiro, a CLI recusa e termina com um erro de utilização, em vez de destruir silenciosamente o código-fonte. Não há flag `--in-place` nesta versão. Sobrescrever a única cópia de um ficheiro que é todo o propósito desta ferramenta transformar é um valor por defeito perigoso para tornar conveniente antes de haver uma razão concreta para precisar disso.

`--edition` não tem valor por defeito e é sempre obrigatório. Escolher um silenciosamente em nome de quem chama arrisca tanto sub-proteger (assumir Free quando quem chamou assumia mais) como fazer mais do que foi pedido (assumir Enterprise, correndo encriptação inesperadamente).

## Não-objectivos

Esta ADR não:

* Decide se `encryption`/`--lean` é exposta como flag da CLI ou puramente como escolha de edição. `--lean` mantém-se um conceito de `tools/diagnose/obfuscate-file.php`, segundo o addendum da ADR-0011; se a CLI de produção alguma vez vai precisar de um equivalente é uma decisão separada e posterior.
* Introduz um validador que verifique a lista de passes de uma edição contra o modelo de quatro camadas, ou desenha um esquema de execução ordenado por camada. Ver "`Layer` é classificação, não um agendador" acima; ambos ficam fora de âmbito pela mesma razão.
* Decide sobre uma API HTTP. "CLI/API" no título e nos diagramas desta ADR refere-se à CLI agora, mantendo a composição suficientemente agnóstica de adaptador para que uma camada HTTP a possa reutilizar mais tarde, e não a um desenho HTTP concreto decidido aqui.
* Toca no `EncryptionPass`, no `MinificationPass`, nos métodos já existentes do `PassRegistry`, ou na lista de passes de qualquer edição. Esses estão encerrados segundo a ADR-0011 e o seu addendum.

## Decidido

1. **A CLI vive em `bin/ember`**, a convenção padrão de executável do Composer (`"bin": ["bin/ember"]`), não sob `src/CLI/`. `src/CLI/` mantém-se reservado para classes, segundo o seu próprio `.gitkeep`, caso a lógica específica da CLI algum dia cresça além do que um script simples consiga suportar; nada nas cinco responsabilidades desta versão precisa de uma.
2. **É criada a `ProtectionException`**, delimitada exactamente à categoria de falha de domínio na tabela acima, e não como um catch-all para qualquer `\Throwable` que a composição possa produzir.
3. **O `AbstractAstPass` traduz `\PhpParser\Error` para `RunStack\Ember\Source\ParseException`**, preservando o original via `getPrevious()`. A CLI captura `ParseException`, não `\PhpParser\Error`, e volta a apresentá-la com o caminho do ficheiro de entrada adicionado, já que o próprio parser nunca vê um caminho de ficheiro, apenas uma string. Isto foi corrigido depois de a primeira tentativa de implementação do `bin/ember` assumir que `\PhpParser\Error` chegaria directamente até ele; não chega, porque o `AbstractAstPass` já a captura e volta a lançar como um simples `\InvalidArgumentException`, indistinguível do caso, sem relação, de "tipo de payload errado" sem esta alteração. Ver "Fronteira de dependência" acima para o raciocínio completo.
4. **O output assume por defeito um nome de ficheiro derivado (`input.ember.php`)**, `--output` sobrepõe-se a isso, e uma colisão entre caminho de entrada e de saída é um erro de utilização recusado, nunca uma sobrescrita silenciosa.

## Estado

Aceite. As quatro questões com que esta ADR começou estão todas decididas acima. O `PassRegistry::withDefaults()` está implementado e testado (91/91, monorepo). A `ProtectionException` e o `bin/ember` são o trabalho de implementação que falta, e que esta ADR autoriza.
