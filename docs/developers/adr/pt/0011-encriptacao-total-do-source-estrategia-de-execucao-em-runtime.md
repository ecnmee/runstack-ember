# ADR-0011: Encriptação total do código-fonte / Estratégia de execução em runtime

* Estado: Aceite
* Data: 2026-08-16
* Decisores: RunStack Ember
* Âmbito: Camada Source, Camada Runtime, Camada Packaging
* Relacionadas: ADR-0002, ADR-0004, ADR-0007, ADR-0009, ADR-0010

## Contexto

O RunStack Ember já fornece transformações ao nível do código-fonte destinadas a aumentar o custo da inspecção casual e da engenharia reversa do código PHP distribuído.

O `StringProtectionPass` protege literais de string individuais com AES-256-GCM. A chave de encriptação é gerada por cada invocação de build e embutida no código gerado junto do ciphertext. O seu modelo de ameaça não afirma, explicitamente, sigilo da chave contra um atacante capaz de inspeccionar o PHP gerado.

O `IntegrityVerificationPass` detecta modificações no código gerado embutindo um digest SHA-256 da representação final e verificando-o em runtime.

A próxima transformação candidata ao nível do código-fonte é o `EncryptionPass`.

Ao contrário do `StringProtectionPass`, que substitui literais individuais deixando intacto o código PHP circundante, a encriptação total do código-fonte substituiria o próprio código PHP protegido por um pequeno loader executável.

O ficheiro distribuído resultante conteria assim:

1. uma representação encriptada do código-fonte PHP original;
2. a informação necessária para o desencriptar;
3. um loader capaz de reconstruir o texto original (plaintext);
4. um mecanismo para executar esse PHP reconstruído.

Isto é materialmente diferente dos passes existentes orientados à AST.

O passe mantém-se uma transformação `SourceCode -> SourceCode`, porque o seu output continua a ser PHP executável válido. No entanto, a transformação não opera de forma significativa sobre a AST do utilizador: o código-fonte original passa a ser um payload encriptado opaco e é substituído por um loader.

A decisão cruza, portanto, a fronteira que o projecto já usou anteriormente para identificar a necessidade de uma Camada Runtime.

## Decisão

O RunStack Ember vai suportar a encriptação total do código-fonte através de um loader de runtime auto-contido, nesta primeira implementação.

O `EncryptionPass` vai:

* receber um `SourceCode` normal;
* encriptar a totalidade do código-fonte PHP protegido;
* substituir esse código-fonte por um pequeno loader executável;
* manter o output gerado como PHP válido;
* desencriptar o código-fonte em runtime;
* materializar o código-fonte desencriptado num ficheiro PHP temporário;
* executá-lo usando `include`;
* remover o ficheiro temporário imediatamente após a execução.

Esta primeira implementação não vai usar `eval()`.

Esta primeira implementação não vai exigir que a aplicação protegida instale o RunStack Ember como dependência de runtime do Composer.

O loader gerado pelo passe será, assim, auto-contido.

## Mecanismo de execução em runtime

### Decisão

Usar materialização em ficheiro temporário seguida de `include`.

O código PHP desencriptado existirá em disco apenas durante a janela de execução exigida pelo loader.

O ficheiro temporário será removido imediatamente após a operação `include()` terminar.

### Justificação

Usar um ficheiro PHP temporário preserva um comportamento naturalmente associado à execução normal de ficheiros PHP de forma mais próxima do que o `eval()`.

Em particular, esta abordagem oferece um ambiente mais convencional para:

* `__FILE__`;
* `__DIR__`;
* stack traces;
* diagnósticos relacionados com ficheiros PHP;
* compilação de opcode;
* comportamento de debugging.

O `eval()` evita a materialização em sistema de ficheiros, mas essa vantagem não é suficiente para justificar torná-lo o mecanismo de execução da v1.

O projecto aceita, assim, o custo de sistema de ficheiros em troca de semântica de execução PHP mais convencional.

## Ciclo de vida do ficheiro temporário

O loader da v1 vai usar um caminho temporário novo para cada execução.

O código-fonte desencriptado não será guardado sob um caminho de aplicação estável e reutilizável.

Depois da execução, o ficheiro temporário será removido.

Isto significa, deliberadamente, que a v1 não tenta tornar a representação desencriptada persistente para reutilização pela OPcache.

A sobrecarga de recompilação resultante é aceite como um compromisso da v1.

Uma cache desencriptada estável poderá ser considerada numa versão futura, se a definição de perfis (profiling) demonstrar que a sobrecarga de compilação é materialmente significativa.

A optimização de performance é, assim, deferida até haver evidência de que é necessária.

