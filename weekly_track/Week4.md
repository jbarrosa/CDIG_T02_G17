# Week 4 — Alinhamento de símbolos (Secção 2.4)

### Como é feito o alinhamento de símbolos?

Cada símbolo OFDM tem 80 amostras: 16 de CP e 64 de dados. O objetivo é saber exatamente onde começa cada símbolo, para
retirar o CP e entregar à FFT blocos de 64 amostras. Isto é feito no **OFDM Sync Long** com o LTS, que tem um GI2 de 32
amostras seguido de duas cópias do padrão LT de 64 amostras.

1. As primeiras N_preamble amostras depois da tag do Sync Short são correlacionadas com o padrão LT (filtro adaptado).
2. A saída tem dois picos estreitos, um no início de cada cópia de LT, separados de 64 amostras.
3. Escolhem-se os maiores picos (eq. 6) e calcula-se o início do primeiro símbolo de dados (eq. 7).
4. A partir daí o bloco envia, para cada símbolo, só as 64 amostras sem CP. A primeira amostra leva uma tag.

No código atual, o bloco procura dois picos afastados de 64 amostras e usa o **primeiro**, para enviar também os dois
LT para a estimação de canal. O resultado é o mesmo, porque min + 128 = max + 64.

### Porque se usa filtro adaptado no alinhamento mas não na deteção?

- **Custo:** a deteção corre sobre todas as amostras, continuamente.
  - A autocorrelação custa cerca de 1 multiplicação por amostra.
  - O filtro adaptado custa 16 a 64 multiplicações por amostra.
  - No alinhamento, o filtro só corre sobre umas centenas de amostras por frame, por isso o custo é aceitável.
- **CFO:** a autocorrelação compara o sinal consigo próprio, por isso o CFO só lhe muda a fase. O filtro adaptado
  compara com o padrão ideal, e com CFO o pico degrada-se. No Sync Long o CFO grosseiro já foi corrigido.
- **Precisão:** a autocorrelação dá um plateau largo, bom para dizer "há um frame". O filtro adaptado dá picos de uma
  amostra, necessários para o alinhamento exato.

### Código que implementa a equação 6

Equação 6: N_P = argmax3_n | Σ_{k=0}^{63} s[n+k] · LT*[k] |

Em `sync_long.cc` (excertos simplificados):

```cpp
// filtro adaptado com os 64 coeficientes do LT
d_fir(gr::filter::kernel::fir_filter_ccc(LONG))

// estado SYNC: correlação das primeiras SYNC_LENGTH amostras
d_fir.filterN(d_correlation, in, std::min(SYNC_LENGTH, std::max(ninput - 63, 0)));
d_cor.push_back(std::pair<gr_complex, int>(d_correlation[i], d_offset));

// search_frame_start(): argmax, ordenando por |correlação|
d_cor.sort(compare_abs);
// procura entre os 4 maiores um par com diff == 64
d_frame_start = min(get<1>(vec[i]), get<1>(vec[k]));
```

### Porque se soma 64 na expressão 7?

O pico do filtro adaptado aparece no **início** de cada cópia de LT. O último pico, max(N_P), marca portanto o início
da segunda cópia. Essa cópia tem 64 amostras, por isso o LTS acaba 64 amostras depois, que é onde começa o primeiro
símbolo seguinte:

  n_P = max(N_P) + 64

---

# O que faz o bloco Stream to Vector?

Junta cada 64 amostras consecutivas num **vetor de 64 complexos**, ou seja, um símbolo OFDM por item. Assim:

- o bloco **FFT (64)**, que trabalha com vetores de 64, recebe um símbolo inteiro de cada vez;
- os blocos seguintes passam a trabalhar símbolo a símbolo.

O Stream to Vector agrupa sempre de 64 em 64 e não sabe onde começam os símbolos. Por isso o Sync Long tem de produzir
sempre um múltiplo de 64 amostras por frame. Quando chega o frame seguinte, completa o último vetor com zeros; sem isso,
um símbolo ficaria dividido entre dois vetores.
