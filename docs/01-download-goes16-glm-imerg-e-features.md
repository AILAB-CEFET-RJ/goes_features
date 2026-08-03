# AtmoSeer — download de GOES-16 e IMERG e geração de features

Este guia descreve a execução dos arquivos do AtmoSeer responsáveis por:

- baixar e recortar os canais ABI do GOES-16;
- baixar o produto GLM e transformá-lo em contagens de flashes sobre uma grade espacial;
- baixar e recortar a precipitação IMERG utilizada como target;
- gerar as features derivadas dos canais ABI;
- verificar os arquivos NetCDF produzidos.

Todos os comandos devem ser executados a partir da raiz do projeto AtmoSeer.

## 1. Arquivos utilizados

| Arquivo | Função na execução |
|---|---|
| `setup.sh` | Cria o ambiente Conda, cria diretórios básicos em `data/` e configura os arquivos locais de autenticação do Earthdata. |
| `config/environment.yml` | Declara o ambiente Conda `atmoseer` e as dependências principais. |
| `src/config/globals.py` | Define os diretórios padrão e os limites geográficos utilizados pelos scripts. |
| `src/goes16/goes16_download_crop.py` | Baixa um canal ABI por execução, reprojeta, recorta e reúne as imagens de cada dia em um NetCDF diário. |
| `src/goes16/goes16_retrieve_glm.py` | Baixa os arquivos GLM, agrupa flashes em intervalos de 30 minutos e gera uma grade diária de frequência de flashes. |
| `src/goes16/goes16_sync_downloader_cropper_for_glm.py` | Baixa e filtra registros GLM pela região. O arquivo não realiza a agregação em grade de 30 minutos utilizada pela feature GLM. |
| `src/gpm/gpm_download_crop.py` | Baixa o IMERG, recorta a variável `precipitation` e reúne os intervalos de cada dia em um NetCDF diário. |
| `src/goes16/main_goes16_features.py` | Interface de linha de comando para gerar as features ABI. |
| `src/goes16/utils.py` | Localiza automaticamente os diretórios de canais e features organizados por ano. |
| `src/goes16/features/` | Contém uma implementação separada para cada feature ABI. |
| `src/goes16/features.py` | Implementação única anterior. Não é o módulo importado pelo comando atual de geração de features. |
| `src/goes16/validate_features.py` | Compara a presença dos arquivos de uma feature com os canais de origem e abre os NetCDF para verificar sua estrutura. |
| `src/goes16/analyze_logs.py` | Resume avisos e erros encontrados em arquivos `.log`. |
| `Makefile` | Disponibiliza atalhos para o downloader ABI, o downloader IMERG, a geração de features e a validação. |

## 2. Preparação do ambiente

### 2.1 Variáveis de autenticação

O `setup.sh` lê o usuário e a senha do Earthdata pelas variáveis abaixo:

```bash
export EARTHDATA_USER="seu_usuario"
export EARTHDATA_PASS="sua_senha"
```

### 2.2 Criação do ambiente

```bash
chmod +x setup.sh
./setup.sh
conda activate atmoseer
```

O ambiente é criado com o nome definido em `config/environment.yml`:

```text
atmoseer
```

O arquivo `src/gpm/gpm_download_crop.py` importa `BeautifulSoup` do pacote `beautifulsoup4`. Instale esse pacote no ambiente antes de executar o download do IMERG:

```bash
conda activate atmoseer
python -m pip install beautifulsoup4
```

### 2.3 Forma de executar os scripts

Os módulos importam os pacotes a partir de `src/`. Por isso, os comandos diretos usam:

```bash
PYTHONPATH=src python caminho/do/script.py
```

## 3. Região utilizada pelos recortes

Os limites estão definidos em `src/config/globals.py`:

```text
latitude mínima:  -23.801876626302175
latitude máxima:  -21.699774257353113
longitude mínima: -45.05290312102409
longitude máxima: -42.35676996062447
```

Os valores podem ser fornecidos ao processo por variáveis de ambiente antes da execução:

```bash
export REGION_LAT_MIN="-23.801876626302175"
export REGION_LAT_MAX="-21.699774257353113"
export REGION_LON_MIN="-45.05290312102409"
export REGION_LON_MAX="-42.35676996062447"
```

