# Chaos NFC

![Build](https://github.com/cybersantana83/chaos-nfc/actions/workflows/build.yml/badge.svg)
![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)

Firmware de referência pra módulo NFC Grove (M5Stack: AtomS3 / M5StickS3 / M5StickC Plus / CardPuter / M5Paper), do laboratório de hardware da **Café com Solda**. Detecta automaticamente qual dos dois chips NFC/RFID da M5 está conectado (`ST25R3916` ou `WS1850S`), lê cartões/tags, e inclui um leitor EMV read-only (PAN + bandeira) validado ponta a ponta sobre ISO14443-4. Projeto **somente leitura** — não emula tags nem grava dumps.

> Construído sobre a base do [whywilson/GroveNFC](https://github.com/whywilson/GroveNFC) — ver [Créditos](#créditos) pra detalhes de atribuição e licença.

## Status

**Fases 0, 1, 1.5 e 2 concluídas** — leitura real validada em hardware nos dois chips, leitor EMV com tela própria no firmware principal extraindo PAN + bandeira de cartão real, dossiê genérico pra qualquer protocolo, leitor FIDO2/CTAP2 via NFC com reconhecimento de fabricante/modelo com tela própria no firmware principal (validado contra uma YubiKey 5C NFC real), identidade visual própria (Café com Solda) e UI com feedback de hold + mapeamento de botões documentado.

- **Detecção automática de chip** — scan de I2C no boot identifica `ST25R3916` (`0x48`, produto "Unit NFC" — ISO14443A, ISO14443B, ISO15693, FeliCa) ou `WS1850S` (`0x28`, produto "RFID Unit 2" — ISO14443A/MIFARE/NTAG), e roteia pro driver certo sem o usuário precisar configurar nada. Antes disso, conectar o módulo "errado" dava um `NFC Fail` genérico sem nenhuma pista do motivo.
- **Leitor EMV (read-only), com tela própria no firmware** — `RATS` → `SELECT PPSE` → escolhe a aplicação de crédito (fallback: menor prioridade) → `SELECT AID` → `GPO` com PDOL neutro → extrai PAN do Track2 e o AIP (tag `82`). Sem escrita, sem manipulação de tag, sem máscara no número (decisão explícita). O resultado é um **dossiê** (bandeira, PAN, AID, AIP e o perfil de autenticação que o AIP indica — SDA/DDA/CDA, com vetor de ataque público conhecido, ano e risco — ver `src/emv_db.h`/`.cpp`), no mesmo espírito do `cards_db` do [ChaosID](https://github.com/cybersantana83/ChaosID). Página `EMV` na home do M5StickS3 (único board com o driver WS1850S ligado hoje) — clique força nova leitura, hold volta pra Home. A lógica em si continua validada em `src/emv_test.cpp` (ambiente isolado) e portada 1:1 pra `src/main.cpp`.
- **Dossiê para qualquer tipo de cartão (não só EMV), direto na página Reader** — `src/cards_db.h`/`.cpp` é o mesmo `cards_db` do ChaosID portado (seção HF, já que os módulos Grove são 13.56MHz — sem LF/125kHz), mapeando o protocolo que o Reader já detecta (`MFC1K`, `MFC4K`, `MFUL`, `NTAG*`, `DESFire`, `FeliCa`, `ISO15693`, `ISO14443B`) pro mesmo tipo de perfil: uso típico, criptografia, vetor de ataque conhecido, ano e risco. Funciona com os dois chips/backends (`ST25R3916`/"Unit NFC" e o driver secundário WS1850S), já que o dossiê só depende da string de protocolo que ambos já produzem — sem precisar de APDU nem de ISO14443-4. Validado primeiro em `src/ws1850s_test.cpp` (ambiente isolado) e depois portado pra tela real do Reader em todo board com tela landscape (AtomS3 fica de fora por falta de espaço na tela quadrada pequena). Essa página aparece na home como **"Mifare and Others"**.
- **Leitor FIDO2/CTAP2 via NFC, com reconhecimento, com tela própria no firmware** — reaproveita a mesma base ISO14443-4 do leitor EMV (`RATS` + I-block com WTX/chaining já resolvidos, via `emvSendIBlock()` compartilhado): `SELECT` no applet FIDO (AID `A0000006472F0001`) seguido de um `authenticatorGetInfo` (CTAP2) sem PIN/toque. A resposta é reconhecida/fingerprinted com o mesmo esquema do [`fido2-nfc-scanner`](https://github.com/cybersantana83/fido2-nfc-scanner) (Flipper FAP, mesma pesquisa Café com Solda): string de fabricante no ATS (`YubiKey`/`uTrust`) + parser de CBOR do `GetInfo` (AAGUID, versões, extensions, options, PIN, firmware). Página `FIDO2` na home do M5StickS3 — clique não faz nada nessa página, hold força nova consulta, botão secundário volta pra Home. Validada primeiro em `src/fido2_test.cpp` (ambiente isolado, contra uma YubiKey 5C NFC real: AAGUID, firmware `5.7.4`, PIN `SET`, resident keys, as 4 versões CTAP2 e as 5 extensions) e depois portada pra `src/main.cpp`. Ferramenta separada, em código e em branding, da pesquisa **Chaos IAM**.
- **Identidade visual Café com Solda** — fundo preto com paleta de destaque creme/vermelho/bege/verde (a mesma linha visual cyberpunk + café de especialidade do canal) em vez das cores genéricas herdadas do projeto original. Home em modo lista (StickS3/StickC Plus) com animação de chevron, em vez do carrossel de ícone único.
- **Usabilidade da UI** — corrigido: cliques perdidos durante a animação de transição da home (~240ms) e falta de feedback visual durante o hold do botão (agora tem uma barra de progresso). Mapeamento de botões por board documentado abaixo.

Detalhes técnicos completos — pegadinhas de protocolo, decisões de escopo, histórico de cada fase — estão no [`ROADMAP.md`](ROADMAP.md).

## Hardware

Dois chips NFC/RFID da M5 são suportados, detectados automaticamente no boot. Nenhum dos dois emula tags neste firmware (suporte a emulação/dump foi removido — ver Créditos/ROADMAP pra histórico):

| Produto M5 | Chip | Endereço I2C | Protocolos |
|---|---|---|---|
| Unit NFC | `ST25R3916` | `0x48` | ISO14443A, ISO14443B, ISO15693, FeliCa |
| RFID Unit 2 | `WS1850S` | `0x28` | ISO14443A (MIFARE, NTAG/Ultralight) |

Pinos I2C por board (selecionados automaticamente em `src/main.cpp` pelo target de build):

- AtomS3: SDA `GPIO2`, SCL `GPIO1`
- M5StickS3: SDA `GPIO9`, SCL `GPIO10`
- M5StickC Plus (1.1 / Plus2): SDA `GPIO32`, SCL `GPIO33`
- CardPuter / CardPuter ADV: SDA `GPIO2`, SCL `GPIO1`
- M5Paper (Porta A): SDA `GPIO25`, SCL `GPIO32`

Notas:

- Grove `G1/G2` do AtomS3 corresponde a `G33/G32` do StickC (ou seja, `SCL=33`, `SDA=32`).
- M5StickC Plus (1.1 / Plus2) usa buzzer PWM, sem caminho de speaker completo.
- No StickS3, o firmware liga `EXT_5V` (`M5.Power.setExtOutput(true)`) no boot pra alimentar o módulo Grove externo.
- Se a sua fiação usa pinos diferentes, ajuste `kSdaPin`/`kSclPin` em `src/main.cpp`.

## Identidade visual

Paleta de destaque (acentos de página, barra de progresso do hold) definida em `src/main.cpp` como `kAccentCream`/`kAccentRed`/`kAccentBeige`/`kAccentGreen`, sobre fundo preto (`TFT_BLACK`) em toda tela. Pra ajustar as cores, edite essas quatro constantes (`uint16_t` RGB565) — estão comentadas com o hex original de referência.

## Estrutura

```
chaos-nfc/
├── platformio.ini            ambientes de build (um por board + 4 isolados de validação)
├── patch_m5unit_nfc.py        extra_script: só roda pro ambiente m5paper
├── build_all.sh                builda todos os ambientes de board em sequência
├── ROADMAP.md                   histórico de fases, decisões de escopo, pegadinhas técnicas
└── src/
    ├── main.cpp                  firmware principal (UI, roteamento de chip, todas as páginas,
    │                             incluindo a página EMV no M5StickS3)
    ├── GroveNFC.cpp / .h          camada de driver do ST25R3916 (upstream)
    ├── M5Paper_Config.h           config específica do target M5Paper
    ├── ws1850s_test.cpp          sketch isolado de validação do driver WS1850S
    ├── emv_test.cpp              sketch isolado de validação do leitor EMV
    ├── fido2_test.cpp            sketch isolado de validação do leitor FIDO2/CTAP2 via NFC
    ├── emv_db.h / emv_db.cpp     dossiê EMV: perfil SDA/DDA/CDA a partir do AIP lido
    ├── cards_db.h / cards_db.cpp dossiê genérico (MIFARE/NTAG/DESFire/FeliCa/ISO15693/ISO14443B)
    └── test_m5paper.cpp         teste de canvas→sprite→push pro e-ink do M5Paper
```

`GroveNFC.cpp`/`.h` mantêm esse nome porque é a camada de driver original (upstream) do chip ST25R3916 — não é o nome do projeto.

## Build (PlatformIO)

```bash
pip install --break-system-packages -U platformio   # ou pipx install platformio
```

Ambientes de board disponíveis:

- `m5stack-atoms3`
- `m5stack-sticks3`
- `m5stack-stickcplus`
- `m5stack-cardputer`
- `m5stack-m5paper`

```bash
pio run -e m5stack-sticks3 -t upload -t monitor
# troca m5stack-sticks3 pelo ambiente do seu board
```

Ambientes isolados de validação (não fazem parte do firmware principal — cada um exclui `main.cpp`/`GroveNFC.cpp` do build):

- `ws1850s-test` — valida leitura MIFARE/NTAG no WS1850S puro
- `emv-test` — valida o fluxo EMV completo (RATS → PPSE → GPO → PAN)
- `fido2-test` — valida SELECT no applet FIDO + `authenticatorGetInfo` (CTAP2) via NFC
- `m5stack-m5paper-test` — valida o pipeline de desenho no e-ink

```bash
pio run -e emv-test -t upload -t monitor
```

## UI e controle de botões

A regra geral é a mesma em todo board: **clique** avança/confirma a ação mais comum da tela atual (trocar card na home, forçar leitura), e **hold** (segurar o botão por um tempo que varia por board — ver tabela) entra na funcionalidade ou volta pra Home, dependendo da tela.

| Board | Botão "clique" | Botão "hold" | Threshold do hold | Botão secundário |
|---|---|---|---|---|
| AtomS3 | `BtnA` | `BtnA` (mesmo botão) | 700ms | nenhum — só tem 1 botão físico |
| M5StickS3 | `BtnA` | `BtnA` | 380ms | `BtnPWR` **ou** `BtnB` (qualquer um dos dois) — tap = `key_b` (equivale a "anterior"), hold = voltar/cancelar |
| M5StickC Plus (1.1 / Plus2) | `BtnA` | `BtnA` | 700ms | `BtnB` — tap = `key_b`, hold = voltar/cancelar |
| CardPuter / CardPuter ADV | teclado | tecla `b`/`B` | 320ms (hold da tecla `b`) | setas ou `W`/`A`/`H`/`I`/`K`/`;`/`,` (anterior), `S`/`D`/`J`/`L`/`.`/`/` (próximo), `Enter` confirma, `ESC`/`Del` volta |
| M5Paper | `BtnB` (ou toque) | `BtnB` | 500ms | `BtnC` — clique = anterior, hold = voltar/cancelar; `BtnA` = próximo |

Notas:

- No StickS3/StickC Plus, o botão secundário só tem *tap* e *hold* — não existe um "clique" equivalente ao botão principal (ele dispara `nav_prev`/"voltar", não "próximo").
- No AtomS3, por ter só um botão físico, a mesma tecla serve pra avançar (clique) e confirmar/voltar (hold) — o firmware decide qual ação cabe ao hold de acordo com a tela atual.
- No M5Paper, tocar direto num card da home também funciona, além dos botões físicos.

Páginas na home (`kHomeOrder`/`kHomeOrderNfcUnit` em `src/main.cpp`):

- **M5StickS3** (com driver WS1850S ligado): `Mifare and Others` (Reader) → `EMV` → `FIDO2` → `About`.
- **Demais boards**: `Mifare and Others` (Reader) → `About`.
- **Diagnose** é a tela de self-test mostrada automaticamente no boot (não faz parte da home, exceto no M5Paper, que vai direto pro touch home sem essa etapa).

Dentro de cada tela:

- **Home**: clique troca de card entre as páginas acima; hold entra na funcionalidade.
- **Mifare and Others (Reader)**: o resultado (dossiê) fica congelado na tela mesmo depois que o cartão sai do campo — não desaparece de imediato. Hold faz uma nova consulta (limpa o resultado e volta a escanear); o botão secundário (quando existe) volta pra Home. No AtomS3 (só 1 botão físico), hold continua voltando pra Home, já que não tem botão secundário.
- **EMV** (só no M5StickS3, com WS1850S): mesmo princípio do Reader — o dossiê (bandeira/PAN/AID/risco) fica na tela depois que o cartão é retirado. Hold faz nova consulta; `BtnPWR`/`BtnB` volta pra Home.
- **FIDO2** (só no M5StickS3, com WS1850S): mesmo princípio do EMV — reconhecimento (fabricante/AAGUID/firmware/PIN/versões CTAP2) fica na tela depois que a chave é retirada. Hold faz nova consulta; `BtnPWR`/`BtnB` volta pra Home.

Feedback visual: uma barra fina no rodapé da tela (cor de destaque Café com Solda) enche durante o hold, proporcional ao tempo até a ação disparar (não disponível no M5Paper — e-ink é lento demais pra overlay por tick).

## Licença

**GPLv3 ou posterior** (GNU General Public License v3.0-or-later).

- Texto completo em [`LICENSE`](LICENSE) (canônico, SHA-256 `3972dc9744f6499f0f9b2dbf76696f2ae7ad8af9b23dde66d6af86c9dfb36986`).
- Cada arquivo fonte tem header SPDX `GPL-3.0-or-later`.
- Derivados devem ser redistribuídos sob a mesma licença.

## Créditos

Este projeto é da **Café com Solda** (canal de hardware hacking e segurança de [@cafecomsoldalab](https://instagram.com/cafecomsoldalab)), mantido por [cybersantana83](https://github.com/cybersantana83).

Construído sobre a base do firmware original [whywilson/GroveNFC](https://github.com/whywilson/GroveNFC) — a camada de driver do chip `ST25R3916` (`src/GroveNFC.cpp`/`.h`) vem de lá, e por exigência da GPLv3 os avisos de copyright e o header SPDX desse código permanecem intactos. A UI, o roteamento de chip, os leitores EMV/FIDO2/dossiê genérico, a identidade visual e tudo em `src/main.cpp` são deste projeto.

Outras ferramentas da mesma pesquisa Café com Solda citadas acima:

- [ChaosID](https://github.com/cybersantana83/ChaosID) — forense de cartões no Flipper Zero (`cards_db` compartilhado)
- [fido2-nfc-scanner](https://github.com/cybersantana83/fido2-nfc-scanner) — FAP de Flipper Zero com o mesmo reconhecimento FIDO2/CTAP2

> Este firmware já divergiu o suficiente do upstream que deixou de ser mantido como fork no GitHub — se você está lendo isso num repositório ainda marcado como fork, a migração pra um repositório independente está em andamento.
