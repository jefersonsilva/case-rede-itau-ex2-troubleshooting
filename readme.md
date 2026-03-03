# 🔴 SRE Incident Response
## payment-api — Latência Crítica em Produção
**EKS-prod / Namespace: payment**

---

> **ALERTA DATADOG:** Latência média da API acima de 2 segundos por 10 minutos | Ambiente: Produção | Cluster: EKS-prod | Namespace: payment | Serviço: payment-api

| Campo | Valor |
|-------|-------|
| **Severidade** | SEV-1 (Crítico) |
| **Impacto** | Pagamentos Bloqueados |
| **SLA Target** | MTTR < 30 min |

---

## 1. Sequência de Diagnóstico

> A abordagem segue o princípio SRE de diagnóstico em camadas: do sintoma para a causa raiz, priorizando mitigação de impacto antes da análise profunda.
> Os exemplos nesse documento tem obejetivo de demonstração de conhecimento técnico, portanto foquei em comandos de kubectl, AWS Console e Datadog. Em um cenário real, seria recomendado utilizar ferramentas que agilizassem as investigações, como k9s ou Lens por exemplo.

### Fase 1 — Situational Awareness (0–5 min)

O primeiro passo é confirmar o escopo e entender o estado atual **sem realizar nenhuma mudança**.

#### 1.1 Verificar o alerta e contexto do incidente

- Confirmar que o alerta ainda está ativo no Datadog e verificar o timestamp de início
- Checar o Datadog APM: distribuição de latência por endpoint, p50, p95, p99
- Verificar dashboard de Error Rate — se há aumento de 5xx correlacionado
- Confirmar ausência de deploys no ArgoCD nas últimas 24h

#### 1.2 Avaliar estado dos pods

```bash
kubectl get pods -n payment -o wide --sort-by=.status.startTime

kubectl get pods -n payment \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,RESTARTS:.status.containerStatuses[0].restartCount,NODE:.spec.nodeName
```

- Verificar se todos os 6 pods estão `Running` e com baixo número de restarts
- Identificar distribuição dos pods pelos nodes do EKS

#### 1.3 Verificar eventos recentes do namespace

```bash
kubectl get events -n payment --sort-by=.lastTimestamp | tail -30
```

- Eventos de `OOMKilled`, `CrashLoopBackOff`, `FailedScheduling` ou `Evicted` indicam pressão de recursos

---

### Fase 2 — Diagnóstico Profundo (5–20 min)

#### 2.1 Análise de Logs

```bash
kubectl logs -n payment -l app=payment-api --tail=200 --timestamps \
  | grep -iE 'timeout|error|exception|slow|rds|database'

kubectl logs -n payment -l app=payment-api --tail=500 --timestamps --prefix \
  | grep -iE 'timeout' | awk '{print $1}' | sort | uniq -c
```

- Quantificar frequência de timeouts de RDS por pod
- Identificar padrão temporal (explosão súbita vs degradação gradual)
- Verificar stack traces completos para entender qual operação está falhando

#### 2.2 Análise de Recursos dos Pods

```bash
kubectl top pods -n payment --sort-by=cpu
kubectl top pods -n payment --sort-by=memory
kubectl describe pod -n payment <pod-name> | grep -A 10 'Requests\|Limits\|Conditions'
```

- Verificar se pods estão próximos dos limits de CPU/Memory
- CPU throttling é causa comum de latência aumentada

#### 2.3 Análise de Recursos dos Nodes

```bash
kubectl top nodes
kubectl describe nodes | grep -A 5 'Allocatable\|Allocated resources'
```

- Identificar nodes com alta pressão de recursos
- Verificar se há node com `NotReady` ou com pressão de memória/disco

#### 2.4 Diagnóstico do Banco RDS

- AWS Console RDS: verificar CPU, FreeStorageSpace, DatabaseConnections, ReadLatency, WriteLatency
- Verificar CloudWatch Insights: queries lentas nos últimos 15 min
- Checar se o número de conexões ativas está próximo do `max_connections` do RDS
- Verificar se há long-running transactions ou locks




#### 2.5 Análise de Conectividade e HPA

```bash
kubectl get hpa -n payment
kubectl describe hpa -n payment payment-api
kubectl get endpoints -n payment payment-api
```