Os três fluxos principais usam esses limites:

- o downloader ABI usa os limites durante a reprojeção e o recorte;
- o agregador GLM descarta flashes fora dos limites e conta os restantes na grade;
- o downloader IMERG seleciona os índices de latitude e longitude contidos nos limites.

## 4. Estrutura de diretórios

A geração de features procura os canais ABI nesta estrutura:

```text
data/
└── goes16/
    ├── CMI/
    │   └── <ano>/
    │       ├── C08/
    │       ├── C09/
    │       ├── C11/
    │       ├── C13/
    │       ├── C14/
    │       └── C15/
    ├── GLM/
    │   ├── temp/
    │   └── aggregated_data/
    │       └── <ano>/
    │           └── <mes>/
    └── features/
        ├── pn/<ano>/
        ├── gtn/<ano>/
        ├── fa/<ano>/
        ├── wv_grad/<ano>/
        ├── li_proxy/<ano>/
        ├── toct/<ano>/
        └── pn_std/<ano>/

data/
└── GPM/
    └── <ano>/
```

Exemplo para criar os diretórios ABI de 2020:

```bash
mkdir -p data/goes16/CMI/2020/{C08,C09,C11,C13,C14,C15}
mkdir -p data/goes16/features
mkdir -p data/GPM
```

O diretório fornecido em `--crop_dir` para `goes16_download_crop.py` deve existir antes da execução.

## 5. Download dos canais ABI do GOES-16

### 5.1 Arquivo executado

```text
src/goes16/goes16_download_crop.py
```

### 5.2 Processamento realizado

Para cada dia do intervalo, o script:

1. acessa o bucket público `noaa-goes16`;
2. percorre as 24 pastas horárias do produto `ABI-L2-CMIPF`;
3. seleciona os arquivos que contêm o canal solicitado;
4. baixa cada arquivo para um diretório temporário;
5. lê a variável informada em `--vars`;
6. aplica `scale_factor` e `add_offset`;
7. reprojeta para latitude/longitude em WGS84;
8. recorta a região configurada em `src/config/globals.py`;
9. reamostra usando a resolução fornecida em `--spatial_resolution`;
10. remove o arquivo original baixado;
11. grava todas as imagens recortadas do dia em um único NetCDF.

### 5.3 Argumentos

| Argumento | Formato | Conteúdo |
|---|---|---|
| `--start_date` | `YYYY-MM-DD` | Primeiro dia processado. |
| `--end_date` | `YYYY-MM-DD` | Último dia processado. |
| `--channel` | inteiro de 1 a 16 | Canal ABI baixado na execução. |
| `--download_dir` | caminho | Diretório temporário informado na linha de comando. Durante a inicialização, o script redefine esse caminho para `./downloads`. |
| `--crop_dir` | caminho | Diretório no qual o NetCDF diário será salvo. Deve existir. |
| `--spatial_resolution` | número decimal | Resolução da grade de saída em graus. O fluxo documentado utiliza `0.1`. |
| `--ignored_months` | lista de inteiros | Meses ignorados. Quando o argumento não é fornecido, o valor padrão é `6 7 8`. |
| `--vars` | uma ou mais strings | Variáveis lidas do produto. Para os canais ABI deste fluxo, utiliza-se `CMI`. |

### 5.4 Canais necessários pelas features

| Canal | Utilizado por |
|---|---|
| `C08` | `WV_grad` |
| `C09` | `PN` e `WV_grad` |
| `C11` | `GTN` |
| `C13` | `PN`, `FA`, `LI_proxy` e `TOCT` |
| `C14` | `GTN` e `LI_proxy` |
| `C15` | `GTN` |

O arquivo `src/goes16/features/tp.py` utiliza `C07`, mas essa feature não possui flag em `main_goes16_features.py`.

### 5.5 Execução de um canal

Exemplo para baixar o canal 8 em janeiro de 2020:

```bash
mkdir -p data/goes16/CMI/2020/C08

PYTHONPATH=src python src/goes16/goes16_download_crop.py \
  --start_date 2020-01-01 \
  --end_date 2020-01-31 \
  --channel 8 \
  --crop_dir data/goes16/CMI/2020/C08 \
  --spatial_resolution 0.1 \
  --vars CMI
```

