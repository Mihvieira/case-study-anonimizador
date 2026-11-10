# Como Anonimizar Holerites sem Quebrar Cálculos: Redação Geométrica em Java 25 vs. NLP (Microsoft Presidio)

> **Última atualização:** 10/11/2026

> [!NOTE]
> **Contexto de Engenharia e Propriedade Intelectual**  
> Este estudo de caso documenta as decisões de arquitetura de software, trade-offs e benchmarks de um sistema de anonimização em produção. Por razões de conformidade de segurança e segredo industrial, a aplicação integral possui código proprietário. As assinaturas e os trechos de código presentes são conceituais, ilustrativos e sanitizados, refletindo os padrões arquiteturais reais sem expor regras de negócio privadas.

---

## Metadados do Projeto

- **Camadas Arquiteturais:** Núcleo de Processamento Determinístico (Java 25), Sensor Óptico Sidecar (Python/ONNX) e Interface de Inspeção Interativa (Angular).
- **Stack Tecnológica:** Java 25, Spring Boot 3.4+, Apache PDFBox 3.x, OpenCV, RapidOCR (PP-OCRv4 / ONNXRuntime), Angular 19, Cloudflare Workers (BFF).
- **Padrões de Engenharia:** Zero-Disk In-Memory Architecture, Punch-Through Cutouts (`java.awt.geom.Area`), Imunidade Contábil Estrutural, Criptografia Efêmera em RAM (AES-256-GCM), Two-Phase Human-in-the-Loop API.

---

## 1. Contexto e o Desafio de Engenharia

O processamento automatizado de documentos trabalhistas (holerites, contracheques e extratos analíticos de FGTS) impõe exigências severas de conformidade com a LGPD (Lei nº 13.709/2018). Tais documentos contêm categorias críticas de dados que não podem trafegar em sua forma original para nuvens públicas ou Modelos de Linguagem (LLMs), classificadas segundo a taxonomia legal estrita:

1. **Dados Pessoais Diretos (Art. 5º, I da LGPD):** CPF, RG, PIS/PASEP, CTPS, matrícula funcional, endereço residencial, agência/conta bancária e nomes completos do empregado e empregador.
2. **Dados Pessoais Sensíveis (Art. 5º, II da LGPD):**
   * **Ocultos em Rubricas de Folha:** Verbas e descontos que revelam filiação a sindicato (*Contribuição Sindical / Assistencial*), dados de saúde ou assistência médica (*Coparticipação Unimed / Odonto / Atestados*) e provimentos de família (*Pensão Alimentícia Judicial*).
   * **Dados Biométricos Gráficos:** Assinaturas manuscritas cursivas, rubricas e carimbos nos canhotos de quitação e recebimento.

### O Dilema Operacional
Para realizar a liquidação de sentenças ou cálculo trabalhista (ex.: alimentação automatizada em motores como o PJe-Calc), o pipeline precisa impreterivelmente de metadados cadastrais como a **Competência** (mês/ano: `05/2024`) e a **Data de Admissão**, além dos valores nominais de vencimentos e descontos.

Tarjamentos cegos tradicionais apagavam todo o cabeçalho, destruindo esses parâmetros vitais; em contrapartida, tarjamentos parciais baseados unicamente em expressões regulares vazavam identificadores não estruturados para os motores externos.

---

## 2. A Abordagem Inicial e o Limite dos Modelos de NLP (Microsoft Presidio)

Inicialmente, avaliou-se o framework **Microsoft Presidio** (Python + spaCy / modelos Transformer), referência de mercado para sanitização textual em prosa livre. Contudo, testes empíricos em documentos tabulares trabalhistas revelaram gargalos estruturais intransponíveis:

