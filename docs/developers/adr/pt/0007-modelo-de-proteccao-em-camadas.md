# ADR-0007: Modelo de protecção, em camadas segundo onde a protecção acontece

* Estado: Aceite
* Data: 2026-07-10
* Autores: RunStack Team
* Substitui: nenhuma
* Substituída por: nenhuma

## Contexto

A ADR-0006 define as técnicas de protecção como passes numa pipeline, o que responde a *como* uma técnica é implementada e ordenada. Não responde a uma pergunta que surge sempre que uma nova técnica é proposta: *onde é que isto pertence?* Sem um modelo explícito, essa pergunta é respondida de forma ad hoc, e a fronteira entre técnicas vai-se desfazendo, da mesma forma que os níveis comerciais e a implementação técnica estavam emaranhados na implementação anterior.

## Decisão

Cada técnica de protecção pertence exactamente a uma de quatro camadas, definidas por quando e onde actua sobre a aplicação:

- **Camada Source (código-fonte)**: transformações aplicadas ao código-fonte PHP antes de este correr, operando sobre a AST. Exemplos: minificação, renomeação de símbolos, protecção de strings, alterações de fluxo de controlo.
- **Camada Intermediate (intermédia)**: transformações aplicadas a uma representação compilada ou serializada do código, depois da camada Source, antes do empacotamento. Exemplos: geração de bytecode, virtualização.
- **Camada Runtime (execução)**: verificações e comportamento que executam enquanto a aplicação protegida corre. Exemplos: verificação de integridade, anti-debugging, aplicação de licenciamento, vinculação a ambiente.
- **Camada Packaging (empacotamento)**: como o artefacto protegido é montado e distribuído. Exemplos: encriptação do payload final, geração do loader, construção de PHAR.

Um passe declara a que camada pertence. Uma técnica que pareça abranger duas camadas é um sinal de que deve ser dividida em dois passes, um por camada, ligados através da pipeline em vez de fundidos num só.

## Consequências

- Quem propõe uma nova técnica de protecção declara a sua camada antes de escrever código, o que faz surgir questões de desenho mais cedo (por exemplo: "esta verificação deve correr no momento do empacotamento ou em runtime?").
- As quatro camadas passam a ser a estrutura organizadora tanto do código (`src/Source`, `src/Intermediate`, `src/Runtime`, `src/Packaging`) como da documentação, substituindo "Nível N" como forma principal de descrever o produto.
- Os níveis comerciais mantêm-se como uma configuração por cima deste modelo, segundo a ADR-0006: um nível é uma selecção de passes através destas quatro camadas, não é uma camada nem um nível em si mesmo.

## Addendum (2026-08-24): o que `Layer` descreve, e o caso `loader-minification`

O texto original desta ADR diz que cada passe pertence a uma camada "definida por quando e onde actua sobre a aplicação". Essa frase nunca foi tornada mais precisa do que as quatro definições em lista já são, e as duas entradas de registo do `MinificationPass`, `minification` e `loader-minification` (ADR-0012), são o primeiro caso em que essa imprecisão importa: ambos os registos resolvem para a mesma classe, com comportamento `process()` idêntico, transformando uma representação que é sintacticamente código-fonte PHP em ambos os casos. Nada na própria classe muda entre os dois. Algo mais tem de explicar porque é que deveriam ter valores de `Layer` diferentes, se é que deveriam.

### A regra: posição na sequência de build E representação tocada, não apenas o tipo de artefacto

"Quando" e "onde" são duas perguntas separadas, e uma regra baseada apenas numa delas é mais restrita do que aquilo a que o texto original desta ADR já se tinha comprometido:

* **Quando**: o programa protegido já foi congelado na sua forma distribuída (depois do `EncryptionPass`), ou ainda não?
* **Onde**: o passe está a tocar no próprio programa protegido, ou num wrapper/loader gerado à sua volta?

Para todos os passes existentes hoje neste código, as duas perguntas coincidem na resposta, pelo que uma regra baseada em qualquer uma delas isoladamente daria o mesmo resultado que a regra combinada. Isso não tem necessariamente de continuar a ser verdade. Um passe hipotético futuro poderia tocar no mesmo tipo de representação (por exemplo, texto de loader gerado) em dois pontos diferentes da sequência por razões que nada têm a ver com empacotamento, e uma regra baseada apenas em "qual artefacto" não teria nada a dizer sobre esse caso. A regra combinada tem.

