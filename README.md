# 🛡️ Wazuh + VirusTotal Integration Lab — Active Response & Automatic Threat Removal

> *"Um ficheiro malicioso entra na rede. O SIEM deteta. O VirusTotal confirma. O agente executa. O ficheiro desaparece. Tudo em menos de 30 segundos. Sem intervenção humana. Sem drama. (Bem, houve bastante drama a configurar isto, mas isso é outra história.)"*

---

## 📋 Índice

- [Visão Geral](#visão-geral)
- [Arquitetura](#arquitetura)
- [Pré-requisitos](#pré-requisitos)
- [Componentes do Sistema](#componentes-do-sistema)
- [Configuração Passo a Passo](#configuração-passo-a-passo)
  - [1. Integração Wazuh + VirusTotal](#1-integração-wazuh--virustotal)
  - [2. Script de Remoção (remove-threat)](#2-script-de-remoção-remove-threat)
  - [3. Active Response no Manager](#3-active-response-no-manager)
  - [4. Decoder Personalizado](#4-decoder-personalizado)
  - [5. Regras de Correlação](#5-regras-de-correlação)
  - [6. Configuração nos Agentes Windows](#6-configuração-nos-agentes-windows)
- [Fluxo de Deteção e Resposta](#fluxo-de-deteção-e-resposta)
- [Validação com EICAR](#validação-com-eicar)
- [Troubleshooting (ou: as armadilhas que eu já caí por ti)](#troubleshooting)
- [Ficheiros de Referência](#ficheiros-de-referência)
- [Contexto do Projeto](#contexto-do-projeto)

---

## Visão Geral

Este repositório documenta a implementação de um pipeline **totalmente automatizado** de deteção e remoção de ameaças, combinando:

- **Wazuh SIEM** — monitorização de integridade de ficheiros (FIM) e motor de correlação de eventos
- **VirusTotal API** — enriquecimento de alertas com reputação de ficheiros (análise de hash MD5/SHA256)
- **Active Response** — execução automática de ações corretivas nos endpoints quando uma ameaça é confirmada

O resultado prático: um ficheiro malicioso depositado num agente Windows é **detetado, analisado e eliminado automaticamente**, com todo o evento registado e correlacionado no SIEM.

### O que este lab demonstra

| Capacidade | Detalhe |
|---|---|
| FIM (File Integrity Monitoring) | Monitorização em tempo real de diretórios sensíveis |
| Threat Intelligence | Integração com VirusTotal via API key |
| Active Response | Execução de script remoto no agente mediante alerta |
| Correlação de Eventos | Cadeia de regras: FIM → VT → Active Response |
| Logging & Auditoria | Registo completo em `active-responses.log` |

---

## Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│                        WAZUH MANAGER                            │
│                                                                 │
│  ossec.conf ──► integração VT ──► rules/local_rules.xml        │
│                                         │                       │
│                                    Active Response              │
│                                         │                       │
└─────────────────────────────────────────┼───────────────────────┘
                                          │ comando remoto
                    ┌─────────────────────┴──────────────────────┐
                    │            AGENTE WINDOWS                   │
                    │                                             │
                    │  FIM deteta ficheiro ──► alerta ──► Manager │
                    │                                             │
                    │  Manager responde ──► remove-threat.exe    │
                    │                           │                 │
                    │                    ficheiro eliminado ✓     │
                    └─────────────────────────────────────────────┘
```

**Cadeia de regras:**
```
Regra 554 (FIM: novo ficheiro)
    └─► Regra 87105 (VirusTotal: hash positivo)
            └─► Regra 100092 (custom: ameaça confirmada)
                    └─► Regra 553 → dispara Active Response
```

---

## Pré-requisitos

| Componente | Versão testada |
|---|---|
| Wazuh Manager | 4.x |
| Wazuh Agent (Windows) | 4.x |
| Sistema Operativo Agente | Windows 10 / Windows Server 2022 |
| VirusTotal API Key | Conta gratuita (60 req/min) |
| Acesso root/Administrator | Necessário em ambos os lados |

> ⚠️ A API gratuita do VirusTotal tem limite de 60 requisições por minuto e 500 por dia. Para ambientes de produção com muitos eventos, considera uma conta premium.

---

## Componentes do Sistema

### Ficheiros envolvidos

```
WAZUH MANAGER
├── /var/ossec/etc/ossec.conf                    # Configuração principal (integração VT + AR)
├── /var/ossec/etc/rules/local_rules.xml         # Regras personalizadas
├── /var/ossec/etc/decoders/local_decoder.xml    # Decoder personalizado para output do script
└── /var/ossec/logs/active-responses.log         # Log de execuções AR (verificar path!)

AGENTE WINDOWS
├── C:\Program Files (x86)\ossec-agent\active-response\bin\remove-threat.exe   # Script de remoção
└── C:\Program Files (x86)\ossec-agent\active-response\active-responses.log    # Log local do agente
```

---

## Configuração Passo a Passo

### 1. Integração Wazuh + VirusTotal

No ficheiro `/var/ossec/etc/ossec.conf` do **Manager**, adiciona o bloco de integração dentro de `<ossec_config>`:

```xml
<integration>
  <name>virustotal</name>
  <api_key>INSERE_A_TUA_API_KEY_AQUI</api_key>
  <rule_id>554</rule_id>
  <alert_format>json</alert_format>
</integration>
```

> 📌 A regra `554` é a regra nativa do Wazuh para **criação de novos ficheiros** detetada pelo FIM. É o ponto de entrada do pipeline.

---

### 2. Script de Remoção (remove-threat)

O Wazuh inclui um script de exemplo para remoção de ameaças. No agente Windows, o executável compilado deve estar em:

```
C:\Program Files (x86)\ossec-agent\active-response\bin\remove-threat.exe
```

O script recebe o **caminho completo do ficheiro** via stdin (em formato JSON) e executa a remoção. O output é registado localmente.

**Verificar permissões:** o agente Wazuh corre como `SYSTEM` — tem permissões suficientes para apagar ficheiros em diretórios monitorizados. Para diretórios com permissões especiais, pode ser necessário ajuste.

---

### 3. Active Response no Manager

Ainda no `ossec.conf`, adiciona **dois blocos** — um para definir o comando, outro para a resposta:

```xml
<!-- Definição do comando -->
<command>
  <name>remove-threat</name>
  <executable>remove-threat.exe</executable>
  <timeout_allowed>no</timeout_allowed>
</command>

<!-- Active Response: dispara quando a regra 100092 é ativada -->
<active-response>
  <disabled>no</disabled>
  <command>remove-threat</command>
  <location>local</location>
  <rules_id>100092</rules_id>
</active-response>
```

> ⚠️ **Atenção ao `<location>`:** `local` significa que o comando é executado **no agente que gerou o alerta**. Outras opções: `server` (no manager), `defined-agent` (num agente específico), `all` (em todos).

**Screenshot sugerido:**
<!-- PLACEHOLDER: screenshot_02_ossec_conf_active_response_command.png -->
> *Blocos `<command>` e `<active-response>` no ossec.conf*

---

### 4. Decoder Personalizado

O `remove-threat.exe` produz output num formato específico que o decoder padrão do Wazuh não reconhece nativamente. Sem decoder personalizado, o pipeline de correlação quebra silenciosamente — e ficas a olhar para logs a perguntar "mas o script correu, porque é que não há alerta?"

Adiciona ao ficheiro `/var/ossec/etc/decoders/local_decoder.xml`:

```xml
<decoder name="ar_log_fields">
  <parent>ar_log</parent>
  <regex>Removed threat located at (\S+)</regex>
  <order>parameters</order>
</decoder>
```

> 🔍 Este decoder captura o caminho do ficheiro removido a partir do output do script, permitindo que o Wazuh crie alertas ricos com o contexto completo da remoção.

**Screenshot sugerido:**
<!-- PLACEHOLDER: screenshot_03_local_decoder_xml.png -->
> *Conteúdo do `local_decoder.xml` com decoder personalizado*

---

### 5. Regras de Correlação

No ficheiro `/var/ossec/etc/rules/local_rules.xml`, adiciona as regras personalizadas:

```xml
<!-- Regra 100092: ameaça confirmada pelo VirusTotal -->
<rule id="100092" level="12">
  <if_sid>87105</if_sid>
  <match>antivirus</match>
  <description>VirusTotal: ficheiro malicioso confirmado - a acionar remoção automática</description>
  <group>virustotal,</group>
</rule>

<!-- Regra 100093: confirmação de remoção bem-sucedida -->
<rule id="100093" level="3">
  <if_sid>553</if_sid>
  <match>remove-threat</match>
  <description>Active Response: ficheiro malicioso removido com sucesso</description>
  <group>virustotal,ar,</group>
</rule>
```

> 📐 **Hierarquia de regras explicada:**
> - `554` → FIM deteta novo ficheiro → envia hash para VirusTotal
> - `87105` → VirusTotal devolve resultado positivo (ameaça detetada)
> - `100092` → regra custom que filtra resultados VT com `antivirus` → **dispara o Active Response**
> - `553` → Wazuh regista a execução do Active Response
> - `100093` → regra custom que confirma a remoção (opcional, para dashboards)

**Screenshot sugerido:**
<!-- PLACEHOLDER: screenshot_04_local_rules_xml.png -->
> *Regras 100092 e 100093 no local_rules.xml*

---

### 6. Configuração nos Agentes Windows

No `ossec.conf` **do agente** (em `C:\Program Files (x86)\ossec-agent\ossec.conf`), configura o FIM para monitorizar o diretório desejado:

```xml
<syscheck>
  <directories realtime="yes" report_changes="yes" check_all="yes">C:\Users\Public\Downloads</directories>
  <!-- Adiciona outros diretórios conforme necessário -->
</syscheck>
```

Confirma também que o Active Response está habilitado no agente:

```xml
<active-response>
  <disabled>no</disabled>
  <repeated_offenders>1,5,10</repeated_offenders>
</active-response>
```

> 💡 `realtime="yes"` é **obrigatório** para deteção imediata. Sem isto, o FIM só verifica no intervalo de scan configurado (por defeito, 6 horas — tempo mais que suficiente para um atacante fazer danos consideráveis).

---

## Fluxo de Deteção e Resposta

Aqui está o pipeline completo, passo a passo, do momento em que um ficheiro malicioso aparece até à sua eliminação:

```
1. Ficheiro depositado no diretório monitorizado pelo FIM
       │
       ▼
2. Wazuh Agent (FIM realtime) deteta a criação → gera evento
       │
       ▼
3. Evento enviado ao Wazuh Manager → regra 554 ativa
       │
       ▼
4. Módulo de integração envia hash MD5/SHA256 à API do VirusTotal
       │
       ▼
5. VirusTotal responde: X engines detetaram como malicioso
       │
       ▼
6. Wazuh processa resposta VT → regra 87105 ativa
       │
       ▼
7. Regra 100092 (custom) confirma ameaça → nível 12 → dispara AR
       │
       ▼
8. Manager envia comando ao agente: executar remove-threat.exe
       │
       ▼
9. Agente executa remove-threat.exe com o caminho do ficheiro
       │
       ▼
10. Ficheiro eliminado ✓ → output registado em active-responses.log
       │
       ▼
11. Regra 553/100093 → alerta de confirmação no SIEM
```

**Screenshot sugerido:**
<!-- PLACEHOLDER: screenshot_06_wazuh_dashboard_alert_chain.png -->
> *Dashboard Wazuh mostrando a cadeia de alertas: 554 → 87105 → 100092 → 553*

---

## Validação com EICAR

O [ficheiro EICAR](https://www.eicar.org/download-anti-malware-testfile/) é o standard da indústria para testar sistemas antivírus e deteção de ameaças **sem usar malware real**. É reconhecido por praticamente todos os motores de AV como ameaça de teste.

### Procedimento de teste

**No agente Windows**, abre o PowerShell como Administrador e executa:

```powershell
# Criar ficheiro EICAR no diretório monitorizado
$eicar = 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*'
Set-Content -Path "C:\Users\Public\Downloads\eicar_test.txt" -Value $eicar
```

**O que deves observar:**

1. **Imediatamente:** Alerta no dashboard Wazuh (regra 554 — FIM deteta novo ficheiro)
2. **Após ~10-30s:** Alerta 87105 (VirusTotal confirma como ameaça)
3. **Após Active Response:** Alerta 553/100093 (ficheiro removido)
4. **No sistema de ficheiros:** ficheiro já não existe

**Verificar logs no Manager:**
```bash
# Log do Active Response (atenção ao path correto!)
tail -f /var/ossec/logs/active-responses.log

# Alertas em tempo real
tail -f /var/ossec/logs/alerts/alerts.json | python3 -m json.tool | grep -A5 "100092\|87105\|remove-threat"
```

**Verificar logs no Agente Windows:**
```powershell
# Log local do Active Response
Get-Content "C:\Program Files (x86)\ossec-agent\active-response\active-responses.log" -Tail 20
```

**Screenshot sugerido:**
<!-- PLACEHOLDER: screenshot_09_file_removed_confirmation.png -->
> *Sistema de ficheiros no Windows mostrando que o ficheiro foi removido*

---

## Troubleshooting

*Também conhecido como: "as armadilhas que eu já caí por ti, para que não precises de sofrer da mesma forma"*

### ❌ O Active Response dispara mas o ficheiro não é removido

**Causa mais provável:** caminho errado para o `remove-threat.exe`.

Verifica:
```bash
# No Manager
grep -r "remove-threat" /var/ossec/etc/ossec.conf
# Deve referenciar remove-threat.exe (sem path absoluto — o Wazuh procura em active-response/bin/)
```

---

### ❌ O script corre mas não há alerta de confirmação

**Causa mais provável:** decoder em falta ou incorreto.

O output do `remove-threat.exe` não é processado pelo decoder padrão `ar_log`. Sem o decoder personalizado (secção 4), o Wazuh não consegue parsear o output e a cadeia de correlação quebra.

Verifica:
```bash
# Testar o decoder
/var/ossec/bin/wazuh-logtest
# Cola uma linha de log do active-responses.log e verifica se é decodificado corretamente
```

---

### ❌ `active-responses.log` está vazio ou não existe

**Causa mais provável:** path incorreto no `ossec.conf`.

> 🚨 **Esta foi a armadilha mais traiçoeira.** O path do `active-responses.log` no Manager é diferente do path no Agente.

| Local | Path correto |
|---|---|
| **Manager** | `/var/ossec/logs/active-responses.log` |
| **Agente Windows** | `C:\Program Files (x86)\ossec-agent\active-response\active-responses.log` |

Verifica se o `ossec.conf` aponta para o path **absoluto e correto**.

---

### ❌ Regras duplicadas causam comportamento inesperado

Se copiaste configurações de múltiplas fontes (documentação, fóruns, Stack Overflow às 2 da manhã), é muito fácil acabar com blocos duplicados no `ossec.conf`. O Wazuh não dá erro explícito — simplesmente o comportamento fica imprevisível.

```bash
# Verificar duplicados
grep -c "<active-response>" /var/ossec/etc/ossec.conf
# Se o número for > 2 (um por bloco legítimo), tens duplicados
```

---

### ❌ VirusTotal não responde / rate limit

A API gratuita tem limites. Se gerares muitos eventos FIM rapidamente (ex: copiar muitos ficheiros), podes atingir o limite.

```bash
# Ver erros de integração
tail -f /var/ossec/logs/integrations.log
```

---

### ❌ FIM não deteta em realtime

Confirma que `realtime="yes"` está na configuração do syscheck. Em Windows, o FIM realtime usa `ReadDirectoryChangesW` — funciona apenas em diretórios locais (não mapeamentos de rede).

---

## Ficheiros de Referência

### ossec.conf (secções relevantes — Manager)

```xml
<ossec_config>

  <!-- FIM: monitorização de integridade -->
  <!-- (configurado nos agentes individualmente) -->

  <!-- Integração VirusTotal -->
  <integration>
    <name>virustotal</name>
    <api_key>SUA_API_KEY</api_key>
    <rule_id>554</rule_id>
    <alert_format>json</alert_format>
  </integration>

  <!-- Comando de remoção -->
  <command>
    <name>remove-threat</name>
    <executable>remove-threat.exe</executable>
    <timeout_allowed>no</timeout_allowed>
  </command>

  <!-- Active Response -->
  <active-response>
    <disabled>no</disabled>
    <command>remove-threat</command>
    <location>local</location>
    <rules_id>100092</rules_id>
  </active-response>

</ossec_config>
```

### local_rules.xml

```xml
<group name="virustotal,">

  <rule id="100092" level="12">
    <if_sid>87105</if_sid>
    <match>antivirus</match>
    <description>VirusTotal: ficheiro malicioso confirmado - remoção automática acionada</description>
    <group>virustotal,</group>
  </rule>

  <rule id="100093" level="3">
    <if_sid>553</if_sid>
    <match>remove-threat</match>
    <description>Active Response: ficheiro malicioso removido com sucesso</description>
    <group>virustotal,ar,</group>
  </rule>

</group>
```

### local_decoder.xml

```xml
<decoder name="ar_log_fields">
  <parent>ar_log</parent>
  <regex>Removed threat located at (\S+)</regex>
  <order>parameters</order>
</decoder>
```

---

## Contexto do Projeto

Este lab foi desenvolvido no âmbito do **Projeto Final CET103 DPR** para o curso **UF9197 — Wargaming** no **CINEL**, sob orientação do Professor Fernando Ruela.

A infraestrutura completa inclui:

- 🔴 **VLAN10 RED/WAN Interna** — rede de ataque simulado
- 🟡 **VLAN20 VENUS** — pfSense + Snort + WireGuard
- 🟢 **VLAN30 MARTE** — OPNsense + Suricata + OpenVPN + IPSec
- 🔵 **VLAN103 GESTÃO** — Wazuh, Nessus, Grafana, Active Directory

**Equipa:** Diego Soares · Ricardo Dias 
**Domínio:** DiePedRic.win

> *"Configurar o Wazuh Active Response com VirusTotal parece simples no papel. Na prática, envolve pelo menos três momentos de 'mas estava a funcionar ontem!', um decoder que não encontras na documentação oficial, e a descoberta de que o path do active-responses.log é diferente no manager e no agente. Consideras-te avisado."*

---

## Referências

- [Wazuh Documentation — VirusTotal Integration](https://documentation.wazuh.com/current/user-manual/capabilities/malware-detection/virus-total-integration.html)
- [Wazuh Documentation — Active Response](https://documentation.wazuh.com/current/user-manual/capabilities/active-response/index.html)
- [Wazuh Documentation — File Integrity Monitoring](https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/index.html)
- [VirusTotal API Documentation](https://developers.virustotal.com/reference/overview)
- [EICAR Test File](https://www.eicar.org/download-anti-malware-testfile/)

---

<div align="center">

**⭐ Se este lab te foi útil, deixa uma estrela no repositório!**

*Feito com ☕, resiliência, e pelo menos 47 recarregamentos do serviço `wazuh-manager`*

</div>

