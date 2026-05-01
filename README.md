# Projetos Northwind Traders: Logística, Vendas e Simulação de Cenários.

## 1. O Desafio de Negócio (O Problema)

No ecossistema de distribuição B2B e varejo, analisar apenas o faturamento bruto é olhar para o passado de forma incompleta. O verdadeiro desafio de um gestor de operações é entender a relação entre **Volume de Vendas**, **Eficiência Logística** e **Impacto de Preços**, garantindo que o crescimento da receita não destrua a margem de lucro ou atrase as entregas.

Este projeto End-to-End atua como uma solução analítica completa para a Northwind Traders. O objetivo foi construir um dashboard executivo que vai além do descritivo, integrando inteligência geográfica, acompanhamento da força de vendas e, principalmente, simulação preditiva de cenários corporativos.

**Perguntas de Negócio respondidas:**

1. Qual é o faturamento total, o volume (quantidade de pedidos) girado pela operação e o ticket médio das vendas?
2. Quantos clientes ativos a empresa possui e quem compõe a "elite" (Top 5) de compradores que sustentam a receita?
3. O aumento nas vendas ao longo dos meses (YoY) impacta negativamente o nosso tempo médio de entrega?
4. Qual transportadora oferece o melhor equilíbrio entre custo de frete, volume transportado e taxa de entregas no prazo?
5. Qual é a concentração geográfica das nossas vendas e quem é o vendedor número 1 responsável por cada país?
6. Qual seria a projeção de faturamento se aplicássemos diferentes níveis de agressividade comercial (descontos) por categoria de produto?

---

## 2. Arquitetura e Stack Tecnológico

Para garantir um processamento rápido, modelagem otimizada e uma interface premium, a solução integrou as seguintes ferramentas:

* **Extração e Transformação (ETL):** Power Query (Linguagem M).
* **Modelagem de Dados:** Star Schema (Fato e Dimensões).
* **Modelagem Analítica:** DAX (Data Analysis Expressions).
* **UI/UX Design:** Figma (criação de backgrounds e assets) e Power BI.
* **Visualização:** Power BI Desktop.

---

## 3. UI/UX Design e Prototipagem

Um dashboard de nível executivo exige leitura instantânea, redução da carga cognitiva e um fluxo de navegação natural. Toda a interface foi pensada sob os princípios de *Progressive Disclosure* (Revelação Progressiva).

* **Dark Mode e Identidade Visual:** Utilização de um fundo chumbo escuro corporativo (`#1b1e24`) com acentos em Azul Ciano (`#00d2ff`). Essa paleta cria um alto contraste para os dados, dando um aspecto de sistema web/aplicativo e reduzindo o cansaço visual.
* **Navegação Estilo App (Home Page):** Criação de uma página inicial de aterrissagem imersiva com botões responsivos (*hover effect*), conduzindo o usuário intuitivamente para as páginas de Visão Executiva, Operacional ou Simulador.
* **Inteligência em Tooltips (Menos é Mais):** Para evitar poluição visual e barras de rolagem, gráficos densos foram transformados em "dossiês" interativos ocultos. O usuário passa o mouse e descobre:
  * *Raio-X do Mês:* Top 5 Produtos mais vendidos no mês selecionado.
  * *Lupa Geográfica:* O faturamento por categoria e o Melhor Vendedor daquele país específico no mapa.
  * *Top Clientes:* Os 5 maiores compradores que compõem a métrica geral de faturamento.
* **Limpeza de Ruído:** Remoção de eixos e títulos óbvios, aplicação de Filtros Top N para "travar" rankings, garantindo telas limpas e focadas puramente na tomada de decisão.

---

## 4. Engenharia e Modelagem de Dados

Para garantir a performance do painel com filtros dinâmicos, os dados transacionais brutos foram estruturados em um modelo **Star Schema** clássico, centralizando a `Fato_Vendas` e conectando-a a dimensões como Tempo, Clientes, Produtos, Vendedores e Transportadoras.

<details>
<summary><b>🛠️ Script 01: Tratamento e Conexão (Power Query M) (Clique para expandir)</b></summary>
```powerquery
let
    // Conexão com a base de dados transacional
    Fonte = Sql.Database("Servidor_Northwind", "NorthwindDB"),
    dbo_Orders = Fonte{[Schema="dbo",Item="Orders"]}[Data],
    
    // Mescla com Detalhes do Pedido para expansão granular
    #"Consultas Mescladas" = Table.NestedJoin(dbo_Orders, {"OrderID"}, OrderDetails, {"OrderID"}, "OrderDetails", JoinKind.Inner),
    #"OrderDetails Expandido" = Table.ExpandTableColumn(#"Consultas Mescladas", "OrderDetails", {"ProductID", "UnitPrice", "Quantity", "Discount"}, {"ProductID", "UnitPrice", "Quantity", "Discount"}),
    
    // Tratamento de Datas para logística
    #"Tipo Alterado" = Table.TransformColumnTypes(#"OrderDetails Expandido",{{"OrderDate", type date}, {"RequiredDate", type date}, {"ShippedDate", type date}}),
    
    // Criação de coluna condicional para atrasos
    #"Atraso Calculado" = Table.AddColumn(#"Tipo Alterado", "Entrega no Prazo", each if [ShippedDate] <= [RequiredDate] then 1 else 0)
in
    #"Atraso Calculado"
```
</details>

