# Atualização periódica dos dados

Este painel não busca dados ao vivo: os números vêm de CSVs processados
localmente e publicados a cada `git push`. Manter o painel atualizado depende
de alguém repetir, periodicamente, o processo abaixo.

## Como o site é publicado

1. Os scripts de coleta (fora deste repositório) atualizam os CSVs em
   `Setor Celulose Papel/3_Dados_Tratados_CSV/`.
2. Copie os CSVs para `3_Dados_Tratados_CSV/` deste repositório.
3. Rode `python build_data.py` e `python build_downloads.py` (ou deixe o CI
   fazer isso) e confira com `pytest`.
4. Dê `git push` para a branch `main`. O GitHub Actions
   (`.github/workflows/build.yml`) reconstrói e republica o site no GitHub
   Pages automaticamente.

## Scripts de coleta

Ficam em `Setor Celulose Papel/2_Scripts/`, **fora deste repositório** (a pasta tem
o `.Renviron` com as credenciais e ~2,5 GB de arquivos brutos). Rodam com R
(`Rscript`) a partir da pasta `Setor Celulose Papel/`; cada um baixa só o que falta.
Tudo o que muda de um setor para outro está em `2_Scripts/config_setor.R`.

| Script | Fonte | Credencial | Periodicidade sugerida |
|---|---|---|---|
| `extrair_ibge.R` | IBGE/SIDRA e Banco Central (PIA-Empresa das divisões 17 e 18 e por classe, CEMPRE, PIM-PF 3.17 e 3.18, IPCA, câmbio, IPP das divisões 17 e 18 e expectativas do boletim Focus) | nenhuma | mensal (PIM, IPCA); anual (PIA) |
| `obter_rais_ftp.R` | Microdados públicos da RAIS no FTP do PDET (`ftp.mtps.gov.br/pdet/microdados/RAIS`): estabelecimentos e vínculos de 2006 em diante, ~45 GB ao todo, baixados em paralelo e apagados depois de lidos; o resultado de cada ano fica em `1_Dados_Manuais_XLS/rais_ftp_agregado/` (só anos novos são baixados de novo) | nenhuma | anual |
| `obter_rais_municipio_ftp.R` | Microdados da RAIS no FTP do PDET (estabelecimentos por município; vínculos do ano em `ano_vinculos`) | nenhuma | anual |
| `obter_caged_ftp_completo.R` | Microdados públicos do CAGED antigo (2008–2019, `CAGEDEST`) e do Novo CAGED (2020 em diante, `CAGEDMOV`) no FTP do PDET; cada mês fica em cache em `1_Dados_Manuais_XLS/caged_ftp_cache/` | nenhuma | mensal |
| `tratar_caged.R` | junta o CAGED antigo e o novo | nenhuma | depois do anterior |
| `obter_pia_produto.R` | IBGE, PIA-Produto (valor da produção por produto): API do SIDRA de 2014 em diante (tabelas 7752 e 10476); arquivos do servidor ftp.ibge.gov.br para 2007–2013 | nenhuma | anual |
| `obter_comex.R` | API do Comex Stat | nenhuma | mensal |
| `processar_comex_municipio.py` | Base bruta do Comex Stat por município (`EXP_/IMP_<ano>_MUN.csv`, em `1_Dados_Manuais_XLS/comexstat/mun/`): exportação e importação por município da empresa, por SH4 (SH4 em que a cesta cobre pelo menos metade das NCMs) | nenhuma | anual |
| `processar_comex_bruto.py` | Base bruta do Comex Stat (`EXP_/IMP_<ano>.csv` de balanca.mdic.gov.br, em `1_Dados_Manuais_XLS/comexstat/`): visão mensal, CIF, série por NCM e cestas fora do setor | nenhuma | mensal |
| `obter_comtrade.R` | API do UN Comtrade: só os recortes que o painel usa (todos os países x Mundo; Brasil x cada parceiro), várias HS por consulta | `COMTRADE_API_KEY` | semestral |
| `obter_bndes.R` | CSVs do portal de dados abertos do BNDES (`operacoes-financiamento-*.csv`, salvos em `1_Dados_Manuais_XLS/`) | nenhuma | trimestral |

Os scripts com BigQuery do painel de borracha (`obter_rais.R`, `obter_caged_antigo.R`,
`obter_novo_caged.R`, `obter_rais_municipio.R`) continuam na pasta como
alternativa mais rápida, mas exigem login do Google (`2_Scripts/autorizar_bigquery.R`)
e acesso ao projeto do `.Renviron`. O FTP dá o mesmo resultado (conferido em
2024 e em 2006/2008) sem login.

