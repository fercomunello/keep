# Plano de execução no Postgres

Quando se trata de **otimizar consultas SQL** lentas visando eficiência (menor custo de CPU) e performance, é importante
atentar-se para o plano de execução gerado pelo banco de dados.

O plano de execução nada mais é do que uma sequência de passos que serão executados pelo SGBD para 
executar uma consulta, ou seja, quais os tipos de processamento que serão realizados nas linhas de uma tabela ou em 
estruturas de índices. O planner/planejador precisa então fazer uso de estatísticas como o número total de registros da tabela, 
o número de blocos de disco ocupados por cada tabela e se há a presença de índices ou não.

Segue abaixo a estrutura do comando:

`EXPLAIN [
    ANALYZE [boolean]
    VERBOSE [boolean]
    COSTS [boolean]
    BUFFERS [boolean]
    TIMING [boolean]  
    SUMMARY [boolean]
    FORMAT { TEXT | XML | JSON | YAML }
] sql_statement;`


**Exemplos simples:**
```plpgsql
EXPLAIN ANALYZE VERBOSE -- plano mais detalhado
SELECT * FROM player ORDER BY row_version DESC LIMIT 100;
```

```plpgsql
EXPLAIN (FORMAT YAML) -- plano formatado em YAML
SELECT * FROM player ORDER BY row_version DESC LIMIT 100;
```

Aqui está um exemplo trivial apenas para mostrar como funciona:
```plpgsql
EXPLAIN SELECT * FROM player;
```

```
OUTPUT >> QUERY PLAN
-----------------------------------------------------------
Seq Scan on player (cost=0.00..458.00 rows=10000 width=244)
```

