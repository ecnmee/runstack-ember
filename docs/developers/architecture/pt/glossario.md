# Glossário

Termos tal como são realmente usados neste código. Onde um termo nomeia uma classe ou ficheiro real, isso é dito explicitamente; onde um termo é uma descrição e não um identificador literal, este glossário não afirma o contrário.

| Termo | Significado no RunStack Ember |
|---|---|
| Pass | Uma única transformação que implementa a interface `Pass` (`name()`, `layer()`, `process()`). |
| Pipeline | Uma sequência ordenada de instâncias de `Pass`, executada por ordem por `Pipeline::run()`. |
| PassRegistry | Resolve um nome estável de pass numa instância de `Pass`, e constrói uma `Pipeline` a partir de uma `Edition`. `PassRegistry::withDefaults()` regista todos os passes que este produto disponibiliza, sob o seu nome canónico. |
| Edition | Uma lista nomeada e ordenada de nomes de passes, carregada de um ficheiro PHP em `src/Editions/` via `Edition::fromFile()`. Declara composição; não implica nada sobre implementação. |
| Layer | Uma de `Source`, `Intermediate`, `Runtime`, `Packaging` (o enum `Layer`). Descreve que tipo de representação um pass actua e quando, segundo a ADR-0007. Não determina a ordem de execução; essa é o que a edition declarar. |
| SourceCode | A implementação de `Representation` que transporta uma string de código PHP através da pipeline. |
| Loader | O código PHP auto-contido que `EncryptionPass` gera para decifrar e executar o programa protegido em runtime. |
| Programa protegido | O código PHP original do cliente, antes ou depois da transformação da camada Source, distinto do loader que o envolve depois de cifrado. |
| Minification | Remoção de whitespace e comentários ao nível de token, implementada por `MinificationPass`. Registada sob dois nomes (`minification`, `loader-minification`) para dois alvos diferentes; ver o docblock dessa classe. |
| String protection | Cifra AES-256-GCM por literal, implementada por `StringProtectionPass`. |
| Control-flow (ofuscação) | Envolver instruções em guardas opacas sempre-verdadeiras (`crc32(__FILE__) === crc32(__FILE__)`), implementada por `ControlFlowPass`. Ofuscação estática da estrutura do programa protegido; não é um mecanismo de integridade em runtime. |
| Symbol rename | Renomear variáveis locais e parâmetros, implementada por `SymbolRenamePass`. Não renomeia propriedades, classes, nem funções. |
| Encryption | Compressão do ficheiro inteiro (`gzdeflate`) mais cifra AES-256-GCM mais geração de loader, implementada por `EncryptionPass`. Layer: `Packaging`. |
| Integrity verification | Um hash SHA-256 auto-verificável embutido no ficheiro distribuído, implementada por `IntegrityVerificationPass`. Layer: `Runtime`. Excluída de qualquer pipeline que também corra `EncryptionPass` (addendum da ADR-0011): o GCM já autentica o ciphertext. |
| CLI | `bin/ember`, o entry point de produção (ADR-0012). Fino: parsing de argumentos, I/O de ficheiros, apresentação de erros. Não constrói nenhum `Pass` directamente. |
| ParseException | Lançada por `AbstractAstPass` quando o `nikic/php-parser` não consegue analisar o input. Estende `InvalidArgumentException`. Permite a consumidores (o CLI) classificar "o input não era PHP válido" sem depender directamente da biblioteca de parsing. |
| ProtectionException | Lançada por `bin/ember`, envolvendo falhas de domínio de `PassRegistry`/`Edition`/`Pipeline` (nome de pass desconhecido, pipeline vazia, etc.). Não é usada para falhas de parsing nem para bugs na composição da pipeline. |
| Enterprise | A edition que actualmente activa `encryption`. `Premium` não activa (decisão de produto deliberada, não uma lacuna). |

## Termos que não devem ser confundidos

* **Pass vs. Layer**: um pass é uma classe; uma layer é uma classificação que essa classe declara sobre si própria. Dois registos da mesma classe (`minification`, `loader-minification`) podem declarar layers diferentes.
* **Edition vs. Pipeline**: uma edition é uma lista declarada de nomes; uma pipeline é o objecto executável que a `PassRegistry` constrói a partir dessa lista.
* **Pipeline vs. PassRegistry**: o registry resolve nomes em instâncias; a pipeline corre as instâncias resultantes por ordem. Nenhum faz o trabalho do outro.
* **Runtime vs. Packaging**: Runtime é comportamento que executa enquanto a aplicação protegida corre (`IntegrityVerificationPass`); Packaging é como o artefacto distribuído é montado (`EncryptionPass`). Confundir estes dois foi exactamente o drift que o addendum da ADR-0007 corrigiu.
* **Layer vs. localização do ficheiro**: `EncryptionPass` e `IntegrityVerificationPass` vivem ambos em `src/Source/`, e ambos reportam uma `Layer` diferente de `Source`. Onde um ficheiro está fisicamente não é prova de que layer lhe pertence; ver o addendum da ADR-0007 sobre porque isto já causou drift uma vez.
