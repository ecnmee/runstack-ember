# ADR-0001: O produto é uma plataforma de protecção de código, não um obfuscador

* Estado: Aceite
* Data: 2026-08-24 (reconstruída; ver nota abaixo)
* Autores: RunStack Team
* Origem: Reconstruída a partir da implementação e da documentação do projecto
* Texto histórico: Não preservado
* Substitui: nenhuma
* Substituída por: nenhuma

**Nota sobre a data e origem deste documento.** Não foi encontrado, em nenhum dos dois repositórios, nenhum documento original da ADR-0001, datado da história mais antiga deste projecto ou próximo dela. O que é recuperável é a decisão em si, visível de forma consistente no código e na sua documentação pública, não a deliberação original nem a sua redacção exacta. Este documento é escrito na data acima e declara a decisão tal como o projecto já a aplica. Não afirma reproduzir um documento histórico que não foi preservado, e onde a evidência se esgota, este documento diz-o em vez de preencher a lacuna com história inventada.

## Contexto

O próprio `composer.json` do produto descreve-o como uma "plataforma de protecção de código PHP (passes de source, intermediate, runtime e packaging)", não como um obfuscador, e o slogan do README público é "Beyond obfuscation" ("mais do que obfuscação"). O namespace do pacote é `RunStack\Ember`, sem qualquer vestígio de um nome enraizado em `Obfuscator` em todo o código actual. A ADR-0007 (modelo de protecção) organiza o código em quatro camadas, Source, Intermediate, Runtime, Packaging, uma estrutura que só faz sentido para uma plataforma com mais do que um tipo de técnica de protecção. A ADR-0006 (as edições são configuração) e a ADR-0008 (verificação de licença assimétrica) pressupõem ambas o licenciamento como uma preocupação de produto de primeira classe, não como um extra acrescentado a um obfuscador.

Além disto, as circunstâncias específicas que levaram a definir o âmbito do produto desta forma, como se chamava antes, o que foi considerado e rejeitado, não são recuperáveis a partir da evidência disponível.

## Decisão sustentada por evidência

O produto é o **RunStack Ember**, descrito de forma consistente nos seus próprios metadados de pacote e na documentação pública como uma plataforma de protecção de código para aplicações PHP, não como um obfuscador. Cada componente distribuído sob este nome corresponde a uma das preocupações já visíveis na própria estrutura e terminologia do código: obfuscação (a camada Source), encriptação, licenciamento, protecção em runtime, verificação de integridade, pipeline de build. O namespace, `RunStack\Ember`, não contém nenhuma referência a obfuscação como termo definidor do produto.

## Raciocínio reconstruído a partir da evidência disponível

Uma implementação anterior, referida por outras ADRs (por exemplo, o relato da ADR-0002 sobre um segredo HMAC partilhado, distribuído dentro do código) mas ausente de ambos os repositórios examinados para esta reconstrução, é descrita noutros pontos como tendo crescido funcionalidade a funcionalidade, sem um âmbito de produto explícito. Definir o âmbito da actual reescrita como uma plataforma, em vez de uma ferramenta única, é consistente com esse relato e com a amplitude de preocupações que o código e as suas ADRs já tratam como centrais (as ADRs 0006 a 0012, no conjunto, cobrem edições, licenciamento, modelação de ameaça e um modelo de protecção multi-camada, e não apenas obfuscação).

Se o produto chegou a ser formalmente definido em torno de um conjunto específico e enumerado de pilares, e, caso sim, qual era esse conjunto originalmente, não é recuperável a partir da evidência disponível. O que pode ser afirmado com confiança é que o código actual e a sua documentação pública tratam consistentemente o produto como mais do que um obfuscador, em todos os artefactos examinados.

## Consequências

- O README, os metadados do pacote e a documentação pública descrevem uma plataforma, não uma ferramenta única, de forma consistente com a evidência acima.
- Espera-se que novas funcionalidades correspondam a uma preocupação já visível na estrutura do código (camadas Source, Intermediate, Runtime, Packaging, segundo a ADR-0007; licenciamento, segundo a ADR-0008), em vez de serem justificadas contra uma lista enumerada que este documento não pode confirmar ter existido, em concreto, nessa forma.
- O namespace e o nome do pacote (`RunStack\Ember`) não contêm nenhuma referência a obfuscação como termo definidor do produto, e qualquer decisão futura de nomenclatura deve preservar isso.
