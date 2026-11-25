# Week 2 — OFDM Sync Short

### O que faz o bloco?

Deteta o início de um frame através do STS e deixa passar as amostras para o resto do recetor. Tem dois estados:

- **SEARCH:** descarta as amostras até c[n] ficar acima de `threshold` durante `min_plateau + 1` amostras seguidas.
  Nessa altura:
  - estima o CFO grosseiro: `arg(a[i]) / 16`;
  - insere a tag `wifi_start`;
  - passa a COPY.
- **COPY:** copia as amostras para a saída, já com o CFO grosseiro corrigido.
  - Volta a SEARCH ao fim de `MAX_SAMPLES` amostras (540 símbolos).
  - Aceita um frame novo se aparecer outro plateau depois de `MIN_GAP` = 480 amostras.

### Qual o significado dos limiares?

- **`threshold` (0.56):** nível de c[n] a partir do qual se considera que há um sinal com período de 16 amostras, ou
  seja, um STS.
  - No código atual, o valor máximo de c[n] é 48/64 = 0.75 (a potência é somada em 64 amostras e a autocorrelação em 48).
  - Com ruído, o plateau fica em c ≈ 0.75 · SNR / (1 + SNR). Para passar 0.56 é preciso SNR ≳ 4.7 dB.
- **`min_plateau` (2):** exige 3 amostras seguidas acima do limiar, para não disparar em picos isolados de ruído.

### Efeito de mudar os limiares

- **`threshold` mais baixo:** deteta frames com SNR mais baixa, mas o ruído começa a passar o limiar e aparecem falsos
  alarmes. Cada falso alarme envia lixo para o resto da cadeia e impede a deteção de um frame verdadeiro durante
  `MIN_GAP` amostras.
- **`threshold` mais alto:** há menos falsos alarmes, mas perdem-se frames fracos. Acima de 0.75 não deteta nada.
- **`min_plateau` maior:** há menos falsos alarmes e quase não se perde sensibilidade, porque o plateau verdadeiro dura
  cerca de 100 amostras. Se for maior do que o plateau, deixa de detetar.

### Limitações e casos em que falha

1. **SNR baixa:** abaixo de alguns dB os frames não são detetados.
2. **Tom contínuo ou offset DC:** um tom é periódico em qualquer atraso, por isso dá c ≈ 0.75. Se for alguns dB mais
   forte do que o ruído, o bloco dispara continuamente, a cada `MIN_GAP` amostras, e os frames verdadeiros perdem-se.
   Com o Pluto pode acontecer se a correção de DC estiver desligada.
3. **Frames próximos ou em colisão:**
   - um segundo frame que comece menos de cerca de `MIN_GAP` amostras depois do primeiro não é detetado;
   - um segundo frame com potência parecida também não, porque o primeiro funciona como ruído;
   - se o segundo for muito mais forte, o bloco muda para ele e perde o primeiro.
4. **Frames longos:** frames maiores do que `MAX_SAMPLES` são cortados e o CRC falha (com BPSK 1/2, a partir de cerca
   de 1600 bytes).
5. **CFO grosseiro limitado:** é estimado a partir de um único valor de a[n] e só cobre ±625 kHz a 20 Msps.

---

# Week 3 — OFDM Sync Long e correção do CFO

### O que faz o bloco OFDM Sync Long?

- Correlaciona as `sync_length` (320) amostras seguintes à tag com o padrão LT de 64 amostras (filtro adaptado).
- Procura dois picos afastados de 64 amostras. Daí tira o início exato do frame e o CFO fino.
- Copia depois os dois LT e os símbolos de dados sem o CP (64 amostras por símbolo), com o CFO corrigido.

### Algoritmo de correção do CFO

Um sinal que se repete com período L, quando tem CFO, aparece com uma rotação de fase de ω·L entre duas repetições.
Daqui sai:

  ω̂ = arg( Σ s[n] · s*[n−L] ) / L

A correção é multiplicar as amostras por e^{−jω̂n}. É feita em duas etapas:

- **Grosseira:** usa o STS, com L = 16. É pouco precisa, mas cobre ±625 kHz.
- **Fina:** usa o LTS, com L = 64. É cerca de 4× mais precisa, mas só cobre ±156 kHz.

Os 4 pilotos de cada símbolo corrigem depois a fase que sobra, no equalizador.

### Onde está implementado?

- **Grosseira:** `sync_short.cc`.
  - Estimação: `d_freq_offset = arg(in_abs[i]) / 16`.
  - Correção: `out[o] = in[o] * exp(gr_complex(0, -d_freq_offset * d_copied))`.
- **Fina:** `sync_long.cc`.
  - Estimação, em `search_frame_start()`: `d_freq_offset = arg(first * conj(second)) / 64`.
  - Correção, no estado COPY: `exp(gr_complex(0, d_offset * d_freq_offset))`.

### Que parâmetros se podem variar?

- **O atraso L:** maior dá mais precisão mas menos gama (a gama é ±fs/2L).
- **O tamanho da janela de soma** (`window_size`): uma janela maior dá menos ruído na estimativa.
- **O instante em que se lê a[n] no Sync Short:** mais tarde no plateau dá uma estimativa melhor.
- **Usar só uma das etapas.**
- **O `sync_length`** do Sync Long.

### Porque há uma entrada atrasada? E se mudar o valor?

O Sync Long só sabe onde começa o frame **depois** de analisar as 320 amostras. Mas, em GNU Radio, as amostras já
consumidas não voltam. Por isso recebe também o mesmo stream atrasado de 320 amostras:

- a entrada normal serve para encontrar os picos;
- quando a decisão está tomada, a entrada atrasada ainda está no início do segmento e pode copiar desde o início do
  frame.

Se o atraso deixar de ser igual a `sync_length`, as janelas da FFT ficam desviadas:

- **Atraso menor:** a janela começa tarde e apanha o símbolo seguinte. Há interferência entre símbolos e o frame perde-se.
- **Atraso um pouco maior (menos do que o CP):** a janela começa dentro do CP. Isso só causa uma rotação de fase, que o
  equalizador corrige, e funciona.
- **Atraso muito maior:** a janela apanha o símbolo anterior e o frame perde-se.

Se mudarem o `sync_length` (e o atraso com ele):

- se for pequeno de mais, os dois picos do LTS não cabem na janela e o alinhamento fica errado;
- se for grande, aumenta a latência.
