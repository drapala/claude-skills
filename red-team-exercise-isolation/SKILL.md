---
name: red-team-exercise-isolation
description: Use when conducting authorized red team exercises against your own infrastructure where the blue team must not identify the attacker as internal — covers VPS provisioning, connection fingerprint sanitization, and operational separation to simulate a realistic external threat.
---

# Red Team Exercise Isolation

## Overview

Goal: make a controlled attack indistinguishable from an external threat actor in your own team's logs — without revealing it's an internal exercise. This is **not** about evading law enforcement; it's about producing realistic telemetry for blue team validation.

**Pre-requisito absoluto:** autorização documentada antes de qualquer ação. Mesmo sendo dono da infra.

---

## Fingerprints que aparecem nos logs e como limpar cada uma

| Fingerprint | Onde aparece | Mitigação |
|-------------|-------------|-----------|
| IP de origem | PostgreSQL logs, firewall, SIEM | VPS externo fora da rede/ASN da empresa |
| ASN / ISP | Firewall, Cloudflare, SIEM | VPS em cloud diferente da usada pela empresa |
| Timezone do cliente | `pg_stat_activity`, timestamp de conexão | Mudar timezone do VPS para Europa/EUA |
| `application_name` | `pg_stat_activity`, pg logs | Imitar nome de app legítimo (ex: `fontedeprecos-api`) |
| Tool fingerprint | WAF, IDS, logs de app | Usar raw sockets ou cliente customizado, não hydra/patator |
| Padrão temporal fixo | SIEM (correlação de horário) | Jitter aleatório entre tentativas |
| Username do SO | Bash history, arquivo de log local | Usar VPS descartável, não máquina pessoal |

---

## Checklist de isolamento — antes do exercício

### 1. Provisionar VPS limpo

```bash
# Critérios de seleção:
# - Fora do Brasil (Europa ou EUA)
# - Cloud diferente da usada pela empresa (se empresa usa Oracle/AWS, use Hetzner/Vultr)
# - Pagar com cartão pessoal (não cartão corporativo)
# - Não associar ao email corporativo

# Opções recomendadas: Hetzner (Frankfurt/Helsinki), Vultr (Amsterdam), Contabo (Europa)
# Ubuntu 22.04 LTS mínimo — sem painel, só SSH
```

### 2. Configurar timezone não-brasileiro

```bash
# No VPS, imediatamente após provisionar:
sudo timedatectl set-timezone Europe/Berlin  # ou America/New_York
date  # confirma

# Isso faz os timestamps nos logs parecerem de fora do BR
```

### 3. Transferir apenas o script — sem rastros de identidade

```bash
# NÃO: git clone do seu repo pessoal (associa sua conta GitHub)
# NÃO: scp de arquivo com metadata pessoal (strip metadata antes)

# Copia apenas o script via stdin (sem arquivo intermediário no VPS)
ssh user@VPS_IP "cat > /tmp/attack.py" < pg-lowrate-bruteforce.py

# Verifica que não tem metadata do git
ssh user@VPS_IP "head -5 /tmp/attack.py"
```

### 4. Sanitizar fingerprints do PostgreSQL

O script já cobre `application_name`. Adicionalmente:

```python
# No startup message do PostgreSQL, inclua timezone realista:
params = (
    b"user\x00"             + USER.encode()        + b"\x00" +
    b"database\x00"         + DBNAME.encode()       + b"\x00" +
    b"application_name\x00" + APP_NAME.encode()     + b"\x00" +
    b"client_encoding\x00UTF8\x00"                             +
    b"TimeZone\x00Europe/Berlin\x00"                           +  # timezone do VPS
    b"\x00"
)
# pg_stat_activity.client_addr mostrará o IP do VPS com timezone europeu
# Indistinguível de um cliente legítimo externo
```

### 5. Padrão temporal realista

```python
# Madrugada BR = horário comercial Europa (UTC-3 vs UTC+1 = 4h diferença)
# 03h00 BRT = 07h00 CET — parece ataque durante horário comercial europeu
# Reforça a narrativa de ator externo

import random, time
DELAY_MIN = 45   # segundos
DELAY_MAX = 120  # jitter — sem padrão fixo
time.sleep(random.uniform(DELAY_MIN, DELAY_MAX))
```

---

## Checklist — durante o exercício

```
[ ] VPS provisionado fora do Brasil, cloud diferente da empresa
[ ] Timezone do VPS configurado (Europe/Berlin ou America/New_York)
[ ] Script transferido sem git clone / sem metadata pessoal
[ ] application_name imitando serviço legítimo
[ ] TimeZone no startup PostgreSQL = timezone do VPS
[ ] Jitter ativo (sem rate fixo)
[ ] Wordlist sem nome/path pessoal no VPS
[ ] Nenhuma janela do seu laptop acessando o target no mesmo período
[ ] Horário do ataque: madrugada BR (03h–05h BRT)
```

---

## Checklist — depois do exercício

```
[ ] Destruir o VPS (sem snapshot, sem preservar logs)
[ ] Remover script do VPS antes de destruir (ou destruir direto)
[ ] Documentar o exercício com timestamp e escopo
[ ] Revelar para a equipe após análise forense deles (dar 24–48h para investigar)
[ ] Comparar o que eles encontraram vs o que foi feito
```

---

## O que seus funcionários vão ver (telemetria realista)

```
Logs PostgreSQL:
  FATAL: password authentication failed for user "fontedeprecos"
  connection received: host=185.x.x.x (Frankfurt, DE) port=54321
  application_name=fontedeprecos-api

pg_stat_activity durante ataque (se monitorarem ao vivo):
  client_addr: 185.x.x.x
  application_name: fontedeprecos-api
  TimeZone: Europe/Berlin

SIEM / firewall:
  src_ip: 185.x.x.x (ASN: Hetzner Online GmbH, DE)
  dst_port: 5432
  protocol: PostgreSQL
  frequency: ~1 req/min (sub-threshold)
```

Indistinguível de um atacante externo que obteve `user` + `dbname` de um vazamento público.

---

## Critério de sucesso do exercício

| O time detectou | Avaliação |
|----------------|-----------|
| Nada | Monitoramento insuficiente — alarmes ausentes ou não funcionam de madrugada |
| Alerta no dia seguinte | Detecção reativa — encontraram nos logs mas sem alerta em tempo real |
| Alerta na madrugada (< 30min) | Monitoramento maduro — on-call funciona |
| Bloquearam o IP e escalaram | Red team bem-sucedido como exercício — equipe está pronta |

---

## Common Mistakes

**Acessar o target do laptop pessoal enquanto o VPS ataca**
→ Dois IPs simultâneos no mesmo target na mesma janela = correlação trivial.

**Usar git clone do repo pessoal no VPS**
→ GitHub access logs + IP do VPS = associação direta com sua identidade.

**Não destruir o VPS após o exercício**
→ VPS persistente com histórico = evidência. Destrua logo após concluir.

**Revelar antes da equipe investigar**
→ Dê 24–48h para eles investigarem de forma independente antes de revelar. Isso é o valor do exercício.