- **Falsos Positivos em Rubricas Contábeis:** O analisador de NER (*Named Entity Recognition*) estatístico frequentemente classificava rubricas salariais e benefícios legítimos (ex.: *"Plano de Saúde Familiar"*, *"Seguro de Vida em Grupo"*, *"Adicional de Insalubridade 20%"*) como entidades a serem destruídas, censurando parcelas financeiras essenciais para a conta.
- **Descasamento Espacial (*Token-to-Box Drift*):** O Presidio analisa fluxos unidimensionais de texto (offsets `start`/`end`). Ao mapear esses offsets de volta para as caixas delimitadoras (*bounding boxes*) geradas pelo OCR em layouts tabulares de múltiplas colunas paralelas, ocorria colapso na ordem de leitura (*reading order collapse*), resultando no tarjamento de colunas numéricas incorretas.
- **Sobrecarga de Latência e Memória:** O pipeline Python composto por OCR + spaCy NLP + Presidio Anonymizer acumulava de **1.800 ms a 3.500 ms por página**, consumindo em torno de ~850 MB de memória RSS no container.

**Decisão de Engenharia:** O pipeline com Presidio foi **descontinuado** no fluxo central. Adotou-se uma abordagem determinística espacial-geométrica executada nativamente em **Java 25**.

---

## 3. Decisão Arquitetural: Centralização Canônica no Java 25

As responsabilidades foram segregadas em duas camadas com fronteiras rígidas:

```text
┌─────────────────────────────────┐
│     1. Ingestão Zero-Disk       │
│  - Documento em RAM (PDF/Img)   │
└────────────────┬────────────────┘
                 │ Stream em Memória
                 ▼
┌─────────────────────────────────┐
│ 2. Sensor Óptico (Sidecar Py)   │
│  - RapidOCR (PP-OCRv4 / ONNX)   │
│  - Emissão de [x, y, w, h, conf]│
└────────────────┬────────────────┘
                 │ Matriz de Caixas
                 ▼
┌─────────────────────────────────────────────────────────────┐
│             3. Núcleo Determinístico (Java 25)              │
│  ┌───────────────────────┐     ┌─────────────────────────┐  │
│  │ Zoneamento Morfológico│ ──> │ Janelas Livres (Punch)  │  │
│  └───────────────────────┘     └───────────┬─────────────┘  │
│                                            ▼                │
│  ┌───────────────────────┐     ┌─────────────────────────┐  │
│  │ Flattening a 300 DPI  │ <── │ Imunidade Contábil (Z2) │  │
│  └───────────────────────┘     └─────────────────────────┘  │
└────────────────┬────────────────────────────────────────────┘
                 │ Stream Final
                 ▼
┌─────────────────────────────────┐
│       4. Saída Blindada         │
│  - PDF Sanitizado sem vazamento │
└─────────────────────────────────┘
```

- **Sensor Óptico Sidecar (Python):** Atua estritamente como sensor de visão desacoplado. Executa o modelo PP-OCRv4 via ONNXRuntime (local em CPU) e emite exclusivamente texto bruto associado às coordenadas espaciais `[x, y, largura, altura, confianca]`. Nenhuma regra de conformidade ou decisão de privacidade reside no serviço Python.
- **Núcleo de Negócio e Segurança (Java 25):** Centraliza 100% da inteligência de domínio: zoneamento morfológico, cálculo de intersecções booleanas, aplicação de janelas livres, mascaramento visual e sanitização criptográfica.

### Benefícios Obtidos
- **Determinismo Absoluto:** Eliminação de variações estocásticas típicas de modelos de linguagem estatísticos.
- **Baixa Latência:** Decisão geométrica e queima física de tarjas em memória inferior a **120 ms por página** (~85 ms em média).
- **Enxugamento de Infraestrutura:** Remoção de dependências pesadas de NLP (`spacy`, `presidio-analyzer`, modelos Transformer) do container principal de produção.

---

## 4. Engenharia do Motor Determinístico: Zoneamento e Janelas Livres

A arquitetura no Java 25 segmenta o documento em zonas morfológicas e aplica operadores geométricos bidimensionais em três passadas coordenadas:

### A. Segmentação em Zonas Estruturais
1. **Zona 1 (Cabeçalho Cadastral):** Metadados do empregador (Razão Social, CNPJ, Endereço) e do trabalhador (Nome, CPF, CTPS, Matrícula, Cargo, Competência e Admissão).
2. **Zona 2 (Corpo Tabular de Rubricas):** Códigos, descrições de verbas, referências percentuais, proventos e descontos.
3. **Zona 3 (Rodapé, Totais e Canhotos):** Totais de vencimentos, deduções, valor líquido, bases fiscais (INSS/FGTS/IRRF), dados bancários e canhotos de assinatura.

### B. Janelas Livres (*Punch-Through Cutouts*) e Expansão Poligonal
Para neutralizar identificadores no cabeçalho sem perder os parâmetros da conta de liquidação, o motor combina navegação espacial e recorte booleano (`java.awt.geom.Area`). O cabeçalho recebe proteção *privacy-first*, preservando janelas transparentes com margem de segurança (*padding*) ao redor dos metadados autorizados para evitar o corte de glifos adjacentes:

```java
// Ilustração conceitual: Subtração booleana e preservação de Janelas Livres
Area tarjaCabecalho = new Area(retanguloCabecalhoSensivel);

for (ElementoGeometria elemento : elementosCabecalho) {
    if (termoMatcher.ehJanelaLivre(elemento.texto())) {
        // Expande o polígono com padding de segurança para assegurar
        // a legibilidade óptica de números (ex.: Competência MM/AAAA e Data de Admissão)
        Polygon janelaLivre = elemento.obterPoligonoComPadding(4, 3);
        tarjaCabecalho.subtract(new Area(janelaLivre)); // Punch-through transparente
    }
}
graphics2D.fill(tarjaCabecalho);

```

### C. Imunidade Contábil Absoluta

O componente de redação do corpo tabular impõe isolamento estrito de contexto: nenhum token contido na **Zona 2** (delimitada visualmente entre o cabeçalho de colunas e a linha de totais) pode ser submetido a heurísticas de censura por nomes próprios. Caso uma linha morfológica de assinatura no rodapé intersecte acidentalmente um valor monetário ou termo protegido, a tarja candidata é automaticamente descartada e registrada em log de auditoria.

### D. Suporte a Holerites de Duas Vias e Deskew Automático

* **Duas Vias:** Para documentos emitidos em duas vias na mesma página (via do empregador e via do empregado), o motor detecta a divisão morfológica via análise de contornos (OpenCV) e aplica o zoneamento de forma independente para cada quadrante.
* **Alinhamento Afim (*Deskew*):** Digitalizações com inclinação ≥ 0,5° passam por rotação inversa da matriz de imagem e transformação trigonométrica das caixas de OCR antes da queima das tarjas.

---

## 5. Ciclo de Vida Zero-Disk & Criptografia Efêmera em RAM

Para garantir conformidade estrita contra vazamentos em repouso (*Data-at-Rest*):

* **Pipeline 100% em Memória (*Zero-Disk I/O*):** Documentos PDF e matrizes de imagem trafegam exclusivamente em buffers de memória (`byte[]`, `ByteArrayInputStream`, `ByteArrayOutputStream` e `BufferedImage`). Nenhum arquivo temporário é gravado em disco ou no diretório `/tmp`.
* **Queima Física Matricial (*Pixel Burning*):** As tarjas são fundidas diretamente na matriz RGB da imagem via `Graphics2D` com `RenderingHints.VALUE_ANTIALIAS_OFF`, garantindo bordas 100% sólidas sem pixels semitransparentes nas extremidades.
* **Aplanamento Definitivo em RAM via Apache PDFBox 3.x (*PDF Flattening*):** Em documentos PDF vetoriais ou híbridos, o serviço rasteriza as páginas a **300 DPI** (`PDFRenderer.renderImageWithDPI`), aplica a queima matricial das tarjas e reconstrói o documento em memória via `LosslessFactory.createFromImage` sobre streams. Isso expurga a árvore de glifos vetoriais subjacentes e impede a recuperação do texto oculto por parsers externos.
* **Criptografia Simétrica Efêmera (AES-256-GCM):** Quando metadados sensíveis precisam ser repassados em memória para automações subsequentes (ex.: robôs RPA), o serviço de criptografia cifra cada campo com chaves simétricas descartadas assim que a requisição é concluída.