<details>
<summary><b>🛠️ Script 02: Geração da Dimensão Calendário (DAX) (Clique para expandir)</b></summary>
```dax
dCalendario = 
ADDCOLUMNS(
    CALENDARAUTO(),
    "Ano", YEAR([Date]),
    "Mês Nome", FORMAT([Date], "mmmm"),
    "Mês Num", MONTH([Date]),
    "Trimestre", "Tri " & FORMAT([Date], "q"),
    "AnoMês", FORMAT([Date], "YYYY-MM")
)
```
</details>

---

## 5. Desenvolvimento Analítico (Dicionário de KPIs em DAX)

Para traduzir a operação em inteligência de negócio, desenvolvi métricas focadas tanto em volume quanto em rentabilidade e logística.
```dax
// 1. Total de Pedidos (Giro da Operação)
Total de Pedidos = 
DISTINCTCOUNT('Fato_Vendas'[OrderID])

// 2. Clientes Ativos (Penetração de Mercado)
Clientes Ativos = 
DISTINCTCOUNT('Fato_Vendas'[CustomerID])

// 3. Faturamento Total Liquido (Receita baseada em Preço, Quantidade e Desconto original)
Faturamento Total = 
SUMX(
    'Fato_Vendas',
    'Fato_Vendas'[Quantity] * 'Fato_Vendas'[UnitPrice] * (1 - 'Fato_Vendas'[Discount])
)

// 4. Ticket Médio (Qualidade da venda)
Ticket Médio = 
DIVIDE([Faturamento Total], [Total de Pedidos], 0)

// 5. Entregas no Prazo % (SLA Logístico)
Entregas no Prazo = 
DIVIDE(
    CALCULATE(COUNTROWS('Fato_Vendas'), 'Fato_Vendas'[Entrega no Prazo] = 1),
    COUNTROWS('Fato_Vendas'),
    0
)

// 6. Faturamento Simulado (Parâmetro What-If para previsão de receita)
Faturamento Simulado = 
SUMX(
    'Fato_Vendas',
    'Fato_Vendas'[Quantity] * 'Fato_Vendas'[UnitPrice] * (1 - 'Desconto Simulado'[Valor Desconto Simulado])
)
```

---

## 6. Storytelling de Dados e Dashboards

O painel segue uma arquitetura de informação focada em funil executivo:

* **Página 1: Visão Executiva (O Macro):** Cartões de indicadores diretos (Faturamento, Ticket, Custos). O coração desta aba é o gráfico de linha dupla que monitora se os picos de faturamento estão afetando o prazo de entrega. Tooltips secretas revelam quem são os Top Clientes puxando essa receita.
* **Página 2: Operacional (O Solo):** Análise logística via gráfico de dispersão (Custo vs. Prazo por transportadora) cruzado com o mapa de abrangência global. Aqui, a diretoria consegue enxergar a performance tática (Rankings dos melhores vendedores e distribuição categórica por país).
* **Página 3: Simulador (O Futuro):** Um ambiente preditivo isolado. Através do ajuste de uma barra interativa, a diretoria altera os descontos comerciais e vê, em tempo real, os gráficos recalcularam o impacto no caixa em relação ao cenário original, produto a produto.

---

## 7. Conclusão e Plano de Ação

Com base na visualização interativa desenvolvida, as recomendações estratégicas imediatas são:

* **Blindagem da Base de Clientes:** Uma parcela gigantesca da receita vem do Top 5 Clientes (ex: *QUICK-Stop* e *Save-a-lot Markets*). É vital criar políticas de retenção exclusivas para essas contas.
* **Otimização de Frete:** Transportadoras no quadrante inferior direito do gráfico de dispersão (baixo custo, porém lentas) devem ser evitadas em categorias perecíveis ou de margem alta. A migração de volume logístico para parceiros com SLA acima de 95% compensa o custo extra a longo prazo.
* **Gestão de Descontos Cirúrgicos:** O Simulador prova que aplicar agressividade comercial (ex: 15% de desconto) de forma linear corrói a margem indevidamente. O foco deve ser aplicar essa alavanca apenas em categorias de baixo giro para limpar estoque.

---

## Contato

| 👨‍💻 Autor | **Arthur Mesquita** |
| :--- | :--- |
| **Cargo** | Analista de Dados |
| **LinkedIn** | [linkedin.com/in/arthur-g-mesquita](https://www.linkedin.com/in/arthur-g-mesquita) |
| **Localização** | 📍 Recife, PE |