## Geração e armazenamento da chave

A chave de encriptação será gerada de novo para cada build.

A chave será embutida no loader gerado.

O desenho da v1 não introduz um serviço externo de gestão de chaves, uma chave de encriptação fornecida por ambiente, um servidor de licenciamento, ou qualquer outra fonte externa de chaves.

Isto segue a fronteira do modelo de ameaça já estabelecida pelo `StringProtectionPass`.

A presença de AES-256-GCM não implica que a chave seja secreta perante um atacante que possua o loader gerado.

Um atacante capaz de inspeccionar o loader pode obter a chave e reproduzir o processo de desencriptação.

O propósito da encriptação total do código-fonte não é, portanto, o sigilo criptográfico da chave.

O seu propósito é impedir que o ficheiro distribuído exponha a implementação PHP protegida directamente como código-fonte legível.

## Modelo de ameaça

A encriptação total do código-fonte está explicitamente limitada ao seguinte objectivo de protecção:

> Remover a implementação PHP em texto plano do ficheiro distribuído, aumentando o custo da inspecção casual e da extracção estática.

Não afirma resistência contra um atacante capaz de executar ou analisar o loader.

Um atacante capaz de executar o loader tem necessariamente acesso ao mecanismo necessário para desencriptar o código-fonte.

O desenho também não afirma protecção contra:

* inspecção de memória;
* instrumentação em runtime;
* interceptação do código-fonte desencriptado;
* modificação do próprio loader;
* extracção da chave de desencriptação a partir do loader;
* observação do ficheiro PHP temporário desencriptado enquanto este existe.

A estratégia de ficheiro temporário introduz, assim, uma janela deliberada e de curta duração de exposição em texto plano no disco.

Essa exposição é aceite para a v1.

## Autenticação e detecção de adulteração

O AES-256-GCM fornece encriptação autenticada.

Consequentemente, quando o payload encriptado é modificado, a tag de autenticação deve fazer a desencriptação falhar, em vez de produzir texto plano aceite como válido.

A encriptação total do código-fonte fornece, assim, detecção de adulteração para o próprio payload encriptado, sem exigir uma verificação de integridade SHA-256 separada para o ciphertext.

Isto altera o papel do `IntegrityVerificationPass`.

O `IntegrityVerificationPass` não deve ser considerado automaticamente redundante apenas porque o GCM autentica o payload encriptado.

O seu papel preciso quando combinado com a encriptação total do código-fonte mantém-se uma questão arquitectural em aberto.

Em particular, o projecto tem de distinguir entre:

1. autenticidade do payload encriptado distribuído;
2. integridade da representação temporária desencriptada;
3. detecção de modificações ao loader;
4. garantias mais amplas de integridade em runtime.

Nesta ADR não é tomada nenhuma decisão de remover o `IntegrityVerificationPass` das edições encriptadas.

Essa decisão exige uma avaliação separada.

## Interacção com o `IntegrityVerificationPass`

A ordem relativa dos dois passes é condicionada por aquilo que cada passe se destina a proteger.

Se o `IntegrityVerificationPass` correr antes da encriptação total do código-fonte, a sua verificação de integridade gerada passa a fazer parte do texto plano encriptado.

O loader encriptado passa então a ser a representação distribuída.

Nessa configuração, a verificação de integridade só é executada depois de o código-fonte encriptado ter sido desencriptado e materializado.

Se o `IntegrityVerificationPass` correr depois da encriptação total do código-fonte, só pode verificar o loader produzido pelo `EncryptionPass`, e não o código-fonte original da aplicação protegida.

Essa distinção não deve ser escondida pelo termo genérico "integridade".

Assim:

> O `EncryptionPass` não deve ser adicionado a uma edição enquanto o papel e a ordem pretendidos para o `IntegrityVerificationPass` não estiverem explicitamente estabelecidos.

A presente ADR regista isto como uma questão arquitectural em aberto, em vez de inventar uma justificação para uma determinada ordem.

## Camada Runtime

O `EncryptionPass` é o segundo caso concreto identificado pelo checkpoint já existente da Camada Runtime.

A primeira geração de passes pode manter-se auto-contida porque o seu comportamento em runtime pode ser injectado directamente no código-fonte resultante:

* o `StringProtectionPass` injecta expressões de desencriptação inline;
* o `IntegrityVerificationPass` injecta uma verificação de integridade inline;
* o `ControlFlowPass` injecta guardas de fluxo de controlo inline.

A encriptação total do código-fonte é diferente.

Introduz um mecanismo completo de desencriptar-e-executar cuja responsabilidade natural é a execução em runtime, e não a transformação da AST.

