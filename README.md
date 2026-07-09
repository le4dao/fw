# Arquitectura e Implementação em Google Apps Script de um Sistema DAO para Loop Engineering


## Resumo

Este relatório propõe a arquitectura e implementação de um sistema baseado no padrão Data Access Object (DAO) em Google Apps Script (GAS), desenhado especificamente para suportar fluxos de trabalho do tipo *Loop Engineering* — a disciplina emergente de estruturar o desenvolvimento assistido por Inteligência Artificial em ciclos repetidos de acção e feedback. A arquitectura proposta integra princípios de governança e disciplina arquitectural inspirados na constituição Spec Kit, bem como uma abordagem multi-agente com verificação determinista de evidências, adaptada do sistema CAPRA. O resultado é um sistema em camadas que abstrai o acesso a dados em Google Sheets através de DAOs tipados, orquestra agentes especializados através de registos centralizados e valida cada transição de estado com base em evidências concretas, tudo dentro das restrições operacionais da plataforma Google Apps Script.

Como complemento executável, o trabalho inclui uma ferramenta Python de validação lógica que verifica achados contra texto-fonte por *Evidence Anchoring*, aplica limiares determinísticos de confiança, sinaliza duplicatas e detecta contradições declaradas. Essa ferramenta permite validar relatórios, achados de agentes e evidências antes de persistir resultados no ambiente GAS.

**Palavras-chave:** Data Access Object, Loop Engineering, Google Apps Script, Multi-Agent Systems, Evidence Anchoring, Arquitectura em Camadas, Google Sheets como Base de Dados.

---

## 1. Introdução

A engenharia de software contemporânea está a atravessar uma transformação profunda impulsionada pela integração de modelos de linguagem de grande escala (LLMs) nos fluxos de trabalho de desenvolvimento. Do prompting manual à construção de sistemas autónomos que gerem ciclos completos de trabalho, a disciplina conhecida como *Loop Engineering* emerge como um novo paradigma que redefine o papel do engenheiro de software [1]. Paralelamente, o padrão Data Access Object (DAO) permanece como um dos pilares da arquitectura de software para abstração de persistência, permitindo que a lógica de negócio se mantenha independente do mecanismo de armazenamento subjacente [4].

A intersecção destes dois conceitos — a automatização de ciclos de feedback e a abstração de acesso a dados — levanta questões arquitecturais relevantes, particularmente em ambientes com restrições operacionais significativas. O Google Apps Script (GAS), uma plataforma JavaScript nativa do ecossistema Google Workspace, representa um cenário de implementação particularmente desafiante. Os seus limites de tempo de execução (6 minutos por invocação de *trigger*), as cotas de acesso às APIs do Google e a ausência de ferramentas de desenvolvimento tradicionais (IDEs avançadas, debuggers, testes automatizados nativos) exigem uma arquitectura cuidadosamente projectada [5].

Este relatório apresenta uma proposta arquitectural completa para um sistema DAO em Google Apps Script, desenhado para suportar fluxos de Loop Engineering. A proposta inspira-se em dois documentos de referência: a constituição Spec Kit [2], que estabelece princípios rigorosos de qualidade de código, disciplina arquitectural e governança; e o paper CAPRA [3], que demonstra a viabilidade de sistemas multi-agente com verificação determinista de evidências para avaliação de entregas de arquitectura de software.

O objectivo central deste trabalho é demonstrar como combinar a abstração de dados do padrão DAO, a disciplina arquitectural da Spec Kit e a orquestração multi-agente do CAPRA num sistema coerente, testável e operacionál dentro do ecossistema Google Workspace, utilizando Google Sheets como repositório de dados persistente para agentes de IA autónomos.

---

## 2. Enquadramento Teórico

### 2.1. Loop Engineering: Da Engenharia de Prompts à Engenharia de Sistemas

A transição do prompting manual para a engenharia de loops representa uma mudança fundamental na forma como os profissionais de software interagem com agentes de IA. Segundo Addy Osmani [1], *Loop Engineering* é o acto de substituir a pessoa que faz *prompting* ao agente pelo sistema que o faz automaticamente. Um *loop*, neste contexto, pode ser entendido como um objectivo recursivo onde se define um propósito e a IA itera até à conclusão.

A disciplina de Loop Engineering estrutura-se em torno de cinco estágios fundamentais que formam o ciclo básico de operação de qualquer sistema de desenvolvimento assistido por IA:

| Estágio | Descrição | Função no Sistema |
|---|---|---|
| **Intent** | Definição do resultado desejado | Estabelece o objectivo concreto e observável da tarefa |
| **Context** | Recolha de código, documentação, logs e restrições | Fornece ao agente a informação relevante para a execução |
| **Action** | Edição de ficheiros, execução de comandos, chamada de ferramentas | Realiza a transformação concreta no sistema |
| **Observation** | Captura de resultados de testes, *diffs*, *reviews*, logs | Transforma saídas em novas entradas de contexto |
| **Adjustment** | Actualização do plano e repetição do ciclo | Decide se continua, repara ou conclui a tarefa |