---

## 6. Arquitetura Human-in-the-Loop em Duas Fases

Para assegurar supervisão humana antes da destruição irreversível dos dados, a API opera em duas fases desacopladas:

1. **Fase 1 — Detecção & Inspeção Interativa (`POST /api/v1/anonimizador/inspecionar-tarjas`):**
* O backend processa o documento e retorna o preview visual acompanhado da malha de caixas aplicadas e descartadas.
* O frontend Angular renderiza o documento em Canvas interativo utilizando build local do PDF.js (Zero CDN externa, prevenindo vazamentos de telemetria).
* O operador pode inspecionar o motivo de cada tarja, redimensionar caixas, desenhar tarjas manuais ou restaurar regiões.


2. **Fase 2 — Queima e Aplanamento (`POST /api/v1/anonimizador/reaplicar-tarjas`):**
* O frontend submete a matriz final de coordenadas homologada pelo operador.
* O backend executa a queima matricial e o *flattening* a 300 DPI em RAM, disponibilizando o arquivo blindado via streaming de resposta.



---

## 7. Resultados Técnicos e Métricas Comparativas

A validação empírica e os benchmarks comparativos foram conduzidos sobre um corpus de homologação composto por **60 documentos trabalhistas** reais e sintéticos (**55 holerites/contracheques** de via única, via dupla e digitalizações ruidosas + **5 extratos analíticos de FGTS**), totalizando **128 páginas processadas** e **66 amostras anotadas**:

| Métrica Avaliada | Microsoft Presidio + spaCy (`pt_core_news_sm`) | Java 25 (Motor Determinístico) | Delta / Ganho |
| --- | --- | --- | --- |
| **Latência de Redação por Página** | ~2.450 ms (1.800 ms – 3.500 ms) | **~85 ms** (< 120 ms p99) | **~28x mais rápido** |
| **Falsos Positivos em Rubricas (Zona 2)** | 14,2% (verbas contábeis censuradas) | **0,0%** (Imunidade Contábil Estrita) | **100% de precisão contábil** |
| **Preservação de Competência / Admissão** | 0% (tarjamento de cabeçalho em bloco) | **100%** (Janelas Livres / *Punch-Through*) | **100% apto para liquidação** |
| **Consumo de Memória (Heap / RSS)** | Alto (~850 MB no runtime Python NLP) | Baixo / Otimizado (Streams efêmeros em RAM) | **Redução de ~65%** |

*Nota sobre a suíte de testes automatizados:* Cobertura integral nos fluxos críticos, validando cenários de documentos inclinados (*deskew*), holerites de duas vias, canhotos laterais destacados, carimbos de assinatura digital de tribunais e imunidade contábil da Zona 2.

---

## 8. Conclusão e Lições Aprendidas

1. **Prioridade da Geometria sobre Semântica Estatística:** Em documentos tabulares altamente estruturados como folhas de pagamento e extratos de FGTS, o zoneamento morfológico aliado a operadores booleanos espaciais supera modelos generalistas de NLP tanto em acurácia quanto em eficiência computacional.
2. **Harmonização entre Privacidade e Utilidade Processual:** O padrão de janelas livres (*punch-through cutouts*) comprova ser viável atender rigorosamente à LGPD sem inutilizar os metadados indispensáveis para o cálculo de liquidação trabalhista.
3. **Desacoplamento Arquitetural Estratégico:** Confinar o Python ao papel de sensor óptico puro (RapidOCR/ONNX) e centralizar as decisões de conformidade, geometria e criptografia no núcleo Java 25 resultou em um sistema determinístico, auditável e de altíssima performance.