O mesmo fluxo deve ser executado separadamente para os canais 9, 11, 13, 14 e 15, alterando `--channel` e `--crop_dir`.

### 5.6 Execução dos seis canais

```bash
YEAR=2020
START_DATE=2020-01-01
END_DATE=2020-01-31

for CHANNEL in 8 9 11 13 14 15; do
  CHANNEL_DIR=$(printf "C%02d" "$CHANNEL")
  mkdir -p "data/goes16/CMI/${YEAR}/${CHANNEL_DIR}"

  PYTHONPATH=src python src/goes16/goes16_download_crop.py \
    --start_date "$START_DATE" \
    --end_date "$END_DATE" \
    --channel "$CHANNEL" \
    --crop_dir "data/goes16/CMI/${YEAR}/${CHANNEL_DIR}" \
    --spatial_resolution 0.1 \
    --vars CMI
done
```

### 5.7 Saída

O nome de cada arquivo diário segue o padrão:

```text
C<canal>_YYYY_MM_DD.nc
```

Exemplo:

```text
data/goes16/CMI/2020/C13/C13_2020_01_01.nc
```

Dentro do arquivo, cada imagem é armazenada como uma variável independente:

```text
CMI_2020_01_01_00_00
CMI_2020_01_01_00_10
CMI_2020_01_01_00_20
...
```

As dimensões de cada variável recebem nomes derivados do próprio timestamp.

### 5.8 Atalho do Makefile

```bash
make goes16-download-crop \
  START=2020-01-01 \
  END=2020-01-31 \
  CHANNEL=13 \
  DIR=data/goes16/CMI/2020/C13
```

O alvo do Makefile fornece `--spatial_resolution 0.1` e `--vars CMI` automaticamente.

## 6. Download e geração da feature GLM

### 6.1 Arquivo executado

```text
src/goes16/goes16_retrieve_glm.py
```

Esse arquivo realiza tanto o download quanto a criação da grade de frequência de flashes.

### 6.2 Processamento realizado

Para cada dia, o script:

1. acessa o produto `GLM-L2-LCFA` no bucket `noaa-goes16`;
2. percorre as 24 horas do dia;
3. baixa os NetCDF GLM para `data/goes16/GLM/temp/`;
4. lê `flash_lat`, `flash_lon` e `time_coverage_start` de cada arquivo;
5. associa cada flash ao início do intervalo de 30 minutos correspondente;
6. cria 48 intervalos entre `00:00` e `23:30`;
7. descarta coordenadas fora da região configurada;
8. converte as coordenadas em índices de uma grade com a resolução solicitada;
9. incrementa uma unidade na célula de cada flash;
10. grava as 48 grades no NetCDF diário;
11. limpa o diretório temporário antes de processar o próximo dia.

O valor de cada célula representa a quantidade de flashes localizados naquela célula durante o intervalo de 30 minutos.

### 6.3 Argumentos

| Argumento | Forma curta | Formato | Conteúdo |
|---|---|---|---|
| `--start_date` | `-b` | `YYYY-MM-DD` | Primeiro dia processado. |
| `--end_date` | `-e` | `YYYY-MM-DD` | Último dia processado. |
| `--spatial_resolution` | `-s` | decimal | Resolução da grade em graus. |
| `--ignored_months` | `-i` | lista de inteiros | Meses que não serão processados. O padrão é uma lista vazia. |

### 6.4 Comando

```bash
PYTHONPATH=src python src/goes16/goes16_retrieve_glm.py \
  --start_date 2020-01-01 \
  --end_date 2020-01-31 \
  --spatial_resolution 0.1
```

Com meses ignorados:

```bash
PYTHONPATH=src python src/goes16/goes16_retrieve_glm.py \
  -b 2020-01-01 \
  -e 2020-12-31 \
  -s 0.1 \
  -i 6 7 8
```

### 6.5 Saída

O script cria automaticamente:

```text
data/goes16/GLM/
├── temp/
└── aggregated_data/
    └── <ano>/
        └── <mes>/
            └── YYYY-MM-DD.nc
```

Exemplo:

```text
data/goes16/GLM/aggregated_data/2020/01/2020-01-01.nc
```

Cada arquivo diário contém 48 variáveis:

```text
2020_01_01_00_00
2020_01_01_00_30
2020_01_01_01_00
...
2020_01_01_23_30
```