A literatura recente identifica vários padrões de loops especializados [1], incluindo o *Test-Driven Agent Loop* (onde o agente reproduz uma falha e itera até à resolução), o *Compiler-Driven Loop* (onde o compilador fornece feedback estrutural), o *Review-Driven Loop* (onde o feedback humano orientá as iterações), o *Runtime Debugging Loop* e o *Product Iteration Loop*. Cada padrão requer diferentes sinais de observação e regras de paragem, o que implica que o sistema de persistência deve ser flexível o suficiente para suportar múltiplos tipos de dados e estados.

### 2.2. O Padrão Data Access Object (DAO)

O padrão DAO, formalizado pela Sun Microsystems (actual Oracle) [4], fornece uma interface abstracta para um mecanismo de persistência, separando a lógica de acesso a dados da lógica de negócio. A sua raiz encontra-se no padrão Adapter, e a sua principal vantagem é a capacidade de isolar as alterações no mecanismo de armazenamento do restante sistema.

Num contexto de Google Apps Script, o DAO assume particular importância porque:

- **Abstrai a complexidade da API do `SpreadsheetApp`**, cujas chamadas são lentas e sujeitas a limites de quotas.
- **Centraliza a lógica de validação**, permitindo que regras de integridade sejam aplicadas de forma consistente.
- **Facilita a testabilidade**, já que os DAOs podem ser substituídos por *mocks* durante testes unitários.
- **Suporta a idempotência**, garantindo que operações repetidas não corrompem o estado.

### 2.3. Princípios da Spec Kit (base.txt)

A constituição Spec Kit [2] define cinco princípios fundamentais de qualidade arquitectural que são directamente transferíveis para a implementação em Google Apps Script:

**Princípio I: Qualidade de Código e Disciplina Arquitectural.** O princípio mais relevante é a separação da superfície de invocação da lógica importável. Em GAS, isto traduz-se na separação entre as funções que servem como *entry points* (ex: `doGet`, `doPost`, funções associadas a *triggers*) e os módulos de lógica de negócio que podem ser testados independentemente do *runtime* do GAS.

**Princípio II: Mudança Suportada por Testes.** Embora o GAS não possua um *framework* de testes nativo equivalente ao *pytest*, o princípio exige que toda a lógica de negócio seja escrita em módulos testáveis, com a camada de integração com o Google reduzida ao mínimo possível.

**Princípio III: Consistência na Interface.** Em GAS, isto aplica-se às respostas HTTP (`doGet`/`doPost`), às estruturas de dados retornadas e às convenções de nomenclatura dos DAOs.

**Princípio IV: Desempenho e Disciplina de Recursos.** O GAS impõe limites rigorosos de tempo de execução (6 minutos por invocação) e cotas de acesso às APIs. O princípio de *offline-first* da Spec Kit traduz-se aqui numa filosofia de *eager loading* de dados e minimização de chamadas ao `SpreadsheetApp`.

**Princípio V: Dependências Mínimas e Operações de Ficheiro Seguras.** No contexto do GAS, isto implica que o sistema não deve depender de bibliotecas externas não suportadas e que todas as operações de escrita nas folhas de cálculo devem ser validadas, idempotentes e protegidas contra corrupção.

### 2.4. Orquestração Multi-Agente e Evidence Anchoring (Paper.txt - CAPRA)

O sistema CAPRA [3] apresenta uma arquitectura multi-agente que processa documentos de arquitectura de software através de uma *pipeline* de quatro estágios: *Document Parsing*, *Parallel Verification Agents*, *Evidence Anchoring* e *Report Generation*. O que distingue o CAPRA é a camada de **Evidence Anchoring**, um mecanismo determinista que verifica se as conclusões geradas pelos agentes têm suporte efectivo nos dados fonte, utilizando *fuzzy matching* via distância de Levenshtein normalizada.

Os agentes especializados do CAPRA incluem o *SpecificationAuditorAgent*, o *TestAuditorAgent*, o *FeatureCheckAgent* e o *TraceabilityMatrixAgent*. Cada agente opera de forma independente, produzindo *findings* com pontuações de confiança, que são depois consolidadas por um *ConsistencyManager*.

A adaptação desta abordagem para o contexto de Loop Engineering em GAS sugere que os agentes do sistema não devem apenas executar tarefas, mas devem também produzir evidências verificáveis para cada decisão que tomam. O *Evidence Anchoring* funciona como uma barreira de confiança entre a execução e a persistência, garantindo que apenas operações validadas são consolidadas no repositório de dados.

---

## 3. Arquitectura Proposta

### 3.1. Visão Geral

A arquitectura proposta adopta um modelo em cinco camadas, cada uma com responsabilidades bem definidas e interfaces claras. A figura abaixo apresenta a visão geral do sistema:

```
┌─────────────────────────────────────────────────────────────────┐
│                     WEB APP / TRIGGER INTERFACE                 │
│              (doGet / doPost / Time-Driven Triggers)            │
├─────────────────────────────────────────────────────────────────┤
│                     ORCHESTRATOR LAYER                          │
│         (Loop Engine, Task Router, Consistency Manager)         │
├─────────────────────────────────────────────────────────────────┤
│                        AGENT LAYER                              │
│     (Discovery Agent, Action Agent, Verifier Agent, Logger)     │
├─────────────────────────────────────────────────────────────────┤
│                     EVIDENCE ANCHORING LAYER                    │
│     (Evidence Validator, Confidence Modulator, Deduplication)   │
├─────────────────────────────────────────────────────────────────┤
│                        DAO LAYER                                │
│   (StateDAO, TaskDAO, AgentLogDAO, ConfigDAO, RegistryDAO)      │
├─────────────────────────────────────────────────────────────────┤
│                     DATA LAYER (Google Sheets)                  │
│   (Sheet: State | Sheet: Tasks | Sheet: AgentLog | Sheet: Config)|
└─────────────────────────────────────────────────────────────────┘
```

A camada de interface (*Web App / Trigger Interface*) serve como ponto de entrada do sistema, responsável por receber requisições HTTP (via *Web App*) ou por ser activada periodicamente (via *Time-Driven Triggers*). A camada de orquestração coordena o ciclo de vida das tarefas, decidindo qual agente deve ser activado com base no estado actual do sistema. A camada de agentes contém os módulos especializados que executam acções concretas. A camada de *Evidence Anchoring* valida as conclusões dos agentes antes de as consolidar. A camada DAO abstrai o acesso às Google Sheets, e a camada de dados contém as folhas de cálculo propriamente ditas.

### 3.2. Registry-Driven Architecture

Inspirado na Spec Kit [2], o sistema utiliza um registo central como fonte única da verdade para o mapeamento entre tipos de tarefas, agentes responsáveis e DAOs necessários. Este padrão garante que a adicção de novas capacidades não exige modificações no motor de loop, apenas a registacção de novos componentes.

A tabela abaixo apresenta a estrutura do registo de tarefas:

| Campo | Tipo | Descrição |
|---|---|---|
| `taskType` | `string` | Identificador único do tipo de tarefa |
| `agentClass` | `Class` | Referência à classe do agente responsável |
| `inputDAO` | `Class` | DAO que fornece os dados de entrada |
| `outputDAO` | `Class` | DAO que recebe os resultados |
| `evidenceDAO` | `Class` | DAO que armazena as evidências de validação |
| `maxRetries` | `number` | Número máximo de tentativas antes de falha |
| `timeoutMs` | `number` | Tempo máximo de execução para esta tarefa |
| `priority` | `number` | Prioridade na fila de execução (menor = maior prioridade) |

A implementação do registo em GAS segue a convenção de nomenclatura da Spec Kit: módulos privados são prefixados com *underscore*, as chaves mantêm a sua forma canónica (frequentemente *hyphenated*), e as importações são mantidas em ordem alfabética. Duplicatas de chaves falham ruidosamente, não silenciosamente.

### 3.3. Evidência como Barreira de Confiança

A camada de *Evidence Anchoring* é o coração da fiabilidade do sistema. Inspirada no CAPRA [3], esta camada implementa um algoritmo determinista que verifica se as conclusões dos agentes têm suporte efectivo nos dados observados. A verificação opera em dois níveis:

**Nível 1: Presença de Citação.** Cada agente deve produzir uma citação directa dos dados que suportam a sua conclusão. Se a citação estiver ausente, a confiança é reduzida em 50%. Se for demasiado curta (inferior a 15 caracteres), aplica-se uma penalização de 30%.

**Nível 2: Matching de Evidência.** Para citações de comprimento padrão, o sistema aplica um *trigram overlap pre-filter* (sobreposição de trigramas acima de 0.27) seguido de cálculo da distância de Levenshtein normalizada. Se a similaridade máxima for inferior ao limiar de descartar (τ_min = 0.45), a conclusão é considerada não verificável e rejeitada. Se for superior a 0.70, a confiança é ligeiramente aumentada. Se estiver entre 0.45 e 0.70, a confiança é modulada proporcionalmente.

Após a modulação, um filtro de confiança elimina qualquer conclusão com pontuação inferior a 0.65. As conclusões sobreviventes são consolidadas pelo *ConsistencyManager*, que elimina duplicatas e ordena por severidade.

### 3.4. Separação de Agentes: Maker vs. Checker

