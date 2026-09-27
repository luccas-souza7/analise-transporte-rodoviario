# Transporte Rodoviário: Análise de Custo de Frete

**Autor:** Luccas Souza | Analista de Dados | **Data:** 26/09/2026

🔗 **[Acessar o dashboard publicado](https://app.powerbi.com/view?r=eyJrIjoiYzRkMjkxMDAtMzQyNy00MTI2LThjMmItNGNlMzI1ZWRjZjM3IiwidCI6IjgxY2I0YzMxLWRiOTUtNDcxNC1hYzhjLTRkMzAwNmVjZjRiMCJ9)**

![Visão geral do dashboard: KPIs do período, evolução mensal do custo e destinos por UF](visao_geral.png)

## 1. Visão Geral do Negócio

**O problema.** Custo por quilômetro é o indicador que toda operação de transporte acompanha, e é
também o que mais engana. Uma frota própria parece imbatível quando se divide o custo dela por todo
quilômetro rodado: R$ 4,34/km contra R$ 8,77/km do frete contratado, 102% de diferença. A conta
esquece que só o caminhão próprio precisa voltar ao CD, e volta vazio. Decidir entre comprar
caminhão e contratar frete com base nesse número é decidir com um viés de 2×.

**O objetivo deste projeto** foi responder à pergunta de gestão que vem depois da conta de R$/km:
**quanto** custa o frete e onde está o dinheiro, **qual trecho** puxa o custo, **com que ativo** se
está gastando, **comprar ou contratar** (e quanto se perde rodando vazio) e **o que esse dinheiro
entregou** em nível de serviço. Cada pergunta é uma página, na ordem em que o raciocínio acontece.

**Usuário final.** O dashboard atende dois perfis:

- **O gestor de transporte ou de compras de frete**, que precisa de um número comparável entre
  modalidades para negociar contrato, dimensionar frota e defender orçamento, e de uma leitura de
  risco regulatório (frete contratado abaixo do piso mínimo da ANTT).
- **O analista**, que precisa cruzar livremente período, CD de origem, região, UF, cidade, rota,
  modalidade de frota, tipo de veículo e status de entrega, sem que nenhuma métrica mude de
  definição no caminho.

> ⚠️ **Nota de transparência:** os dados são **inteiramente sintéticos**, gerados por script Python
> com semente fixa (seed 42). Os parâmetros foram calibrados com referências públicas do setor
> (coeficientes de piso mínimo da ANTT, preço do diesel S10 da ANP, consumo por configuração de
> eixos, Pesquisa CNT de Rodovias) para produzir ordens de grandeza plausíveis, mas **não derivam de
> dados de nenhuma empresa**. Os números aqui demonstram método analítico; não são benchmark de
> mercado.

**Escopo.** 37.782 pernas de viagem entre agosto de 2024 e julho de 2026 (24 meses), 88 veículos
em três modalidades (frota própria, agregado e terceiro), 83 rotas saindo de dois centros de
distribuição (Suzano-SP e Contagem-MG) para 42 cidades em 24 UFs. Custo total no período:
**R$ 181,9 milhões**.

## 2. Racional Analítico

Cada indicador entrou no dashboard porque responde a uma pergunta que muda uma decisão. Os que não
mudavam nenhuma ficaram de fora.

### O grão é a perna de viagem, não a entrega

Uma linha do fato é uma perna: um CT-e carregado **ou** um retorno vazio da frota própria. Confundir
esse grão com "1 linha = 1 entrega" apagaria o retorno vazio da base, e com ele o custo que ele
gera. É a decisão que sustenta todo o resto: sem a perna vazia no fato, não existe como medir
ociosidade nem corrigir a comparação entre modalidades.

### Uma métrica única de custo para naturezas contábeis diferentes

A frota própria gera custo por componente (diesel, pedágio, manutenção, motorista, custo fixo
rateado); agregado e terceiro geram um valor de frete contratado. As duas colunas são mutuamente
exclusivas por regra de negócio. O `custo_total` é derivado na carga (frete contratado quando
existe, senão a soma dos componentes) e é a única métrica de custo do dashboard. É o que torna as
três modalidades comparáveis na mesma escala.

### Por que custo por km **carregado**, e não por km rodado

É o indicador central do projeto. O denominador desconta o quilômetro que não transportou nada:

```dax
Custo KM Carregado Própria =
DIVIDE (
    [Custo Frota Própria],
    CALCULATE ( [KM Carregado], D_Veiculo[modalidade_frota] = "Frota Própria" )
)
```

```dax
Gap Contratado vs Própria =
DIVIDE ( [Custo KM Carregado Contratado], [Custo KM Carregado Própria] ) - 1
```

A frota própria roda 38,5% dos quilômetros, mas 28,4% deles voltando vazia. Corrigindo o
denominador, o custo dela sobe de R$ 4,34 para **R$ 6,05 por km carregado**, contra R$ 8,77 do
contratado: a vantagem cai de 102% para **44,9%**. Ela continua existindo, mas é menos da metade do
que a conta ingênua sugere, e é esse o número que sustenta uma decisão de investimento em frota.

![Frota e ociosidade: a comparação ingênua (102%) contra a comparação justa (44,9%)](frota_ociosidade.png)

### Quanto custa rodar vazio

O retorno vazio custa **R$ 10,8 milhões** no período, 6,0% do gasto total. A ociosidade não é
homogênea: sobe com a distância e com a falta de carga de volta. O percentual de km vazio vai de
9,1% nos destinos do Sudeste a 13,4% no Norte. É o insumo para negociar carga de retorno nas rotas
longas, antes de discutir preço de frete.

### Por que o OTIF é multiplicativo

```dax
OTIF = [% No Prazo (OTD)] * [% In Full]
```

OTIF mede a entrega que chegou **no prazo e completa**, sem avaria ou devolução. Tirar a média das
duas taxas daria 92,4% e contaria como sucesso uma entrega que falhou em um dos critérios. O produto
(88,0% × 96,8% = **85,2%**) é a proporção que passou nos dois. O denominador exclui viagens ainda em
trânsito e retornos vazios, que não têm entrega a avaliar.

### Compliance: frete abaixo do piso ANTT como risco mensurável

Dos 22.139 CT-e contratados, **3.753 (17,0%)** foram fechados abaixo do piso mínimo de referência.
Na média a carteira paga 13,3% acima do piso, e é exatamente por isso que a média não serve aqui: ela
esconde uma cauda relevante que expõe a empresa a autuação. O indicador conta CT-e, não reais,
porque o risco é por documento emitido.

### Onde o dinheiro está e onde ele rende menos

Sudeste (32,3%) e Nordeste (29,0%) somam 61% do custo, mas por motivos opostos: o Sudeste por
volume de viagens curtas, o Nordeste por distância. São dois problemas com o mesmo tamanho de conta
e ações diferentes. A cauda de rotas é longa: as cinco maiores, todas de longa distância saindo de
Suzano, concentram 14,6% do custo, com Suzano → Manaus à frente (R$ 6,8 milhões).

Por tipo de veículo, a leitura de eficiência contradiz a de volume:

| Tipo de veículo | Participação no custo | R$ por km carregado |
|---|---|---|
| Toco | 4,6% | 5,17 |
| Truck | 12,1% | 6,12 |
| Carreta LS | 23,6% | 8,19 |
| Carreta 3 eixos | **36,6%** | **8,81** |
| Rodotrem | 5,1% | 11,37 |

A configuração que mais consome orçamento está entre as menos eficientes por quilômetro carregado.
É a linha que merece a primeira revisão de contrato.

![Mapa de rotas: custo por UF de destino e arcos saindo dos dois CDs](mapa_rotas.png)

## 3. Decisões de Design/UX

O layout foi tratado como parte da análise, não como acabamento. Cada decisão abaixo existe para
reduzir esforço de leitura ou impedir uma conclusão errada.

- **Termos técnicos explicados no próprio visual.** Cada KPI e cada título de gráfico tem um ícone
  de informação com o que o indicador significa, como é calculado e um exemplo. OTIF, In Full, km
  carregado e retorno vazio não são vocabulário de quem aprova orçamento; um número que o leitor
  não entende não sustenta decisão.
- **Filtro cruzado em painel suspenso, com contador.** Nove dimensões que se combinam entre si e
  valem para todas as páginas, com aplicação imediata e sem botão "aplicar". Os filtros ativos
  aparecem como etiquetas removíveis no topo da página e em um contador no botão: um filtro
  esquecido é a causa mais comum de leitura errada de um dashboard.
- **Filtro de veículo pelo desenho do caminhão.** Cada tipo aparece ilustrado com a configuração de
  eixos e capacidade. "Bitrem" e "Rodotrem" não são nomes que todo leitor distingue; a silhueta com
  7 e 9 eixos resolve sem legenda.
- **Nenhum cartão com espaço vazio.** Gráficos são desenhados no tamanho real do cartão; listas se
  distribuem na altura disponível e ganham rolagem quando crescem. Espaço vazio sugere dado
  faltando.
- **KPIs com a mesma estrutura e altura.** Rótulo, valor, contexto e minigráfico de tendência
  sempre na mesma posição, para que a comparação entre cartões seja visual e não de leitura.
- **Rótulos de dado em todas as séries**, abreviados e com densidade adaptativa: se não cabem
  todos, ficam os extremos, o primeiro, o último e os intermediários alternados. Linhas de
  referência (média, meta) vão para a legenda para não disputar espaço com o dado.
- **Mapa com arcos por CD de origem.** A pergunta "qual trecho puxa o custo" é geográfica; uma
  tabela de 83 rotas não mostra que o problema está no eixo Suzano → Norte/Nordeste.
- **Modo noturno** com paleta própria para gráficos, mapa e ilustrações, não uma inversão de cores.
- **Estado preservado.** Página, filtros e tema sobrevivem às atualizações do visual pelo Power BI,
  para que uma interação não devolva o leitor ao ponto de partida.

## 4. Arquitetura e Stack

- **Power BI Desktop / Power BI Service**: modelagem, publicação e distribuição.
- **Power Query (M)**: ingestão e limpeza de três arquivos CSV (fato de 37.970 × 26 e duas
  dimensões). Decisões tomadas a partir do dado inspecionado:
  - 151 pernas reemitidas pelo sistema de origem e 37 com distância inválida foram removidas; sem
    distância não existe custo por km.
  - 56 pesos impossíveis foram **anulados mantendo a viagem**: o custo e a quilometragem dela são
    válidos, e excluir a linha por uma coluna auxiliar distorceria o custo total.
  - A mesma placa aparecia em 176 grafias (com hífen, minúsculas, espaço no fim) para 88 veículos;
    a padronização nos dois lados do relacionamento é o que zera os órfãos.
  - O tipo de carga tinha 10 grafias para 7 categorias reais.
  - O fato vem no padrão brasileiro (vírgula decimal) e o cadastro de veículos no invariante (ponto).
    A tipagem automática leu o consumo de 2,5 km/l como 25 sem erro nenhum; a cultura de conversão é
    declarada explicitamente por arquivo.
- **Modelo em star schema**: fato de pernas de viagem (37.782 linhas, 31 colunas com as derivadas)
  ligado a três dimensões (veículo, rota e calendário), todos os relacionamentos N:1 e de direção
  única. A distância de referência da dimensão de rota foi renomeada na carga para não coincidir com
  a distância rodada do fato e evitar a autodetecção de um relacionamento espúrio.
- **DAX**: 35 medidas de negócio em quatro grupos funcionais (KPIs, frota e ociosidade, nível de
  serviço, compliance e eficiência), além das medidas da camada de apresentação. Decisões que valem
  registro:
  - **Filtro de contexto preservado no km carregado.** Na abertura por tipo de viagem, sobrescrever
    o filtro faria a linha "retorno vazio" exibir o km carregado total; o filtro é aplicado de forma
    aditiva.
  - **Campo vazio não é nulo.** Ocorrência sem registro chega do CSV como texto vazio; a contagem
    ingênua por "não em branco" devolvia 37.782 ocorrências em vez de 2.324.
  - **Todo o HTML vive em medida, nunca em coluna:** coluna trunca texto em 32.766 caracteres na
    carga, e a camada de apresentação passa de 500 mil caracteres.
  - **Sanitização de texto na serialização**, exigida pelo dado real: duas rotas terminam em "Santa
    Bárbara d'Oeste", e o apóstrofo encerraria o literal no lugar errado sem nenhum erro no DAX.
  - **Um único cubo agregado para o filtro cruzado.** Os dados vão para o visual no grão mês × rota
    × veículo e modalidade × status (18.653 linhas). É o menor grão que permite combinar os nove
    filtros entre si; a interação acontece no navegador, sem nova consulta ao modelo.
- **Python (pandas, numpy)**: geração do dataset sintético com semente fixa, o que torna todo número
  deste documento reproduzível.
- **HTML, CSS e JavaScript**: camada de apresentação servida por um visual customizado, com mapa em
  SVG construído a partir da malha territorial do IBGE.

> **Sobre a escolha da camada visual:** a interface foi construída como aplicação HTML/CSS/JS em um
> único visual, e não com visuais nativos. A escolha é deliberada para uma peça de portfólio:
> demonstra domínio do modelo semântico e da apresentação de ponta a ponta. **Em ambiente
> corporativo**, com governança, certificação de visuais e um time que mantém o artefato depois que
> o autor sai do projeto, o default seria visuais nativos do Power BI, pelo custo de manutenção.

## Contato

- 📧 luccasnsouza1@gmail.com
- 🔗 [linkedin.com/in/luccas-souza7](https://www.linkedin.com/in/luccas-souza7/)
- 💻 [github.com/luccas-souza7](https://github.com/luccas-souza7)
- 📱 [+55 11 93201-8859](https://wa.me/5511932018859)