### 6.6 Papel de `goes16_sync_downloader_cropper_for_glm.py`

O arquivo:

```text
src/goes16/goes16_sync_downloader_cropper_for_glm.py
```

baixa `GLM-L2-LCFA`, filtra os registros pela caixa geográfica e tenta reuni-los em um arquivo diário chamado:

```text
GLM_YYYY_MM_DD.nc
```

Ele trabalha com as variáveis originais dos arquivos GLM. A criação das 48 grades de frequência de flashes é implementada em `goes16_retrieve_glm.py`.

## 7. Download e recorte do IMERG

### 7.1 Arquivo executado

```text
src/gpm/gpm_download_crop.py
```

### 7.2 Autenticação usada pelo comando

O usuário é obrigatório. A senha pode ser fornecida por argumento:

```bash
--user "seu_usuario" --pwd "sua_senha"
```

ou pela variável `GPMPWD`:

```bash
export GPMPWD="sua_senha"
```

O `setup.sh` também cria os arquivos locais `.netrc` e `.urs_cookies` a partir de `EARTHDATA_USER` e `EARTHDATA_PASS`.

### 7.3 Processamento realizado

Para o intervalo informado, o script:

1. exige que a data inicial e a data final pertençam ao mesmo ano;
2. converte os dias do intervalo em dias julianos;
3. percorre os diretórios do produto `GPM_3IMERGHH.07`;
4. baixa cada arquivo correspondente aos dias selecionados;
5. lê o timestamp inicial do grânulo;
6. lê `lat`, `lon` e `precipitation`;
7. seleciona somente os índices contidos na região configurada;
8. transpõe o recorte para a orientação usada pelo projeto;
9. adiciona o campo ao NetCDF diário;
10. remove o arquivo intermediário baixado.

### 7.4 Argumentos

| Argumento | Forma curta | Formato | Conteúdo |
|---|---|---|---|
| `--begin_date` | — | `YYYY/MM/DD` | Primeiro dia processado. |
| `--end_date` | — | `YYYY/MM/DD` | Último dia processado. Deve estar no mesmo ano de `--begin_date`. |
| `--user` | `-u` | texto | Usuário do Earthdata. |
| `--pwd` | `-p` | texto | Senha. É opcional quando `GPMPWD` está definida. |
| `--ignored_months` | — | lista de inteiros | Meses que não serão processados. O padrão é uma lista vazia. |
| `--debug` | `-d` | flag | Exibe mensagens de depuração. |

### 7.5 Comando com senha no ambiente

```bash
export GPMPWD="sua_senha"

PYTHONPATH=src python src/gpm/gpm_download_crop.py \
  --begin_date 2020/01/01 \
  --end_date 2020/01/31 \
  --user "seu_usuario"
```

### 7.6 Comando com senha no argumento

```bash
PYTHONPATH=src python src/gpm/gpm_download_crop.py \
  --begin_date 2020/01/01 \
  --end_date 2020/01/31 \
  --user "seu_usuario" \
  --pwd "sua_senha"
```

### 7.7 Intervalo que atravessa anos

O comando deve ser separado por ano:

```bash
PYTHONPATH=src python src/gpm/gpm_download_crop.py \
  --begin_date 2020/12/20 \
  --end_date 2020/12/31 \
  --user "seu_usuario"

PYTHONPATH=src python src/gpm/gpm_download_crop.py \
  --begin_date 2021/01/01 \
  --end_date 2021/01/10 \
  --user "seu_usuario"
```

### 7.8 Saída

```text
data/GPM/<ano>/YYYY-MM-DD.nc
```

Exemplo:

```text
data/GPM/2020/2020-01-01.nc
```

Cada intervalo do produto é armazenado como uma variável:

```text
2020_01_01_00_00
2020_01_01_00_30
2020_01_01_01_00
...
```

O log geral é gravado na raiz configurada por `AUSREFDIR`, ou na raiz atual quando a variável não está definida:

```text
gpm_update_log.txt
```

### 7.9 Atalho do Makefile

```bash
make gpm-download-crop \
  BEGIN=2020/01/01 \
  END=2020/01/31 \
  USER=seu_usuario \
  PWD=sua_senha
```

## 8. Geração das features ABI