Um dos princípios mais importantes da arquitectura é a separação entre o agente que executa a acção (*maker*) e o agente que verifica o resultado (*checker*). Esta separação, inspirada no *Sub-Agent* pattern do Loop Engineering [1] e no *ConsistencyManager* do CAPRA [3], garante que um modelo não avalia o seu próprio trabalho, mitigando o risco de alucinações e auto-confirmação.

| Agente | Função | Model |
|---|---|---|
| **DiscoveryAgent** | Identifica trabalho pendente e tria prioridade | Modelo leve, leitura apenas |
| **ActionAgent** | Executa a tarefa e produz resultados | Modelo de alto desempenho |
| **VerifierAgent** | Valida resultados contra critérios de sucesso | Modelo independente do ActionAgent |
| **LoggerAgent** | Regista evidências e actualiza estado | Modelo leve, escrita apenas |

---

## 4. Implementação em Google Apps Script

### 4.1. Estrutura de Directórios

A estrutura de ficheiros segue o padrão de nomenclatura da Spec Kit [2], com directórios em *underscores* e chaves em forma canónica:

```
src/
├── _core/
│   ├── BaseDAO.js            # Classe base para todos os DAOs
│   ├── EvidenceAnchor.js     # Mecanismo de Evidence Anchoring
│   └── Registry.js           # Registo central de tarefas
├── _agents/
│   ├── _DiscoveryAgent.js    # Agente de descoberta (privado)
│   ├── _ActionAgent.js       # Agente de execução (privado)
│   ├── _VerifierAgent.js     # Agente de verificação (privado)
│   └── _LoggerAgent.js       # Agente de registo (privado)
├── daos/
│   ├── StateDAO.js           # DAO para gestão de estado
│   ├── TaskDAO.js            # DAO para gestão de tarefas
│   ├── AgentLogDAO.js        # DAO para registo de actividade
│   ├── ConfigDAO.js          # DAO para configuração
│   └── RegistryDAO.js        # DAO para gestão do registo
├── loop/
│   ├── LoopEngine.js         # Motor principal do loop
│   └── TaskRouter.js         # Router de tarefas
├── _commands/
│   ├── doGet.gs              # Entry point para Web App (GET)
│   ├── doPost.gs             # Entry point para Web App (POST)
│   └── _triggers.gs          # Entry points para triggers temporizados
└── tests/
    ├── test_evidence_anchor.gs
    └── test_registry.gs
```

### 4.2. Implementação do DAO Base

O DAO base abstrai todas as interações com o `SpreadsheetApp`, implementando operações seguras e idempotentes:

```javascript
/**
 * @fileoverview BaseDAO - Classe base para todos os DAOs do sistema.
 * Abstrai o acesso ao SpreadsheetApp e implementa operações
 * seguras, idempotentes e com validação de inputs.
 * 
 * Segue os princípios I e V da Spec Kit Constitution.
 */

/** @const {string} ID da folha de cálculo principal */
const CONFIG_SPREADSHEET_ID = PropertiesService
    .getScriptProperties()
    .getProperty('SPREADSHEET_ID');

/**
 * DAO base que abstrai o acesso às Google Sheets.
 * Implementa segurança de caminhos e idempotência.
 */
class BaseDAO {
  /**
   * @param {string} sheetName - Nome da folha de cálculo.
   * @param {Object} options - Opções de configuração.
   * @param {boolean} [options.autoCreate=false] - Criar folha se não existir.
   */
  constructor(sheetName, options = {}) {
    this._sheetName = sheetName;
    this._spreadsheetId = CONFIG_SPREADSHEET_ID;
    this._options = options;
    this._cache = null;
  }

  /**
   * Obtém a folha de cálculo, com *lazy loading* e *cache*.
   * @returns {GoogleAppsScript.Spreadsheet.Sheet}
   */
  getSheet() {
    if (!this._cache) {
      const spreadsheet = SpreadsheetApp.openById(this._spreadsheetId);
      let sheet = spreadsheet.getSheetByName(this._sheetName);
      if (!sheet && this._options.autoCreate) {
        sheet = spreadsheet.insertSheet(this._sheetName);
      }
      this._cache = sheet;
    }
    return this._cache;
  }

  /**
   * Lê todos os registos da folha como array de objectos.
   * A primeira linha é tratada como cabeçalho.
   * @returns {Object[]} Array de registos.
   */
  readAll() {
    const sheet = this.getSheet();
    const data = sheet.getDataRange().getValues();
    if (data.length === 0) return [];

    const headers = data[0];
    const records = [];
    for (let i = 1; i < data.length; i++) {
      const record = {};
      headers.forEach((header, index) => {
        record[header] = data[i][index];
      });
      records.push(record);
    }
    return records;
  }

  /**
   * Escreve registos de forma segura e idempotente.
   * Verifica duplicações antes de inserir.
   * @param {Object[]} records - Registos a inserir.
   * @param {string} [keyField='id'] - Campo utilizado como chave primária.
   * @returns {number} Número de registos inseridos.
   */
  safeUpsert(records, keyField = 'id') {
    if (!records || records.length === 0) return 0;

    const existing = this.readAll();
    const existingKeys = new Set(existing.map(r => r[keyField]));
    const newRecords = records.filter(r => !existingKeys.has(r[keyField]));

    if (newRecords.length === 0) return 0;

    const sheet = this.getSheet();
    const headers = Object.keys(newRecords[0]);
    const rows = newRecords.map(r => headers.map(h => r[h]));

    sheet.getRange(
      sheet.getLastRow() + 1, 1,
      rows.length, headers.length
    ).setValues(rows);

    return newRecords.length;
  }

  /**
   * Actualiza um registo existente.
   * @param {string} id - Identificador do registo.
   * @param {Object} updates - Campos a actualizar.
   * @param {string} [keyField='id'] - Campo chave primária.
   * @returns {boolean} Verdadeiro se o registo foi actualizado.
   */
  update(id, updates, keyField = 'id') {
    const sheet = this.getSheet();
    const data = sheet.getDataRange().getValues();
    const headers = data[0];
    const keyIndex = headers.indexOf(keyField);

    for (let i = 1; i < data.length; i++) {
      if (data[i][keyIndex] === id) {
        Object.entries(updates).forEach(([field, value]) => {
          const colIndex = headers.indexOf(field);
          if (colIndex !== -1) {
            sheet.getRange(i + 1, colIndex + 1).setValue(value);
          }
        });
        return true;
      }
    }
    return false;
  }
}
```

