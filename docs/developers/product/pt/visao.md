# Visão

O RunStack Ember é uma plataforma de protecção de código para aplicações PHP.

O Ember combina transformações de AST, encriptação, protecção em runtime, licenciamento e reforço de aplicação (application hardening) numa única pipeline de build, em vez de tratar cada uma destas coisas como uma ferramenta separada acrescentada a um obfuscador de código-fonte.

## Definição

> O RunStack Ember é uma plataforma moderna de protecção de código para PHP, combinando transformações de AST, encriptação, protecção em runtime, licenciamento e reforço de aplicação numa única pipeline de build.

## Slogan

Beyond obfuscation. ("Mais do que obfuscação.")

## O que o Ember é

- Uma pipeline de passes de protecção (ADR-0006), organizada em quatro camadas: source, intermediate, runtime, packaging (ADR-0007).
- Uma plataforma cujos níveis comerciais são configuração, não bases de código separadas: cada nível activa um conjunto específico de passes.
- Um produto cujas afirmações públicas são sempre sustentadas por uma implementação e um teste (ADR-0003), e cujas afirmações de protecção são declaradas contra um modelo de ameaça explícito (ADR-0009).

## O que o Ember não é

- Não é uma ferramenta que prometa impedir um engenheiro reverso dedicado e com bons recursos. Ver a ADR-0009 para a fronteira exacta.
- Não é um sítio para criptografia proprietária ou "inventada". Ver a ADR-0004.
- Não é organizado em torno da expressão "Nível N" como conceito técnico; essa linguagem mantém-se apenas como contexto histórico para quem estiver a comparar com a implementação anterior.

## Estrutura

```text
RunStack Ember

Engine
├── Parser
├── Pipeline
├── Runtime
├── Cryptography
├── Licensing
└── Packaging

Passes de Protecção
├── Minification
├── Symbol Renaming
├── String Protection
├── Control Flow
├── Runtime Guards
├── Integrity Verification
├── Encryption
└── Virtualization (planeada)
```

As edições comerciais (Free, Basic, Premium, Enterprise) activam subconjuntos diferentes de Passes de Protecção. Não são implementações separadas.