- Verificar se o HPA está tentando escalar e por quê
- Confirmar que todos os endpoints do Service estão saudáveis


---

### Fase 3 — Correlação e Root Cause (20–30 min)

Com os dados coletados nas fases anteriores, correlacionar os sintomas e formular hipóteses priorizadas. Cruzar os timestamps dos timeouts com métricas do RDS, nodes e eventos do cluster.

---

## 2. Ferramentas e Comandos

| Ferramenta | Uso no Incidente | Comando / Acesso |
|------------|-----------------|------------------|
| `kubectl` | Estado dos pods, logs, eventos, recursos | `kubectl get/describe/logs/top/exec` |
| Datadog APM | Latência p99, traces, error rate, flamegraph | app.datadoghq.com — APM > Services |
| Datadog Monitors | Histórico do alerta e correlações | Monitors > Triggered |
| AWS Console RDS | CPU, Connections, Latência, Storage | RDS > Databases > payment-db |
| AWS CloudWatch | Métricas granulares RDS, Logs Insights | CloudWatch > Metrics > RDS |
| ArgoCD | Histórico de sync, health de Helm release | `argocd app get payment-api --show-operation` |
| AWS Cost Explorer | Verificar mudança de tipo de instância RDS | Billing > Cost Explorer |
| Netcat / nslookup | Diagnóstico de rede e DNS | `kubectl exec -it <pod> -- nc -zv <host> 5432` |

---

## 3. Hipóteses Prováveis

> As hipóteses são ordenadas por probabilidade considerando o contexto: sem mudanças de código, timeouts ao acessar o RDS, e 6 réplicas em execução.

### 🔴 Hipótese 1 — Connection Pool Esgotado no RDS (Alta Probabilidade)

**Evidências que confirmam:** `DatabaseConnections >= max_connections`; requisições na fila aguardando conexão; timeout após tempo configurado no pool.

**Como verificar:** CloudWatch: `DatabaseConnections`; RDS Performance Insights; logs do app com `connection timeout`.

---

### 🟠 Hipótese 2 — CPU do RDS Saturado / Query Lenta (Alta Probabilidade)

**Evidências que confirmam:** CPU RDS > 80%; aumento de `ReadLatency`/`WriteLatency`; pico de volume de transações (segunda-feira, retomada de atividade).

**Como verificar:** RDS Performance Insights; slow query log; CloudWatch `CPUUtilization`.

---

### 🟡 Hipótese 3 — CPU Throttling nos Pods (Média Probabilidade)

**Evidências que confirmam:** `kubectl top pods` mostrando pods no limite de CPU; `container_cpu_cfs_throttled_periods_total` alto no Datadog.

**Como verificar:** `kubectl top pods`; Datadog metric: `container.cpu.throttled`.

---

### 🔵 Hipótese 4 — Node do EKS Sob Pressão de Recursos (Baixa-Média Probabilidade)

**Evidências que confirmam:** `kubectl top nodes` mostrando node(s) com alta utilização; múltiplos pods concentrados no mesmo node.

**Como verificar:** `kubectl top nodes`; `kubectl describe nodes`; distribuição de pods por node.

---

### ⚪ Hipótese 5 — Manutenção Automática RDS ou Backup em Andamento (Baixa Probabilidade)

**Evidências que confirmam:** Janela de manutenção configurada para segunda-feira; backup automático rodando na mesma janela que pico de uso.

**Como verificar:** AWS Console RDS > Maintenance & Backup tab; CloudWatch Events.

---

## 4. Ações Imediatas de Mitigação

> ⚡ **Princípio:** Restaurar o serviço primeiro, investigar depois. Nunca realizar mudanças destrutivas sem aprovação durante o incidente.

### 4.1 Escalar o Número de Réplicas (Ação Imediata)

Aumentar réplicas distribui a carga e reduz o volume de requisições simultâneas ao banco por pod, aliviando pressão no connection pool.

```bash
kubectl scale deployment payment-api -n payment --replicas=9
kubectl rollout status deployment/payment-api -n payment
```

- Monitorar se a latência começa a reduzir após 2-3 minutos
- Verificar se o novo número de réplicas não agrava o connection pool (`connection count = replicas x pool_size`)

