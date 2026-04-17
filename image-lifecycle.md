# Ciclo de Vida de Imagens de Container

> **Escopo:** Este documento define o ciclo de vida completo de imagens de container na organização — desde a criação até a remoção — cobrindo gestão de vulnerabilidades, enforcement no OpenShift, processo de remediação e mecanismos para forçar atualizações nos repositórios de desenvolvedores de forma segura e controlada.
>
> **Relação com o Renovate:** A automação de PRs de atualização é descrita em [`readme.md`](./readme.md). Este documento cobre o ciclo mais amplo no qual o Renovate é apenas uma peça.

---

## Índice

1. [Visão Geral do Ciclo de Vida](#1-visão-geral-do-ciclo-de-vida)
2. [Stages de uma Imagem](#2-stages-de-uma-imagem)
3. [Gestão de Vulnerabilidades](#3-gestão-de-vulnerabilidades)
4. [Enforcement no OpenShift](#4-enforcement-no-openshift)
5. [Forçar Atualização nos Repos de Dev](#5-forçar-atualização-nos-repos-de-dev)
6. [Papéis e Responsabilidades](#6-papéis-e-responsabilidades)
7. [Runbooks](#7-runbooks)

---

## 1. Visão Geral do Ciclo de Vida

Uma imagem de container passa por cinco stages desde que é criada até ser removida. O diagrama abaixo mostra o fluxo completo e os atores envolvidos em cada transição:

```
  registry.redhat.io
         │
         │  Renovate detecta nova versão
         ▼
  ┌─────────────────────────────┐
  │  1. CRIAÇÃO                 │  Equipe de Plataforma reconstrói
  │  base-images (GitHub)       │  a imagem pai com a nova base
  │  PR revisado + mergeado     │  Red Hat e publica no ACR
  └────────────┬────────────────┘
               │
               ▼
  ┌─────────────────────────────┐
  │  2. USO ATIVO               │  Imagem está em produção,
  │  ACR → OpenShift            │  sendo monitorada e escaneada
  │  Renovate propaga para devs │  continuamente
  └────────────┬────────────────┘
               │
               │  CVE crítico detectado
               │  ou nova versão major disponível
               ▼
  ┌─────────────────────────────┐
  │  3. REMEDIAÇÃO              │  Rebuild da imagem pai,
  │  Patch ou upgrade           │  propagação controlada
  │  controlado via Renovate    │  para imagens filhas
  └────────────┬────────────────┘
               │
               │  Red Hat anuncia EOL
               │  ou a imagem não é mais suportada
               ▼
  ┌─────────────────────────────┐
  │  4. DEPRECAÇÃO              │  Aviso formal de 30 dias,
  │  Comunicação + prazo        │  migração planejada,
  │  para times de dev          │  nova imagem pai disponível
  └────────────┬────────────────┘
               │
               ▼
  ┌─────────────────────────────┐
  │  5. EOL / REMOÇÃO           │  Imagem removida do ACR,
  │  Imagem retirada do ACR     │  Renovate para de monitorar,
  │  e do cluster               │  workloads migrados
  └─────────────────────────────┘
```

---

## 2. Stages de uma Imagem

### 2.1 Criação e publicação no ACR

Uma nova imagem pai nasce quando:
- A Red Hat publica uma nova versão no `registry.redhat.io` e o Renovate abre um PR no `base-images`
- A equipe de Plataforma cria proativamente uma nova imagem base (ex: nova stack de runtime)

**Critérios obrigatórios antes de publicar no ACR:**

| Critério | Verificação |
|---|---|
| Build bem-sucedido | Gate `image-build` no CI |
| Scan sem CVEs críticos | Gate `image-scan` com trivy/grype |
| Tag segue convenção semver (`major.minor.patch-build`) | Gate `tag-validator` no CI |
| FROM com digest sha256 | Gate `validate-dockerfile` no CI |
| Release notes linkadas na descrição do PR | Revisão humana |

Somente após todos os gates passarem e o PR ser mergeado o CI faz push da imagem para o ACR.

### 2.2 Uso ativo

Após publicada no ACR, a imagem entra em uso ativo. Neste stage:

- O Renovate monitora o ACR e propaga atualizações para os repos de dev via PRs automáticos
- Scans de vulnerabilidade são executados continuamente — tanto no CI (a cada PR) quanto em varreduras agendadas nas imagens já em execução no cluster
- A equipe de Plataforma monitora avisos de segurança da Red Hat (errata) e do NVD

**Scan contínuo no cluster — como verificar imagens em execução:**

```bash
# Listar todas as imagens únicas em execução no cluster
oc get pods -A -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' \
  | sort -u

# Escanear uma imagem específica com trivy
trivy image myacr.azurecr.io/base/ubi8:2.1 \
  --severity CRITICAL,HIGH \
  --exit-code 1
```

### 2.3 Deprecação

Uma imagem entra em deprecação quando:
- A Red Hat anuncia fim de vida (EOL) da versão base (ex: RHEL 8 → RHEL 9)
- A stack de runtime atingiu EOL upstream (ex: Java 11 → Java 21)
- Uma vulnerabilidade estrutural não pode ser corrigida sem upgrade de geração

**Processo de deprecação:**

```
Dia 0   → Equipe de Plataforma abre Issue no base-images
           com label `deprecation` documentando:
           - Imagem afetada e motivo
           - Nova imagem substituta
           - Prazo final de migração (mínimo 60 dias)
           - Repositórios filhos afetados

Dia 0   → Comunicação formal para todos os times de dev
           via canal de Plataforma (Slack, email, etc.)

Dia 30  → Lembrete automático para times que ainda
           não abriram PR de migração

Dia 45  → Escalação para tech leads dos times atrasados

Dia 60  → Prazo final — enforcement ativo no cluster
           (ver seção 4 e 5)

Dia 75+ → Remoção do ACR (somente após confirmação
           de que nenhum workload usa a imagem)
```

### 2.4 EOL e remoção do ACR

Antes de remover uma imagem do ACR, a equipe de Plataforma deve confirmar que nenhum workload no cluster ainda a referencia:

```bash
# Verificar se algum pod ainda usa a imagem depreciada
oc get pods -A -o json \
  | jq -r '.items[] | select(.spec.containers[].image | contains("ubi8:1.")) | .metadata.name'
```

Somente com resultado vazio a imagem pode ser removida do ACR. A remoção é irreversível — documentar data e motivo na Issue de deprecação antes de executar.

---

## 3. Gestão de Vulnerabilidades

### 3.1 SLAs de remediação por severidade

| Severidade | CVSS | SLA de remediação | O que acontece se vencer |
|---|---|---|---|
| Crítico | ≥ 9.0 | 24h após detecção | Escalação imediata para liderança técnica |
| Alto | 7.0–8.9 | 72h | Notificação para tech lead do time |
| Médio | 4.0–6.9 | Próxima sprint | Registrado no backlog de segurança |
| Baixo | < 4.0 | Próximo ciclo de patch | Monitoramento passivo |

> **Detecção** conta a partir do momento em que o scan (CI ou agendado) identifica o CVE, não da publicação no NVD.

### 3.2 Processo de remediação

O fluxo padrão de remediação de um CVE em imagem base:

```
1. CVE detectado pelo scan (trivy/grype)
         │
         ▼
2. Equipe de Plataforma avalia:
   - O CVE está na imagem pai ou em uma dependência da aplicação?
   - A Red Hat já publicou patch (errata)?
         │
    ┌────┴────┐
    │         │
   Sim       Não (aguardando Red Hat)
    │         │
    │         ▼
    │    Abre Issue com label `cve-pending`
    │    Documenta CVE, CVSS, workaround temporário
    │    Monitora errata da Red Hat
    │
    ▼
3. Renovate detecta nova versão patcheada no registry.redhat.io
   Abre PR no base-images com prPriority: 10
         │
         ▼
4. Equipe de Plataforma revisa e mergea o PR
   CI reconstrói imagem pai e faz push no ACR
         │
         ▼
5. Renovate detecta nova tag no ACR
   Abre PRs nos repos de dev (automerge se CI verde)
         │
         ▼
6. Scan pós-update confirma que CVE foi removido
   Issue de CVE fechada com link para o PR de fix
```

### 3.3 Exceções e waiver temporário

Se um time não consegue remediar dentro do SLA por bloqueio técnico legítimo (ex: incompatibilidade da aplicação com a versão patcheada), deve abrir uma Issue com:

- CVE afetado e severidade
- Motivo técnico do bloqueio
- Plano de mitigação temporária (ex: WAF rule, network policy)
- Data-alvo de resolução definitiva
- Aprovação do tech lead e da equipe de Segurança

Waivers não podem ultrapassar 30 dias para CVEs críticos.

---

## 4. Enforcement no OpenShift

### 4.1 Política — somente imagens do ACR são permitidas

Nenhum workload no cluster deve executar imagens de registries externos. Toda imagem deve passar pelo pipeline do ACR e pelos gates de qualidade da organização.

**Implementação com Kyverno:**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-acr-images
  annotations:
    policies.kyverno.io/description: >
      Bloqueia pods que referenciam imagens fora do ACR interno.
      Todas as imagens devem vir de myacr.azurecr.io.
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-image-registry
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: >
          Imagens externas não são permitidas. Use apenas
          myacr.azurecr.io. Imagem rejeitada: {{ request.object.spec.containers[].image }}
        pattern:
          spec:
            containers:
              - image: "myacr.azurecr.io/*"
```

> **Modo de rollout:** Inicie com `validationFailureAction: Audit` para mapear violações existentes antes de ativar `Enforce`. Isso evita interrupções em workloads legados.

### 4.2 Auditoria de imagens em execução no cluster

**Listar todas as imagens e suas tags em execução:**

```bash
oc get pods -A -o json \
  | jq -r '.items[] | .metadata.namespace + " | " + .metadata.name + " | " + .spec.containers[].image' \
  | sort | column -t -s "|"
```

**Identificar imagens desatualizadas — comparar com o ACR:**

```bash
# Para cada imagem em execução, verificar se a tag é a mais recente no ACR
az acr repository show-tags \
  --name myacr \
  --repository base/ubi8 \
  --orderby time_desc \
  --top 1
```

**Scan em lote de todas as imagens em execução:**

```bash
oc get pods -A -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' \
  | sort -u \
  | xargs -I{} trivy image {} --severity CRITICAL,HIGH --quiet
```

### 4.3 Bloquear imagens com CVE crítico no admission

Além de exigir que imagens venham do ACR, você pode bloquear imagens que não passaram no scan de segurança usando anotações no ACR integradas ao admission controller:

```yaml
# Kyverno — bloqueia imagens sem a anotação de scan aprovado
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-scan-annotation
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-scan-label
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: >
          A imagem não possui anotação de scan aprovado.
          Execute o pipeline de CI antes de fazer deploy.
        pattern:
          metadata:
            annotations:
              security.acr/scan-status: "passed"
```

---

## 5. Forçar Atualização nos Repos de Dev

### 5.1 Como o controle de major/minor/patch funciona sem quebrar apps

O modelo de governança usa três mecanismos complementares para garantir que devs atualizem sem risco de quebrar aplicações:

| Tipo | Mecanismo | Risco de quebra | Ação do dev |
|---|---|---|---|
| Patch | Automerge se CI verde | Mínimo | Nenhuma — automático |
| Minor | Automerge se CI verde | Baixo | Nenhuma — automático |
| Major | Bloqueado no Dependency Dashboard | Alto | Aprovação explícita + testes |

Para major, o dev não é surpreendido — o PR existe mas fica congelado até que ele decida avaliar e aprovar. O prazo de 30 dias de aviso da equipe de Plataforma garante que nenhuma mudança de geração chega sem comunicação prévia.

### 5.2 Fluxo de notificação antes do prazo

```
Dia 0   → Equipe de Plataforma mergea imagem pai major no base-images
           Renovate abre PRs BLOQUEADOS em todos os repos de dev
           Notificação via label subscription (breaking-change)

Dia 7   → Relatório: quais repos ainda não aprovaram no Dashboard

Dia 21  → Lembrete direto para tech leads dos times atrasados

Dia 30  → Prazo final — times que não migraram entram em
           enforcement ativo (ver 5.3)
```

### 5.3 O que acontece se o dev não atualizar dentro do prazo

Quando o prazo de migração vence e um repo ainda executa a imagem depreciada/vulnerável no cluster, a equipe de Plataforma aplica enforcement progressivo:

**Nível 1 — Audit (dias 0–30):**
A política Kyverno está em modo `Audit`. Pods com imagens antigas continuam funcionando mas são registrados em relatório de violações.

**Nível 2 — Warn (dias 30–45):**
```yaml
validationFailureAction: Audit
# + anotação de warning no pod
```
Deploy de novos pods com a imagem depreciada gera warning visível no `oc apply`. Pods existentes continuam.

**Nível 3 — Enforce (dia 45+):**
```yaml
validationFailureAction: Enforce
```
Novos deploys com a imagem depreciada são **bloqueados pelo cluster**. Pods existentes continuam rodando até o próximo restart/redeploy. A partir daí, o time não consegue fazer deploy de nenhuma nova versão da aplicação sem migrar a imagem.

> **Importante:** O Enforce nunca derruba pods em execução imediatamente — apenas bloqueia novos deploys. Isso evita interrupções em produção enquanto o time migra.

### 5.4 Controlar rollout de major sem quebrar apps

Para atualizações major que exigem mudanças na aplicação (ex: ubi8 → ubi9 com incompatibilidade de biblioteca), o processo recomendado é:

```
1. Dev aprova o PR major no Dependency Dashboard
         │
         ▼
2. Dev cria branch de migração no repo da aplicação
   Atualiza o FROM para a nova imagem pai
         │
         ▼
3. CI executa build + scan + testes de integração
   na branch de migração
         │
         ▼
4. Deploy em ambiente de staging com a nova imagem
   Validação funcional da aplicação
         │
    ┌────┴────┐
    │         │
   OK        Falha
    │         │
    ▼         ▼
  Merge    Abre Issue documentando incompatibilidade
           Solicita waiver com prazo e plano de fix
    │
    ▼
5. Deploy em produção via pipeline normal
   Rollout gradual (rolling update padrão do OpenShift)
```

---

## 6. Papéis e Responsabilidades

| Papel | Responsabilidades no ciclo de vida |
|---|---|
| **Equipe de Plataforma / Imagens** | Mantém o `base-images`. Publica e retira imagens do ACR. Gerencia todo o ciclo de deprecação. Executa scans agendados no cluster. Mantém as políticas Kyverno. |
| **Equipes de desenvolvedores** | Mantêm as imagens filhas atualizadas dentro dos prazos. Não contornam as políticas de imagem. Documentam incompatibilidades e solicitam waivers quando necessário. |
| **Arquiteto / Tech Lead** | Aprova atualizações major no Dependency Dashboard. Aprova waivers de CVE. Garante que os times de dev têm capacidade para executar migrações dentro do prazo. |
| **Equipe de Segurança** | Define e revisa os SLAs de remediação. Aprova waivers de CVE crítico e alto. Monitora o relatório de violações do Kyverno. Realiza auditorias periódicas de imagens em produção. |
| **Administrador do OpenShift** | Mantém e atualiza as políticas Kyverno/OPA no cluster. Gerencia o modo Audit → Enforce durante rollouts de enforcement. |

### 6.1 Matriz RACI — decisões críticas

| Decisão | Plataforma | Dev | Arquiteto | Segurança |
|---|---|---|---|---|
| Publicar nova imagem pai no ACR | **R** | I | C | C |
| Mergear PR de patch/minor em imagem pai | **R** | I | I | I |
| Aprovar major no Dependency Dashboard (pai) | C | I | **R** | C |
| Aprovar major no Dependency Dashboard (filho) | I | **R** | C | I |
| Iniciar processo de deprecação | **R** | I | C | C |
| Aprovar waiver de CVE crítico | C | I | C | **R** |
| Ativar Enforce no Kyverno | **R** | I | C | A |
| Remover imagem do ACR | **R** | I | C | I |

> R = Responsável, A = Aprovador, C = Consultado, I = Informado

---

## 7. Runbooks

### 7.1 CVE crítico descoberto em imagem em produção

**Sintomas:** Scan agendado ou alerta externo identifica CVE com CVSS ≥ 9.0 em imagem em execução no cluster.

**SLA:** Remediação em até 24h.

```
1. Identificar escopo
   oc get pods -A -o json | jq -r '...' | grep <imagem-afetada>
   → Quantos pods? Quais namespaces? Quais aplicações?

2. Verificar se a Red Hat já publicou patch
   → Consultar https://access.redhat.com/security/cve/<CVE-ID>
   → Verificar se o Renovate já abriu PR no base-images

3a. Red Hat tem patch disponível:
    → Priorizar review do PR do Renovate (prPriority: 10 já garante posição na fila)
    → Mergear PR no base-images
    → CI reconstrói e publica nova imagem no ACR
    → Renovate abre PRs nos repos de dev (automerge se CI verde)
    → Confirmar com scan que CVE foi removido

3b. Red Hat NÃO tem patch:
    → Abrir Issue com label cve-pending
    → Avaliar mitigação temporária (network policy, WAF, isolamento de namespace)
    → Documentar workaround e prazo estimado
    → Notificar Segurança e Arquitetura

4. Após remediação:
   → Re-scan de todas as imagens afetadas
   → Fechar Issue com link para PR de fix
   → Registrar no histórico de incidentes
```

### 7.2 Imagem base chegando ao EOL da Red Hat

**Trigger:** Anúncio de EOL no portal Red Hat ou alerta de fim de suporte no errata.

```
1. Confirmar data de EOL oficial
   → https://access.redhat.com/product-life-cycles

2. Identificar imagem substituta
   → Ex: ubi8 → ubi9, openjdk-11 → openjdk-21
   → Validar que a substituta é suportada pelo OpenShift em produção

3. Criar major update na imagem pai
   → Abrir PR manual no base-images com a nova base
   → PR passa por todos os gates de CI
   → Equipe de Plataforma valida compatibilidade

4. Iniciar processo de deprecação (seção 2.3)
   → Issue com label deprecation
   → Prazo mínimo de 60 dias
   → Comunicação formal para todos os times

5. Monitorar adoção
   → Relatório semanal de repos ainda na imagem depreciada
   → Escalação progressiva (seção 5.3)

6. Remoção do ACR somente após confirmar
   que nenhum workload usa a imagem depreciada
```

### 7.3 Dev recusando atualização de major

**Cenário:** Time de dev não aprova PR de major no Dependency Dashboard e não abre Issue documentando o motivo.

```
1. Verificar se existe Issue de waiver aberta pelo time
   → Se sim: verificar se está dentro do prazo aprovado
   → Se não: contatar tech lead do time

2. Entender o bloqueio
   → É incompatibilidade técnica real? → Orientar processo de waiver
   → É falta de capacidade/priorização? → Escalação para liderança
   → É desconhecimento do processo? → Treinamento + documentação

3. Se o prazo vencer sem ação:
   → Ativar Nível 2 (Warn) no Kyverno para o namespace do time
   → Notificar formalmente tech lead e gestão

4. Se persistir após Warn:
   → Ativar Nível 3 (Enforce) — novos deploys bloqueados
   → O time não consegue fazer deploy até migrar a imagem
   → Isso cria pressão operacional sem derrubar produção

5. Registrar todo o histórico na Issue de deprecação
   para auditoria e aprendizado
```

### 7.4 Imagem não autorizada detectada em execução no cluster

**Cenário:** Kyverno em modo Audit identifica pod usando imagem fora do ACR.

```
1. Identificar o workload
   oc get pods -A -o json \
     | jq -r '.items[] | select(.spec.containers[].image | contains("myacr.azurecr.io") | not) | .metadata.namespace + "/" + .metadata.name + ": " + .spec.containers[].image'

2. Identificar o time responsável pelo namespace

3. Notificar o time — imagem precisa ser migrada para o ACR
   → Orientar sobre o pipeline de publicação de imagens
   → Dar prazo para migração (máximo 30 dias para imagens em produção)

4. Se a imagem tiver CVEs críticos:
   → Tratar como incidente (runbook 7.1)
   → Prazo reduzido para 24-72h

5. Após migração, confirmar que o Kyverno não registra mais violações
   para aquele namespace
```

---

*Última atualização: Abril de 2026 — Equipe de Platform Engineering*
*Ciclo de revisão: Semestral ou após qualquer mudança de política de segurança*
*Documento relacionado: [`readme.md`](./readme.md) — Governança do Renovate Bot*
