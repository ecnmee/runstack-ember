# ADR-0002: Sem chaves geridas externamente para licenciamento

* Estado: Aceite
* Data: 2026-08-24 (reconstruída; ver nota abaixo)
* Autores: RunStack Team
* Substitui: nenhuma
* Substituída por: nenhuma

**Nota sobre a data e origem deste documento.** Esta ADR é citada pelo número numa discussão de desenho do projecto (uma conversa sobre o desenho do `EncryptionPass` observou explicitamente que obter uma chave de encriptação a partir de "uma variável de ambiente, um servidor de licenças" iria "reabrir a gestão de chaves que a ADR-0002 já bane para o caso do licenciamento"), mas o documento original nunca foi encontrado em nenhum dos dois repositórios. Isto reconstrói a decisão que essa citação descreve, na data acima. O texto original exacto não é preservado; só a decisão em si, e o raciocínio já visível em como o resto do código se comporta de forma consistente com ela, são.

## Contexto

O licenciamento (segundo a ADR-0008, verificação de licença assimétrica) precisa de alguma forma de distinguir uma cópia validamente licenciada de um artefacto protegido de uma que não o é. A forma mais flexível de o fazer envolveria gerir chaves nalgum sítio fora do próprio artefacto protegido: uma variável de ambiente que o servidor do cliente tem de definir, um servidor de licenças que o artefacto chama em runtime, ou algum tipo de armazenamento externo de chaves. Essa flexibilidade tem um custo: exige que o produto opere, proteja e suporte infra-estrutura para além do artefacto que produz, e exige que a implantação do cliente confie e dependa dessa infra-estrutura estar acessível.

## Decisão

A verificação de licenciamento não depende de chaves geridas externamente. Nenhuma variável de ambiente, nenhuma chamada a servidor de licenças, nenhum armazenamento externo de chaves faz parte do modelo de licenciamento. Seja qual for o mecanismo que a ADR-0008 especifica para verificação assimétrica, o material de chave envolvido é embutido no artefacto no momento da build, a mesma forma de gestão de chaves já usada noutros sítios deste código para um propósito diferente: `StringProtectionPass` e `EncryptionPass` geram ambos uma chave nova por build e embutem-na (ou, no caso do encryption, geram e embutem chave e ciphertext juntos) em vez de recorrer a algo externo.

## Consequências

* Um artefacto protegido é auto-contido no que toca à verificação de licenciamento: não precisa de acesso à rede, de um ambiente configurado, nem de um serviço acessível para verificar se está validamente licenciado.
* Isto é uma fronteira de âmbito, não uma afirmação de segurança: não significa que a verificação de licença seja inquebrável, só que este produto não assume o fardo operacional de correr ou exigir infra-estrutura de servidor de licenças para tornar essa verificação possível.
* Qualquer proposta futura de acrescentar verificação de licença do lado do servidor, revogação, ou vinculação a ambiente precisa da sua própria ADR; não é compatível com esta decisão tal como está enunciada; e deve ser pesada contra o custo operacional que esta decisão foi tomada especificamente para evitar.
