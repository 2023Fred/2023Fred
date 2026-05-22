# 📊 Relatório Executivo — Laboratório Oracle 19c com IA Claude

> **Administração de banco de dados Oracle orquestrada por IA.**
> Um estudo prático de Database Administration em Oracle 19c Enterprise Edition,
> conduzido por nove skills especializadas e operadas pelo modelo Claude da Anthropic
> — pensado para gestores, CIOs e estudantes de TI que querem entender como a IA
> agêntica está transformando o ofício do DBA.

| Edição | Plataforma | Skills | Achados | Score Geral |
|---|---|---|---|---|
| #001 — Maio 2026 | Oracle 19c EE | 9 agentes | 20 itens | **6 / 10** |

*Classificação: Laboratório · Educacional · Público*

---

## § 01 — O contexto

### Um *laboratório* deliberadamente imperfeito

Este relatório documenta uma sessão real de administração Oracle conduzida em um ambiente controlado — uma máquina virtual Oracle Linux 8 rodando três instâncias de banco em paralelo, configurada para fins **exclusivamente didáticos**.

> ⚠️ **Aviso importante**
> Trata-se de um **laboratório de testes controlado**. Todas as vulnerabilidades de segurança, alertas de capacidade e configurações fora do padrão foram **deliberadamente deixadas sem hardening** para servir de base aos exercícios práticos. Nenhum dado real ou sensível existe neste ambiente. Os achados aqui descritos não representam falhas operacionais, e sim **oportunidades pedagógicas** mapeadas pelas skills de auditoria.

### ACID — os quatro pilares que governam o ambiente

| | Pilar | O que garante |
|---|---|---|
| **A** | *Atomicidade* | Uma transação acontece por inteiro ou não acontece. Garantida pelos *undo segments* do Oracle. |
| **C** | *Consistência* | O banco passa sempre de um estado válido para outro. Constraints, triggers, foreign keys e regras de integridade. |
| **I** | *Isolamento* | Transações concorrentes não enxergam dados sujos umas das outras. Implementado via MVCC. |
| **D** | *Durabilidade* | Uma vez confirmada, a transação sobrevive a qualquer falha. *Redo logs* e *archivelog mode* são a espinha dorsal. |

---

## § 02 — Para quem é este relatório

### Quatro *olhares*, um mesmo banco

| Público | Foco | Entrega |
|---|---|---|
| 👔 **Gestores** | O painel de controle | Indicadores agregados, score por dimensão e plano de ação priorizado. |
| 🏢 **CIOs & Heads** | O caso de negócio | Riscos de licenciamento, exposição de segurança, custo-benefício de upgrades. |
| 🎓 **Estudantes de TI** | O passo-a-passo | Comandos reais, resultados reais, decisões reais — cada skill é uma aula. |
| 🛠️ **DBAs Juniores** | O playbook | Checklists prontos, queries de diagnóstico e método multi-agente replicável. |

---

## § 03 — A topologia

### Três *instâncias*, um único servidor

O laboratório roda em um host Oracle Linux 8 com 3,5 GB de RAM e 2 cores Intel, hospedando simultaneamente um banco primário, uma duplicata RMAN e um standby físico Data Guard.

```text
ol8-dba.localdomain  (Oracle Linux 8 · Kernel 5.15 · 3.5 GB RAM · 2 cores Intel)
│
├─ [ orcl ]      PRIMARY   — CDB + 1 PDB (ORCLPDB)
│   ├─ Container Database (CDB$ROOT)
│   ├─ PDB$SEED ......... READ ONLY  (template)
│   ├─ ORCLPDB .......... READ WRITE (banco da aplicação)
│   ├─ Archive → FRA      (/u01/fast_recovery_area)
│   ├─ Archive → standby  (ASYNC, LGWR)
│   └─ SGA: 956 MB · PGA: 654 MB
│
├─ [ ORCLDUP ]   DUPLICATE — RMAN duplicate (treino de restore)
│   └─ SGA: 400 MB
│
└─ [ ORCL_ST ]   PHYSICAL STANDBY — Data Guard em MOUNTED + MRP
    └─ SGA: 956 MB

Backups RMAN: /u02/orcl_backup/  (incremental · retenção 7d · compressão BASIC)
Listener:     porta 1521         (serviços: orcl, orclpdb, orcl_st)
```