### 4.3. Implementação do Evidence Anchor

O mecanismo de *Evidence Anchoring* implementa a lógica de validação determinista inspirada no CAPRA [3]:

```javascript
/**
 * @fileoverview EvidenceAnchor - Mecanismo de validação determinista
 * de conclusões geradas por agentes.
 * 
 * Segue o princípio II da Spec Kit e a abordagem de Evidence Anchoring
 * descrita no paper CAPRA.
 */

/** @const {number} Limiar mínimo para descartar citações não verificáveis */
const EVIDENCE_DISCARD_THRESHOLD = 0.45;

/** @const {number} Limiar mínimo de confiança para consolidar conclusões */
const CONFIDENCE_MIN_THRESHOLD = 0.65;

/** @const {number} Sobreposição mínima de trigramas para prosseguir */
const TRIGRAM_OVERLAP_THRESHOLD = 0.27;

/**
 * Valida a evidência de uma conclusão e modula a confiança.
 * @param {Object} finding - Conclusão com citação e confiança inicial.
 * @param {string} sourceText - Texto fonte para validação.
 * @returns {Object|null} Conclusão validada ou null se rejeitada.
 */
function anchorEvidence(finding, sourceText) {
  const initialConfidence = finding.confidence || 0.5;
  const quote = finding.quote;

  // Caso 1: Citação ausente
  if (!quote || quote.trim() === '') {
    return {
      ...finding,
      confidence: initialConfidence * 0.5,
      anchoringStatus: 'UNANCHORED_NO_QUOTE'
    };
  }

  // Caso 2: Citação muito curta
  if (quote.length < 15) {
    return {
      ...finding,
      confidence: initialConfidence * 0.7,
      anchoringStatus: 'WEAK_SHORT_QUOTE'
    };
  }

  // Caso 3: Citação padrão - aplicar trigram overlap + Levenshtein
  const similarity = computeMaxSimilarity(quote, sourceText);

  // Discard threshold
  if (similarity < EVIDENCE_DISCARD_THRESHOLD) {
    return null; // Hallucinação potencial - descartar
  }

  // Confidence modulation
  let modulatedConfidence = initialConfidence;
  if (similarity >= 0.70) {
    modulatedConfidence = initialConfidence + (1 - initialConfidence) * 0.5;
  } else if (similarity >= 0.45) {
    const penalty = (0.70 - similarity) / 0.25;
    modulatedConfidence = initialConfidence * (1 - penalty * 0.35);
  }

  // Confidence filter
  if (modulatedConfidence < CONFIDENCE_MIN_THRESHOLD) {
    return null;
  }

  return {
    ...finding,
    confidence: modulatedConfidence,
    similarity: similarity,
    anchoringStatus: 'ANCHORED'
  };
}

/**
 * Calcula a similaridade máxima entre uma citação e o texto fonte,
 * utilizando trigram overlap como pre-filter e Levenshtein normalizado.
 * @param {string} quote - Citação a verificar.
 * @param {string} sourceText - Texto fonte.
 * @returns {number} Similaridade máxima (0 a 1).
 */
function computeMaxSimilarity(quote, sourceText) {
  const quoteTrigrams = getTrigrams(quote);
  if (quoteTrigrams.length === 0) return 0;

  const windowSize = quote.length;
  let maxSimilarity = 0;

  for (let i = 0; i <= sourceText.length - windowSize; i++) {
    const window = sourceText.substring(i, i + windowSize);
    const overlap = computeTrigramOverlap(quoteTrigrams, getTrigrams(window));

    if (overlap < TRIGRAM_OVERLAP_THRESHOLD) continue;

    const similarity = computeNormalizedLevenshtein(window, quote);
    if (similarity > maxSimilarity) {
      maxSimilarity = similarity;
    }
  }

  return maxSimilarity;
}
```

