Este documento define a especificação técnica (Blueprint) para a arquitetura **AuraCEAF v4**, integrando rigorosamente os princípios de Separação Dimensional (DST), Engenharia de Representação (RepE) e Análise Topológica de Dados (TDA).

O objetivo é eliminar heurísticas arbitrárias (hardcoded limits) e substituí-las por controle geométrico dinâmico do espaço latente.

---

# Blueprint Arquitetural: AuraCEAF v4 (Neuro-Topological Framework)

## 1. Camada de Representação (DST Compliance)

O pré-requisito fundamental (conforme DST Theorem 1) é garantir volume dimensional suficiente para evitar o colapso representacional ($CS > 0$).

### 1.1. Modelo de Embedding
*   **Requisito:** Dimensionalidade $d \ge 768$ para maximizar a capacidade $C(S) \le (L/\epsilon)^d$.
*   **Implementação:** Substituição do `all-MiniLM-L6-v2` (384d) por `nomic-embed-text-v1.5` (768d, Matryoshka) ou `bge-m3` (1024d).
*   **Codificação Posicional Híbrida:**
    Para permitir a análise topológica de estruturas não-lineares (Graph-of-Thought), o vetor de entrada $x_i$ é definido como:
    $$x_i = E_{sem}(t_i) \oplus E_{struct}(L_{eigen})$$
    Onde $E_{sem}$ é o embedding semântico e $E_{struct}$ utiliza os autovetores do Laplaciano do grafo de raciocínio atual.

### 1.2. Recuperação de Memória (MBS)
*   **Remoção de Heurística:** Eliminação de `top_k=5`.
*   **Algoritmo de Recuperação:** Busca por raio ($\epsilon$-ball search).
    $$R = \{m \in M \mid \text{cosine\_dist}(q, m) < \epsilon_{dynamic}\}$$
*   **Consolidação Dinâmica:**
    Se $|R| > \text{limit}_{ctx}$, aplica-se clustering (DBSCAN) no conjunto recuperado $R$. Retorna-se apenas os centróides $C_j$ de cada cluster, preservando a topologia da informação enquanto reduz a carga de tokens.

---

## 2. Camada de Monitoramento (TDA Engine)

Este módulo opera em tempo real sobre o buffer de saída e o histórico de contexto, calculando a estabilidade estrutural do raciocínio.

### 2.1. Pipeline de Homologia Persistente
1.  **Input:** Janela deslizante $W$ dos últimos $N$ vetores de pensamento.
2.  **Complexo Simplicial:** Construção do complexo de Vietoris-Rips $VR_\epsilon(W)$.
3.  **Cálculo de Invariantes:**
    *   **$H_0$ (Conectividade):** Número de componentes conexos.
        *   *Sinal:* Se $H_0 \gg 1$ em escalas $\epsilon$ altas $\rightarrow$ Fragmentação Semântica (Drift).
    *   **$H_1$ (Ciclicidade):** Número de buracos/loops.
        *   *Sinal:* Se a persistência (death - birth) de um gerador $H_1$ excede threshold $\tau$ $\rightarrow$ Raciocínio Circular/Redundante.

### 2.2. Métrica de Tensão Epistêmica ($\xi$)
A tensão $\xi$ deixa de ser uma heurística e torna-se uma função escalar da entropia persistente:
$$\xi = \alpha \cdot \mathcal{E}(H_0) + \beta \cdot \mathcal{E}(H_1)$$
Onde $\mathcal{E}$ é a Entropia Persistente normalizada.

---

## 3. Camada de Controle (RepE Actuators)

Implementação de intervenções diretas no stream residual do Transformer, baseada em vetores de direção extraídos via Linear Artificial Tomography (LAT).

### 3.1. Vetores de Controle (Drivers)
O sistema mantém um banco de vetores pré-computados ($\mathbf{v}$) extraídos via PCA das diferenças de ativação:
*   $\mathbf{v}_{coherence}$: Vetor de coerência global.
*   $\mathbf{v}_{novelty}$: Vetor de exploração/criatividade.
*   $\mathbf{v}_{summary}$: Vetor de síntese/conclusão.
*   $\mathbf{v}_{critic}$: Vetor de autocrítica/verificação.

### 3.2. Mecanismo de Injeção
Hook na camada $L$ (onde $L$ é determinado empiricamente via LAT, geralmente camadas intermediárias-tardias):
$$h_L' = h_L + \lambda \cdot \mathbf{v}_{target}$$
Onde $\lambda$ é o coeficiente de intensidade, determinado dinamicamente pelo loop de feedback.

---

## 4. O Loop Homeostático (Integração TDA-RepE)

A lógica de controle (`agency_module.py`) deixa de ser baseada em texto (prompts) e passa a ser baseada em tensores.

### 4.1. Máquina de Estados Finitos (FSM) Guiada por Topologia

**Estado A: Divergência (Exploração)**
*   **Condição:** $H_0$ baixo, $H_1$ moderado (topologia de grafo saudável).
*   **Ação:** Nenhuma intervenção ou injeção leve de $\mathbf{v}_{novelty}$ para maximizar cobertura do espaço.

**Estado B: Fragmentação (Alucinação/Drift)**
*   **Condição:** $H_0$ alto (componentes desconexos), $\xi > 0.7$.
*   **Diagnóstico:** O modelo perdeu a coerência contextual.
*   **Intervenção RepE:** Injeção agressiva de $\mathbf{v}_{coherence}$.
*   **Intervenção RAG:** Trigger imediato para chamada de ferramenta de busca para re-ancoragem (grounding).

**Estado C: Estagnação (Loop)**
*   **Condição:** $H_1$ com alta persistência (ciclo estável).
*   **Diagnóstico:** O modelo está repetindo lógica sem progresso informacional.
*   **Intervenção RepE:** Injeção de $\mathbf{v}_{summary}$ ou $\mathbf{v}_{critic}$ para forçar o colapso da topologia atual e transição para o próximo token de conclusão.

---

## 5. Especificação de Fluxo de Dados (Pipeline)

1.  **Input:** Query do usuário.
2.  **Vector Retrieval:** Busca $\epsilon$-ball no MBS. Recuperação de subespaço vetorial relevante.
3.  **Initial Generation:** LLM inicia geração de tokens.
4.  **Real-time Monitoring (Async):**
    *   A cada $N$ tokens, extrair embeddings.
    *   Calcular Barcode de Persistência ($H_0, H_1$).
    *   Atualizar escalar $\xi$.
5.  **Control Gate:**
    *   Se $\xi < \text{threshold}$: Continuar geração.
    *   Se $\xi \ge \text{threshold}$:
        1.  Interromper geração.
        2.  Selecionar vetor $\mathbf{v}$ baseado na assinatura topológica (Fragmentação vs. Loop).
        3.  Aplicar Hook RepE.
        4.  Reiniciar geração dos últimos $K$ tokens (backtrack) ou continuar com novo viés.
6.  **Output:** Resposta gerada.
7.  **Memory Consolidation:** O grafo final de raciocínio é armazenado. Se $H_0$ final for alto, aplica-se sumarização antes do armazenamento para garantir densidade informacional no futuro (compressão lossy baseada em relevância).

## Resumo Técnico

A **AuraCEAF v4** não "pensa" que está raciocinando; ela **mede a geometria** de suas próprias ativações. Se a geometria se desfaz (fragmentação) ou se fecha (loop), o sistema intervém algebricamente no fluxo residual, forçando o modelo a convergir para uma topologia de variedade (manifold) saudável, correlacionada estatisticamente com a verdade e a coerência lógica.
