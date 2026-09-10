# ADR-0009: Modelo de ameaça

* Estado: Aceite
* Data: 2026-07-10
* Autores: RunStack Team
* Substitui: nenhuma
* Substituída por: nenhuma

## Contexto

A ADR-0003 exige que toda a afirmação de marketing seja sustentada por uma garantia específica e testável. Essa exigência fica incompleta sem um modelo de ameaça declarado, porque "protecção" não tem significado sem dizer contra quem protege. Sem isto, é fácil recair no tipo de afirmação sem qualificação ("99% de protecção") que a ADR-0003 existe precisamente para evitar, bastando descrever uma funcionalidade real de forma ilimitada.

## Decisão

O RunStack Ember está desenhado para aumentar o custo de ataques casuais e semi-automatizados. Não afirma, explicitamente, impedir um atacante dedicado e com recursos suficientes.

Dentro do âmbito, o produto está desenhado para atrasar ou bloquear de forma significativa:

- A inspecção casual do código-fonte por alguém com conhecimento básico de PHP e sem ferramentas especializadas.
- Análise estática automatizada e decompiladores genéricos que não foram construídos especificamente para o output do Ember.
- A redistribuição ou reutilização não autorizada de código licenciado por utilizadores finais da aplicação protegida.
- A adulteração casual de licenças (editar um ficheiro de licença, contornar uma verificação óbvia).

Fora do âmbito, o produto não afirma impedir:

- Um atacante dedicado e tecnicamente competente, com tempo ilimitado, disposto a fazer engenharia reversa manual do runtime e de quaisquer passes de protecção aplicados a uma build específica.
- Um atacante com acesso root ou físico à máquina que corre a aplicação protegida no momento em que esta desencripta e executa.
- Ataques de canal lateral (side-channel) contra o runtime PHP ou o sistema operativo subjacente.

Qualquer funcionalidade ou afirmação de marketing declara em que lado desta fronteira se situa. As afirmações são escritas em termos do atacante que se destinam a impedir, não em termos de uma percentagem abstracta.

## Consequências

- A documentação e o texto de marketing descrevem capacidades contra um adversário declarado ("impede ferramentas de análise estática automatizada que não conhecem o Ember") em vez de afirmações de força sem qualificação ("99% de protecção").
- As conversas de vendas podem definir expectativas correctas do cliente desde o início, o que reduz o risco de credibilidade que motivou a ADR-0003 em primeiro lugar.
- Futuros passes de protecção são avaliados contra este modelo de ameaça: uma funcionalidade proposta que só ajuda contra um atacante já fora do âmbito é despriorizada a favor de funcionalidades que aumentem o custo para atacantes dentro do âmbito.