### 4.2 Se Causa for Connection Pool Esgotado

```bash
kubectl get configmap -n payment payment-api-config -o yaml | grep -i pool
```

- Verificar variável de ambiente ou configmap com configuração do pool
- Se RDS suportar, aumentar `max_connections` modificando o Parameter Group (requer restart do RDS — avaliar impacto)
- **Alternativa preferencial:** habilitar PgBouncer/RDS Proxy se disponível para pooling externo

### 4.3 Se Causa for CPU Saturada no RDS

- Considerar upgrade temporário do tipo de instância RDS (requer failover ~1-2 min em Multi-AZ)

### 4.4 Se Causa for CPU Throttling nos Pods

- Aumentar temporariamente os limits de CPU para evitar throttling 

---

## 5. Comunicação do Incidente

### 5.1 Abertura do Incidente (T+0 min)

```
#inc-payment-api-latency

🔴 [INCIDENTE ABERTO] SEV-1 — payment-api com latência crítica
Início: [HH:MM]
Impacto: Processamento de pagamentos degradado — timeouts para usuários
Responsável: [NOME DO TIME RESPONSÁVEL]
War room: #inc-payment-api-latency
Próxima atualização: em 10 minutos
```

### 5.2 Atualizações de Progresso (a cada 10 min)

```
⏱ [UPDATE T+10] — payment-api
Status: Investigando | RDS com alto número de conexões identificado
Métricas: Latência p99 = 4.2s (pico), agora 2.8s
Próxima atualização: em 10 minutos
```

### 5.3 Comunicação para Stakeholders Não-Técnicos

```
📋 ATUALIZAÇÃO DE SERVIÇO — [HH:MM]

Estamos com lentidão no processamento de pagamentos. Alguns clientes
podem estar vendo demora ao finalizar compras.

Estamos trabalhando para resolver. Estimativa de normalização: próximos 20-30 minutos.

Nenhum dado financeiro foi comprometido. Atualizaremos em 15 minutos.
```

### 5.4 Resolução do Incidente

```
✅ [INCIDENTE RESOLVIDO] — payment-api |

Latência normalizada (p99 < 400ms). Serviço 100% operacional.
Causa raiz: Esgotamento do connection pool RDS.
Ação tomada: Escalonamento de réplicas + ajuste do pool size.
Post-mortem: Agendado para [Data].
```

---

## 6. Post-Mortem Blameless

| Campo | Informação |
|-------|-----------|
| **Título** | Latência crítica no payment-api por esgotamento de connection pool — RDS |
| **Início** | Segunda-feira, [DATA] às [HH:MM] |
| **Resolução** | [DATA] às [HH:MM] |
| **Duração Total** | ~28 minutos |
| **Severidade** | SEV-1 — Impacto direto em faturamento |
| **Detectado por** | Alerta Datadog — Latência média > 2s por 10 minutos |
| **Serviços Afetados** | payment-api (payment namespace, EKS-prod) |
| **Impacto ao Usuário** | Timeout em ~X% das transações durante o período |

### 6.1 Linha do Tempo

- Montar cronologicamente os eventos principais: início do alerta, ações tomadas, mudanças de estado, resolução, etc. Incluir timestamps precisos.

### 6.2 Análise da Causa Raiz

1. **Por que a latência da API aumentou?** 
2. **Por que ocorreram timeouts?** 
3. **Por que o pool estava esgotado?** 
4. **Por que o sistema não se adaptou?** 
5. **Por que não havia proteção contra isso?** 

### 6.3 O Que Funcionou Bem

- Documentar o que funcionou bem durante o incidente: comunicação, ferramentas, processos, etc. Isso reforça boas práticas e motiva o time.

### 6.4 O Que Pode Melhorar

- Documentar estratégias de melhorias para previnir outros incidentes similares

### 6.5 Action Items

- Listar ações concretas, responsáveis e prazos para implementação das melhorias identificadas

> ✅ **Este post-mortem é blameless** — os problemas identificados são sistêmicos, não individuais. O objetivo é aprender e prevenir. Nenhuma pessoa ou time será responsabilizado individualmente por este incidente.

---
