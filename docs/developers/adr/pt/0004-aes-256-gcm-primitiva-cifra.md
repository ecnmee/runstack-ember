# ADR-0004: AES-256-GCM como primitiva de cifra deste projecto

* Estado: Aceite
* Data: 2026-08-24 (reconstruída; ver nota abaixo)
* Autores: RunStack Team
* Substitui: nenhuma
* Substituída por: nenhuma

**Nota histórica.** O documento original da ADR não foi preservado. Ao contrário de outras decisões reconstruídas neste conjunto, o pareamento deste número com esta decisão não é inferido: o `StringLiteralEncryptor.php` cita-o directamente no seu próprio docblock ("a self-invoking closure that decrypts it (AES-256-GCM, ADR-0004)"). O que não é preservado é a redacção histórica original, só a decisão e a sua citação directa.

## Decisão apoiada por evidência

Todas as operações de cifra neste código, sem excepção, usam AES-256-GCM através da extensão `openssl` do PHP:

* `StringLiteralEncryptor` (usado por `StringProtectionPass`) cifra cada literal string protegido com `openssl_encrypt($plaintext, 'aes-256-gcm', $this->key, OPENSSL_RAW_DATA, $iv, $tag, '', 16)`, uma chave nova gerada por build.
* `EncryptionPass` reutiliza mais tarde exactamente a mesma cifra e o mesmo modelo de chave-por-build para a cifra do ficheiro inteiro, por cima da compressão `gzdeflate`.

Nenhuma outra cifra, modo, ou biblioteca de encriptação aparece em lado nenhum de `src/`.

## Racional reconstruído

AES-256-GCM é uma cifra de encriptação autenticada padrão, disponível através da extensão `openssl` do PHP sem acrescentar nenhuma dependência, e a sua tag de autenticação fornece detecção de adulteração como efeito colateral da própria cifra. Esse efeito colateral é estrutural noutros pontos deste projecto: o addendum da ADR-0011 depende directamente dele para justificar excluir `IntegrityVerificationPass` de qualquer pipeline que também corra `EncryptionPass`, já que o GCM já autentica o ciphertext.

Se outras cifras foram avaliadas e rejeitadas, ou se AES-256-GCM foi escolhida sem comparação formal, não é recuperável a partir da evidência disponível. O que se pode afirmar com confiança é que este projecto usou uma cifra, de forma consistente, em todos os sítios onde cifra, desde antes do artefacto mais antigo examinado para esta reconstrução.

## Consequências

* Uma única escolha de cifra, usada em todo o lado, significa que as propriedades e limitações de segurança do AES-256-GCM (um atacante que obtenha a chave, que viaja ao lado do ciphertext segundo o threat model da ADR-0002, consegue decifrar; um ciphertext modificado falha a autenticação em vez de decifrar silenciosamente para algo errado) se aplicam uniformemente a `StringProtectionPass` e `EncryptionPass`, não uma propriedade aqui e outra diferente ali.
* Qualquer pass futuro que precise de cifrar algo deve usar por defeito esta mesma primitiva, a menos que uma razão específica e enunciada exija o contrário; introduzir uma segunda cifra neste código seria uma decisão significativa o suficiente para precisar da sua própria ADR.
