# Construção do dataset e treinamento do modelo

# Parte 1: Construção do Dataset para o STConvS2S

Esta etapa do pipeline consolida as features meteorológicas derivadas dos dados do GOES-16 e o dado de precipitação utilizado como **target**, organiza os dados em uma escala temporal comum e constrói amostras no formato necessário para o modelo STConvS2S.

## 1. Arquivos envolvidos

O script responsável por esta etapa está no repositório **atmoseer**:

```text
goes16_generate_samples_to_stconvs2s.py
```

## 2. Estrutura de entrada

Após a etapa de download e construção das features, os dados deverão estar organizados da seguinte forma:

```text
data/
├── GPM/
│   └── 2020/
│       ├── 2020-01-01.nc
│       └── 2020-01-02.nc
└── goes16/
    ├── CMI/
    │   └── 2020/
    │       ├── C08/C08_2020_01_01.nc
    │       ├── C09/C09_2020_01_01.nc
    │       ├── C11/C11_2020_01_01.nc
    │       ├── C13/C13_2020_01_01.nc
    │       ├── C14/C14_2020_01_01.nc
    │       └── C15/C15_2020_01_01.nc
    ├── GLM/
    │   └── aggregated_data/
    │       └── 2020/
    │           └── 01/
    │               ├── 2020-01-01.nc
    │               └── 2020-01-02.nc
    └── features/
        ├── pn/2020/PN_2020_01_01.nc
        ├── gtn/2020/GTN_2020_01_01.nc
        ├── fa/2020/FA_2020_01_01.nc
        ├── wv_grad/2020/WV_grad_2020_01_01.nc
        ├── li_proxy/2020/LI_proxy_2020_01_01.nc
        ├── toct/2020/TOCT_2020_01_01.nc
        └── pn_std/2020/PNstd_2020_01_01.nc
```

A estrutura acima representa os dados disponíveis ao final da etapa anterior. O script de geração de samples, por sua vez, trabalha com dois caminhos de entrada separados:

- `--features-path`: diretório contendo as features;
- `--target-path`: diretório contendo o(s) target(s).

Por padrão, o script utiliza `./src/features` e `./src/target`, portanto os caminhos precisam ser ajustados à organização real do projeto. O script não usa `data/GPM/` diretamente como valor padrão de `--target-path`.

## 3. Comandos e argumentos

O script recebe os seguintes argumentos:

| Argumento | Obrigatório | Padrão | Descrição |
|---|---|---|---|
| `--features-path` | Não | `./src/features` | Diretório contendo as features |
| `--target-path` | Não | `./src/target` | Diretório contendo o target |
| `--output-path` | Não | `./src/output_dataset.nc` | Arquivo NetCDF de saída |
| `--lat-dim` | **Sim** | — | Número de pontos na latitude |
| `--lon-dim` | **Sim** | — | Número de pontos na longitude |
| `--max-gap` | Não | `60` | Maior intervalo permitido entre timestamps, em minutos |
| `--timestep` | Não | `5` | Número de passos temporais em `X` e `Y` |

Os parâmetros são definidos diretamente no `argparse` do script.

### Exemplo de execução

Considerando que a estrutura efetivamente utilizada pelo projeto seja:

```text
./data/goes16/features
./data/GPM
```

um exemplo seria:

```bash
python goes16_generate_samples_to_stconvs2s.py \
    --features-path ./data/goes16/features \
    --target-path ./data/GPM \
    --output-path ./data/dataset_stconvs2s.nc \
    --lat-dim 94 \
    --lon-dim 121 \
    --max-gap 60 \
    --timestep 5
```

> **Atenção:** o comando acima é um exemplo de configuração baseado na estrutura de diretórios apresentada anteriormente. A compatibilidade exata dos arquivos de GPM com `process_target()` deve ser confirmada, pois o código procura timestamps no padrão `IMERG_YYYY_MM_DD_HH_MM`.

## 4. Processamento executado

A construção do dataset ocorre nas seguintes etapas:

1. Leitura dos arquivos NetCDF de cada feature.
2. Extração dos timestamps presentes nos nomes das variáveis.
3. Organização dos dados cronologicamente.
4. Agregação dos dados em escala horária.
5. Interseção dos timestamps entre todas as features.
6. Leitura e agregação do target.
7. Interseção dos timestamps entre features e target.
8. Criação de janelas temporais deslizantes.
9. Remoção de janelas com grandes lacunas temporais.
10. Conversão das amostras para arrays NumPy.
11. Replicação do target entre os canais, quando necessário.
12. Salvamento do dataset final em NetCDF.