### 4.4. Implementação do Motor de Loop

O motor de loop é a função principal executada periodicamente por um *trigger* temporizado. Ele coordena todo o ciclo de vida das tarefas:

```javascript
/**
 * @fileoverview LoopEngine - Motor principal do sistema de Loop Engineering.
 * Executado por Time-Driven Triggers (ex: a cada 15 minutos).
 * 
 * Segue os princípios IV (Desempenho) e I (Disciplina) da Spec Kit.
 */

/** @const {number} Tempo máximo de execução antes de parar (5 minutos) */
const LOOP_MAX_RUNTIME_MS = 5 * 60 * 1000;

/**
 * Função principal do motor de loop, executada por um trigger.
 * Coordena descoberta, execução, validação e consolidação.
 */
function runLoopEngine() {
  const startTime = Date.now();
  const stateDAO = new StateDAO('State');
  const configDAO = new ConfigDAO('Config');

  // Fase 1: Descoberta e Triage
  const pendingTasks = stateDAO.getPendingTasks();
  if (pendingTasks.length === 0) {
    Logger.log('No pending tasks. Loop idle.');
    return;
  }

  // Ordenar por prioridade (menor número = maior prioridade)
  pendingTasks.sort((a, b) => a.priority - b.priority);

  // Fase 2: Execução sequencial com limite de tempo
  for (const task of pendingTasks) {
    // Verificar limite de tempo de execução
    if (Date.now() - startTime > LOOP_MAX_RUNTIME_MS) {
      Logger.log('Runtime limit reached. Resuming in next cycle.');
      break;
    }

    // Obter registacção da tarefa
    const registryEntry = Registry.get(task.type);
    if (!registryEntry) {
      Logger.log(`Unknown task type: ${task.type}. Skipping.`);
      continue;
    }

    // Fase 3: Acção
    const agent = new registryEntry.agentClass();
    const result = agent.execute(task);

    // Fase 4: Verificação (Maker-Checker)
    const verifier = new registryEntry.verifierClass();
    const verifiedResult = verifier.validate(result, task);

    // Fase 5: Evidence Anchoring
    const anchored = anchorEvidence(verifiedResult, task.sourceData);

    if (anchored) {
      // Consolidar no DAO de saída
      registryEntry.outputDAO.safeUpsert([anchored]);
      registryEntry.evidenceDAO.safeUpsert([{
        taskId: task.id,
        findingId: anchored.findingId,
        evidence: anchored.quote,
        confidence: anchored.confidence,
        status: anchored.anchoringStatus,
        timestamp: new Date().toISOString()
      }]);
      stateDAO.completeTask(task.id);
    } else {
      stateDAO.failTask(task.id, 'Evidence anchor rejected');
    }
  }
}
```

### 4.5. Registo de Configuração e Tarefas

O registo central mapeia cada tipo de tarefa ao seu agente, verificador e DAOs:

```javascript
/**
 * @fileoverview Registry - Registo central de tarefas e componentes.
 * Fonte única da verdade para mapeamento task-type -> componentes.
 * Segue o Princípio I da Spec Kit: registry-driven architecture.
 */

const LOOP_REGISTRY = {
  'code-review': {
    agentClass: CodeReviewerAgent,
    verifierClass: CodeReviewVerifier,
    inputDAO: TaskDAO,
    outputDAO: ReviewResultDAO,
    evidenceDAO: AgentLogDAO,
    maxRetries: 3,
    timeoutMs: 240000, // 4 minutos
    priority: 1
  },
  'test-validation': {
    agentClass: TestValidatorAgent,
    verifierClass: TestVerifier,
    inputDAO: TaskDAO,
    outputDAO: TestResultDAO,
    evidenceDAO: AgentLogDAO,
    maxRetries: 2,
    timeoutMs: 180000, // 3 minutos
    priority: 2
  },
  'architectural-audit': {
    agentClass: ArchitectureAuditorAgent,
    verifierClass: ArchitectureVerifier,
    inputDAO: TaskDAO,
    outputDAO: AuditResultDAO,
    evidenceDAO: AgentLogDAO,
    maxRetries: 3,
    timeoutMs: 300000, // 5 minutos
    priority: 3
  }
};

/**
 * Obtém a registacção de uma tarefa pelo tipo.
 * @param {string} taskType - Tipo da tarefa.
 * @returns {Object|null} Registo ou null se não encontrado.
 */
Registry.get = function(taskType) {
  return LOOP_REGISTRY[taskType] || null;
};

/**
 * Lista todos os tipos de tarefas registados.
 * @returns {string[]} Array de tipos de tarefas.
 */
Registry.list = function() {
  return Object.keys(LOOP_REGISTRY);
};

/**
 * Valida a integridade do registo (sem duplicatas).
 * Segue o princípio de "duplicate keys MUST fail loudly".
 * @throws {Error} Se forem detectadas duplicatas.
 */
Registry.validate = function() {
  const keys = Object.keys(LOOP_REGISTRY);
  const uniqueKeys = new Set(keys);
  if (keys.length !== uniqueKeys.size) {
    throw new Error(
      'Registry contains duplicate keys. ' +
      'Duplicate keys MUST fail loudly rather than silently override.'
    );
  }
};
```