### 8.1 Arquivo executado

```text
src/goes16/main_goes16_features.py
```

O arquivo importa as funções do pacote:

```text
src/goes16/features/
```

A localização dos canais e das features é obtida em `src/goes16/utils.py`, usando os diretórios definidos em `src/config/globals.py`:

```text
GOES16_DATA_DIR=./data/goes16/CMI/
GOES16_FEATURES_DIR=./data/goes16/features/
```

Esses valores também podem ser definidos no ambiente:

```bash
export GOES16_DATA_DIR="./data/goes16/CMI/"
export GOES16_FEATURES_DIR="./data/goes16/features/"
```

### 8.2 Estrutura de entrada

O código procura um diretório de canal dentro de cada ano:

```text
data/goes16/CMI/<ano>/<canal>/
```

Exemplo:

```text
data/goes16/CMI/2020/C09/C09_2020_01_01.nc
data/goes16/CMI/2020/C13/C13_2020_01_01.nc
```

Os arquivos de canais usados juntos devem ter o mesmo sufixo diário. O sufixo é toda a parte localizada depois do primeiro `_`:

```text
2020_01_01.nc
```

### 8.3 Flags disponíveis

| Flag | Módulo executado | Canais/entrada |
|---|---|---|
| `--pn` | `features/pn.py` | C09 e C13 |
| `--gtn` | `features/gtn.py` | C11, C14 e C15 |
| `--fa` | `features/fa.py` | C13 |
| `--wv_grad` | `features/wv_grad.py` | C09 e C08 |
| `--li_proxy` | `features/li_proxy.py` | C14 e C13 |
| `--toct` | `features/toct.py` | C13 |
| `--pn_std` | `features/pn_std.py` | Feature PN já calculada |
| `--verbose` | `main_goes16_features.py` | Exibe o nome da feature e os diretórios processados. |

### 8.4 Comando completo

```bash
PYTHONPATH=src python src/goes16/main_goes16_features.py \
  --pn \
  --gtn \
  --fa \
  --wv_grad \
  --li_proxy \
  --toct \
  --verbose
```

Atalho equivalente:

```bash
make goes16-features \
  FEATS="--pn --gtn --fa --wv_grad --li_proxy --toct --verbose"
```

É possível executar somente uma feature:

```bash
PYTHONPATH=src python src/goes16/main_goes16_features.py --pn --verbose
```

### 8.5 Saída geral

Cada módulo cria:

```text
data/goes16/features/<feature>/<ano>/<PREFIXO>_YYYY_MM_DD.nc
```

Exemplos:

```text
data/goes16/features/pn/2020/PN_2020_01_01.nc
data/goes16/features/gtn/2020/GTN_2020_01_01.nc
data/goes16/features/fa/2020/FA_2020_01_01.nc
data/goes16/features/wv_grad/2020/WV_grad_2020_01_01.nc
data/goes16/features/li_proxy/2020/LI_proxy_2020_01_01.nc
data/goes16/features/toct/2020/TOCT_2020_01_01.nc
```

As variáveis internas mantêm os nomes temporais dos arquivos ABI de origem, como:

```text
CMI_2020_01_01_00_00
CMI_2020_01_01_00_10
...
```

## 9. Cálculo de cada feature

### 9.1 PN — profundidade das nuvens

Arquivo:

```text
src/goes16/features/pn.py
```

Entrada:

```text
C09 e C13
```

Cálculo aplicado a cada variável temporal presente nos dois arquivos:

```text
PN = C09 - C13
```

Saída:

```text
data/goes16/features/pn/<ano>/PN_YYYY_MM_DD.nc
```

Log gerado no diretório de trabalho:

```text
pn.log
```

### 9.2 GTN — glaciação do topo das nuvens

Arquivo:

```text
src/goes16/features/gtn.py
```

Entrada:

```text
C11, C14 e C15
```

Cálculo:

```text
GTN = (C11 - C14) - (C14 - C15)
```

Saída:

```text
data/goes16/features/gtn/<ano>/GTN_YYYY_MM_DD.nc
```

Log:

```text
gtn.log
```

### 9.3 FA — derivada temporal do fluxo ascendente

Arquivo:

```text
src/goes16/features/fa.py
```

Entrada:

```text
C13
```

