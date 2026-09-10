# ADR-0010: Construir para o segundo caso real, não para o primeiro caso especulativo

* Estado: Aceite
* Data: 2026-08-24 (reconstruída; ver nota abaixo)
* Autores: RunStack Team
* Substitui: nenhuma
* Substituída por: nenhuma

**Nota histórica.** O documento original da ADR não foi preservado. O pareamento deste número com esta decisão é corroborado de forma independente em quatro ficheiros separados: `Representation.php`, `Edition.php`, `SourceCode.php`, e `AbstractAstPass.php` citam cada um "ADR-0010" directamente, descrevendo o mesmo princípio em cada caso. O que não é preservado é a redacção histórica original, só a decisão, corroborada a partir de quatro sítios de citação independentes, e aplicada de forma consistente em todas as decisões arquitecturais tomadas também no trabalho posterior deste projecto.

## Decisão apoiada por evidência

Uma abstracção, um membro de interface, um campo de configuração, ou uma camada estrutural é introduzida quando se conhece uma segunda necessidade real e concreta para ela, não quando é imaginada pela primeira vez como plausivelmente útil. Esperar por um segundo caso, em vez de construir para um primeiro caso especulativo, é a regra; quatro sítios independentes no código aplicam-na explicitamente:

* `Representation`, a interface marcadora que todo o payload da pipeline implementa, está deliberadamente vazia: "Concrete representations... are introduced once each layer has its first real pass and their actual shape is known, instead of being designed speculatively today."
* `Edition` transporta hoje só um nome e uma lista de passes: "Additional fields (version, feature flags, limits) are added here once a real edition needs them, rather than speculatively now."
* `SourceCode`, a primeira `Representation` concreta, foi "introduced now because MinificationPass, the first real Source layer pass, needs it, not before."
* O próprio `AbstractAstPass` foi "introduced after four concrete passes had already implemented this exact parse/transform/print sequence independently, not before... four was well past that bar."

## Racional reconstruído

Estrutura especulativa, construída para uma necessidade que ainda não apareceu, tende a adivinhar mal a forma que essa necessidade acaba por tomar, e a adivinhação tem de ser desfeita ou contornada mais tarde. Esperar por uma segunda instância concreta antes de generalizar significa que a abstracção é construída a partir de dois exemplos reais em vez de um imaginado, o que a própria história deste código confirma directamente: `AbstractAstPass` não existiu até quatro passes separados terem já, independentemente, implementado a mesma sequência de parse-transform-print, altura em que a forma partilhada deixou de ser uma adivinhação.

O limiar exacto ("segundo caso", especificamente, em vez de terceiro ou quinto) não é justificado em separado em lado nenhum recuperável; é simplesmente o número usado de forma consistente em todas as citações encontradas.

## Consequências

* Este princípio foi aplicado repetidamente no trabalho neste projecto depois de este esforço de reconstrução ter começado, ainda que nem sempre com este número de ADR citado explicitamente na altura: a decisão de não introduzir uma classe facade para o entry point na ADR-0012 ("construir a abstracção quando aparecer uma segunda necessidade concreta, não preventivamente"), e a decisão de não abrir uma implementação da Runtime Layer no addendum da ADR-0011 até aparecer uma segunda necessidade concreta de comportamento de runtime partilhado, reafirmam ambas esta mesma regra por outras palavras.
* Quem propõe uma nova abstracção, interface, ou campo de configuração deve conseguir nomear o segundo caso concreto que a motiva, não só o primeiro. "Isto pode vir a ser útil mais tarde" não é, por si só, justificação suficiente ao abrigo desta decisão.
* Isto é um valor por defeito geral do projecto, não uma regra absoluta sem excepções: um argumento para construir antes do segundo caso concreto deve ser defendido explicitamente, nos seus próprios termos, em vez de ser assumido como proibido à partida.
