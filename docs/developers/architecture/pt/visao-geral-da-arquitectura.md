# Visão geral da arquitectura

* Estado: Documento vivo
* Última actualização: 2026-09-09
* Público: Desenvolvedores que trabalham no motor RunStack Ember, ou com ele

Este documento dá uma visão de topo de como o RunStack Ember está montado. Não substitui as ADRs; onde uma decisão tem o seu próprio raciocínio, este documento aponta para essa ADR em vez de o repetir.

## A ideia central: uma pipeline de passes

O RunStack Ember protege um ficheiro PHP fazendo-o passar por uma sequência ordenada de **passes**. Cada passe implementa a interface `Pass` (`name()`, `layer()`, `process()`) e transforma uma `Representation` (hoje, sempre `SourceCode`, uma string de código-fonte PHP) noutra `Representation` do mesmo tipo. Uma `Pipeline` corre os passes em ordem e regista o que correu (ver o glossário para os termos exactos).

Três colaboradores fazem isto funcionar, cada um com exactamente uma responsabilidade (ADR-0012):

```text
Edition        declara O QUE corre (uma lista nomeada e ordenada de nomes de passes)
PassRegistry   resolve QUAL IMPLEMENTAÇÃO um nome mapeia
Pipeline       executa COMO os passes correm (em ordem, registando o que correu)
```

Uma `Edition` é um ficheiro PHP simples que devolve uma lista de nomes de passes (`src/Editions/free.php`, `basic.php`, `premium.php`, `enterprise.php`), ver ADR-0006. O `PassRegistry::withDefaults()` é o único sítio que conhece todos os nomes de passes que este produto distribui e qual a classe que implementa cada um, ver ADR-0012.

## As quatro camadas

Cada passe declara a que uma das quatro camadas pertence, com base em quando e onde actua sobre o programa protegido (ADR-0007):

- **Source**: actua sobre o código-fonte PHP antes de este correr, via AST ou tokens em bruto. `SymbolRenamePass`, `StringProtectionPass`, `ControlFlowPass`, `MinificationPass`.
- **Intermediate**: actua sobre uma representação compilada ou serializada, depois de Source e antes de Packaging. Ainda não implementada.
- **Runtime**: verificações e comportamento que correm enquanto a aplicação protegida executa. `IntegrityVerificationPass`.
- **Packaging**: como o artefacto protegido é montado e distribuído. `EncryptionPass`, e o segundo registo do `MinificationPass` (`loader-minification`, addendum da ADR-0007).

`Layer` é classificação, não um agendador: responde a "que tipo de coisa é este passe", não a "quando corre". A ordem de execução é a que uma edição declarar, por nome (ADR-0012). A pipeline encriptada actual, por exemplo, corre `minification` duas vezes, uma vez como `Source` e outra (como `loader-minification`) como `Packaging`, com `encryption` no meio:

```text
symbol-rename -> string-protection -> control-flow -> minification -> encryption -> loader-minification
```

A pipeline não encriptada termina em `integrity-verification`, já que o `IntegrityVerificationPass` e o `EncryptionPass` nunca correm juntos (addendum da ADR-0011).

## Licenciamento

A assinatura de licenças usa um par de chaves Ed25519, não um segredo partilhado (ADR-0008). A chave privada nunca sai do serviço de emissão de licenças; a chave pública é embutida em todos os artefactos distribuídos e verifica assinaturas totalmente offline, pelo que um artefacto protegido nunca precisa de acesso à rede para verificar a sua própria licença. Nenhuma chave gerida externamente, de qualquer tipo, variável de ambiente, servidor de licenças, ou armazenamento de chaves, faz parte do modelo de licenciamento (ADR-0002).

## Modelo de ameaça

O RunStack Ember aumenta o custo de ataques casuais e semi-automatizados: inspecção casual do código-fonte, análise estática genérica e decompiladores, redistribuição não autorizada, adulteração casual de licenças. Não afirma impedir um atacante dedicado e tecnicamente competente com tempo ilimitado, um atacante com acesso root ou físico no momento em que o artefacto corre, ou ataques de canal lateral (ADR-0009). Cada funcionalidade e cada afirmação de marketing é escrita contra esta fronteira, não como uma afirmação de força sem qualificação.

## Ponto de entrada de produção

`Edition`, `PassRegistry` e `Pipeline` compõem-se directamente; nenhuma fachada as envolve, porque nenhuma se justifica ainda (ADR-0012). A CLI (`bin/ember`) é um adaptador simples: faz parsing dos argumentos, lê o ficheiro de entrada, corre a composição de três linhas, escreve o resultado, e transforma exceptions em mensagens claras através de quatro categorias (erro de utilização, PHP malformado via `ParseException`, falha de composição de protecção via `ProtectionException`, e tudo o resto deixado por capturar como sinal de bug). Uma futura API HTTP, se construída, seria um segundo adaptador a convergir para a mesma composição, não uma segunda implementação dela.

## Para onde ir a seguir

- Para terminologia exacta, ver o glossário.
- Para saber porque cada decisão foi tomada, ver o índice das ADRs.
- Para o que está planeado mas ainda não construído, ver o roadmap.