Esta ADR considera, portanto, o checkpoint da Camada Runtime satisfeito.

No entanto, esta primeira implementação do `EncryptionPass` ainda não vai introduzir uma API de Runtime partilhada.

O loader manter-se-á auto-contido.

A Camada Runtime será aberta como componente arquitectural explícito quando a implementação demonstrar uma necessidade concreta de comportamento de runtime partilhado.

Isto evita criar uma abstracção apenas porque o roadmap prevê que ela possa, eventualmente, ser útil.

## Camada Packaging

O ficheiro PHP encriptado gerado não pode exigir que a aplicação protegida dependa do RunStack Ember em runtime.

O loader da v1 é, portanto, auto-contido.

Uma futura Camada Packaging poderá fornecer ficheiros de suporte de runtime partilhados, quando partilhar a implementação de runtime entre outputs gerados se tornar vantajoso.

Esse empacotamento seria um mecanismo de implementação e distribuição, não uma dependência de runtime do Composer imposta à aplicação protegida.

## Classificação do passe

O `EncryptionPass` vai implementar `Pass` directamente, em vez de estender `AbstractAstPass`.

A razão é estrutural.

O passe não transforma a AST do utilizador da mesma forma que:

* o `SymbolRenamePass`;
* o `StringProtectionPass`;
* o `ControlFlowPass`.

Transforma a representação completa do código-fonte num payload encriptado mais um loader.

O seu contrato mantém-se:

```text
SourceCode -> SourceCode
```

mas a sua implementação opera sobre a representação completa do código-fonte.

Isto é, assim, análogo à posição arquitectural já ocupada pelo `MinificationPass`, e não à dos passes de transformação da AST.

Nenhum novo tipo de representação se justifica apenas por esta distinção.

## Restrições de ordem

A ordem eventual do `EncryptionPass` tem de preservar o seguinte princípio:

> Todas as transformações destinadas a proteger o código-fonte PHP original têm de terminar antes de o código-fonte ser encriptado.

Uma vez que o `EncryptionPass` substitui o código-fonte original por um payload encriptado opaco, as transformações de código-fonte subsequentes deixam de conseguir transformar de forma significativa o código original da aplicação.

A encriptação é, por isso, uma transformação terminal da pipeline de código-fonte protegido.

A ordem final exacta relativa ao `IntegrityVerificationPass` mantém-se não resolvida, como descrito acima.

As transformações de código-fonte já estabelecidas podem, conceptualmente, manter-se:

```text
symbol-rename
-> string-protection
-> control-flow
-> minification
-> [ordem integrity-verification / encryption a decidir]
```

A ordem final da edição não pode ser activada até a interacção integridade/encriptação ter sido testada de ponta a ponta.

## Não-objectivos

Esta ADR não introduz:

* gestão externa de chaves;
* obtenção de chaves do lado do servidor;
* aplicação de licenciamento;
* anti-debugging;
* protecção de memória;
* cache persistente do código-fonte desencriptado;
* optimização de OPcache;
* uma API de Runtime partilhada;
* uma dependência obrigatória de Runtime via Composer;
* um novo tipo de representação de código-fonte;
* execução via `eval()`.

Estas podem vir a ser consideradas de forma independente em ADRs futuras.

## Consequências

### Positivas

* O código-fonte PHP distribuído deixa de ser directamente legível.
* A primitiva AES-256-GCM já existente pode fornecer encriptação autenticada da totalidade do código-fonte.
* O loader mantém-se auto-contido.
* A aplicação protegida não precisa de ter o RunStack Ember instalado em runtime.
* A execução de PHP continua baseada num ficheiro real e em `include()`, em vez de `eval()`.
* A cache em runtime pode ser introduzida mais tarde, se medições de performance o justificarem.
* A Camada Runtime tem agora um segundo caso de uso concreto, em vez de ser uma abstracção criada de forma especulativa.

### Negativas

* PHP em texto plano existe temporariamente em disco durante a execução.
* O ficheiro desencriptado tem de ser removido de forma fiável.
* A v1 aceita a sobrecarga de recompilação causada por um caminho temporário novo em cada execução.
* A chave de encriptação continua recuperável a partir do loader gerado.
* A execução em runtime torna-se mais complexa do que a execução normal de código-fonte PHP.
* O `IntegrityVerificationPass` passa a ter uma relação mais estreita e mais complicada com a representação encriptada.
* O comportamento de debugging e operacional tem de ser testado, e não assumido como idêntico à execução PHP normal.

## Consequências para o roadmap

