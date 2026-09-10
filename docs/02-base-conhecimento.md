# Base de Conhecimento

## Dados Utilizados

> Meus dados de aquecimento para meu Agente "Primo pobre" foram criados conforme a nescessidade de testes!

| Arquivo | Formato | Utilização no Agente |
|---------|---------|---------------------|
| `transacoes_historico.csv` | CSV | Contextualizar interações anteriores |
| `perfil_usuario` | JSON | todos os dados nescessario do usuario para sincronizar com os dados obtidos |




---

## Adaptações nos Dados



 Meus dados de aquecimento para meu Agente "Primo pobre" foram criados com Chats para melhor aproveito mediante as nescessidades do Agente Primo Probre.



---

## Estratégia de Integração

### Como os dados são carregados?

Os dados do agente são armazenados na pasta `data/` e carregados quando a aplicação é iniciada. Os arquivos CSV são utilizados para armazenar as transações financeiras, enquanto os arquivos JSON guardam informações sobre o usuário, os orçamentos, as metas e as regras de funcionamento do agente.
``` Python
import json
from pathlib import Path
import pandas as pd

# Caminhos dos arquivos
BASE_DIR = Path(__file__).resolve().parent.parent
DATA_DIR = BASE_DIR / "data"
TRANSACOES_PATH = DATA_DIR / "transacoes_historico.csv"
PERFIL_PATH = DATA_DIR / "perfil_usuario.json"

# ------------------------------------------------------------
# 1. CSV com as entradas e saídas financeiras
# ------------------------------------------------------------
def detectar_configuracao_csv():
    """Detecta a codificação e o separador do CSV."""
    for codificacao in ["utf-8-sig", "utf-8", "cp1252", "latin1"]:
        try:
            conteudo = TRANSACOES_PATH.read_text(encoding=codificacao)
            primeira_linha = conteudo.splitlines()[0]
            separador = (
                ";" if primeira_linha.count(";") > primeira_linha.count(",")
                else ","
            )
            return codificacao, separador
        except (UnicodeDecodeError, FileNotFoundError):
            continue
    return "utf-8-sig", ","

def carregar_transacoes():
    if not TRANSACOES_PATH.exists():
        return pd.DataFrame()  
    codificacao, separador = detectar_configuracao_csv()
    return pd.read_csv(
        TRANSACOES_PATH,
        sep=separador,
        encoding=codificacao,
        on_bad_lines="skip",  
        engine="python"
    )

transacoes = carregar_transacoes()

# ------------------------------------------------------------
# 2. JSON com o perfil e as preferências do usuário
# ------------------------------------------------------------

def carregar_perfil():
    try:
        with open(PERFIL_PATH, "r", encoding="utf-8") as arquivo:
            return json.load(arquivo)
    except UnicodeDecodeError:
        with open(PERFIL_PATH, "r", encoding="cp1252") as arquivo:
            return json.load(arquivo)
    except FileNotFoundError:
        return {"nome": "Usuário"}

perfil_usuario = carregar_perfil()

## Como os dados são usados no prompt?

Os dados não são enviados ao agente de forma desorganizada. Primeiro, o sistema identifica o que o usuário deseja fazer e consulta apenas as informações necessárias para responder.As regras do agente e o perfil do usuário são carregados no contexto principal, pois são usados com frequência. Já as transações, os orçamentos e as metas são consultados dinamicamente conforme a solicitação.Por exemplo:
* Para registrar uma entrada ou saída, o agente consulta as regras de registro e classificação.
* Para informar o saldo, o sistema consulta as transações do período solicitado.
* Para verificar um orçamento, compara o valor realizado com o valor planejado.
* Para acompanhar uma meta, consulta o valor esperado e o valor já alcançado.
* Para identificar padrões, agrupa as transações por categoria, mês ou tipo.
* Para gerar alertas, utiliza os limites definidos no perfil do usuário e nas regras do agente.
  
O contexto enviado ao agente deve conter apenas os dados necessários para a solicitação atual. Dessa forma, a resposta fica mais organizada e o agente reduz o risco de misturar informações ou criar conclusões sem base nos dados.

```python
def montar_contexto_financeiro(
    perfil_usuario,
    regras_agente,
    transacoes,
    orcamentos_metas
):
    ultimas_transacoes = (
        transacoes
        .tail(10)
        .to_dict(orient="records")
    )

    contexto = {
        "perfil_usuario": perfil_usuario,
        "regras_agente": regras_agente,
        "ultimas_transacoes": ultimas_transacoes,
        "orcamentos": orcamentos_metas.get("orcamentos", []),
        "metas": orcamentos_metas.get("metas", [])
    }

    return contexto