Ordem do CAGED: `obter_caged_ftp_completo.R` → `tratar_caged.R`. Energia Industrial é comum a
todos os painéis setoriais e não é coletada por estes scripts.

**Credenciais:** ficam apenas em `Setor Celulose Papel/.Renviron`. Nunca copie
chaves para código, arquivos versionados, mensagens de commit ou issues.

## Se os CSVs vierem como planilha (.xlsx)

Baixar a pasta pelo Google Drive pode converter os CSVs em planilhas. O script
`Setor Celulose Papel/2_Scripts/converter_xlsx_para_csv.py` desfaz a conversão e
confere valor a valor. Os arquivos de energia **não** podem ser recuperados
assim (a conversão do Drive corrompe o ponto decimal): use os CSVs originais
de outro painel.

## Mapas, CEMPRE e comparativos

- **CEMPRE** (`extrair_ibge.R`, `obter_cempre`): empresas ativas, pessoal ocupado
  e salários por classe CNAE, Brasil e UF, 2006 em diante (SIDRA 6449 e 9418),
  mais a indústria de transformação. Grava `Cempre_<Rotulo>.csv`.
- **RAIS por município** (`obter_rais_municipio_ftp.R`): lê os arquivos públicos
  do PDET em `1_Dados_Manuais_XLS/ftp_pdet/` (`RAIS_ESTAB_PUB_<ano>.7z` e
  `RAIS_VINC_PUB_<região>_<ano>.7z`) e grava `RAIS_Municipio_Estab_<Rotulo>.csv`
  e `RAIS_Municipio_Remun_<Rotulo>.csv` (setor e indústria de transformação por
  município). Alternativa com BigQuery: `obter_rais_municipio.R` (exige login Google).
- **Malha municipal** (`assets/geo/mun-<UF>.json`): contorno simplificado dos
  municípios por estado, derivado da malha do IBGE (cópia pública geodata-br).
- **PIM-PF comparativa** (`extrair_ibge.R`, `obter_pim_comparativos`): indústria
  geral e de transformação, `Producao_Fisica_Comparativos.csv`.

## Indicadores de desempenho (PIA-Empresa por classe)

`extrair_ibge.R` (função `obter_pia_indicadores`) grava
`PIA_Empresa_Indicadores_<Rotulo>.csv`: PIA-Empresa por classe CNAE, das
tabelas SIDRA 7241 a 7245 (2007–2023) e 10473 (2024 em diante, nova série),
classificação 12762, para as classes do setor
(`cnae_sidra`), as divisões (`divisoes`, recortes `div_17` e `div_18`) e a indústria de transformação
(`cnae_comparacao_sidra`, código 116910). `build_data.py` calcula a tabela
"Pontos de destaque" do diagnóstico setorial da FIESP (seção Financeiro), com a
metodologia da planilha DRE: R$ pelo IPCA de dezembro, lucro bruto = valor
adicionado − gastos de pessoal, margem bruta = lucro bruto / receita líquida,
investimento líquido = aquisições + melhorias − baixas. O painel mostra
2007/2023 (como no diagnóstico) e 2007/último ano (nova série, com quebra).

## PIA-Produto e mercado doméstico

`obter_pia_produto.R` grava `PIA_Produto_<Rotulo>.csv` com o valor da produção
dos produtos da cesta PRODLIST (`prodlist_cestas` em `config_setor.R`):

- **2014 em diante:** API do SIDRA, tabelas 7752 (2014–2023) e 10476 (2024+),
  variável 215 (valor da produção), classificação 1264 (produtos Prodlist),
  configurados em `pia_produto_sidra` no `config_setor.R`. É a mesma série do
  diagnóstico setorial da FIESP (bate com ela até 0,2%).
- **2007 a 2013:** Tabela 1 dos arquivos anuais do servidor do IBGE (a tabela
  5806 do SIDRA usa outra classificação). Diferença para a FIESP de até ~6%
  em 2007–2011, por revisões das listas de produtos antigas.

A PIA-Produto sai com cerca de dois anos de defasagem: o aviso de "sem dados"
para o ano mais recente é esperado até o IBGE publicar.

`cesta_componentes.csv` (neste repositório) liga cada NCM e cada PRODLIST ao
mesmo produto ("componente" do algoritmo da FIESP), para a tabela por produto
da seção Mercado doméstico. Ao mudar a cesta de NCM ou de PRODLIST, atualize
os três lugares juntos: `ncm_cestas`, `prodlist_cestas` e este CSV.

## Cesta de produtos (NCM/HS)

Não existe correspondência automática entre atividade (CNAE) e produto (NCM).
As cestas atuais estão em `config_setor.R` e descritas no `config_painel.json`
(campo `cesta_ncm`). Qualquer mudança precisa ser aprovada pelo dono do projeto
antes da coleta.
