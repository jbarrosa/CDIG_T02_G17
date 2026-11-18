# Recetor IEEE 802.11a/g/p em GNU Radio — Week 1 (última pergunta), Week 2 e Week 3

> Base: artigo *An IEEE 802.11a/g/p OFDM Receiver for GNU Radio* (Bloessl et al., SRIF'13) e o código atual do
> `gr-ieee802-11` (branch `maint-3.10`: `lib/sync_short.cc`, `lib/sync_long.cc`, `examples/wifi_phy_hier.grc`).
>
> Todas as figuras foram geradas com um simulador em Python (pasta `sim/`) que reproduz **amostra a amostra** o
> flowgraph `wifi_phy_hier.grc` e os blocos `sync_short`/`sync_long` (port direto do C++). O frame tem preâmbulo
> 802.11a real, SIGNAL BPSK e dados QPSK aleatórios (sem codificação convolucional — chega para estudar
> deteção e sincronização). Para reproduzir: `python3 sim/exp_week1.py`, `exp_week2.py`, `exp_week3.py`.

**Nota importante — o código mudou desde o artigo.** Há três diferenças que aparecem em várias respostas:

| | Artigo (2013) | Código atual (`maint-3.10`) |
|---|---|---|
| Normalização de c[n] | p[n] somado na **mesma** janela N_win (eq. 2) → plateau ≈ 1 | p[n] somado em **N_win + 16 = 64** amostras → plateau ≈ 48/64 = **0.75** |
| CFO | só com o STS, dentro do Sync Long (eq. 4) | **grosseiro** no Sync Short (STS, lag 16) + **fino** no Sync Long (LTS, lag 64) |
| Alinhamento | início = max(N_P) + 64 (eq. 7) | início = **mínimo** dos dois picos (início do 1.º LTS); os 2 LTS também são enviados para a estimação de canal |

---

## Week 1 — última pergunta: efeito de N_win (gráfico tipo Figura 2)

### Como o flowgraph calcula c[n]

No `wifi_phy_hier.grc` (`window_size = 48`):

| Grandeza | Blocos GRC | Equação |
|---|---|---|
| autocorrelação a[n] | `Delay(16)` → `Conjugate` → `Multiply` (com s[n]) → `Moving Average(window_size)` | a[n] = Σ_{k=0}^{W−1} s[n−k]·s*[n−k−16] |
| \|a[n]\| | `Complex to Mag` | |
| potência p[n] | `Complex to Mag²` → `Moving Average(window_size + 16)` | p[n] = Σ_{k=0}^{W+15} \|s[n−k]\|² |
| c[n] | `Divide` | c[n] = \|a[n]\| / p[n] |

A potência é somada em W+16 amostras porque a soma de a[n] "toca" W+16 amostras (W produtos, cada um com lag 16).
Num sinal perfeitamente periódico (STS sem ruído), \|a\| = W·P e p = (W+16)·P, por isso:

**c_plateau = W / (W + 16)** → para W = 48 dá **0.75** (e não 1, como na Figura 2 do artigo).

Com ruído (SNR = S), o termo de ruído de a[n] tem média nula mas p[n] soma sinal + ruído:

**c_plateau ≈ W/(W+16) · S/(1+S)**

### Réplica da Figura 2 (W = 48)

![Réplica da Figura 2](figs/w1_fig2_replica.png)

Plateau a ≈ 0.75 durante o STS (amostras 0–160), queda rápida no LTS (período 64, não 16) e valores baixos (~0.1–0.2)
nos dados OFDM, porque o CP só correlaciona 16 amostras em cada 80.

### c[n] para vários N_win (mesmo frame, SNR 10 dB)

![c[n] para vários N_win](figs/w1_nwin_traces.png)

### Métricas vs N_win

![Métricas vs N_win](figs/w1_nwin_metrics.png)

| N_win | teto W/(W+16) | plateau medido (20 dB) | P(det) a 3 dB — GRC | P(det) a 3 dB — artigo | falsos alarmes/s — artigo* |
|---:|---:|---:|---:|---:|---:|
| 8   | 0.33 | 0.33 | 0    | 1.00 | 37 550 |
| 16  | 0.50 | 0.50 | 0    | 1.00 | 17 450 |
| 24  | 0.60 | 0.59 | 0    | 1.00 | 2 600 |
| 32  | 0.67 | 0.66 | 0.03 | 1.00 | 350 |
| **48** | **0.75** | **0.74** | **0.45** | 1.00 | 0 |
| 64  | 0.80 | 0.79 | 0.74 | 1.00 | 0 |
| 96  | 0.86 | 0.85 | 0.85 | 1.00 | 0 |
| 128 | 0.89 | 0.88 | 0.91 | 1.00 | 0 |
| 144 | 0.90 | 0.89 | 0.88 | 0.99 | 0 |
| 192 | — | 0.89 | 0.28 | 0.46 | 0 |
| 256 | — | 0.88 | 0.00 | 0.01 | 0 |

\* limiar 0.56, `min_plateau = 2`, 20 ms de ruído. Com a normalização do GRC houve **0** falsos alarmes para todos os W.

### Conclusões

1. **W pequeno (≤ 16): o recetor deixa de detetar.** Com a normalização do GRC o teto W/(W+16) fica abaixo do
   limiar 0.56 (W=16 → 0.50). O limiar está "afinado" para W = 48; se mudar W tem de mudar também o `sensitivity`.
2. **W pequeno com a normalização do artigo: dispara sem parar.** O teto passa a 1, mas a variância de c[n] em ruído
   é ∝ 1/W e o ruído passa o limiar milhares de vezes por segundo (37 550/s com W = 8).
3. **W maior → mais sensível e c[n] mais "limpo"** (a média de 48 → 128 produtos reduz o ruído), por isso P(det) a 3 dB
   sobe de 0.45 (W=48) para ~0.9 (W=128).
4. **Mas o plateau encolhe.** O valor máximo só existe enquanto a janela (W+16) está toda dentro dos 160 amostras do STS:
   dura 145 − W amostras (97 para W=48, 1 para W=144). Para **W > 144** a janela apanha o LTS/dados e o plateau
   nunca chega ao teto → a deteção a baixa SNR cai a pique (0.28 com W=192, ~0 com W=256).
5. **Latência e estimação grosseira do CFO**: com SNR alta o trigger dá-se ~39 amostras depois do início do frame,
   quase independentemente de W (é a rampa de subida que interessa). A estimativa grosseira de CFO usa a[n] nesse
   instante (ver Week 3), por isso W também afeta a precisão do CFO.
6. W = 48 é um compromisso razoável: plateau largo (~100 amostras, dá folga ao `min_plateau` e ao Sync Long),
   ruído baixo, e janela pequena face ao STS. W entre 64 e 128 ganha ~1–2 dB de sensibilidade.

### Como fazer em GNU Radio

1. Abrir `examples/wifi_phy_hier.grc`, mudar a variável `window_size`, gerar o bloco hierárquico (▶/"Generate")
   e reabrir o flowgraph que o usa (`wifi_rx.grc` ou `wifi_loopback.grc`).
2. Para ver c[n]: numa cópia do hier block, ligar a saída do `Divide` a um `QT GUI Time Sink` (float) com
   *Trigger mode = Normal*, nível 0.5 — o gráfico fica igual à Figura 2. Alternativa: `File Sink` e plotar em Python.
3. Comparar W = 16, 48, 96 com `wifi_loopback.grc` (bloco `Channel Model`: *noise voltage* para baixar a SNR).

---

## Week 2 — OFDM Sync Short

### O que faz o bloco

Entradas (3): `in[0]` = amostras atrasadas de 16 (saída do `Delay(16)`), `in[1]` = a[n] complexo (saída do
`Moving Average`), `in[2]` = c[n] (saída do `Divide`). Saída: amostras complexas. É uma **"válvula"** com dois estados:

- **SEARCH** — descarta tudo e procura um plateau: c[n] > `threshold` em `min_plateau + 1` amostras **consecutivas**
  (`min_plateau = 2` → 3 amostras, como diz o artigo). Quando encontra:
  - estima o **CFO grosseiro** `d_freq_offset = arg(a[i]) / 16` (rad/amostra),
  - insere uma tag `wifi_start` (valor = CFO grosseiro) na amostra de saída,
  - passa a COPY.
- **COPY** — copia as amostras para a saída **já com o CFO grosseiro corrigido**:
  `out[o] = in[o] · exp(−j · d_freq_offset · d_copied)`. Continua a vigiar c[n]: se aparecer outro plateau **e**
  já tiver copiado mais de `MIN_GAP = 480` amostras, considera que é um novo frame (nova tag, novo CFO, `d_copied = 0`).
  Ao fim de `MAX_SAMPLES = 540·80 = 43 200` amostras (2.16 ms a 20 Msps) volta a SEARCH.

O bloco não sabe o comprimento do frame (só o SIGNAL, mais à frente, o diz), por isso copia sempre "um frame grande".

### Significado dos limiares

| Parâmetro | Onde | Valor | Significado |
|---|---|---|---|
| `threshold` (`sensitivity` no hier block) | argumento do bloco | 0.56 | nível de c[n] a partir do qual há "correlação de período 16". Como o teto é 0.75, exige 0.75·S/(1+S) > 0.56 → **SNR ≳ 4.7 dB** |
| `min_plateau` | argumento | 2 | nº de amostras extra acima do limiar → evita disparar em picos isolados de ruído |
| `MIN_GAP` | constante no `.cc` | 480 | nº mínimo de amostras copiadas antes de aceitar um novo frame (24 µs ≈ preâmbulo 16 µs + SIGNAL 4 µs + margem) |
| `MAX_SAMPLES` | constante no `.cc` | 43 200 | comprimento máximo copiado = 540 símbolos OFDM |

### Efeito de mudar os limiares

![Deteção vs SNR e falsos alarmes](figs/w2_threshold.png)

Linhas pontilhadas: SNR onde 0.75·S/(1+S) = limiar (previsão teórica do joelho).

| limiar | SNR com P(det) ≈ 50 % | SNR com P(det) ≈ 100 % | falsos alarmes/s (só ruído, min_plateau 2) |
|---:|---:|---:|---:|
| 0.30 | < −2 dB | ~ −1 dB | **500** |
| 0.45 | 0 dB | 2 dB | 0 |
| **0.56** | **3 dB** | **5 dB** | **0** |
| 0.65 | 6.5 dB | 9 dB | 0 |
| 0.70 | 9.5 dB | 11 dB | 0 |

- Em ruído puro, o máximo de c[n] em 50 ms foi **0.36**. Abaixo de ~0.35 os falsos alarmes crescem explosivamente
  (21 000/s a 0.2). Cada falso alarme envia ≥ 480 amostras de lixo para o Sync Long/equalizador (CPU) e, durante
  essas 480 amostras, um frame verdadeiro que chegue **não é detetado**.
- Acima de ~0.65 perdem-se frames que seriam descodificáveis (BPSK 1/2 descodifica-se a poucos dB de SNR).
- `min_plateau` maior reduz falsos alarmes (~8× entre 0 e 8 a limiar 0.3) mas pouco muda a sensibilidade,
  porque o plateau verdadeiro dura ~100 amostras.
- 0.56 fica no "joelho": 0 falsos alarmes com margem e deteção a partir de ~4–5 dB.

### Limitações e casos onde falha

**1. SNR baixa.** Como o teto é 0.75 e p[n] inclui o ruído, abaixo de ~3 dB perde-se metade dos frames com o limiar
por omissão (figura acima). Com o Pluto isto aparece quando se afasta o AP ou se baixa o ganho.

**2. Tom CW / offset DC → disparos contínuos (o caso mais fácil de demonstrar).** Qualquer sinal periódico com
período que divida 16 amostras dá c ≈ 0.75 — um tom puro e^{jωn} é periódico em *qualquer* lag. Com um tom
T dB acima do ruído, c ≈ 0.75·T/(T+N):

![Tom CW](figs/w2_tone.png)

- A partir de um tom ~3 dB acima do ruído o bloco dispara **a cada 481 amostras** (41 600 triggers/s = 20 Msps / 481):
  dispara, copia `MIN_GAP` amostras, dispara de novo…
- O frame verdadeiro só é apanhado se, por sorte, um re-trigger cair no seu STS. No exemplo, o último trigger antes do
  frame caiu 69 amostras antes do início, o Sync Long procurou os picos numa janela errada e o **EVM foi +4.8 dB**
  (frame perdido; sem tom: −14 dB).
- **No Pluto** isto acontece com o *LO leakage*/offset DC se desligar no bloco `PlutoSDR Source` as correções
  automáticas de DC (RF DC / BB DC), ou com um tom de um gerador / outro transmissor de banda estreita. Em simulação
  (`wifi_loopback.grc`) basta somar um `Signal Source` (cosseno complexo) à entrada.

**3. Dois frames próximos / colisão.** Frame B a chegar Δ amostras depois do início de A (A tem 60 símbolos):

![Colisão](figs/w2_collision.png)

- **B nunca é detetado se Δ ≲ 370 amostras**: B só pode ser aceite depois de `d_copied > MIN_GAP` e o seu plateau
  dura ~110 amostras (480 − 110 ≈ 370).
- **B com a mesma potência que A nunca é detetado**, porque A atua como ruído: c ≈ 0.75·½ = 0.375 < 0.56.
  É preciso B ≳ 5 dB mais forte (com 4 dB fica no limite e depende do conteúdo de A).
- Se B for mais forte, o bloco **muda para B e A perde-se** (o resto de A é truncado).
- O caso normal do 802.11 (ACK depois de SIFS = 16 µs = 320 amostras) **funciona**: o ACK tem 480 amostras e o
  frame seguinte foi detetado na posição certa. Esta é uma melhoria face ao artigo, onde o frame seguinte se perdia.

**4. Frames mais longos que `MAX_SAMPLES`.** Um frame com 600 símbolos de dados (48 400 amostras) foi cortado em
43 200 → só **532 símbolos** chegaram ao Sync Long; o resto do PSDU perde-se e o CRC falha. Com BPSK 1/2 (6 Mbit/s,
24 bits/símbolo) isto acontece para PSDUs acima de ~1 590 bytes; o 802.11a permite até 4095 bytes.

**5. Outras limitações de desenho.**
- A estimativa grosseira de CFO usa **um único valor** a[i], no instante do trigger, quando a janela ainda não está
  toda dentro do STS (ver Week 3) → é ruidosa (~10 kHz RMS).
- Gama de CFO limitada a ±π/16 rad/amostra = **±625 kHz** a 20 Msps (±312.5 kHz no 802.11p a 10 Msps).
- O instante do trigger não é preciso (depende da SNR e do limiar) — só serve para "abrir a válvula"; a sincronização
  fina é responsabilidade do Sync Long, que precisa de ter os dois picos do LTS na janela de `sync_length` (Week 3).
- Se não houver sinal (amostras a zero), c = 0/0; não é um problema real com ruído do ADC.

---

## Week 3 — OFDM Sync Long e correção do desvio de frequência

### O que faz o bloco (Section 2.3/2.4, versão atual)

Entradas: `in[0]` = saída do Sync Short; `in[1]` = a mesma saída atrasada `sync_length = 320` amostras (`Delay`).
Máquina de estados:

- **SYNC** — quando chega uma tag `wifi_start`, filtra as próximas `sync_length` amostras com o **filtro adaptado ao
  LTS** (vetor `LONG`, 64 coeficientes, `fir_filter_ccc`). Guarda (valor, índice) de cada saída.
- `search_frame_start()` — ordena por \|correlação\|, procura entre os 4 maiores um par afastado de **64** amostras
  (aceita 63/65 como recurso). Daí tira:
  - `d_frame_start = min(índices)` → início do 1.º símbolo LTS;
  - **CFO fino**: `d_freq_offset = arg(first · conj(second)) / 64` — os dois picos são correlações com a mesma
    sequência separadas de 64 amostras, logo diferem de uma rotação de 64·ω.
  - Se não encontrar par, usa `d_frame_start = sync_length` (valor por omissão).
- **COPY** — lê `in_delayed` a partir de `d_frame_start`, multiplica por `exp(j · d_offset · d_freq_offset)` e
  envia os 2×64 amostras do LTS e depois, para cada símbolo de 80 amostras, só as 64 sem CP
  (`rel < 128 || ((rel−128) % 80) > 15`). Na primeira amostra insere a tag `wifi_start` com o CFO total
  (`d_freq_offset_short − d_freq_offset`).
- **RESET** — quando chega a tag do frame seguinte, completa o vetor de 64 com zeros (para o `Stream to Vector`
  não ficar desalinhado) e volta a SYNC.

### Algoritmo de correção do CFO

O CFO Δf roda a amostra n por e^{j2πΔf·n/fs}. Num sinal periódico de período L, s[n]·s*[n−L] tem fase 2πΔf·L/fs,
logo:

  ω̂ = arg( Σ s[n]·s*[n−L] ) / L    [rad/amostra],   Δf̂ = ω̂·fs / 2π

Somar os produtos **antes** de tirar o argumento (eq. 4 do artigo) pondera automaticamente as amostras fortes e é
muito mais robusto do que fazer a média dos ângulos.

O código atual faz isto em **duas etapas**:

| Etapa | Onde | Sequência / lag | Gama sem ambiguidade (20 Msps) | Precisão |
|---|---|---|---|---|
| grosseira | `sync_short.cc`, `d_freq_offset = arg(in_abs[i]) / 16`; aplicada em `out[o] = in[o]*exp(−j f d_copied)` | STS, L = 16 | ±fs/(2·16) = **±625 kHz** | baixa |
| fina | `sync_long.cc`, `search_frame_start()`; aplicada em `out[o] = in_delayed[i]*exp(j d_offset f)` | LTS, L = 64 | ±fs/(2·64) = **±156 kHz** | ~4× melhor |
| residual (por símbolo) | `frame_equalizer` com os 4 pilotos (Week 5) | — | — | segue o desvio de fase restante |

O lag 64 dá 4× mais rotação por erro → mais precisão, mas dá a volta (aliasing) mais cedo. Por isso a etapa
grosseira traz o erro para dentro de ±156 kHz e a fina limpa o resto. Referência de necessidade: o 802.11 exige
±20 ppm por estação → no pior caso **40 ppm**: 97 kHz a 2.4 GHz, 232 kHz a 5.8 GHz e 236 kHz a 5.9 GHz (802.11p,
onde a 10 Msps a etapa fina só cobre ±78 kHz). A 5 GHz **só a etapa fina não chega**.

![Gama de captura](figs/w3_cfo_range.png)

- Grosseira + fina: estimativa correta e EVM ≈ −24 dB em **±600 kHz**; aliasing a ±625 kHz.
- Só fina (correção grosseira desligada): correta só em **±150 kHz**; fora disso estima um valor errado de
  múltiplos de 312.5 kHz e o frame perde-se.

![Erro RMS do CFO](figs/w3_cfo_rms.png)

- A estimativa grosseira no instante do trigger tem um patamar de erro de ~5–13 kHz (≈2–4 % do espaçamento de
  312.5 kHz — sozinha já degradaria QAM).
- A estimativa final (grosseira + fina) fica em 4.3 kHz a 0 dB e 0.3 kHz a 25 dB.
- Usar a[n+64] (janela inteiramente no STS) melhora a grosseira 2–6× acima de 6 dB, mas a SNR baixa o trigger
  chega tarde e a janela cai no LTS → erro enorme. É um exemplo de parâmetro que se pode afinar.

### Que parâmetros se podem variar

- **Lag L** (16 no STS, 64 no LTS): compromisso gama ↔ precisão (gama = fs/2L). Poder-se-ia usar lag 32/48 no STS.
- **Janela/número de produtos somados** (`window_size` para a etapa grosseira; 64 no LTS): mais produtos → menos ruído.
- **Instante em que se lê a[n]** no Sync Short (no trigger vs. mais tarde no plateau) — figura acima.
- **Usar só uma das etapas** (desligar a grosseira → gama ±156 kHz; desligar a fina → precisão ~10 kHz).
- **Tolerância do par de picos** (63/64/65) e **`sync_length`** (janela de pesquisa, ver abaixo).
- Fora do recetor: **calibrar o oscilador do Pluto** (atributo `xo_correction` do AD9361) para reduzir o CFO de
  partida — o erro do cristal do Pluto pode ser de vários ppm, o que a 2.4/5 GHz são dezenas de kHz.

### Porque é que há uma entrada atrasada? E se mudar o atraso?

**Porquê.** O Sync Long só sabe onde começa o frame (e qual o CFO fino) **depois** de ter visto as `sync_length` = 320
amostras e calculado os picos. Mas num bloco GNU Radio as amostras consumidas desaparecem — não se pode "voltar
atrás". A solução é ter o mesmo stream atrasado de `sync_length` amostras: quando a decisão é tomada (`d_offset` volta
a 0 na amostra 320 de `in[0]`), `in_delayed[i]` está a mostrar exatamente a amostra 0 do segmento, e
`in_delayed` na posição `d_frame_start` é o início do LTS. Por outras palavras, o `Delay` é um buffer que permite
olhar para o futuro (`in[0]`) e copiar o passado (`in[1]`). Custa 320 amostras de latência (16 µs a 20 Msps).