| Indicador | Valor | Observação |
|---|---|---|
| Instâncias ativas | **3** | primary + duplicate + standby |
| SGA total alocado | **2,3 GB** | soma das três instâncias |
| PDBs no primário | **1** | dentro da licença gratuita |
| Modo de proteção | **MAX PERF** | Data Guard async |

---

## § 04 — A metodologia

### Nove *skills*, uma orquestradora

Cada área crítica do trabalho de um DBA Oracle foi encapsulada em uma skill — um módulo especializado com prompts, queries e critérios de diagnóstico próprios. A nona skill, *central-agentes*, é o maestro: consolida os achados das outras oito em um relatório executivo único.

| # | Skill | Responsabilidade | Veredito |
|---|---|---|---|
| 01 | `check-oracle-health` | Health check completo: tablespaces, sessões, wait events, SGA/PGA, jobs e alert log. | 🟡 FRA em 79%, backups com falha |
| 02 | `gerenciar-tablespaces` | Criação, redimensionamento e gestão de tablespaces. | 🟢 Espaço saudável, UNDO ajustável |
| 03 | `gerenciar-redo-logs` | Dimensionamento, multiplexação e redimensionamento de online redo logs. | 🔴 Sem multiplexing — risco alto |
| 04 | `criar-pdbs` | Provisionamento e administração de PDBs em arquitetura Multitenant. | 🟡 PDB sem SAVE STATE |
| 05 | `configurar-hugepages` | Configuração de HugePages no Linux para otimização da SGA. | 🔴 Não configurado (4K pages) |
| 06 | `auditoria-seguranca` | Hardening, Unified Auditing, TDE, perfis de senha e privilégios. | 🔴 Senha default, PUBLIC exposto |
| 07 | `capacity-planning` | Projeção de crescimento via AWR (tablespaces, FRA, sessões, CPU). | 🟡 FRA esgota em ~11 dias |
| 08 | `licenciamento-oracle` | Auditoria de edition, options pagas e features ativas. | 🔵 1 licença Processor · revisar packs |
| **09** | **`central-agentes`** ★ | **Orquestrador multi-agente.** Consolida o output das 8 skills em um plano executivo priorizado. | ⭐ Sintetiza · prioriza · entrega |

---

## § 05 — A evidência

### Os *prints* que contam a história

#### Print #01 — Health check do banco primário
*Skill `check-oracle-health` resume o estado geral em 11 blocos.*

```text
❯ check-oracle-health

● Skill(check-oracle-health) — Successfully loaded
● Vou executar o health check completo agora.
────────────────────────────────────────────────────────────
 RESULTADO — orcl (2026-05-18)
────────────────────────────────────────────────────────────
 Bloco                   Situação    Observação
 ─────────────────────   ──────────  ─────────────────────────────────
 Status básico           OK          OPEN / ARCHIVELOG / Flashback ON
 Tablespaces             OK          Máximo 23% (SYSTEM)
 FRA                     ATENÇÃO     79.27% usada (7.93 GB de 10 GB)
 Sessões / Locks         OK          1 sessão ativa, sem bloqueios
 Wait events             OK          Perfil normal (Data Guard ativo)
 SGA / PGA               OK          Buffer Hit Ratio 97.26%
 Archivelog              OK          Geração normal, 2 destinos VALID
 Jobs Scheduler          OK          Sem falhas nos últimos 7 dias
 Backups RMAN            ATENÇÃO     1 backup FAILED + 2 WARNINGS
 Objetos inválidos       OK          Nenhum
 Alert log               INVESTIGAR  ORA-01210 / ORA-01565 históricos
```

#### Print #02 — Diagnóstico de tablespaces
*Skill `gerenciar-tablespaces` identifica oportunidades de tuning de UNDO.*