O checkpoint da Camada Runtime é agora considerado atingido conceptualmente.

O próximo passo de implementação não é escrever imediatamente o `EncryptionPass`.

O próximo passo é transformar esta ADR em restrições arquitecturais executáveis através de testes que cubram:

1. encriptação completa do código-fonte;
2. desencriptação bem-sucedida;
3. execução bem-sucedida através de um ficheiro temporário;
4. limpeza do ficheiro temporário;
5. falha de autenticação após modificação do ciphertext;
6. comportamento de falha do loader;
7. preservação de namespaces;
8. semântica de `__FILE__` e `__DIR__`;
9. exceptions e stack traces;
10. interacção com o `IntegrityVerificationPass`;
11. ordem dentro da pipeline completa de uma edição.

Só depois de estas restrições estarem estabelecidas é que o `EncryptionPass` concreto deve ser implementado.

## Estado

Aceite.

A encriptação total do código-fonte é aprovada como a próxima grande capacidade da Camada Source, mas a implementação fica condicionada aos testes de execução em runtime e de ordem de integridade definidos acima.

## Addendum (2026-08-21): IntegrityVerificationPass e ordem final dos passes

Os testes de execução em runtime e de compressão descritos acima estão agora implementados e a passar (`EncryptionPass`, `EncryptionPassTest`). Duas questões que esta ADR deixou explicitamente em aberto ficam agora encerradas.

### O IntegrityVerificationPass é excluído quando o EncryptionPass corre

O AES-256-GCM é uma cifra autenticada: o `openssl_decrypt()` falha directamente perante qualquer modificação ao ciphertext, incluindo modificação do loader que o envolve. Correr o `IntegrityVerificationPass` depois do `EncryptionPass` só autenticaria o loader, algo que o GCM já autentica em cada tentativa de desencriptação; corrê-lo antes do `EncryptionPass` embute a verificação dentro do texto plano encriptado, onde só consegue verificar o ficheiro temporário depois de esse ficheiro já ter sido desencriptado e escrito em disco, um cenário mais estreito e mais raro do que o caso de artefacto adulterado em repouso para o qual o `IntegrityVerificationPass` existe. Nenhuma das duas posições justifica o seu custo.

Decisão: o `IntegrityVerificationPass` não participa em nenhuma pipeline onde o `EncryptionPass` também corra. Mantém-se disponível, sem alterações, para edições que não usem encriptação total do código-fonte.

### Ordem final dos passes para uma pipeline encriptada

A regra que esta ADR implicava originalmente, "o `EncryptionPass` tem de ser o último passe", é mais forte do que aquilo que o projecto realmente precisa e não é o que a implementação faz. A regra que importa é mais restrita:

> O `EncryptionPass` tem de ser o último passe que transforma o programa protegido. Um passe só pode correr depois dele se for demonstravelmente incapaz de inspeccionar, analisar (parse) ou modificar o payload encriptado, operando exclusivamente sobre a representação do próprio loader gerado.

O `MinificationPass` satisfaz isto. Ao contrário do `SymbolRenamePass`, do `StringProtectionPass`, do `ControlFlowPass` e do `IntegrityVerificationPass`, não constrói nem depende de uma AST do programa protegido; segundo a sua própria classificação de passe (ver "Classificação do passe" acima), opera sobre texto PHP em bruto, ao nível de tokens. Quando corre depois do `EncryptionPass`, o texto que recebe é o loader, não o programa protegido, que nesse ponto já é ciphertext opaco dentro de um literal de string. O `MinificationPass` não consegue ver através disso, tal como um humano a ler o código-fonte do loader também não conseguiria.

Isto não é apenas permitido; vale a pena fazê-lo duas vezes, de forma mensurável. Correr o `MinificationPass` uma vez antes do `EncryptionPass` (reduzindo o texto plano que vai ser comprimido e encriptado) e outra vez depois (reduzindo o boilerplate do próprio loader gerado) são poupanças independentes e aditivas, não escolhas redundantes ou em competição:

| Configuração (mesma fixture, `--lean`, sem `IntegrityVerificationPass`) | Tamanho final |
|---|---|
| Nenhuma | 3686 bytes |
| Minificar só antes | 3366 bytes |
| Minificar só depois | 2415 bytes |
| Minificar antes **e** depois | **2095 bytes** |

Uma comparação anterior sobre este assunto concluiu que minificar antes do `EncryptionPass` era activamente contraproducente; essa conclusão veio de uma experiência que alterou duas variáveis ao mesmo tempo (comparou "minificar antes, não depois" com "não antes, minificar depois" como se fossem opostos), em vez de isolar o efeito de cada posição. Medido isoladamente, minificar antes ajuda pelo seu próprio mérito: entrega ao `gzdeflate()` menos bytes de texto plano para comprimir. Ajuda menos do que minificar o loader depois, mas as duas coisas não estão em tensão.