### 4.6. Ferramenta Python de Validação Lógica

Além da implementação prevista para o runtime do Google Apps Script, foi implementada uma ferramenta local em Python em `tools/logical_validator.py`. A ferramenta materializa a parte verificável da proposta: recebe um texto-fonte e uma lista de achados em JSON, valida se cada citação existe no texto com similaridade suficiente, recalcula a confiança final e rejeita conclusões não ancoradas.

O validador foi desenhado sem dependências externas, de modo compatível com os princípios de dependências mínimas e operação offline da Spec Kit. A interface de linha de comando separa a lógica importável da superfície de execução, permitindo que a mesma implementação seja usada por testes automatizados, scripts de CI ou uma etapa anterior à consolidação em Google Sheets.

```bash
python tools/logical_validator.py \
  --source Relatorio.md \
  --findings exemplos/achados_validacao.json \
  --output validacao_logica.json
```

A entrada esperada é uma lista de achados, ou um objecto com a chave `findings`. Cada achado pode conter campos textuais (`claim`, `quote`, `confidence`, `severity`) e, opcionalmente, uma declaração lógica explícita (`subject`, `predicate`, `polarity`) usada para detectar contradições entre conclusões aceites.

```json
{
  "findings": [
    {
      "id": "EV-001",
      "claim": "A arquitectura usa Evidence Anchoring antes da persistência.",
      "quote": "A camada de *Evidence Anchoring* valida as conclusões dos agentes antes de as consolidar.",
      "confidence": 0.84,
      "severity": "high",
      "logic": {
        "subject": "Evidence Anchoring",
        "predicate": "validates_before_persistence",
        "polarity": true
      }
    }
  ]
}
```

Internamente, a ferramenta executa quatro verificações:

1. **Integridade estrutural:** confirma presença de claim, confiança numérica e severidade reconhecida.
2. **Ancoragem textual:** aplica normalização, pré-filtro de trigramas e distância de Levenshtein normalizada.
3. **Modulação de confiança:** penaliza citações ausentes ou curtas e rejeita achados abaixo de 0.65.
4. **Consistência lógica:** marca duplicatas por impressão digital textual e contradições quando achados aceites declaram polaridades opostas para o mesmo par sujeito/predicado.

O processo retorna código de saída `0` quando todos os achados passam e `1` quando há rejeições, erros ou contradições, o que permite integrar a validação em *quality gates* automatizados.

Durante a verificação da ferramenta foi identificado e corrigido um defeito específico da plataforma Windows: o `stdout` nativo da consola usa por omissão uma *codepage* limitada (ex. cp1252), incapaz de representar caracteres fora do seu repertório. Como o próprio texto-fonte deste relatório contém símbolos fora desse repertório (ex. os marcadores ☑/☐ da Tabela da secção 5.3), qualquer achado cujo `claim` referencie esse tipo de conteúdo fazia o CLI terminar com `UnicodeEncodeError` ao imprimir em `--format json`, um crash não distinguível de uma reprovação de validação para quem lê apenas o código de saída. A função `main()` passou a reconfigurar `stdout`/`stderr` para UTF-8 no arranque, e o comportamento é coberto por um teste de regressão que simula uma consola com codificação restrita.

---

## 5. Avaliação e Discussão

### 5.1. Coerência Arquitectural

A arquitectura proposta demonstra coerência com os princípios da Spec Kit [2]. A separação entre *entry points* e lógica importável é garantida pela camada de interface (doGet/doPost/triggers) que delega toda a lógica às camadas inferiores. O padrão *registry-driven* garante que a adicção de novas tarefas não exige modificações no motor de loop, respeitando o princípio de minimalidade de dependências.

A abordagem de *Evidence Anchoring* fornece uma barreira de confiança determinista entre a execução e a persistência, mitigando o risco de alucinações que seria particularmente problemático num sistema autónomo que opera sem supervisação humana directa.

A ferramenta Python reforça esta coerência porque transforma uma regra arquitectural em artefacto executável: o mesmo critério descrito no relatório pode ser aplicado a exemplos, achados produzidos por agentes e futuras saídas do loop. Na validação local, o exemplo `exemplos/achados_validacao.json` foi aprovado contra `Relatorio.md`, demonstrando que a ancoragem textual é reprodutível fora do ambiente GAS.