```text
Tablespace   Usado    Livre      %       Status
──────────   ──────   ────────   ─────   ──────
SYSTEM       907 MB   2.957 MB   23%     OK
SYSAUX       580 MB   3.284 MB   15%     OK
USERS          3 MB   3.862 MB   0,07%   OK
TEMP           2 MB   3.863 MB   0,05%   OK
UNDOTBS1       —      —          —       OK

Recomendações geradas pela IA:
 1. UNDOTBS1 tem 242 MB recuperáveis — candidato a RESIZE
 2. UNDO retention em 900s — advisor sugere 2400s
 3. Autoextend sem MAXSIZE definido (risco operacional)
```

#### Print #03 — Online redo logs
*Skill `gerenciar-redo-logs` aponta ausência de multiplexing.*

```text
Group   Thread   Tamanho   Members   Status      Arquivado
─────   ──────   ───────   ───────   ──────────  ─────────
1       1        200 MB    1         INACTIVE    YES
2       1        200 MB    1         CURRENT     NO
3       1        200 MB    1         INACTIVE    YES
4 (SRL) 1        200 MB    —         UNASSIGNED  —
5 (SRL) 1        200 MB    —         UNASSIGNED  —
6 (SRL) 0        200 MB    —         UNASSIGNED  —
7 (SRL) 0        200 MB    —         UNASSIGNED  —
```

#### Print #04 — Auditoria de segurança
*Skill `auditoria-seguranca` revela violações deliberadamente plantadas.*

```text
Área                              Status                          Criticidade
────────────────────────────────  ──────────────────────────────  ───────────
Modo de auditoria                 Mixed Mode                      Médio
Policies ativas                   ORA_SECURECONFIG + LOGON_FAIL   OK
CTXSYS com senha default          (conta administrativa exposta)  CRÍTICO
Permissões PUBLIC em pacotes      11 pacotes expostos             ALTO
Profile de senha (DEFAULT)        Sem PASSWORD_VERIFY_FUNCTION    ALTO
sqlnet.ora / listener.ora         Sem criptografia de rede        ALTO
remote_os_authent                 FALSE                           OK
sec_case_sensitive_logon          TRUE                            OK
```

#### Print #05 — A orquestradora consolida tudo
*Skill `central-agentes` entrega o relatório executivo priorizado.*

```text
❯ central-agentes
● Vou consolidar todos os achados em um relatório executivo único,
  priorizando por criticidade.

Semana 0 (crítico):
  ├── [1] FRA — limpar archivelogs aplicados / aumentar tamanho
  ├── [2] CTXSYS — trocar senha default e LOCK da conta
  └── [8] Investigar e reexecutar backups com falha

Semana 1 (segurança):
  ├── [4] Revogar PUBLIC de UTL_*/DBMS_SQL
  ├── [5] Habilitar PASSWORD_VERIFY_FUNCTION
  └── [6] Criar sqlnet.ora com criptografia de rede

Semana 2 (infraestrutura):
  ├── [3]  Configurar HugePages
  ├── [7]  Adicionar segundo member nos redo logs
  └── [12] SAVE STATE no ORCLPDB

Semana 3-4 (otimização):
  ├── [9]  Redimensionar redo logs para 500 MB
  ├── [10] Definir MAXSIZE nos datafiles
  ├── [11] Redimensionar UNDOTBS1
  ├── [13] Ajustar undo_retention para 2400s
  └── [16] Aumentar retenção AWR para 90 dias

Próximo ciclo (planejamento estratégico):
  ├── [17] Atualizar para RU 19.18+ antes de criar 2º PDB
  └── [18] Confirmar licenciamento Diagnostic+Tuning
```

---

## § 06 — Os achados consolidados

### Vinte *oportunidades* mapeadas pela IA

A lista abaixo é o produto final da skill orquestradora: a totalidade dos achados de todas as outras oito, ordenada por criticidade e cruzada para eliminar duplicidades.

