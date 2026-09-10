# Documento de Requisitos de Produto: RunStack Ember

* Estado: Rascunho
* Data: 2026-09-09
* Autores: RunStack Team
* Origem: Escrito a partir da visão, do roadmap e das ADRs actuais; não é uma reconstrução de um documento histórico

**Nota sobre este documento.** Ao contrário das ADRs, este não é uma reconstrução de algo que já existisse antes. Não foi encontrado nenhum PRD anterior em nenhum dos dois repositórios. Este documento é uma síntese nova, escrita agora, de requisitos que a visão, o roadmap e as ADRs do projecto já estabelecem. Onde um requisito remete para uma ADR específica, essa ADR é citada; onde este documento afirma algo ainda não sustentado por uma ADR ou por um teste, di-lo explicitamente.

## Propósito

O RunStack Ember é uma plataforma de protecção de código para aplicações PHP: transformações de AST, encriptação, protecção em runtime, licenciamento e reforço de aplicação numa única pipeline de build, em vez de ferramentas separadas acrescentadas a um obfuscador de código-fonte (Visão).

## Utilizadores-alvo

- Fornecedores de aplicações PHP que distribuem o seu código a clientes ou a instalações self-hosted, e que precisam de aumentar o custo da inspecção casual, da redistribuição e da engenharia reversa.
- Equipas que hoje dependem apenas de obfuscação, e que precisam de licenciamento e detecção de adulteração como preocupações de produto de primeira classe, e não como extras externos.

## Âmbito do produto

### Dentro do âmbito (o que o Ember é, segundo a Visão e a ADR-0007)

- Uma pipeline de passes de protecção organizada em quatro camadas: Source, Intermediate, Runtime, Packaging.
- Níveis comerciais (edições) que são configuração, não bases de código separadas: cada nível é uma lista nomeada e ordenada de nomes de passes (ADR-0006).
- Verificação de licença offline e assimétrica, sem segredo partilhado e sem chave gerida externamente (ADR-0002, ADR-0008).
- Toda a afirmação de protecção declarada contra um modelo de ameaça explícito, não como uma percentagem sem qualificação (ADR-0003, ADR-0009).

### Fora do âmbito (o que o Ember não é, segundo a Visão e a ADR-0009)

- Impedir um atacante dedicado e tecnicamente competente, com tempo ilimitado e acesso root ou físico à aplicação em execução (ADR-0009).
- Criptografia proprietária ou "inventada"; apenas primitivas padrão e validadas (ADR-0004).
- Um único modelo monolítico de "Nível N"; o produto está organizado por camada e por passe, não por um nível de protecção indiferenciado.

## Edições

As edições activam subconjuntos diferentes dos mesmos passes; há um motor, não um motor por nível (Visão, ADR-0006).

| Edição | Encriptação (`EncryptionPass`) | Notas |
|---|---|---|
| Free | Não activada | Protecção base da camada Source. |
| Basic | Não activada | |
| Premium | Não activada, por decisão deliberada | A encriptação total do código-fonte acarreta um custo operacional real (materialização em ficheiro temporário, desencriptação em cada request); o posicionamento/preço do Premium não absorve actualmente esse custo (addendum da ADR-0011). |
| Enterprise | Activada | Corre `symbol-rename -> string-protection -> control-flow -> minification -> encryption -> loader-minification`; o `integrity-verification` não é usado em conjunto com a encriptação (addendum da ADR-0011). |

Esta tabela reflecte o código actual (`src/Editions/*.php`) à data das ADRs citadas, e não um compromisso de manter exactamente esta forma à medida que o produto evolui.

## Requisitos funcionais

1. **Proteger código-fonte PHP sem alterar o seu comportamento.** O output de cada passe tem de ser PHP semanticamente equivalente ao seu input, verificado por testes, não apenas afirmado pela documentação.
2. **Licenciar uma build sem chamada de rede.** A verificação tem de ter sucesso totalmente offline, usando uma chave pública embutida (ADR-0008).
3. **Detectar adulteração de forma adequada à edição.** As edições não encriptadas usam o `IntegrityVerificationPass`; as edições encriptadas dependem da autenticação embutida do AES-256-GCM, já que os dois mecanismos se sobrepõem naquilo contra que protegem (addendum da ADR-0011).
4. **Produzir output protegido através de um único ponto de entrada por interface.** A CLI (e qualquer API futura) tem de usar a mesma composição `Edition` / `PassRegistry` / `Pipeline`; nenhuma das duas pode construir directamente um `Pass` nem fixar a ordem de passes de uma edição fora de um ficheiro de edição (ADR-0012).
5. **Falhar de forma legível.** Erros de utilização, PHP de entrada malformado, e falhas de composição de protecção têm de ser distinguíveis por quem chama; falhas inesperadas têm de aparecer com o seu tipo real e stack trace, em vez de serem mascaradas (ADR-0012).
6. **Nunca destruir a única cópia do código-fonte de um cliente.** O output assume por defeito um nome de ficheiro derivado; uma colisão entre caminho de entrada e de saída é recusada, não sobrescrita silenciosamente (ADR-0012).

## Não-objectivos (para o âmbito actual deste documento)

- Comprometer-se com um desenho de API HTTP. A composição tem de se manter suficientemente agnóstica de adaptador para suportar uma mais tarde, sem decidir a sua forma agora (ADR-0012).
- Um agendador de execução ordenado por camada. `layer()` é classificação, não um mecanismo de agendamento; a ordem de execução é a que uma edição declarar (ADR-0012).
- Uma camada Intermediate de bytecode/virtualização. Planeada (Visão, Roadmap Fase 2), ainda não desenhada ao nível de requisitos.

## Questões em aberto

- Se `--lean` (ou equivalente) chega alguma vez a ser exposta como flag da CLI, versus puramente como escolha de edição (ADR-0012).
- A forma e o âmbito de um futuro servidor de licenças para limites de activação e revogação, para além da verificação offline de assinaturas já decidida (Roadmap Fase 3).

## Referências

Visão, Roadmap, ADR-0002, ADR-0003, ADR-0004, ADR-0006, ADR-0007, ADR-0008, ADR-0009, ADR-0011, ADR-0012.
