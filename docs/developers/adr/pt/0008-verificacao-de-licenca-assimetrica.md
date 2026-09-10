# ADR-0008: A verificação de licença é assimétrica

* Estado: Aceite
* Data: 2026-07-10
* Autores: RunStack Team
* Substitui: nenhuma
* Substituída por: nenhuma

## Contexto

A ADR-0002 bane, como regra geral, segredos partilhados e fixos no código para a assinatura de licenças, em resposta a uma falha específica: a implementação anterior assinava e verificava licenças com o mesmo segredo HMAC, e esse segredo era distribuído dentro do código que todos os clientes recebiam. O licenciamento é suficientemente importante e específico para justificar a sua própria decisão, para que o esquema correcto fique documentado uma vez, com precisão, em vez de ser re-derivado de um princípio geral cada vez que alguém toca em código de licenciamento.

## Decisão

A assinatura e verificação de licenças usa um par de chaves assimétrico, não um segredo partilhado:

- **Algoritmo**: Ed25519.
- **Chave privada**: gerada uma vez por ambiente de assinatura, nunca sai do serviço de emissão de licenças, nunca aparece em nenhum repositório, pacote ou artefacto distribuído.
- **Chave pública**: embutida no código distribuído a todos os clientes, independentemente do nível. A sua presença em código do lado do cliente é segura por desenho, já que uma chave pública não pode ser usada para forjar uma assinatura.
- **Verificação**: totalmente offline. A aplicação de um cliente verifica localmente a assinatura de uma licença usando a chave pública embutida, sem uma chamada de rede a um servidor de licenças, embora um servidor de licenças possa existir separadamente para outros fins (limites de activação, telemetria, listas de revogação).
- **Formato de licença**: uma estrutura determinística e versionada (por exemplo: ID de licença, nível, emitida para, expiração, feature flags) que é serializada sempre da mesma forma antes de ser assinada, para que a verificação seja reproduzível entre versões de PHP e plataformas.

## Consequências

- A emissão de licenças exige acesso à chave privada de assinatura, o que significa que só pode acontecer a partir de um ambiente controlado, não de qualquer máquina que calhe ter o código-fonte.
- Um ficheiro de licença de cliente que seja exposto permite que alguém veja o que essa licença contém, mas não crie uma nova licença válida, ao contrário do esquema HMAC anterior, em que o próprio validador já era suficiente.
- Revogar uma chave privada comprometida exige rodar o par de chaves e reemitir a chave pública para lançamentos futuros; instalações offline existentes com a chave pública antiga continuam a validar licenças antigas até actualizarem, o que é um compromisso conhecido e aceite da verificação offline.