O código **assume Delay = sync_length**. Variando o atraso para `sync_length + k`, a janela de cada símbolo desloca-se
k amostras:

![Efeito do atraso](figs/w3_delay.png)

- **k < 0** (atraso menor): a janela FFT começa **tarde** → apanha o CP do símbolo seguinte → ISI. Basta k = −2
  para o EVM passar de −22.5 dB para −9 dB.
- **0 ≤ k ≲ 16 − espalhamento do canal** (aqui ~4 amostras): a janela começa **cedo**, dentro do CP. Isto só causa uma
  rampa de fase linear nos subportadores, igual no LTS e nos dados → o equalizador absorve-a. EVM inalterado até
  k ≈ 14.
- **k > 16**: a janela sai do CP e apanha o símbolo anterior → ISI → frame perdido.

Ou seja, um atraso ligeiramente **maior** do que `sync_length` é tolerado (dá margem contra o espalhamento),
um atraso menor não.

**Mudar `sync_length` (com Delay = sync_length):**

![sync_length](figs/w3_sync_length.png)

- A janela tem de conter os **dois** picos do LTS. Com limiar 0.56 o trigger dá-se ~39 amostras depois do início do
  frame e, como a saída do Sync Short vem atrasada de 16, a janela começa na amostra ~23 do frame; por isso os picos estão nos índices ~169 e ~233 da janela → é preciso
  **sync_length ≳ 240**. Com limiar 0.72 o trigger é mais tardio e 224 já chega.
