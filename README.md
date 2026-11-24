# Painel do Setor de Celulose, Papel e Gráfico — CNAE 17 e 18

Painel executivo estático (sem framework, sem build step de JS) sobre o setor de
celulose, papel e gráfico no Brasil. Construído a partir do painel de borracha
(`pedroazevedofiesp/dashboard-borracha`), que por sua vez veio do painel de
fundição (`samuelmro/dashboard-fundicao`).

O seletor de segmento é uma caixa de seleção com 18 recortes:

| Recorte | Código no painel | O que soma |
|---|---|---|
| Celulose, papel e gráfico - Total | `total` | as 15 classes |
| Celulose e papel - Total | `17` | classes 17.10 a 17.49 |
| Gráfico - Total | `18` | classes 18.11 a 18.30 |
| 15 classes CNAE | `1710`, `1721`, `1722`, `1731`, `1732`, `1733`, `1741`, `1742`, `1749`, `1811`, `1812`, `1813`, `1821`, `1822`, `1830` | todas as subclasses da classe |

Os totais somam os dados de origem das classes (bloco `agregados` do
`config_painel.json`). Cada NCM está em uma só classe, então não há dupla contagem
no comércio exterior.

Blocos: produção física (índice IBGE por divisão), financeiro (PIA), emprego
formal (RAIS), CAGED, comércio exterior (Comex Stat e Comtrade), mercado
doméstico, BNDES, estudos especiais, simulador de cenários, cadeia produtiva,
diagnóstico setorial (COPAGREM) e energia industrial (EPE/MME-ANEEL).

## Estrutura

```
config_painel.json      Tudo o que é específico do setor (divisões, CNAEs, totais, rótulos, prefixos de arquivo, notas)
cadeia_config.json      Aba Cadeia Produtiva e pesos do cenário de demanda do simulador
cesta_componentes.csv   NCM e PRODLIST de cada produto (componente do algoritmo da FIESP) e a classe CNAE dele
3_Dados_Tratados_CSV/   CSVs tratados, separador ';' e decimal ','
build_data.py           Lê os CSVs e gera data/data.json, data/setores/*.json e data/energia/*.json
build_downloads.py      Lê os CSVs e gera data/downloads/*.xlsx (botão "Baixar dados" de cada seção)
extrair_cadeia.py       Gera os CSVs Cadeia_* a partir das tabelas do IBGE (TRU/MIP)
index.html, css/, js/   O site (js/charts.js: gráficos SVG; js/main.js: filtros, cálculos e gráficos)
tests/test_build_data.py Testes de fumaça do pipeline (pytest)
ATUALIZACAO.md          Como atualizar os dados, fonte por fonte
```

### Divisões CNAE

O IBGE só publica alguns dados por divisão (17 e 18), não por classe: o índice
de produção física (PIM-PF), o IPP e a série histórica da PIA-Empresa. Eles
ficam em `shared` (`producao.por_divisao`, `financeiro.grupos`, `macro[].ipp`)
e cada recorte usa os da sua divisão (campo `divisao` no `config_painel.json`).
O total do setor, que abrange as duas, mostra a divisão 17 nos gráficos mensais
e as duas no comparativo anual, com aviso no texto.

### Um arquivo por recorte

Com 18 recortes, um `data.json` único passaria de 20 MB. Por isso
`data/data.json` (~0,6 MB) traz só o que é comum e os limites de período de cada
recorte (`meta.setores`); os dados de cada recorte ficam em
`data/setores/<codigo>.json`, e o painel só baixa o recorte que for aberto.

## Como regenerar os dados

Pré-requisito: Python 3.10+.

```powershell
python -m venv .venv
.venv\Scripts\pip install -r requirements.txt
.venv\Scripts\python build_data.py
.venv\Scripts\python build_downloads.py
```

## Como rodar localmente

```powershell
.venv\Scripts\python -m http.server 8793
```

Abre em `http://localhost:8793`.

## Testes

```powershell
.venv\Scripts\python -m pytest
```

Os testes rodam `build_data.py` contra os CSVs reais e checam, para cada
recorte: schema, simetria entre recortes, séries não vazias e em ordem
cronológica, CAGED sem meses faltando, totais iguais à soma das classes,
mercado doméstico, simulador, CEMPRE, municípios e energia industrial.

## Cesta de produtos (comércio exterior)

Não existe correspondência oficial entre atividade (CNAE) e produto (NCM). A
cesta segue o algoritmo por produto do diagnóstico setorial da FIESP (COPAGREM):
componentes 862 a 916 e 1014 (celulose e papel) e 905, 908, 910, 917 a 925, 1577
e 1799 a 1805 (impressão e gravação). Cada NCM vai para a classe do seu PRODLIST
(4 primeiros dígitos); quando a mesma NCM serve a duas classes (papel 17.21 e
papel-cartão 17.22; embalagens 17.31 e 17.32), fica na classe principal. Por
isso 17.32 não tem cesta própria e 18.22 (acabamento gráfico, um serviço) não
tem NCM: nesses recortes o painel mostra uma nota no lugar dos gráficos de
comércio exterior. A lista completa está em `cesta_componentes.csv`.

## Aba Cadeia Produtiva

Diagrama de onde vêm os insumos e para quem vai o papel, com as Contas
Nacionais do IBGE (TRU nível 68, 2010–2021, e MIP 2015). Configurada em
`cadeia_config.json`: produto 17002 (papel, papelão, embalagens e artefatos de
papel) e atividade 1700. Os CSVs `Cadeia_*` saem de `extrair_cadeia.py`, que lê
as tabelas do IBGE em `../Setor Celulose Papel/1_Dados_Manuais_XLS/` (fora do
git). Ver `COMO_REPLICAR_CADEIA.md`.

## Fontes de dados

| Bloco | Base | Cobertura |
|---|---|---|
| Produção física | IBGE — PIM-PF, divisões 3.17 e 3.18 (índice) | mensal, desde 2002 (17) |
| Financeiro | IBGE — PIA-Empresa (divisões 17 e 18; por classe em 2007–2024) | anual |
| Emprego formal | RAIS (microdados públicos do PDET, FTP) | 2006–2025 (anual) |
| CAGED | CAGED + Novo CAGED (microdados públicos do PDET, FTP) | jan/2008 em diante (mensal) |
| Comércio exterior | MDIC — Comex Stat, por cesta de NCM | 2006 em diante (anual, ano corrente parcial) |
| Comércio exterior (contexto mundial) | UN Comtrade (proxy por HS) | 2015–2024 (anual) |
| Mercado doméstico | IBGE — PIA-Produto + Comex Stat | 2007–2024 (anual) |
| Energia Industrial | EPE (consumo) + MME/ANEEL (tarifa média faturada) | 2012 em diante (mensal) |
| BNDES | Desembolsos por UF/porte/instrumento | anual |
| Cadeia Produtiva | IBGE — TRU nível 68 + MIP 2015 | 2010–2021 (anual) |

## Publicação (GitHub Pages)

`.github/workflows/build.yml` roda os testes, regenera `data/` e publica o site
a cada push na branch `main`. Em **Settings → Pages**, a fonte ("Source") é
**GitHub Actions**. A pasta `data/` é gerada pelo CI e não fica no repositório.

CSS, JS e os JSON de dados são referenciados com `?v=dev`; o workflow troca isso
pelo commit a cada publicação, para o navegador não misturar arquivo novo com
arquivo velho guardado em cache.
