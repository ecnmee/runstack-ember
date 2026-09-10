# ADR-0006: Editions são configuração, não código

* Estado: Aceite
* Data: 2026-08-24 (reconstruída; ver nota abaixo)
* Autores: RunStack Team
* Substitui: nenhuma
* Substituída por: nenhuma

**Nota sobre a data deste documento.** Esta ADR é citada pelo nome em todo o código-fonte (`src/Editions/free.php` e todos os outros ficheiros de edition: "Editions are configuration, not code (ADR-0006)") e nos próprios docblocks da `PassRegistry`, mas o documento original nunca foi encontrado em nenhum dos dois repositórios. Isto é uma reconstrução da decisão que essas citações já descrevem de forma consistente, escrita na data acima, não uma afirmação de que este texto exacto existiu antes. Onde o texto original é desconhecido, este documento apresenta a decisão tal como o código já a aplica, não como história inventada.

## Contexto

Antes desta decisão (como ADRs posteriores, incluindo a ADR-0007, descrevem), os níveis comerciais e a implementação técnica estavam emaranhados. Adicionar uma técnica de protecção a um nível significava mexer em código que também decidia que clientes a podiam usar, tornando os dois assuntos difíceis de separar ou raciocinar independentemente.

## Decisão

Uma edition é uma lista simples e ordenada de nomes de passes, nada mais:

```php
return [
    'symbol-rename',
    'string-protection',
    'control-flow',
    'minification',
    'encryption',
    'loader-minification',
];
```

Todos os ficheiros de edition em `src/Editions/` (`free.php`, `basic.php`, `premium.php`, `enterprise.php`) seguem exactamente esta forma: um ficheiro PHP que devolve uma `list<string>`. Não contém instanciação de classes, lógica condicional, nem referência a qual classe implementa cada pass que nomeia. A `PassRegistry` (ver `PassRegistry::withDefaults()`, ADR-0012) é a única responsável por resolver cada nome numa instância concreta de `Pass`; `Edition::fromFile()` é o único responsável por carregar e validar a própria lista (ficheiro existe, devolve um array, cada elemento é uma string).

Esta separação significa que uma edition pode ser lida, comparada e compreendida por qualquer pessoa, sem precisar de saber uma única linha de PHP além do que significa uma lista de strings, e a implementação de um pass pode mudar por completo (como já aconteceu com `EncryptionPass` e `MinificationPass`) sem tocar num único ficheiro de edition.

## Consequências

* Um nível comercial é uma selecção de nomes de passes, nunca um sítio onde vive lógica de protecção. Adicionar "esta edition passa a ter encryption" é uma edição de uma linha numa lista, não uma alteração de código a nenhum pass.
* A `PassRegistry` rejeita um nome que não reconhece (`OutOfBoundsException`) e um nome duplicado dentro da mesma edition (`LogicException`, `assertNoDuplicates`), por isso uma edition mal formada falha ruidosamente no momento de construção, em vez de silenciosamente fazer menos do que era pretendido.
* É por isto que `MinificationPass` precisou de um segundo nome registado (`loader-minification`) em vez de um caso especial dentro de algum ficheiro de edition, quando passou a correr duas vezes na pipeline cifrada (addendum da ADR-0011, addendum da ADR-0007): o formato de edition não tem espaço para nada além de uma lista plana de nomes, por desenho, por isso a segunda utilização teve de ser um segundo nome, não um parâmetro ou uma flag.
