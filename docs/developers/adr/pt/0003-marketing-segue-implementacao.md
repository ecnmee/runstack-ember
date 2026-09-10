# ADR-0003: O marketing segue a implementação

* Estado: Aceite
* Data: 2026-08-24 (reconstruída; ver nota abaixo)
* Autores: RunStack Team
* Substitui: nenhuma
* Substituída por: nenhuma

**Nota sobre a origem deste documento.** Esta ADR é citada quase textualmente no README público anterior do projecto: "Per ADR-0003, a capability is documented here only once it has a working implementation and a test that exercises the specific claim, so this README will grow slowly and deliberately rather than all at once." O número e esta frase específica aparecem juntos, o que é uma evidência mais forte do que a maioria das decisões reconstruídas neste conjunto. A estrutura em redor (Contexto, Consequências) abaixo foi escrita para corresponder ao formato de ADR deste projecto; a decisão em si são as próprias palavras deste projecto, recuperadas, não inventadas.

## Contexto

É fácil que a documentação pública de um projecto, sobretudo um README destinado a representar o produto, descreva capacidades como se já existissem só porque estão planeadas, em curso, ou meramente plausíveis. Uma vez que isso aconteça, cada afirmação futura no mesmo documento fica suspeita: quem lê não tem forma de distinguir que frases descrevem software a funcionar e que frases descrevem intenção.

## Decisão

Uma capacidade só é documentada no material público deste projecto depois de ter uma implementação real e um teste que exercite especificamente a alegação feita. Não "a funcionalidade está planeada", não "a arquitectura suporta isso", não "devia funcionar": primeiro, um teste específico a passar.

É por isto que o README público actual não afirma que existe um CLI até `bin/ember` e o `EmberCliTest` (ADR-0012) existirem os dois, não afirma que `encryption` está disponível em todas as editions até o ficheiro de cada edition o dizer e um teste de integração o exercitar, e não afirma que o licenciamento está a ser aplicado (ADR-0008) enquanto `src/Licensing/` continuar a ser um directório vazio e reservado.

## Consequências

* A documentação pública cresce devagar e deliberadamente, ao ritmo do que é realmente construído, não toda de uma vez à frente disso.
* Uma alegação desactualizada ou aspiracional num README ou noutro documento público é tratada como um bug de documentação, a mesma categoria de problema que uma classificação de ADR desactualizada (ver o addendum da ADR-0007), não uma simplificação inofensiva.
* Isto não proíbe discutir planos, roadmaps, ou direcção. Restringe especificamente afirmações de capacidade presente: uma secção de roadmap que diz "planeado" não viola esta decisão; uma lista de funcionalidades que omite essa palavra e deixa uma capacidade planeada ler-se como já entregue, viola.