### Resolução: `loader-minification` é `Layer::Packaging`

Aplicar as duas perguntas a todos os passes actualmente registados, não apenas ao que está em questão, mostra a mesma regra a manter-se consistente, em vez de ser inventada para se ajustar a um único caso:

| Registo | Quando | Onde | Camada |
|---|---|---|---|
| `minification` | antes do programa protegido ser congelado | o próprio programa protegido | `Source` |
| `encryption` | enquanto o artefacto é montado | payload mais loader | `Packaging` |
| `loader-minification` | depois do programa protegido ser congelado | o loader que o `EncryptionPass` produziu | `Packaging` |
| `integrity-verification` | enquanto a aplicação protegida executa | a aplicação em execução | `Runtime` |

O `loader-minification` reportar `Layer::Source` estava incorrecto. Deveria reportar `Layer::Packaging`: corre depois do programa protegido já estar congelado num payload opaco, e toca apenas no loader que o `EncryptionPass` gerou, segundo os próprios exemplos de Packaging desta ADR ("como o artefacto protegido é montado"). Reduzir o boilerplate do loader faz parte de montar o artefacto final distribuído, no mesmo sentido em que a geração do loader pelo `EncryptionPass` já o é.

Para ser explícito sobre o sentido deste raciocínio, já que é fácil enunciá-lo ao contrário: a classificação decorre daquilo sobre que a instância opera e de quando o faz, não do nome que calha ter. O `loader-minification` é `Layer::Packaging` porque a instância desse registo transforma um artefacto da camada Packaging; o nome não causa essa classificação, existe para que a configuração possa declarar em voz alta aquilo que já era verdade. Um registo chamado, por exemplo, `second-minification-pass`, tocando exactamente no mesmo loader, teria exactamente o mesmo valor `Layer::Packaging`, pela mesma razão exacta. Nada em `layer()` deve ser justificado apontando para `name()`.

### Porque isto não é "uma técnica a abranger duas camadas", e não exige dividir o passe

O texto original desta ADR diz que uma técnica que pareça abranger duas camadas deve ser dividida em dois passes, um por camada, em vez de fundida num só. O `MinificationPass` usado sob dois nomes parece, à primeira vista, exactamente esse caso. Não é, por uma razão específica que vale a pena tornar explícita em vez de deixar implícita: essa orientação protege contra um passe cujo comportamento `process()` bifurca a partir da camada em que está a participar, tipicamente inspeccionando o contexto da pipeline para decidir "estou a correr antes ou depois de X" e a agir de forma diferente em consequência. Isso é um verdadeiro sinal de mau desenho, porque esconde uma decisão dependente da camada dentro do fluxo de controlo de uma única classe, em vez de a expor como dois passes declarados e pensados separadamente.

O `MinificationPass` não faz isto. O seu método `process()` não tem nenhuma bifurcação sobre sob que nome foi registado, nem nenhuma bifurcação sobre algo parecido com "em que camada estou"; a remoção de espaços em branco e comentários ao nível dos tokens é idêntica independentemente da representação que lhe seja entregue. Dividi-lo agora em duas classes produziria duas classes com implementações byte a byte idênticas, diferindo apenas nos metadados (`name()`, e agora `layer()`) associados, o que é duplicação sem qualquer redução correspondente de bifurcação escondida, já que nunca houve bifurcação nenhuma a remover. Uma classe, registada duas vezes com metadados distintos por registo, é a forma melhor para este caso específico. A orientação de divisão mantém-se válida para o caso para que foi escrita; este não é esse caso.

### A implementação não é decidida aqui

Este addendum resolve a questão semântica e a classificação a que ela conduz. Não decide como é que `layer()` deve reportar um valor que agora precisa de variar por registo em vez de ser fixo por classe, da mesma forma que `name()` já faz através da sobreposição no construtor do `MinificationPass`. Uma implementação de `layer()` que faça um switch interno sobre `$this->name` faria com que `name()` controlasse implicitamente a classificação arquitectural, um acoplamento indirecto que vale a pena evitar. A alternativa (um segundo parâmetro explícito no construtor, `new MinificationPass('loader-minification', Layer::Packaging)`, ou um mecanismo explícito equivalente) é uma pequena alteração de desenho, não apenas uma correcção de uma linha, e fica reservada para o seu próprio passo de implementação depois de este addendum ser aceite, não decidida como efeito colateral dele.
