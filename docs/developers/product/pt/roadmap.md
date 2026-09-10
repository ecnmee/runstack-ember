# Roadmap

Este roadmap descreve intenção, não funcionalidade já distribuída. Segundo a ADR-0003, nada aqui é anunciado como funcionalidade de produto até ter uma implementação a funcionar e um teste que exercite a afirmação específica.

## Fase 0: Fundação

- Pipeline e gestor de passes (ADR-0006).
- Passes da camada Source: minificação, renomeação de símbolos, protecção de strings, usando `nikic/php-parser` (ADR-0005).
- Assinatura de licenças assimétrica e verificação offline (ADR-0008).
- Pipeline de CI a correr testes unitários ao nível dos passes.

## Fase 1: Runtime e packaging

- Camada Runtime: verificação de integridade, verificações de aplicação de licenciamento.
- Camada Packaging: encriptação do artefacto final, usando AES-256-GCM (ADR-0004), com gestão de chaves que satisfaz os padrões banidos na ADR-0002.
- Passes de fluxo de controlo na camada Source.
- **Estado**: a encriptação total do código-fonte (`EncryptionPass`) está implementada e activada na edição Enterprise, correndo através de um loader de runtime auto-contido (ADR-0011). O `IntegrityVerificationPass` e o `EncryptionPass` não correm juntos; as edições escolhem um ou o outro (addendum da ADR-0011). O Premium não activa a encriptação, por decisão de produto deliberada, e não como uma lacuna ainda por fechar (addendum da ADR-0011).

## Fase 2: Camada intermédia

- Representação em bytecode e uma máquina virtual real baseada em dispatch, substituindo a VM não funcional da implementação anterior.
- Benchmarks comparando código protegido e não protegido, publicados junto da metodologia.

## Fase 3: Plataforma

- Superfície de CLI e API REST.
- Servidor de licenças para limites de activação e revogação, separado da verificação offline de assinaturas.
- Documentação comparativa contra outras ferramentas de protecção PHP, sem afirmações depreciativas.
- **Estado**: o contrato da CLI está decidido (`bin/ember`, `PassRegistry::withDefaults()`, `ProtectionException`, `ParseException`), mas ainda não está totalmente implementado; um ponto de entrada real, que receba um caminho de código-fonte e um nome de edição e produza output protegido, é o próximo trabalho concreto, antes de mais técnicas na camada Source (ADR-0012). Ainda não foi tomada nenhuma decisão sobre uma API HTTP, além de manter a composição subjacente suficientemente agnóstica de adaptador para poder vir a suportar uma mais tarde.

Os itens só saem deste ficheiro e passam para as notas de lançamento depois de estarem implementados e testados.