Decisão: a pipeline encriptada da Camada Source é

```text
symbol-rename
-> string-protection   (omitido em --lean)
-> control-flow
-> minification         (reduz o programa protegido antes de este ser congelado)
-> encryption            (congela o programa protegido como um loader opaco)
-> minification         (reduz o boilerplate do próprio loader; não consegue ver o payload)
```

A pipeline não encriptada mantém-se inalterada:

```text
symbol-rename
-> string-protection   (omitido em --lean)
-> control-flow
-> minification
-> integrity-verification
```

Os dois passos de `minification` na pipeline encriptada são o mesmo passe, o `MinificationPass`, executado duas vezes com input diferente; não são dois passes diferentes e não são renomeados para sugerir o contrário. Qualquer passe futuro que queira correr depois do `EncryptionPass` (um hipotético `LoaderOptimizationPass`, por exemplo) tem de se justificar perante o mesmo contrato que o `MinificationPass` satisfaz aqui: operação ao nível de texto/tokens sobre a representação do próprio loader, sem visibilidade sobre nem modificação do payload encriptado. Essa justificação pertence a esta ADR ou a uma sua própria, não deve ser assumida por precedente.

### Camada Runtime: continua a não ser necessária

Nada do que foi dito acima exige abrir a Camada Runtime. O loader mantém-se totalmente auto-contido, segundo a decisão original desta ADR. A Camada Runtime abre-se quando surgir uma segunda necessidade concreta de comportamento de runtime partilhado, não de forma preventiva.

### Activação em produto (2026-08-21, mais tarde no mesmo dia): decidido

A versão anterior desta secção deixava em aberto se o `EncryptionPass` seria activado em alguma edição. Isso foi entretanto decidido:

- **Enterprise**: activado. O `src/Editions/enterprise.php` corre agora `symbol-rename -> string-protection -> control-flow -> minification -> encryption -> loader-minification`, com o `integrity-verification` removido pela razão dada acima. O `MinificationPass` ganhou uma sobreposição opcional de nome no construtor (`'loader-minification'`) especificamente para que o `PassRegistry` -- que exige que `pass->name()` corresponda exactamente ao nome com que foi registado -- consiga resolver a mesma classe sob dois nomes sem enfraquecer o `PassRegistry::assertNoDuplicates()`. Coberto de ponta a ponta pelo `EnterpriseEditionIntegrationTest`, que constrói um `PassRegistry` real, carrega `enterprise.php` através de `Edition::fromFile`, e verifica tanto a ordem de execução dos passes como a correcção funcional do output.
- **Premium**: não activado, propositadamente, não apenas por ainda não se ter chegado lá. O `premium.php` mantém-se inalterado. A encriptação total do código-fonte acarreta um custo operacional real (materialização em ficheiro temporário, desencriptação em cada request, sem reutilização persistente de OPcache nesta versão) que o posicionamento/preço do Enterprise consegue absorver sem mais dados de performance em produção do que os que existem actualmente. Esta é uma distinção de linha de produto que vale a pena ter, não uma lacuna a fechar mais tarde por simetria.

### Próximo bloqueio: ainda não há ponto de entrada de produção

Os testes a passar (actualmente 88/88 em todo o monorepo) demonstram que a pipeline está correctamente implementada e correctamente ligada através do `PassRegistry` e do `Edition`. Não demonstram que um consumidor real a consiga usar: nenhum `PassRegistry` neste código está populado com as classes `Pass` de produção fora de ficheiros de teste. O `PassRegistryTest`, o `FreeEditionIntegrationTest` e o `EnterpriseEditionIntegrationTest` constroem cada um o seu próprio registo ad hoc inline; não existe nenhum `bin/`, comando de CLI, ou ficheiro de bootstrap que faça isto uma vez para uma build real.

Isto já era verdade antes de o Enterprise activar a encriptação, mas importa mais agora: o `enterprise.php` deixou de ser configuração validada e passou a ser uma capacidade de produto não distribuída no momento em que `encryption` lhe foi adicionado. Fechar essa lacuna -- um ponto de entrada real, que recebe um caminho de código-fonte e um nome de edição, e produz output protegido -- é o próximo passo arquitectural, não mais técnicas de obfuscação da Camada Source. Esse ponto de entrada ainda não está desenhado; este parágrafo é um apontador para esse trabalho, não uma decisão sobre a sua forma.