### Processamento das features

Para cada diretório localizado em `--features-path`, o script identifica os arquivos `.nc` e processa as variáveis que possuam timestamp no nome.

O padrão reconhecido é:

```text
CMI_YYYY_MM_DD_HH_MM
```

ou, de forma mais geral, o timestamp contido nesse padrão.

Cada variável é lida como uma matriz espacial com dimensões:

```text
(lat, lon)
```

O script verifica se o tamanho da matriz corresponde aos valores informados nos parâmetros `--lat-dim` e `--lon-dim`. Caso contrário, a execução apresenta erro.

Após a leitura, timestamps e dados são ordenados cronologicamente.

#### Agregação temporal

Os dados são agrupados por hora.

Para as features normais, o script calcula a **média dos valores encontrados dentro de cada hora**.

A `densidade_flashes` possui tratamento diferente: seus valores são **somados dentro de cada hora**.

Depois da agregação, os dados assumem a forma:

```text
(time, lat, lon, channel)
```

com um único canal associado a cada feature.

### Alinhamento temporal entre as features

Cada feature pode possuir timestamps diferentes. Para garantir que todas as variáveis estejam disponíveis nos mesmos instantes, o script calcula a **interseção dos timestamps** de todas as features.

Somente os horários presentes em todas as features são mantidos.

Em seguida, as features são concatenadas ao longo da dimensão `channel`.

O resultado é um dataset contendo todas as features nos mesmos timestamps.

### Processamento do target

O target é processado de maneira semelhante às features.

O script procura timestamps no padrão:

```text
IMERG_YYYY_MM_DD_HH_MM
```

A matriz espacial do target também deve possuir dimensões `(lat, lon)`.

Ao contrário das features normais, o target é agregado por hora utilizando **soma**.

Depois, caso existam múltiplos targets, os timestamps comuns entre eles são obtidos e os targets são concatenados na dimensão `channel`.

> **Importante:** embora a estrutura de dados anterior do projeto apresente `GPM`, o código fornecido chama essa etapa de `target` e procura variáveis com padrão `IMERG_...`. A forma exata como os arquivos de GPM devem ser apresentados ao script precisa ser conferida no restante do projeto.

### Alinhamento entre features e target

Depois que features e target foram processados separadamente, o script calcula a interseção entre seus timestamps:

```text
common_times = features_times ∩ target_times
```

Somente esses horários são utilizados na construção das amostras.

Isso garante que cada instante utilizado em `X` também possua um correspondente no target.

### Construção das amostras temporais

O parâmetro padrão utilizado pelo pipeline é:

```text
timestep = 5
```

O script também define:

```text
offset = timestep
```

portanto, com o valor padrão:

```text
offset = 5
```

Para cada posição `i` da série temporal, são construídas duas janelas:

```text
X = [i, i+1, i+2, i+3, i+4]
Y = [i+5, i+6, i+7, i+8, i+9]
```

Ou seja, o modelo recebe **5 instantes consecutivos de entrada** e o target corresponde aos **5 instantes seguintes**. O código implementa essas duas fatias temporais diretamente.

#### Exemplo

Considerando timestamps horários:

```text
10:00
11:00
12:00
13:00
14:00
15:00
16:00
17:00
18:00
19:00
```

uma amostra terá:

```text
X = 10:00, 11:00, 12:00, 13:00, 14:00
Y = 15:00, 16:00, 17:00, 18:00, 19:00
```

A janela seguinte será deslocada em um passo temporal.

### Verificação de lacunas temporais

Nem todas as sequências de timestamps são necessariamente contínuas. Para impedir a criação de amostras com intervalos temporais muito grandes, o script calcula a maior diferença entre timestamps de cada janela.

O parâmetro padrão é:

```text
max_gap = 60 minutos
```

Uma amostra é descartada quando:

```text
maior_gap(X) > max_gap
```

ou:

```text
maior_gap(Y) > max_gap
```

Esse mecanismo é importante principalmente para períodos em que há ausência de dados nos satélites ou no produto de precipitação.

### Verificação das dimensões das amostras

O script verifica se todas as amostras possuem a mesma forma espacial e temporal.

A forma da primeira amostra válida é utilizada como referência. Amostras posteriores com dimensões diferentes são descartadas.

As amostras aceitas são armazenadas juntamente com seus índices temporais.

### Formato das matrizes `X` e `Y`