- Com 192–224 só um pico (ou o pico parcial do GI2, 64 amostras antes) cabe na janela → par errado → alinhamento
  errado de um símbolo inteiro.
- O ponto "bom" em 160 é **coincidência**: sem par válido o código usa `d_frame_start = sync_length = 160`, que neste
  caso caiu dentro do GI2 do LTS.
- Valores grandes funcionam (até 640 na simulação) mas aumentam a latência e o buffer. Há ainda um risco no código
  real: se chegar uma nova tag enquanto o bloco ainda está em SYNC, ele lança `runtime_error("wtf")`. Como o Sync Short
  pode gerar uma nova tag ao fim de `MIN_GAP = 480` amostras, **sync_length > ~480 pode fazer o flowgraph abortar**
  com dois frames seguidos ou com um falso alarme (não modelado no simulador).

### Resumo Week 3

1. Sync Long = filtro adaptado ao LTS + escolha de dois picos a 64 amostras → início exato do frame e CFO fino.
2. CFO em duas etapas (STS lag 16 no Sync Short, LTS lag 64 no Sync Long) + seguimento por pilotos no equalizador.
3. A entrada atrasada é o "buffer" que permite decidir com o futuro e copiar o passado; o atraso tem de ser
   ≥ `sync_length` e no máximo ~`sync_length` + (CP − espalhamento do canal).

---

### Ficheiros

- `figs/*.png` — figuras.
- `sim/wifi_sim.py` — preâmbulo 802.11a, canal, autocorrelação (blocos GRC), port de `sync_short` e `sync_long`, EVM.
- `sim/exp_week1.py`, `sim/exp_week2.py`, `sim/exp_week3.py` — experiências que geram as figuras e números deste documento.