| # | Achado | Skill(s) | Severidade |
|---|---|---|---|
| 01 | **FRA em 79% — esgota em ~11 dias.** Fast Recovery Area com 7,93 GB usados de 10 GB. Backup pieces ocupam 71% do espaço. | check-oracle-health · capacity-planning | 🔴 Crítico |
| 02 | **CTXSYS com senha default.** Conta administrativa do Oracle Text rodando com credencial pública conhecida. | auditoria-seguranca | 🔴 Crítico |
| 03 | **HugePages não configurado.** Linux operando com pages de 4K para uma SGA de 956 MB — penalidade de performance. | configurar-hugepages | 🟠 Alto |
| 04 | **11 pacotes sensíveis expostos no PUBLIC.** UTL_HTTP, UTL_TCP, UTL_SMTP, UTL_FILE, DBMS_SQL, DBMS_SCHEDULER e outros. | auditoria-seguranca | 🟠 Alto |
| 05 | **Profile DEFAULT sem PASSWORD_VERIFY_FUNCTION.** Política de complexidade não é aplicada para usuários default. | auditoria-seguranca | 🟠 Alto |
| 06 | **sqlnet.ora ausente — tráfego sem criptografia.** Conexões cliente-servidor em texto plano na rede. | auditoria-seguranca | 🟠 Alto |
| 07 | **Redo logs sem multiplexing.** Cada um dos 3 grupos tem 1 member. Falha de disco causa ORA-00313 e parada total. | gerenciar-redo-logs | 🟠 Alto |
| 08 | **Backup RMAN com FAILED + 2 WARNINGS.** Job de 17/05 16:59 falhou; outros dois com COMPLETED WITH WARNINGS. | check-oracle-health | 🟠 Alto |
| 09 | **Redo logs com 200 MB — pico de 15 switches/hora.** Tamanho atual produz switches a cada 4 min; ideal 15–20 min. | gerenciar-redo-logs | 🟡 Médio |
| 10 | **Autoextend ilimitado em todos os datafiles.** MAXSIZE de 32 GB equivale a UNLIMITED — runaway pode encher o disco. | gerenciar-tablespaces | 🟡 Médio |
| 11 | **UNDOTBS1 com 242 MB recuperáveis.** Datafile UNDO ocupando 285 MB mas usando apenas 43 MB no mínimo. | gerenciar-tablespaces | 🟡 Médio |
| 12 | **ORCLPDB sem SAVE STATE.** PDB volta para MOUNTED a cada restart do CDB. | criar-pdbs | 🟡 Médio |
| 13 | **undo_retention curto — risco de ORA-01555.** Advisor detectou queries de até 25 min. Snapshot too old iminente. | gerenciar-tablespaces | 🟡 Médio |
| 14 | **Falhas de login do DGMGRL.** Broker Data Guard tentando conectar com credencial errada às 01:05h. | auditoria-seguranca | 🟡 Médio |
| 15 | **Usuário HR com profile DEFAULT.** Sem limite de tentativas, sem expiração, sem complexidade. | criar-pdbs · auditoria-seguranca | 🟡 Médio |
| 16 | **AWR com apenas 2 dias de histórico.** Insuficiente para projeções confiáveis. Recomendação: 90 dias. | capacity-planning | 🔵 Planejar |
| 17 | **Versão 19.3 sem isenção de 3 PDBs gratuitos.** Isenção só existe a partir do RU 19.18. | licenciamento-oracle | 🔵 Planejar |
| 18 | **Diagnostic + Tuning Pack ativos.** Configuração habilita features pagas. Confirmar contrato. | licenciamento-oracle | 🔵 Planejar |
| 19 | **memlock sem entrada explícita em limits.conf.** Ulimit atual OK, mas controle não versionado em arquivo. | configurar-hugepages | 🔵 Planejar |
| 20 | **SYSTEM e SYSAUX com growth aparente alto.** 411 MB/dia e 166 MB/dia — provável artefato de setup. Remonitorar após 30 dias. | capacity-planning | 🔵 Planejar |

---

### 🔗 Próxima parada

> O relatório acima conta a história.
> O **[dashboard interativo](https://htmlpreview.github.io/?https://github.com/2023Fred/2023Fred/blob/main/relatorio-executivo.html)** coloca cada métrica em gráfico, cada achado em contexto, cada score na sua dimensão.

---

<sub>**Oracle DBA Lab Report** · Edição #001 — Maio 2026 · Frederick Moura · Goiânia/GO</sub>