```

O agente deve seguir estas instruções ao utilizar os dados:

* Responder somente com base nas informações disponíveis.
* Não inventar valores, datas, categorias ou movimentações.
* Informar quando não houver dados suficientes.
* Diferenciar informações registradas de estimativas.
* Pedir esclarecimentos quando uma movimentação estiver incompleta ou ambígua.
* Separar fatos, cálculos e sugestões.
* Não julgar as decisões financeiras do usuário.
  ---

> Exemplo de Contexto Montado

CONTEXTO DO AGENTE FINANCEIRO

Perfil do usuário:
- Nome:  Felipe
- Perfil: Profissional autônomo
- Área de atuação: Publicidade e tecnologia
- Moeda principal: BRL
- Período analisado: Janeiro a Junho de 2026
- Separação entre despesas pessoais e profissionais: Ativada
- Frequência de resumo: Mensal
- Tom de comunicação: Informal, próximo, educativo e proativo

Objetivos financeiros:
- Organizar as entradas e saídas do negócio.
- Acompanhar o saldo mensal.
- Identificar padrões de consumo.

Regras importantes:
- Toda movimentação precisa ter tipo, valor, data e descrição.
- O tipo deve ser entrada ou saída.
- Quando a categoria não estiver clara, perguntar antes de registrar.
- Valores retirados pelo proprietário devem ser separados das despesas operacionais.
- Não inventar dados para completar uma análise.
- Informar quando o histórico for insuficiente.
- Não fazer recomendações personalizadas de investimento.
- Não julgar as escolhas financeiras do usuário.

Resumo financeiro do período:
- Janeiro de 2026:
  - Entradas: R$ 8.900,00
  - Saídas: R$ 4.254,90
  - Saldo: R$ 4.645,10

- Fevereiro de 2026:
  - Entradas: R$ 10.200,00
  - Saídas: R$ 4.449,90
  - Saldo: R$ 5.750,10

- Março de 2026:
  - Entradas: R$ 9.550,00
  - Saídas: R$ 5.104,90
  - Saldo: R$ 4.445,10

- Abril de 2026:
  - Entradas: R$ 11.900,00
  - Saídas: R$ 5.349,90
  - Saldo: R$ 6.550,10

- Maio de 2026:
  - Entradas: R$ 10.400,00
  - Saídas: R$ 4.819,90
  - Saldo: R$ 5.580,10

- Junho de 2026:
  - Entradas: R$ 13.000,00
  - Saídas: R$ 5.819,90
  - Saldo: R$ 7.180,10

Últimas transações:
- 03/06/2026: Entrada de R$ 6.800,00
  - Descrição: Campanha de lançamento
  - Categoria: Prestação de serviços

- 05/06/2026: Saída de R$ 1.200,00
  - Descrição: Aluguel do escritório
  - Categoria: Custos fixos

- 06/06/2026: Saída de R$ 289,90
  - Descrição: Assinatura de ferramenta de gestão
  - Categoria: Ferramentas e softwares

- 08/06/2026: Saída de R$ 690,00
  - Descrição: Combustível e deslocamentos
  - Categoria: Transporte

- 10/06/2026: Entrada de R$ 3.600,00
  - Descrição: Desenvolvimento de automação
  - Categoria: Prestação de serviços

- 13/06/2026: Saída de R$ 1.650,00
  - Descrição: Anúncios patrocinados
  - Categoria: Marketing

- 18/06/2026: Saída de R$ 780,00
  - Descrição: Contratação de editor freelancer
  - Categoria: Equipe e colaboradores

- 25/06/2026: Saída de R$ 900,00
  - Descrição: Retirada do proprietário
  - Categoria: Retirada pessoal





