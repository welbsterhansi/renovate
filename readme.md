# Renovate Bot — Governança de Imagens de Container

> **Escopo:** Este documento define o modelo de governança para atualizações de imagens de container gerenciadas pelo Renovate Bot nos repositórios pai (`base-images`) e filhos (repositórios de desenvolvedores), com foco na prevenção de incompatibilidades de imagens em ambientes OpenShift/Kubernetes.

---

## Índice

1. [Visão Geral](#1-visão-geral)
2. [Definições](#2-definições)
3. [Hierarquia de Imagens](#3-hierarquia-de-imagens)
4. [Classificação de Atualizações](#4-classificação-de-atualizações)
5. [Regras de Governança](#5-regras-de-governança)
6. [Configuração do Renovate](#6-configuração-do-renovate)
7. [Fluxo de Aprovação](#7-fluxo-de-aprovação)
8. [Contrato de Compatibilidade](#8-contrato-de-compatibilidade)
9. [Gates de Validação](#9-gates-de-validação)
10. [Resposta a Incidentes](#10-resposta-a-incidentes)
11. [Papéis e Responsabilidades](#11-papéis-e-responsabilidades)
12. [Perguntas Frequentes](#12-perguntas-frequentes)

---

## 1. Visão Geral

A empresa opera um modelo de imagens de container em dois níveis:

- **Imagens pai** são mantidas no repositório `base-images`. Elas são construídas diretamente a partir do registry oficial da Red Hat (`registry.redhat.io`) e publicadas no Azure Container Registry (ACR) interno.
- **Imagens filhas** são mantidas nos repositórios individuais dos desenvolvedores. A diretiva `FROM` do `Dockerfile` aponta para uma imagem pai no ACR.

O Renovate Bot automatiza as atualizações de dependências em ambos os níveis. Sem governança, uma atualização não controlada no nível pai pode quebrar silenciosamente dezenas de imagens filhas simultaneamente — ou introduzir uma mudança de geração de SO (ex: UBI 8 → UBI 9) sem nenhuma decisão humana.

Este documento estabelece as regras, configurações, fluxos e responsabilidades para prevenir que isso aconteça.

---

## 2. Definições

| Termo | Definição |
|---|---|
| **Imagem pai** | Imagem base construída no repositório `base-images`, originada do `registry.redhat.io` e publicada no ACR. |
| **Imagem filha** | Imagem de aplicação construída em um repositório de desenvolvedor cujo `FROM` referencia uma imagem pai no ACR. |
| **Digest pin** | Bloqueio de uma imagem ao seu hash `sha256` exato: `image:tag@sha256:abc…`. Garante reprodutibilidade byte a byte, independentemente de mutações de tag. |
| **Referência por tag** | Referenciar uma imagem apenas pela tag: `image:tag`. O conteúdo subjacente pode mudar sem que a tag mude. |
| **Atualização major** | Mudança no primeiro segmento de versão: `ubi8 → ubi9`, `nginx-118 → nginx-126`. Alto risco — nova geração de SO ou runtime. |
| **Atualização minor** | Mudança no segundo segmento de versão: `8.9 → 8.10`. Risco médio — novos pacotes, possíveis mudanças de comportamento. |
| **Atualização patch** | Mudança no terceiro segmento ou rebuild de segurança com a mesma tag: `8.9-110 → 8.9-115`. Baixo risco — correções de CVE e bugs. |
| **Dependency Dashboard** | Issue do GitHub criada e mantida pelo Renovate listando todas as atualizações pendentes e bloqueadas. Atualizações major exigem que uma checkbox seja marcada aqui antes do PR ser desbloqueado. |
| **Image drift** | Estado em que diferentes ambientes ou réplicas executam conteúdo de imagem diferente apesar de referenciar a mesma tag, devido a mutação de tag. |

---

## 3. Hierarquia de Imagens

```
registry.redhat.io
        │
        │  Renovate monitora aqui
        │  Abre PR com: image:tag@sha256:…
        │
        ▼
  ┌─────────────────────────┐
  │   base-images (GitHub)  │   ← imagens pai
  │   Dockerfile FROM       │
  │   fixado ao digest sha256│
  └────────────┬────────────┘
               │
               │  Pipeline CI: build + docker push
               ▼
  ┌─────────────────────────┐
  │  Azure Container        │
  │  Registry (ACR)         │   ← repositório central de imagens
  └────────────┬────────────┘
               │
               │  Renovate monitora aqui
               │  Abre PR com: image:tag (sem sha)
               │
               ▼
  ┌─────────────────────────┐
  │  Repositórios dos       │   ← imagens filhas
  │  desenvolvedores        │
  │  Dockerfile FROM apenas │
  │  por tag, sem digest     │
  └────────────┬────────────┘
               │
               │  Pipeline CI: build + docker push
               ▼
  ┌─────────────────────────┐
  │  ACR                    │   ← imagens filhas publicadas
  └────────────┬────────────┘
               │
               ▼
  Workloads OpenShift / Kubernetes
```

### Por que imagens pai usam sha256 e imagens filhas usam apenas tag

**Imagens pai** consomem conteúdo externo da Red Hat. Uma tag como `ubi8:8.9` pode ser silenciosamente reconstruída pela Red Hat (ex: um patch de segurança é aplicado e a tag é movida). Sem digest pin, dois builds da imagem pai em dias diferentes poderiam produzir resultados diferentes usando exatamente o mesmo `FROM` — isso é image drift. Fixar ao `sha256` significa que o Renovate só abrirá um PR quando a Red Hat publicar um novo digest, tornando cada mudança upstream explícita e auditável.

**Imagens filhas** referenciam imagens pai armazenadas no ACR, que está sob controle interno. A tag só é movida quando o nosso próprio pipeline de CI faz push explícito de um novo build após um PR revisado e mergeado. Não há risco de mutação silenciosa. Fazer digest pin das imagens filhas no ACR quebraria a cadeia de propagação: após uma imagem pai ser reconstruída e publicada no ACR, os repositórios filhos nunca receberiam a atualização.

---

## 4. Classificação de Atualizações

### 4.1 Imagens pai (origem: `registry.redhat.io`)

O gate de qualidade principal acontece **aqui**. Toda atualização de imagem pai exige revisão humana porque ela é a fundação de todas as imagens da empresa. Mesmo um patch pode atualizar `glibc` ou `openssl` e quebrar uma aplicação que depende de uma versão específica dessas bibliotecas.

| Tipo | Exemplo | Risco | Comportamento |
|---|---|---|---|
| Patch | `ubi8:8.9-110 → 8.9-115` | 🟢 Baixo | PR aberto — **revisão humana obrigatória** |
| Minor | `ubi8:8.9 → 8.10` | 🟡 Médio | PR aberto — **revisão humana obrigatória** |
| **Major** | `ubi8 → ubi9` | 🔴 Alto | PR aberto — **bloqueado até aprovação no Dependency Dashboard** |
| **Major** | `nginx-118 → nginx-126` | 🔴 Alto | PR aberto — **bloqueado até aprovação no Dependency Dashboard** |

> **Regra:** Não há automerge em nenhum tipo de atualização de imagem pai. O custo de uma imagem ruim chegando à produção é maior do que o custo de uma revisão humana.

### 4.2 Imagens filhas (origem: ACR)

O conteúdo já passou pelo gate de qualidade no nível pai. Patch e minor vindos do ACR podem ter automerge habilitado condicionado à aprovação do CI, pois a decisão de qualidade já foi tomada upstream.

| Tipo | Exemplo | Risco | Comportamento |
|---|---|---|---|
| Patch | `base-img:v1.2-3 → v1.2-4` | 🟢 Baixo | ✅ **Automerge se CI aprovado** |
| Minor | `base-img:v1.2 → v1.3` | 🟢 Baixo–Médio | ✅ **Automerge se CI aprovado** |
| **Major** | `ubi8-base:1.x → ubi9-base:2.x` | 🔴 Alto | PR aberto — **bloqueado até aprovação no Dependency Dashboard** |

> **Justificativa do automerge nas filhas:** A imagem pai no ACR só existe porque passou por revisão humana no `base-images`. Fazer revisão manual novamente em cada repositório filho para a mesma atualização cria atrito sem adicionar segurança real — apenas atrasa a chegada de patches de segurança às aplicações.

### 4.3 A regra dos dois gates para atualizações major

Uma atualização major no nível pai cria uma cascata de PRs bloqueados. A sequência é sempre:

```
1. Renovate abre PR BLOQUEADO em base-images              (gate 1 — equipe de Plataforma)
2. Equipe de Plataforma aprova no Dependency Dashboard
3. PR é revisado, mergeado, CI constrói nova imagem pai e faz push no ACR
4. Renovate detecta nova tag no ACR, abre PRs BLOQUEADOS
   em todos os repositórios filhos                        (gate 2 — cada equipe de dev)
5. Cada equipe de dev aprova seu próprio PR no Dependency Dashboard
6. Cada PR filho é revisado, mergeado, CI constrói nova imagem filha
```

Isso significa que uma mudança `ubi8 → ubi9` exige sign-off explícito no nível pai **e** em cada repositório filho. Nenhuma imagem filha executa uma nova geração de SO silenciosamente.

---

## 5. Regras de Governança

### Regra 1 — Sem automerge em imagens pai

Nenhuma atualização de imagem pai, independentemente do tipo, será mergeada automaticamente. Toda PR exige revisão humana. Isso é inegociável para imagens base.

```json
"automerge": false  // aplicado a todas as regras de imagens pai
```

### Regra 2 — Automerge permitido em imagens filhas para patch e minor

Imagens filhas podem ter automerge habilitado para patch e minor, desde que o CI passe. O gate de qualidade já aconteceu no nível pai.

```json
"automerge": true,
"automergeType": "pr",
"platformAutomerge": true  // usa o automerge nativo do GitHub (requer branch protection)
```

### Regra 3 — Imagens pai devem sempre ter digest pin

Todo `FROM` no repositório `base-images` deve incluir o digest `sha256`. Um gate de CI deve rejeitar qualquer PR que introduza ou modifique um `FROM` sem digest.

```dockerfile
# ✅ Correto — com digest pin
FROM registry.redhat.io/ubi8/ubi:8.10-1132@sha256:3d6b4e8c...

# ❌ Rejeitado pelo gate de CI — apenas tag
FROM registry.redhat.io/ubi8/ubi:8.10
```

### Regra 4 — Imagens filhas NÃO devem usar digest pin

Os `FROM` das imagens filhas devem referenciar imagens do ACR apenas por tag. Fazer digest pin no nível filho quebraria a cadeia de propagação automática.

```dockerfile
# ✅ Correto — apenas tag
FROM myacr.azurecr.io/base/ubi8:2.1

# ❌ Incorreto — quebra a propagação
FROM myacr.azurecr.io/base/ubi8:2.1@sha256:abc123...
```

### Regra 5 — Atualizações major exigem aprovação explícita

Qualquer atualização classificada como `major` deve ter `dependencyDashboardApproval: true`. O PR será criado, mas permanecerá bloqueado até que um membro da equipe de Plataforma ou Arquitetura marque manualmente a checkbox no Dependency Dashboard.

### Regra 6 — PRs devem sempre ser recriados

O Renovate deve ser configurado para recriar PRs mesmo quando forem fechados sem merge. Isso garante que atualizações não desapareçam silenciosamente da fila.

```json
"recreateClosed": true
```

### Regra 7 — Todas as PRs de atualização de imagem devem ter labels

Labels são obrigatórias em todas as PRs de atualização de imagem para facilitar filtragem, relatórios e alertas no GitHub:

- `docker` — todas as atualizações de imagem
- `redhat` — atualizações de `registry.redhat.io`
- `security` — atualizações patch (provavelmente relacionadas a CVE)
- `major-update` + `breaking-change` — bumps de versão major

### Regra 8 — Tags de imagens pai seguem convenção de nomenclatura semver

Tags publicadas no ACR pelo pipeline do `base-images` devem seguir versionamento semântico: `<major>.<minor>.<patch>-<build>`. Isso permite que o Renovate classifique corretamente as atualizações nos repositórios filhos.

```
# Exemplos
myacr.azurecr.io/base/ubi8:8.10.0-1
myacr.azurecr.io/base/nginx126:1.26.2-3
```

---

## 6. Configuração do Renovate

### 6.1 Repositório `base-images` — `renovate.json`

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "dependencyDashboard": true,
  "dependencyDashboardTitle": "Renovate — Dependency Dashboard (base-images)",
  "docker": {
    "enabled": true,
    "pinDigests": true
  },
  "recreateClosed": true,
  "packageRules": [
    {
      "description": "Red Hat — patch e minor: abre PR, revisão humana obrigatória, sem automerge",
      "matchDatasources": ["docker"],
      "matchPackagePatterns": ["registry.redhat.io/.*"],
      "matchUpdateTypes": ["patch", "minor"],
      "versioning": "loose",
      "automerge": false,
      "addLabels": ["docker", "redhat", "security"]
    },
    {
      "description": "Red Hat — major: bloqueado até aprovação manual",
      "matchDatasources": ["docker"],
      "matchPackagePatterns": ["registry.redhat.io/.*"],
      "matchUpdateTypes": ["major"],
      "versioning": "loose",
      "automerge": false,
      "dependencyDashboardApproval": true,
      "addLabels": ["docker", "redhat", "major-update", "breaking-change"]
    }
  ]
}
```

### 6.1.1 Por que `versioning: "loose"` para imagens Red Hat

O Renovate usa por padrão o versioning `docker`, que faz uma comparação lexicográfica básica entre tags. O problema é que as tags da Red Hat não seguem semver estrito:

```
ubi8:8.9-1105                        → formato: major.minor-build
ubi8:8.10-1132                       → idem
ubi8/openjdk-21:1.20-2.1726696548    → timestamp no build
```

Sem `versioning: "loose"`, o Renovate pode classificar incorretamente o tipo de atualização — por exemplo, interpretar `8.9-1105 → 8.10-1132` como major em vez de minor, ou ignorar o componente de build completamente. Isso faria com que as regras de `matchUpdateTypes` não se aplicassem ao tipo correto, quebrando a governança de patch vs minor vs major.

O `loose` é um semver permissivo que aceita o formato `major.minor-build` e mapeia corretamente:

| Segmento | Exemplo | Classificação |
|---|---|---|
| Primeiro número | `8` → `9` | major |
| Segundo número | `8.9` → `8.10` | minor |
| Sufixo de build | `8.9-1105` → `8.9-1132` | patch |

Para imagens com tags completamente opacas (ex: baseadas em timestamp ou hash), o `loose` ainda pode não ser suficiente — nesses casos use `versioning: "regex"` com um padrão customizado que mapeie explicitamente os segmentos da tag.

### 6.2 Repositórios dos desenvolvedores — `renovate.json`

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "dependencyDashboard": true,
  "dependencyDashboardTitle": "Renovate — Dependency Dashboard",
  "docker": {
    "enabled": true,
    "pinDigests": false
  },
  "recreateClosed": true,
  "prConcurrentLimit": 5,
  "prHourlyLimit": 2,
  "packageRules": [
    {
      "description": "ACR imagens pai — patch e minor: automerge se CI aprovado",
      "matchDatasources": ["docker"],
      "matchPackagePatterns": ["myacr.azurecr.io/.*"],
      "matchUpdateTypes": ["patch", "minor"],
      "versioning": "loose",
      "automerge": true,
      "automergeType": "pr",
      "platformAutomerge": true,
      "addLabels": ["docker", "base-image-update"]
    },
    {
      "description": "ACR imagens pai — major: bloqueado até aprovação manual",
      "matchDatasources": ["docker"],
      "matchPackagePatterns": ["myacr.azurecr.io/.*"],
      "matchUpdateTypes": ["major"],
      "versioning": "loose",
      "automerge": false,
      "dependencyDashboardApproval": true,
      "addLabels": ["docker", "major-update", "breaking-change"]
    }
  ]
}
```

**Por que `prConcurrentLimit` e `prHourlyLimit` estão aqui e não no `base-images`?**

Os repos de dev são o ponto de amplificação do modelo. Quando uma imagem pai é mergeada e publicada no ACR, o Renovate detecta a nova tag e dispara PRs em **todos os repos filhos ao mesmo tempo**. Sem limites, uma única atualização de imagem pai pode gerar dezenas de PRs simultâneos, saturando runners de CI, gerando ruído de notificações para os times e dificultando a triagem de falhas.

O `base-images` não precisa desses limites — ele recebe um PR por atualização de imagem upstream, sem efeito cascata.

| Parâmetro | Valor | Efeito |
|---|---|---|
| `prConcurrentLimit` | `5` | Máximo de 5 PRs abertos ao mesmo tempo por repo |
| `prHourlyLimit` | `2` | Máximo de 2 PRs criados por hora por repo |

Os valores `5` e `2` são um ponto de partida razoável. Ajuste conforme a capacidade do seu CI — times com runners dedicados e rápidos podem subir esses valores sem problema.

> **Pré-requisito para automerge:** O branch protection do GitHub deve ter status checks obrigatórios (`image-build`, `image-scan`). Sem isso, o Renovate pode mergear uma PR com build quebrado.

### 6.3 Preset compartilhado — centralização via `extends`

O Renovate tem um sistema de herança de configuração via presets. Em vez de cada repositório de desenvolvedor manter seu próprio `renovate.json` completo, todos estendem um único arquivo mantido pela equipe de Plataforma. Quando a regra muda, você altera **um arquivo** — na próxima execução do Renovate (ciclo padrão: a cada hora), todos os repos já pegam a mudança sem nenhum PR nos repos de dev.

#### Estrutura do repo centralizador

Crie um repositório dedicado, por exemplo `sua-org/renovate-config`, com a seguinte estrutura:

```
renovate-config/
└── presets/
    └── acr-child-images.json   ← preset das imagens filhas
```

O arquivo `presets/acr-child-images.json` contém as regras completas que todos os repos filhos devem herdar:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "packageRules": [
    {
      "description": "ACR imagens pai — patch e minor: automerge se CI aprovado",
      "matchDatasources": ["docker"],
      "matchPackagePatterns": ["myacr.azurecr.io/.*"],
      "matchUpdateTypes": ["patch", "minor"],
      "versioning": "loose",
      "automerge": true,
      "automergeType": "pr",
      "platformAutomerge": true,
      "addLabels": ["docker", "base-image-update"]
    },
    {
      "description": "ACR imagens pai — major: bloqueado até aprovação manual",
      "matchDatasources": ["docker"],
      "matchPackagePatterns": ["myacr.azurecr.io/.*"],
      "matchUpdateTypes": ["major"],
      "versioning": "loose",
      "automerge": false,
      "dependencyDashboardApproval": true,
      "addLabels": ["docker", "major-update", "breaking-change"]
    }
  ]
}
```

#### `renovate.json` mínimo em cada repo de dev

Com o preset centralizado, cada repositório de desenvolvedor precisa apenas de:

```json
{
  "extends": [
    "config:base",
    "github>sua-org/renovate-config//presets/acr-child-images"
  ]
}
```

O Renovate busca e interpreta o preset remoto automaticamente antes de processar qualquer regra local.

#### Sintaxe do `extends`

```
"github>org/repo//caminho/para/arquivo"
          │         │
          │         └── caminho dentro do repo, sem extensão .json
          └── org/repo no GitHub
```

Para apontar para o arquivo raiz do repo (`renovate.json` ou `default.json`) sem especificar caminho:

```json
"github>sua-org/renovate-config"
```

#### Ordem de precedência

O Renovate aplica as configurações na ordem em que aparecem no array `extends`, da esquerda para a direita, e regras locais do `renovate.json` sempre sobrescrevem o preset. Isso permite que um repo de dev sobrescreva pontualmente uma regra específica sem quebrar o restante:

```json
{
  "extends": [
    "config:base",
    "github>sua-org/renovate-config//presets/acr-child-images"
  ],
  "packageRules": [
    {
      "description": "Override local: desabilita automerge para esta imagem específica",
      "matchPackageNames": ["myacr.azurecr.io/base/ubi8-legacy"],
      "automerge": false
    }
  ]
}
```

#### Por que isso importa operacionalmente

Sem preset centralizado, qualquer mudança de governança — adicionar um label, ajustar `prConcurrentLimit`, adicionar uma nova regra de imagem — exige um PR em cada repositório de desenvolvedor. Com o preset:

| Cenário | Sem preset | Com preset |
|---|---|---|
| Adicionar label em todas as PRs de patch | PR em cada repo de dev | 1 commit no `renovate-config` |
| Ajustar limite de PRs simultâneos | PR em cada repo de dev | 1 commit no `renovate-config` |
| Onboarding de novo repo de dev | `renovate.json` completo | 3 linhas de `extends` |
| Auditoria de configuração vigente | Comparar N arquivos | 1 arquivo de referência |

> **Pré-requisito:** O GitHub App do Renovate instalado na organização precisa ter acesso de leitura ao repositório `renovate-config`. Se o repo for privado, verifique as permissões do app antes de referenciar o preset.

---

## 7. Fluxo de Aprovação

### 7.1 Patch / Minor — imagem pai (`base-images`)

```
Renovate detecta nova versão no registry.redhat.io
          │
          ▼
  PR aberto automaticamente
  (labels: docker, redhat, security)
          │
          ▼
  Equipe de Plataforma revisa o PR
  - Verifica errata / changelog da Red Hat
  - Verifica se CI passou
  - Avalia impacto potencial nas imagens filhas
          │
          ▼
       Mergea o PR
          │
          ▼
  CI constrói imagem pai, faz push no ACR com nova tag
          │
          ▼
  Renovate detecta nova tag no ACR
  Abre PRs nos repositórios filhos (automerge se CI verde)
```

### 7.2 Patch / Minor — imagem filha (repositórios de dev)

```
Renovate detecta nova tag no ACR
          │
          ▼
  PR aberto automaticamente
  (labels: docker, base-image-update)
          │
          ▼
  CI executa (build + scan)
          │
    ┌─────┴─────┐
  CI verde    CI vermelho
    │               │
    ▼               ▼
  Automerge    PR permanece aberto
  automático   Equipe de dev investiga
               e corrige antes de mergear
```

### 7.3 Major — qualquer nível

```
Renovate detecta mudança de versão major
          │
          ▼
  PR aberto — BLOQUEADO
  (labels: docker, major-update, breaking-change)
  Descrição do PR inclui:
  - Versão atual e versão alvo
  - Link para release notes da Red Hat
  - Lista de repositórios filhos afetados (se nível pai)
          │
          ▼
  Equipe de Plataforma / Arquitetura é notificada
  (via subscription de label no GitHub ou integração com Slack)
          │
          ▼
  Avaliação de compatibilidade:
  - Nova imagem é suportada pela versão atual do OpenShift?
  - Todas as dependências (RPMs, runtimes, libs) são compatíveis?
  - Quais repositórios filhos consomem este pai?
  - Qual é o plano de rollout?
          │
    ┌─────┴──────┐
  Aprovado    Rejeitado
    │               │
    ▼               ▼
  Marca checkbox  Fecha o PR
  no Dependency   Adiciona comentário
  Dashboard       explicando a rejeição
    │
    ▼
  PR desbloqueado
  Equipe revisa e mergea
    │
    ▼
  CI constrói imagem pai, faz push no ACR
    │
    ▼
  Renovate abre PRs BLOQUEADOS em todos os repositórios filhos
    │
    ▼
  Cada equipe de dev repete a avaliação
  e aprova/rejeita independentemente
```

---

## 8. Contrato de Compatibilidade

### 8.1 O que a equipe de Plataforma garante

Ao publicar uma nova imagem pai no ACR, a equipe de Plataforma garante que:

- A imagem passou por todos os scans de segurança internos.
- A imagem é suportada pela versão atual do OpenShift em produção.
- A tag segue a convenção de nomenclatura acordada (`major.minor.patch-build`).
- Release notes ou entrada de changelog está linkada na descrição do PR.
- Para atualizações major: pelo menos 30 dias de aviso são dados às equipes de desenvolvedores antes do merge do PR pai.

### 8.2 O que as equipes de desenvolvedores são responsáveis

Quando uma PR do Renovate atualiza o `FROM` de uma imagem filha:

- A equipe não deve confiar apenas no automerge para atualizações que impactem comportamento crítico da aplicação — o CI deve cobrir esses cenários.
- A equipe não deve fechar ou dispensar uma PR de major update sem documentar o motivo.
- A equipe é responsável por testar o comportamento da sua aplicação contra a nova imagem pai quando o CI não cobre todos os casos.
- A equipe não deve editar manualmente o `FROM` para contornar a tag gerenciada pelo Renovate.

### 8.3 Práticas proibidas

| Prática | Risco | Alternativa |
|---|---|---|
| Fazer digest pin do `FROM` filho no ACR | Quebra a propagação automática de atualizações | Usar apenas tag; deixar o Renovate gerenciar |
| Usar tag `latest` em qualquer `FROM` | Completamente imprevisível | Sempre usar tag versionada |
| Mergear PR do Renovate com CI falhando | Imagem não testada em produção | Corrigir o CI antes de mergear |
| Fechar PR de major update silenciosamente | Cria dívida técnica invisível | Documentar a rejeição com comentário |
| Executar versões diferentes da imagem pai entre ambientes | Image drift, builds não reprodutíveis | Todos os ambientes devem usar a mesma tag ao mesmo tempo |
| Desabilitar o Renovate em um repositório | Patches de segurança nunca chegam à aplicação | Abrir issue documentando motivo e data-alvo de migração |

---

## 9. Gates de Validação

Os seguintes checks automatizados devem estar em vigor para reforçar as regras de governança.

### 9.1 Gate de CI — Imagens pai (pipeline do `base-images`)

```yaml
# Exemplo: GitHub Actions
- name: Valida digest pin no Dockerfile
  run: |
    if grep -E "^FROM" Dockerfile | grep -qv "@sha256:"; then
      echo "ERRO: A linha FROM está sem digest pin sha256."
      echo "Todos os FROM de imagens pai devem incluir @sha256:..."
      exit 1
    fi
```

### 9.2 Gate de CI — Imagens filhas (pipeline dos repositórios de dev)

```yaml
# Exemplo: GitHub Actions
- name: Valida ausência de digest pin no Dockerfile
  run: |
    if grep -E "^FROM.*myacr\.azurecr\.io" Dockerfile | grep -q "@sha256:"; then
      echo "ERRO: FROM de imagem filha não deve incluir digest sha256."
      echo "Use apenas a tag: FROM myacr.azurecr.io/base/ubi8:2.1"
      exit 1
    fi
```

### 9.3 Gate de CI — Convenção de nomenclatura de tags (pipeline do `base-images`)

```bash
TAG="${IMAGE_VERSION}"
if ! echo "$TAG" | grep -qE '^[0-9]+\.[0-9]+\.[0-9]+-[0-9]+$'; then
  echo "ERRO: Tag '$TAG' não segue o formato obrigatório: major.minor.patch-build"
  exit 1
fi
```

### 9.4 Status checks obrigatórios no GitHub

Os seguintes checks devem ser configurados como **obrigatórios** no branch protection do branch principal de todos os repositórios:

| Check | Descrição |
|---|---|
| `validate-dockerfile` | Valida as regras de `FROM` descritas acima |
| `image-build` | A imagem deve ser construída com sucesso |
| `image-scan` | Scan de segurança deve passar sem CVEs críticos |
| `renovate-config-validator` | Valida o schema do `renovate.json` em cada PR |

> **Atenção:** O automerge nas imagens filhas só é seguro se `image-build` e `image-scan` forem status checks obrigatórios. Sem isso, o Renovate pode mergear uma PR com build quebrado.

---

## 10. Resposta a Incidentes

### 10.1 Imagem incompatível implantada em produção

**Sintomas:** Crash da aplicação, erros de inicialização, bibliotecas ou RPMs faltando, comportamento inesperado após atualização do `FROM`.

**Resposta imediata:**

1. Identificar a tag de imagem em execução: `oc get pod <pod> -o jsonpath='{.spec.containers[*].image}'`
2. Fazer rollback do deployment para a tag anterior: `oc rollout undo deployment/<nome>`
3. Abrir uma Issue no GitHub no repositório afetado com labels `incident` e `image-compatibility`
4. Notificar a equipe de Plataforma para investigar a imagem pai

**Investigação da causa raiz:**

1. Comparar os dois digests de imagem pai usando `skopeo inspect`
2. Identificar quais RPMs ou bibliotecas mudaram entre os builds
3. Determinar se as dependências da aplicação na imagem filha são incompatíveis com a mudança
4. Documentar os achados na Issue do GitHub

**Resolução:**

- Se a imagem pai for a causa: reverter o PR pai, reconstruir e publicar a versão anterior no ACR com uma nova tag de patch.
- Se a imagem filha for a causa: atualizar a aplicação para ser compatível, depois re-mergear o PR da filha.

### 10.2 Renovate abre centenas de PRs simultaneamente

Pode acontecer após uma pausa longa do Renovate ou uma atualização major de imagem pai propagando para muitos repositórios filhos.

**Prevenção:** Configurar `prConcurrentLimit` e `prHourlyLimit` no preset compartilhado:

```json
{
  "prConcurrentLimit": 5,
  "prHourlyLimit": 2
}
```

**Se acontecer:** Usar o Dependency Dashboard para aprovar ou fechar PRs em lote. Não mergear tudo de uma vez — escalonar os merges para que a capacidade do CI absorva a carga.

### 10.3 Renovate para de criar PRs

**Causas prováveis:**
- `renovate.json` tem erro de sintaxe — validar com `renovate-config-validator`
- Token do GitHub App expirou ou perdeu permissões
- O bot do Renovate foi limitado por rate limit da API do GitHub

**Como verificar:** Abrir a Issue do Dependency Dashboard e procurar mensagens de erro. Verificar os logs do bot do Renovate no seu runner de CI ou no dashboard do Renovate Cloud.

---

## 11. Papéis e Responsabilidades

| Papel | Responsabilidades |
|---|---|
| **Equipe de Plataforma / Imagens** | Mantém o repositório `base-images`. Revisa e mergea todos os PRs de imagens pai. Possui os gates de CI e a convenção de nomenclatura de tags. Dá aviso de 30 dias antes de mergear atualizações major de imagens pai. |
| **Equipes de desenvolvedores** | Revisam e mergeam PRs de imagens filhas nos seus próprios repositórios quando o automerge não cobre o caso. Responsáveis por testes de compatibilidade no nível da aplicação. Não devem contornar as linhas `FROM` gerenciadas pelo Renovate. |
| **Arquiteto / Tech Lead** | Aprova atualizações major no Dependency Dashboard em ambos os níveis. Possui o processo de avaliação de compatibilidade. Revisa este documento de governança anualmente. |
| **Equipe de Segurança** | Recebe notificações por subscription de label `security`. Verifica que patches de CVE críticos são mergeados dentro do SLA acordado (ver abaixo). |
| **Administrador do Renovate** | Gerencia a instalação do GitHub App do Renovate, presets compartilhados e configuração do bot. Responde a interrupções ou configurações incorretas do Renovate. |

### 11.1 SLA para merge de patches de segurança

| Severidade | Tempo-alvo de merge |
|---|---|
| CVE Crítico (CVSS ≥ 9.0) | Dentro de 24 horas após criação do PR |
| CVE Alto (CVSS 7.0–8.9) | Dentro de 72 horas |
| Médio / Baixo | Na próxima sprint agendada |

---

## 12. Perguntas Frequentes

**P: Por que patch e minor nas imagens pai não têm automerge, mas nas filhas têm?**

R: O gate de qualidade acontece **uma vez, no nível pai**. Quando a equipe de Plataforma revisa e mergea um patch ou minor em `base-images`, ela está tomando a decisão de que aquela atualização é segura para a empresa. Os repositórios filhos podem confiar nessa decisão — exigir revisão humana novamente em cada filho cria atrito sem adicionar segurança real, e atrasa a chegada de patches de segurança às aplicações. Para imagens pai, no entanto, não há gate upstream — qualquer coisa da Red Hat pode chegar, incluindo mudanças que quebram libs ou RPMs usados pelas nossas imagens.

---

**P: Um desenvolvedor quer fixar o `FROM` filho a um sha256 específico para maior estabilidade. Por que isso não é permitido?**

R: O digest pin no nível filho corta a cadeia de atualização automatizada. O Renovate não conseguiria mais abrir PRs para aquela imagem porque o digest nunca mudaria da perspectiva do Renovate (ele não sabe que a tag foi movida). A garantia de estabilidade deve vir da governança da imagem pai, não de congelar a filha.

---

**P: O que acontece se a Red Hat descontinuar uma imagem?**

R: O Renovate não abrirá um PR porque não há nova versão. No entanto, a imagem continuará sendo construída usando o último digest conhecido até que seja explicitamente removida do registry da Red Hat. A equipe de Plataforma deve monitorar os anúncios de ciclo de vida da Red Hat e abrir proativamente um PR de migração quando uma imagem atingir o fim de vida.

---

**P: Um desenvolvedor pode fixar uma versão específica da imagem pai e optar por não receber as atualizações do Renovate?**

R: Não. Todos os repositórios de desenvolvedores devem ter o Renovate habilitado e não devem desabilitar ou excluir pacotes de imagem do ACR do scan. Optar por sair cria dívida de dependência invisível e significa que patches de segurança nunca chegam àquela aplicação. Se um desenvolvedor tiver um motivo legítimo de compatibilidade para permanecer em uma versão específica da imagem pai, deve abrir uma Issue no GitHub documentando o motivo e a data-alvo de migração.

---

**P: Como lidar com imagens que não seguem semver? Por exemplo, `ubi8/openjdk-21`.**

R: O `versioning: "loose"` do Renovate lida com tags não-semver fazendo uma comparação de melhor esforço. Para imagens com tags opacas (ex: baseadas em data ou hash), a equipe de Plataforma deve definir uma regra de `versioning` customizada no preset compartilhado e documentar o padrão de tag esperado. Se o versionamento não puder ser analisado de forma confiável, a imagem deve ser excluída das atualizações automáticas e gerenciada manualmente com um ciclo de revisão documentado.

---

**P: Temos um ambiente de staging. Podemos ter automerge lá e exigir aprovação manual apenas em produção?**

R: O automerge por ambiente não é suportado nativamente pelo Renovate (ele age no repositório, não no alvo de deployment). O padrão recomendado é ter um branch `staging` separado onde atualizações de patch e minor podem ser mergeadas com revisão mais leve, e só promover para `main` após validação. Isso está fora do escopo do Renovate e deve ser tratado pelo pipeline de GitOps / promoção.

---

*Última atualização: Abril de 2026 — Equipe de Platform Engineering*
*Ciclo de revisão: Anual ou após qualquer upgrade de versão major do Renovate*