Para cada variável `CMI_...` do NetCDF diário, o código procura no mesmo arquivo a variável correspondente a dez minutos depois e calcula:

```text
FA(t) = [C13(t + 10 min) - C13(t)] / 10
```

O intervalo padrão é dez minutos. O cálculo ocorre entre variáveis do mesmo arquivo diário. Uma variável sem o timestamp seguinte dentro do arquivo não é gravada na saída.

Saída:

```text
data/goes16/features/fa/<ano>/FA_YYYY_MM_DD.nc
```

Log:

```text
fa.log
```

### 9.4 WV_grad — gradiente de vapor d'água

Arquivo:

```text
src/goes16/features/wv_grad.py
```

Entrada:

```text
C09 e C08
```

Cálculo:

```text
WV_grad = C09 - C08
```

Saída:

```text
data/goes16/features/wv_grad/<ano>/WV_grad_YYYY_MM_DD.nc
```

Log:

```text
wv_grad.log
```

### 9.5 LI_proxy — proxy de estabilidade

Arquivo:

```text
src/goes16/features/li_proxy.py
```

Entrada:

```text
C14 e C13
```

Cálculo:

```text
LI_proxy = C14 - C13
```

Saída:

```text
data/goes16/features/li_proxy/<ano>/LI_proxy_YYYY_MM_DD.nc
```

Log:

```text
li_proxy.log
```

### 9.6 TOCT — temperatura do topo das nuvens

Arquivo:

```text
src/goes16/features/toct.py
```

Entrada:

```text
C13
```

O módulo copia os campos temporais do canal C13 para o arquivo da feature.

Saída:

```text
data/goes16/features/toct/<ano>/TOCT_YYYY_MM_DD.nc
```

### 9.7 PN_STD — textura espacial da profundidade

Arquivo:

```text
src/goes16/features/pn_std.py
```

Entrada:

```text
arquivos PN
```

Para cada grade PN, o módulo aplica um desvio padrão local com janela `3 x 3` e bordas tratadas com o modo `nearest`:

```text
PN_STD = desvio_padrao_local(PN, janela=3x3)
```

O comando `--pn_std` procura a entrada em:

```text
data/goes16/features/profundidade_nuvens/<ano>/
```

A flag `--pn` grava a PN em:

```text
data/goes16/features/pn/<ano>/
```

Para executar `--pn_std` com a organização esperada pelo código atual, os arquivos PN devem estar disponíveis no diretório lido por `build_feature_paths_by_year("profundidade_nuvens")`:

```bash
mkdir -p data/goes16/features/profundidade_nuvens/2020
cp data/goes16/features/pn/2020/*.nc \
   data/goes16/features/profundidade_nuvens/2020/

PYTHONPATH=src python src/goes16/main_goes16_features.py \
  --pn_std \
  --verbose
```

Saída:

```text
data/goes16/features/pn_std/<ano>/PNstd_YYYY_MM_DD.nc
```

Log:

```text
pn_std.log
```

### 9.8 TP — tamanho das partículas

Arquivo:

```text
src/goes16/features/tp.py
```

Entrada:

```text
C07
```

O módulo copia as dimensões, variáveis, atributos e valores do C07 para arquivos com prefixo `TP`. Ele não é importado por `main_goes16_features.py` e não possui flag na interface de linha de comando atual.

## 10. Relação das resoluções

### 10.1 ABI

A resolução de saída é definida em cada execução pelo argumento:

```text
--spatial_resolution 0.1
```

O recorte é reamostrado pelo GDAL para uma grade regular em latitude e longitude.

### 10.2 GLM

A grade de contagem é criada usando o mesmo valor de resolução:

```text
--spatial_resolution 0.1
```

O tempo é dividido em 48 janelas diárias de 30 minutos.

### 10.3 IMERG

O script preserva os pontos do produto contidos nos limites geográficos e agrupa os intervalos em arquivos diários. Os nomes temporais seguem intervalos de 30 minutos.

## 11. Validação das features

### 11.1 Arquivo executado

```text
src/goes16/validate_features.py
```

O validador executa duas verificações:

1. compara os sufixos dos arquivos da feature com a interseção dos arquivos disponíveis nos canais informados;
2. abre cada NetCDF e verifica se existem variáveis e se elas possuem o atributo `description`.