Como a consulta acima não possuí a cláusula WHERE, todas as linhas da tabela serão escaneadas, logo o planejador de
consultas do Postgres escolherá um plano de leitura simples e sequencial. Os números representados são, respectivamente:
- 0.00 - Custo estimado de start-up (início), esse é o tempo gasto antes do cursor começar a retornar as linhas;
- 458.00 - Custo estimado total, é declarado na suposição de que o nó (node) do plano é executado até a conclusão, 
ou seja, todas as linhas disponíveis são retornadas. Na prática, o nó pai pode parar antes mesmo de ler todas 
as linhas disponíveis (exemplo: cláusula LIMIT).
- 10000 - Número **estimado** de linhas retornadas. Novamente, supõe-se que o nó seja executado até a conclusão.
- 244 - Média do tamanho estimado de cada linha retornada (em bytes). É possível ver essas estatísticas na tabela
pg_stats, veja os [exemplos](https://www.plpgsql.org/docs/current/row-estimation-examples.html).

Uma prática tradicional é medir os custos em unidades de buscas de página no disco, ou seja, o parâmetro 
[seq_page_cost ](https://www.plpgsql.org/docs/current/runtime-config-query.html#GUC-SEQ-PAGE-COST)
é definido convencionalmente valendo 1.0 e os outros parâmetros de custo são definidos com base neste valor. 

É importante entender que o custo de um nó de nível superior (upper-level node) inclui o custo de todos os nós que 
estão abaixo (child nodes) dele.

Também é importante perceber que o custo reflete apenas coisas com as quais o planejador se preocupa -
o custo não considera o tempo gasto transmitindo as linhas para o cliente.

```
OUTPUT >> QUERY PLAN
-----------------------------------------------------------
Seq Scan on player (cost=0.00..458.00 rows=10000 width=244)
```

O custo de leitura de cada linha é determinado pelo parâmetro abaixo:

[cpu_tuple_cost](https://www.plpgsql.org/docs/current/runtime-config-query.html#GUC-CPU-TUPLE-COST) (ponto flutuante)
= 0.01 é o valor padrão

Você descobrirá também que a tabela tem 358 páginas no disco e 10.000 linhas.
Esses números são derivados de maneira muito direta. Para verificar, basta executar:
```plpgsql
SELECT relpages AS disk_pages, -- páginas no disco
       reltuples AS num_of_rows --nº de linhas
FROM pg_class 
WHERE relname = 'player'; -- nome da tabela = jogador
```

O custo estimado total é calculado da seguinte forma:

`Custo Estimado = (disk_pages * seq_page_cost) + (num_of_rows * cpu_tuple_cost)`

Considerando que disk_pages = **358**, num_of_rows = **10000**, 
seq_page_cost = **1.0**
e cpu_tuple_cost = **0.01**, então:
```
Custo Estimado = (358 * 1.0) + (10000 * 0.01)
Custo Estimado = 458
```

Agora vamos adicionar um filtro na consulta:
```plpgsql
EXPLAIN SELECT * FROM player WHERE wins < 9000;
```
```
OUTPUT >> QUERY PLAN
----------------------------------------------------------
Seq Scan on player (cost=0.00..483.00 rows=7001 width=244)
    Filter: (wins < 9000)
```

Perceba que o plano mostra a clásula WHERE usando um algoritmo chamado "Filter" vinculado no node "Seq Scan", isso
significa que o nó principal executa a condição para cada linha escaneada e retorna somente aquelas que passam
na condição declarada, logicamente a quantidade de linhas foi reduzida de 10.000 para 7.001 por conta do nosso filtro.

No entanto, o nó principal ("Seq Scan") ainda terá que visitar as 10.000 linhas, logo o custo não diminuiu; na
realidade ele até subiu um pouco, aumentou 25 pontos 
(10000 * [cpu_operator_cost](https://www.plpgsql.org/docs/current/runtime-config-query.html#GUC-CPU-OPERATOR-COST)
= 10000 * 0.0025 para ser exato), esses 25 pontos refletem o tempo extra gasto pela CPU para checar a condição
(salary < 9000).

Ok, agora vamos tornar esta condição mais restritiva!

```plpgsql
EXPLAIN SELECT * FROM player WHERE wins < 100;
```
```
OUTPUT >> QUERY PLAN
---------------------------------------------------------------------------
Bitmap Heap Scan on player  (cost=5.07..229.20 rows=101 width=244)
    Recheck Cond: (wins < 100)
    ->  Bitmap Index Scan on player_wins (cost=0.00..5.04 rows=101 width=0)
        Index Cond: (wins < 100)
```

Veja que interessante, neste caso o planejador optou por um **caminho de 2 etapas** usando o mecanismo chamado "Bitmap":

* O plano filho visita um índice para localizar as **localizações das linhas** que correspondem à condição do índice 
e, em seguida, o nó do plano superior realmente (upper-level node) busca essas linhas da própria tabela.
* Claro, buscar linhas separadamente é muito mais caro do que lê-las sequencialmente, mas como nem todas as páginas 
da tabela precisam ser visitadas, isso ainda é muito **mais barato do que uma varredura sequencial**.
* A razão para usar 2 níveis de plano é que o nó do plano superior classifica os locais de linha identificados 
pelo índice em ordem física antes de lê-los, para minimizar o custo de buscas separadas. O "bitmap" mencionado 
nos nomes dos nós é o mecanismo que faz essa classificação.

Ok, entendido. Agora vamos adicionar mais uma condição:
```plpgsql
EXPLAIN SELECT * FROM player WHERE wins < 100 AND player_group = 'xyz';
```
```
OUTPUT >> QUERY PLAN
---------------------------------------------------------------------------
 Bitmap Heap Scan on player  (cost=5.04..229.43 rows=1 width=244)
   Recheck Cond: (wins < 100)
   Filter: (player_group = 'xyz'::name)
   ->  Bitmap Index Scan on player_wins  (cost=0.00..5.04 rows=101 width=0)
         Index Cond: (wins < 100)
```

* A condição adicionada `player_group='xyz'` reduz a estimativa de contagem de linhas retornadas, mas não o custo porquê
ainda temos que iterar o mesmo conjunto de linhas anterior!
* Observe também que a cláusula `player_group='xyz'` não pode ser aplicada como uma condição de índice, 
pois esse índice já foi computado na coluna `wins`. 
* Em vez disso, é aplicado como um filtro nas linhas retornadas pelo índice. Assim, o custo aumentou ligeiramente 
para refletir essa verificação extra.

Em alguns casos, o planejador pode preferir um caminho mais "simples":
```plpgsql
EXPLAIN SELECT * FROM player WHERE wins = 42;
```
```
OUTPUT >> QUERY PLAN
---------------------------------------------------------------------------
Index Scan using tenk1_unique1 on tenk1  (cost=0.29..8.30 rows=1 width=244)
   Index Cond: (unique1 = 42)
```

Nesse tipo de plano, as linhas da tabela são buscadas na ordem do índice, o que faz dele um plano ainda mais custoso,
mas são tão poucas leituras que esse custo extra de encontrar as localizações das linhas não vale a pena. 
* Adicionar uma clásula `ORDER BY` neste caso não irá mudar o plano de execução;
* Você verá esse tipo de plano com mais frequência para consultas que buscam apenas uma única linha.

**IMPORTANTE:** Para ter um plano de execução mais preciso execute com `EXPLAIN ANALYZE`, pode levar mais tempo
pois vai executar o statement antes de tudo.

* Lembre-se de que, como EXPLAIN ANALYZE realmente executa a consulta, quaisquer efeitos colaterais ocorrerão 
normalmente. Se você deseja analisar uma consulta de modificação de dados sem alterar suas tabelas, pode reverter 
o comando posteriormente, por exemplo:


```plpgsql
BEGIN;

EXPLAIN ANALYZE 
    UPDATE tenk1 SET hundred = hundred + 1 
    WHERE unique1 < 100;


/*
OUTPUT >> QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------
Update on tenk1  (cost=5.08..230.08 rows=0 width=0) (actual time=3.791..3.792 rows=0 loops=1)
    ->  Bitmap Heap Scan on tenk1  (cost=5.08..230.08 rows=102 width=10) (actual time=0.069..0.513 rows=100 loops=1)
        Recheck Cond: (unique1 < 100)
        Heap Blocks: exact=90
        ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.05 rows=102 width=0) (actual time=0.036..0.037 rows=300 loops=1)
            Index Cond: (unique1 < 100)
Planning Time: 0.113 ms
Execution Time: 3.850 ms
*/

ROLLBACK;
```

**Referências:**

https://www.plpgsql.org/docs/current/sql-explain.html
https://www.plpgsqltutorial.com/plpgsql-tutorial/plpgsql-explain/
https://www.plpgsql.org/docs/current/runtime-config-query.html