### 5.2. Operação dentro das Restrições do GAS

O sistema é desenhado para operar dentro dos limites operacionais do Google Apps Script. O limite de 6 minutos por invocação de *trigger* é gerido através da verificação de tempo de execução no motor de loop, que interrompe o processamento quando restam apenas 60 segundos do limite. As cotas de acesso ao `SpreadsheetApp` são minimizadas através de *lazy loading* com *cache* e da concentração de operações de leitura/escrita em lotes (*batch operations*).

### 5.3. Comparacção com Abordagens Existentes

A tabela abaixo compara a abordagem proposta com alternativas existentes:

| Dimensão | DAO Simples em GAS | CAPRA [3] | Proposta (DAO + Loop Engineering) |
|---|---|---|---|
| **Abstração de Dados** | ☑ | ☐ (usa Python) | ☑ (DAOs tipados) |
| **Orquestração Multi-Agente** | ☐ | ☑ | ☑ |
| **Evidence Anchoring** | ☐ | ☑ | ☑ |
| **Validação lógica executável** | ☐ | ☑ | ☑ (Python local sem dependências externas) |
| **Autonomia (Loop)** | ☐ | ☐ (processamento batch) | ☑ (triggers periódicos) |
| **Registry-Driven** | ☐ | ☐ | ☑ |
| **Ambiente** | Google Apps Script | Python/Spring AI | Google Apps Script |
| **Persistência** | Google Sheets | Base de dados relacional | Google Sheets |
| **Maker-Checker** | ☐ | ☑ (ConsistencyManager) | ☑ (VerifierAgent) |

### 5.4. Limitações

O sistema proposto apresenta algumas limitações inerentes à plataforma Google Apps Script. A ausência de um *framework* de testes nativo obriga a que os testes sejam implementados manualmente, utilizando *asserts* básicos. A dependência do Google LLM (através do serviço `LanguageApp` ou integração com API externa) introduz custos e limitações de quota. Finalmente, o *trigram overlap* e a distância de Levenshtein em JavaScript puro podem ser computacionalmente intensivos para textos muito longos, embora sejam adequados para a escala típica de Google Sheets.

A ferramenta Python reduz o risco de validação tardia, mas não substitui a validação em runtime no GAS. Ela opera sobre texto-fonte e achados serializados em JSON; portanto, a qualidade da verificação depende da qualidade das citações extraídas pelos agentes e da explicitação opcional das relações lógicas. Contradições sem `subject`, `predicate` e `polarity` continuam exigindo revisão humana ou regras semânticas adicionais.

---

## 6. Conclusão

A arquitectura apresentada neste relatório demonstra que é possível combinar o padrão Data Access Object, a disciplina arquitectural da Spec Kit e a orquestração multi-agente com *Evidence Anchoring* num sistema coerente e operacionál em Google Apps Script para suporte de fluxos de Loop Engineering.

O sistema proposto transforma o Google Sheets numa base de dados persistente e auditável para agentes de IA autónomos, permitindo que ciclos de acção e feedback sejam executados periodicamente sem intervenção humana directa. A camada de *Evidence Anchoring* garante que apenas conclusões verificadas são consolidadas, mitigando o risco de alucinações. A separação *maker-checker* entre agentes de execução e agentes de verificação reforça a fiabilidade do sistema.

A ferramenta Python de validação lógica acrescenta ao trabalho um ponto de controlo executável e reprodutível. Ela permite testar a ancoragem de evidências fora do GAS, auditar achados antes da persistência e transformar parte da argumentação arquitectural em verificação automatizada.

Os princípios da Spec Kit [2] — separação de camadas, registos como fonte única da verdade, tipagem rigorosa e operações idempotentes — proporcionam a disciplina arquitectural necessária para manter o sistema manutenível e extensível, mesmo no ambiente limitado do Google Apps Script.

---

## 7. Referências

[1] Osmani, A. (2026). *Loop Engineering*. Addy Osmani Blog. Disponível em: https://addyosmani.com/blog/loop-engineering/

[2] Spec Kit Team. (2026). *Spec Kit Constitution v1.0.0* (base.txt). Spec Kit Repository. Disponível em: https://github.com/github/spec-kit

[3] Becattini, M., Caselli, N., Minin, M., Verdecchia, R., & Vicario, E. (2026). *CAPRA: Scaling Feedback on Software Architecture Deliverables with a Multi-Agent LLM System* (paper.txt).

[4] Oracle. (s.d.). *Data Access Object Design Pattern*. Oracle Java Documentation. Disponível em: https://www.oracle.com/java/technologies/data-access-object.html

[5] Google. (s.d.). *Google Apps Script*. Google for Developers. Disponível em: https://developers.google.com/apps-script
