# AtmoSeer — Documentação de reprodução

Este repositório reúne a documentação operacional utilizada para reproduzir as etapas do trabalho de predição de precipitação utilizando os dados do GOES16 através do Atmoseer.

Os comandos consideram que o código-fonte do Atmoseer está disponível localmente e que a execução é feita a partir da raiz desse projeto.

## Organização

```text
atmoseer-documentacao/
├── README.md
├── docs/
│   ├── 01-download-goes16-glm-imerg-e-features.md
│   ├── 02-construcao-dataset-e-treinamento.md
│   └── assets/
└── .gitignore
```

## Ordem de leitura

1. **Download de GOES-16 e IMERG e geração de features**  
   Arquivo: [`docs/01-download-goes16-glm-imerg-e-features.md`](docs/01-download-goes16-glm-imerg-e-features.md)

2. **Construção do dataset e treinamento do modelo**  
   Arquivo: [`docs/02-construcao-dataset-e-treinamento.md`](docs/02-construcao-dataset-e-treinamento.md)

## Convenções dos guias

- Todos os caminhos são relativos à raiz do Atmoseer.
- Todos os exemplos usam terminal Bash.
- Diretórios `data/`, arquivos NetCDF, credenciais, logs e modelos não fazem parte deste repositório documental.
- Cada guia descreve os arquivos executados, os argumentos recebidos, a estrutura de entrada e os arquivos produzidos.