Depois da coleta, as listas de amostras são convertidas para arrays NumPy:

```python
X_array = np.stack(X_samples)
Y_array = np.stack(Y_samples)
```

A estrutura resultante é:

```text
(sample, time, lat, lon, channel)
```

Assim, `X` contém as features meteorológicas e `Y` contém o target.

### Replicação dos canais do target

O código contém uma exigência específica para o modelo STConvS2S: `X` e `Y` precisam possuir o mesmo número de canais.

Quando `Y` possui apenas um canal e `X` possui vários canais, o target é replicado:

```python
Y_array = np.repeat(Y_array, required_channels, axis=-1)
```

Por exemplo, se:

```text
X → 9 canais
Y → 1 canal
```

o resultado será:

```text
X → 9 canais
Y → 9 canais
```

Nesse caso, cada canal de `Y` possui a mesma informação de precipitação.

## 5. Estrutura de saída

O NetCDF final contém duas variáveis principais:

```text
x
```

e

```text
y
```

Ambas possuem as dimensões:

```text
sample × time × lat × lon × channel
```

conforme definido no script.

Também são armazenados:

- `sample_x_timestamps`: timestamps reais correspondentes à janela de entrada de cada amostra;
- `sample_y_timestamps`: timestamps reais correspondentes à janela do target;
- `lat`: coordenadas espaciais utilizadas;
- `lon`: coordenadas espaciais utilizadas;
- `channel`: nomes das features.

O arquivo também recebe metadados contendo, entre outras informações, o `timestep`, o `max_gap_minutes` e o número total de amostras.

Ao final da execução, o script salva o dataset em:

```text
--output-path
```

e exibe as dimensões de `X` e `Y` no terminal:

```text
X_array shape: (...)
Y_array shape: (...)
```

Também exibe os nomes dos canais utilizados.

O arquivo é finalmente salvo com:

```python
output_ds.to_netcdf(output_path)
```

## 6. Sequência completa de execução

1. Organizar os dados de GPM e GOES-16 conforme a estrutura de entrada.
2. Definir os caminhos de `--features-path` e `--target-path`.
3. Executar `goes16_generate_samples_to_stconvs2s.py` com os parâmetros correspondentes à grade espacial, `max-gap` e `timestep`.
4. O script lê e agrega as features e o target.
5. Os timestamps comuns são identificados.
6. São construídas as janelas de 5 passos de entrada e 5 passos seguintes de target.
7. Amostras com lacunas temporais superiores a 60 minutos ou dimensões incompatíveis são descartadas.
8. O target é replicado entre os canais, quando necessário.
9. O dataset final é salvo em NetCDF.

# Parte 2: Treinamento com o STConvS2S

> **Aviso:** esta seção descreve o procedimento utilizado neste projeto na versão do repositório STConvS2S disponível à época do desenvolvimento (2025). O repositório recebeu atualizações posteriores, portanto a estrutura de diretórios, parâmetros e comandos pode ter sido alterada. Antes de executar o treinamento, recomenda-se consultar o `README.md` e os arquivos de configuração da versão atual.
>
> Repositório: https://github.com/AILAB-CEFET-RJ/stconvs2s

## 1. Arquivos envolvidos

Passando agora para o repositório **stconvs2s**, o dataset gerado na etapa anterior deve ser colocado na pasta `/data`.

Antes de executar o treinamento, é necessário substituir o caminho do dataset na função **`__get_dataset_file`**, no arquivo **`ml_builder.py`**.

## 2. Estrutura de entrada

O dataset gerado na etapa anterior deve estar na pasta `/data` do repositório **stconvs2s**.

## 3. Comandos e argumentos

Com o dataset no diretório esperado, o treinamento é iniciado a partir da raiz do repositório com:

```bash
python main.py # Com os demais parâmetros disponíveis no README do repositório
```

## 4. Processamento executado

O treinamento utiliza o dataset gerado na etapa anterior através do caminho configurado na função **`__get_dataset_file`**, no arquivo **`ml_builder.py`**.

## 5. Estrutura de saída

O conteúdo desta etapa depende da configuração e da versão do repositório **stconvs2s** utilizada.

## 6. Sequência completa de execução

1. Passar para o repositório **stconvs2s**.
2. Colocar o dataset gerado na etapa anterior na pasta `/data`.
3. Substituir o caminho do dataset na função **`__get_dataset_file`**, no arquivo **`ml_builder.py`**.
4. Iniciar o treinamento a partir da raiz do repositório com `python main.py`.