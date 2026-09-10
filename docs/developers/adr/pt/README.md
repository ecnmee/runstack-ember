# Registos de Decisão de Arquitectura (ADRs)

Este índice lista todas as ADRs do RunStack Ember. "Origem" indica se a decisão foi reconstruída a partir da implementação e da documentação existente (não sobreviveu nenhum documento original) ou se é um documento original escrito na altura em que a decisão foi tomada. Ver o cabeçalho de cada ADR para mais detalhe.

| ID | Título | Origem |
|---|---|---|
| [ADR-0001](0001-plataforma-de-proteccao-de-codigo-nao-um-obfuscador.md) | O produto é uma plataforma de protecção de código, não um obfuscador | Reconstruída |
| [ADR-0002](0002-sem-chaves-externas-licenciamento.md) | Sem chaves geridas externamente para licenciamento | Reconstruída |
| [ADR-0003](0003-marketing-segue-implementacao.md) | O marketing segue a implementação | Reconstruída |
| [ADR-0004](0004-aes-256-gcm-primitiva-cifra.md) | AES-256-GCM como primitiva de cifra deste projecto | Reconstruída |
| [ADR-0005](0005-ast-exigida-transformacoes-estruturais.md) | É exigida uma AST para transformações estruturais | Reconstruída |
| [ADR-0006](0006-editions-sao-configuracao.md) | Editions são configuração, não código | Reconstruída |
| [ADR-0007](0007-modelo-de-proteccao-em-camadas.md) | Modelo de protecção, em camadas segundo onde a protecção acontece | Original, com um addendum de 2026-08-24 que resolve a classificação do `loader-minification` |
| [ADR-0008](0008-verificacao-de-licenca-assimetrica.md) | A verificação de licença é assimétrica | Original |
| [ADR-0009](0009-modelo-de-ameaca.md) | Modelo de ameaça | Original |
| [ADR-0010](0010-segundo-caso-real-nao-especulativo.md) | Construir para o segundo caso real, não para o primeiro caso especulativo | Reconstruída |
| [ADR-0011](0011-encriptacao-total-do-source-estrategia-de-execucao-em-runtime.md) | Encriptação total do código-fonte / Estratégia de execução em runtime | Original, com dois addenda de 2026-08-21 (integridade/ordem, activação por edição, e o bloqueio do ponto de entrada de produção) |
| [ADR-0012](0012-ponto-de-entrada-da-pipeline-de-producao.md) | Ponto de entrada da pipeline de produção | Original |
| [ADR-0013](0013-camada-intermediate-bytecode-vm.md) | Camada Intermediate: representação em bytecode e máquina virtual | Proposta, ainda não aceite; duas das suas questões de desenho estão decididas |

As ADRs reconstruídas declaram, no seu próprio cabeçalho, que o texto histórico não foi preservado, e separam o que é sustentado por evidência do que é raciocínio reconstruído. Não devem ser lidas como documentos históricos literais.
