# ADR-0005: É exigida uma AST para transformações estruturais

* Estado: Aceite
* Data: 2026-08-24 (reconstruída; ver nota abaixo)
* Autores: RunStack Team
* Substitui: nenhuma
* Substituída por: nenhuma

**Nota histórica.** O documento original da ADR não foi preservado. O pareamento deste número com esta decisão é corroborado de forma independente, não inferido uma única vez: `SymbolRenamePass`, `ControlFlowPass`, `StringProtectionPass`, e `AbstractAstPass` citam cada um "(ADR-0005)" directamente, nos seus próprios docblocks, todos a descrever a mesma regra. `MinificationPass` cita-a também, para explicar porque é o único pass que não a segue. O que não é preservado é a redacção histórica original, só a decisão, corroborada a partir de cinco sítios de citação independentes.

## Decisão apoiada por evidência

Um pass que transforma estruturalmente código PHP, no sentido de mudar o que o código significa ou como os símbolos se relacionam entre si, tem de operar sobre uma AST produzida pelo `nikic/php-parser`, não sobre tokens, não sobre expressões regulares, e não sobre nenhuma outra representação ao nível do texto. `SymbolRenamePass`, `ControlFlowPass`, `StringProtectionPass`, e `IntegrityVerificationPass` estendem todos `AbstractAstPass`, que possui a sequência partilhada de parse-transform-print, especificamente para que esta regra seja aplicada estruturalmente, não só por convenção.

`MinificationPass` é a excepção documentada e deliberada: segundo o seu próprio docblock, "ADR-0005 requires an AST for structural transformations; minification is not structural, it never changes what the code means, only how much whitespace surrounds it." Usa o tokenizer do próprio PHP em vez disso, porque a regra que esta ADR enuncia está limitada a mudança estrutural, e a minificação não muda nada estruturalmente.

## Racional reconstruído

Expressões regulares e manipulação ingénua de texto erram a própria sintaxe do PHP em casos que um padrão escrito à mão não vai antecipar: literais string que contêm texto parecido com código, heredocs, chavetas aninhadas, e nomes qualificados por namespace são os casos concretos já tratados correctamente noutros sítios deste código precisamente porque foi usada uma AST em vez disso. Porque é que este projecto se decidiu especificamente pelo `nikic/php-parser`, em vez de outro parser, não é recuperável a partir da evidência disponível; só que todos os passes baseados em AST neste código o usam, de forma consistente.

## Consequências

* Qualquer pass futuro cujo trabalho seja mudar o que o código significa, não só o seu aspecto, precisa de uma razão enunciada se propuser não estender `AbstractAstPass` e não trabalhar a partir de uma AST real. Excepções ao estilo da minificação existem, e são legítimas, mas são excepções a defender individualmente, tal como o próprio docblock do `MinificationPass` já faz, não uma opção por defeito.
* É também por isto que `EncryptionPass`, que substitui o ficheiro inteiro por um loader opaco em vez de transformar o significado do código lá dentro, foi classificado como não precisando de estender `AbstractAstPass` também (ADR-0011): não está a fazer transformação estrutural ao nível da AST do programa protegido, está a substituir a representação inteira, um tipo de operação diferente que o âmbito desta ADR nunca pretendeu cobrir.