### 11.2 Comandos

PN:

```bash
PYTHONPATH=src python src/goes16/validate_features.py \
  --feature pn \
  --canais C09 C13
```

GTN:

```bash
PYTHONPATH=src python src/goes16/validate_features.py \
  --feature gtn \
  --canais C11 C14 C15
```

WV_grad:

```bash
PYTHONPATH=src python src/goes16/validate_features.py \
  --feature wv_grad \
  --canais C09 C08
```

Atalho do Makefile:

```bash
make validate-feature FEAT=pn CANAIS="C09 C13"
```

### 11.3 Inspeção direta de um NetCDF

```bash
PYTHONPATH=src python - <<'PY'
from netCDF4 import Dataset

path = "data/goes16/features/pn/2020/PN_2020_01_01.nc"

with Dataset(path) as dataset:
    print("dimensões:", list(dataset.dimensions))
    print("variáveis:", list(dataset.variables))
    for name, variable in dataset.variables.items():
        print(name, variable.shape, variable.dtype)
PY
```

### 11.4 Leitura dos logs das features

Os módulos `pn.py`, `gtn.py`, `fa.py`, `wv_grad.py`, `li_proxy.py` e `pn_std.py` criam arquivos `.log` no diretório de trabalho em que o comando de geração foi iniciado.

O analisador procura esses arquivos em:

```text
src/goes16/features/
```

Para reuni-los e executar o resumo:

```bash
cp ./*.log src/goes16/features/
PYTHONPATH=src python src/goes16/analyze_logs.py
```

A saída é impressa no terminal, separada por arquivo, com a quantidade de avisos, erros e exceções encontradas.

## 12. Sequência completa de execução

### 12.1 Preparar ambiente e diretórios

```bash
export EARTHDATA_USER="seu_usuario"
export EARTHDATA_PASS="sua_senha"
export GPMPWD="sua_senha"

./setup.sh
conda activate atmoseer
python -m pip install beautifulsoup4

mkdir -p data/goes16/CMI/2020/{C08,C09,C11,C13,C14,C15}
mkdir -p data/goes16/features
mkdir -p data/GPM
```

### 12.2 Baixar os canais ABI

```bash
YEAR=2020
START_DATE=2020-01-01
END_DATE=2020-01-02

for CHANNEL in 8 9 11 13 14 15; do
  CHANNEL_DIR=$(printf "C%02d" "$CHANNEL")

  PYTHONPATH=src python src/goes16/goes16_download_crop.py \
    --start_date "$START_DATE" \
    --end_date "$END_DATE" \
    --channel "$CHANNEL" \
    --crop_dir "data/goes16/CMI/${YEAR}/${CHANNEL_DIR}" \
    --spatial_resolution 0.1 \
    --vars CMI
done
```

### 12.3 Gerar a feature GLM

```bash
PYTHONPATH=src python src/goes16/goes16_retrieve_glm.py \
  --start_date 2020-01-01 \
  --end_date 2020-01-02 \
  --spatial_resolution 0.1
```

### 12.4 Baixar o IMERG

```bash
PYTHONPATH=src python src/gpm/gpm_download_crop.py \
  --begin_date 2020/01/01 \
  --end_date 2020/01/02 \
  --user "seu_usuario"
```

### 12.5 Gerar as features ABI

```bash
PYTHONPATH=src python src/goes16/main_goes16_features.py \
  --pn \
  --gtn \
  --fa \
  --wv_grad \
  --li_proxy \
  --toct \
  --verbose
```

### 12.6 Preparar e gerar PN_STD

```bash
mkdir -p data/goes16/features/profundidade_nuvens/2020
cp data/goes16/features/pn/2020/*.nc \
   data/goes16/features/profundidade_nuvens/2020/

PYTHONPATH=src python src/goes16/main_goes16_features.py \
  --pn_std \
  --verbose
```

### 12.7 Conferir as saídas

```bash
find data/goes16/CMI/2020 -type f -name '*.nc' | sort
find data/goes16/GLM/aggregated_data/2020 -type f -name '*.nc' | sort
find data/GPM/2020 -type f -name '*.nc' | sort
find data/goes16/features -type f -name '*.nc' | sort
```

## 13. Estrutura final produzida

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
