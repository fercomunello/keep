# PL/PGSQL - Bitcoin Trade Script (exemplo)

Processamento e consulta de Trades de Bitcoin usando features avançadas do PostgreSQL.

Alterando o timezone da sessão para **'America/Sao_Paulo'**.
```postgresql
SHOW TIMEZONE;
SELECT name FROM pg_timezone_names WHERE name LIKE '%America%';
SET TIMEZONE = 'America/Sao_Paulo';
```

Criando a tabela referente as **Negociações de Bitcoin**:
**Campos:** código, descrição, valor fiduciário (R$), valor em satoshis (é um inteiro que representa
a menor unidade de 1.0 BTC, pois cada um é dividido em 100.000.000 de sats),
cotação (último preço negociado na corretora, pode ser consultado na API Mercado Bitcoin),
código do trader/usuário, tipo da oferta (Compra / Venda), data e hora.
```postgresql
CREATE TYPE offer_type AS ENUM ('BUY', 'SELL'); -- Compra e Venda

CREATE TABLE bitcoin_trade (
    id SERIAL UNIQUE NOT NULL,
    description TEXT,
    value DECIMAL(16, 2),
    satoshis BIGINT,
    last_price DECIMAL(16, 2),
    trader_id INTEGER NOT NULL,
    offer_type offer_type NOT NULL,
    date_time TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT pk_trade_id PRIMARY KEY (id)
);
```

Inserindo 10 milhões de trades na tabela.
```postgresql
-- ################## TESTES ##################   
DO $$
    BEGIN
        FOR i IN 1..10000000 LOOP
                INSERT INTO bitcoin_trade (description, value, satoshis, last_price, trader_id, offer_type)
                VALUES ('TESTE', 500.50::DECIMAL, 200200, 250000.00::DECIMAL, 10, 'BUY');
            END LOOP;
    END;
$$;
```

Plano de execução da consulta de saldo para o user/trader com 10 milhões de trades.
Esta consulta possui um custo elevado mesmo que seja otimizada, vai depender do contexto
e do volume de registros lidos e calculados dentro das CTEs.
```postgresql
EXPLAIN ANALYZE
    WITH
        summary (total_spent, satoshis, avg_price) AS (
            SELECT
                SUM(trade.value) AS total_spent,
                SUM(trade.satoshis) AS satoshis,
                AVG(trade.last_price) AS avg_price
            FROM
                bitcoin_trade trade
            WHERE
                (trade.trader_id = 10)
                AND (trade.offer_type = 'BUY')
        ),
        sell_summary (total_sold, satoshis_sold) AS (
            SELECT
                SUM(trade.value) AS total_sold,
                SUM(trade.satoshis) AS satoshis_sold
            FROM
                bitcoin_trade trade
            WHERE
                (trade.trader_id = 10)
                AND (trade.offer_type = 'SELL')
        )
    SELECT
        summary.total_spent - sell_summary.total_sold AS total_spent,
        summary.satoshis - sell_summary.satoshis_sold AS satoshis_balance,
        summary.avg_price AS average_price
    FROM
        summary, sell_summary
    ;
```

### **1ª estratégia de otimização**

1) Cria índice para otimizar a consulta anterior. Com os índices, melhora um pouco, leva metade do tempo 
para executar a mesma consulta, mas ainda é uma consulta com custo muito alto e levará alguns segundos 
para calcular os valores.

2) Cria a view materializada da consulta anterior.

3) Ao criar/atualizar a view materializada, a consulta pela view é muito rápida, custo baixo.

4) No entanto, após inserir um registro na tabela, sempre teremos que atualizar a view materializada, 
e esse processo pode demorar muito dependendo da quantidade de usuários e trades cadastrados...

O trade-off aqui é otimizar ao custo de inconsistência no cálculo de resumo das negociações, pode ser disparado
um job no lado da aplicação para atualizar a view materializada de hora em hora por exemplo, mas o trade-off
continua existindo.

```postgresql
CREATE INDEX idx_bitcoin_trade ON bitcoin_trade (trader_id);

CREATE MATERIALIZED VIEW bitcoin_trade_summary_view (
    trader_id, total_spent, satoshis_balance, average_price) AS
WITH
    summary (trader_id, total_spent, satoshis, avg_price) AS (
        SELECT
            trade.trader_id,
            SUM(trade.value) AS total_spent,
            SUM(trade.satoshis) AS satoshis,
            AVG(trade.last_price) AS avg_price
        FROM
            bitcoin_trade trade
        WHERE
            (trade.trader_id = 10)
            AND (trade.offer_type = 'BUY')
        GROUP BY
            trade.trader_id
    ),
    sell_summary (total_sold, satoshis_sold) AS (
        SELECT
            SUM(trade.value) AS total_sold,
            SUM(trade.satoshis) AS satoshis_sold
        FROM
            bitcoin_trade trade
        WHERE
            (trade.trader_id = 10)
            AND (trade.offer_type = 'SELL')
    )
    SELECT
        summary.trader_id,
        summary.total_spent - sell_summary.total_sold AS total_spent,
        summary.satoshis - sell_summary.satoshis_sold AS satoshis_balance,
        summary.avg_price AS average_price
    FROM
        summary, sell_summary
;

EXPLAIN ANALYZE
    SELECT * FROM bitcoin_trade_summary_view WHERE trader_id = 10;

REFRESH MATERIALIZED VIEW bitcoin_trade_summary_view;
```

### **1ª estratégia de otimização:** Extraindo a melhor performance!

1) Deletando todos os registros anteriores.
2) Criando tabela referente ao Resumo das Negociações de Bitcoin:
   => Campos: código do trader, valor fiduciário investido (R$),
   saldo em satoshis, preço médio de compra (R$).
3) Function responsável por calcular o resumo do trader após ocorrer uma
trigger operation de INSERT/UPDATE/DELETE na tabela das negociações (bitcoin_trade).
- Com o resumo do trader é possível saber o preço médio de compra (average price),
     saldo em satoshis (para saber quanto é na unidade BTC (8 casas decimais após a vírgula),
     basta dividir por 100.000.000 e multiplicar pela cotação atual em R$ por exemplo).
- Docs: https://www.postgresql.org/docs/9.2/plpgsql-trigger.html
4) Realizando testes para verificar se os cálculos estão corretos.

```postgresql
DELETE FROM bitcoin_trade;

CREATE TABLE bitcoin_trader_summary (
    trader_id INTEGER UNIQUE NOT NULL,
    total_spent DECIMAL(16, 2) NOT NULL,
    satoshis_balance BIGINT NOT NULL,
    avg_price DECIMAL(16, 2) NOT NULL
);

CREATE OR REPLACE FUNCTION process_bitcoin_trader_summary() RETURNS TRIGGER AS $$
DECLARE
    v_trader_id INTEGER;
    v_total_spent DECIMAL(16, 2) DEFAULT 0;
    v_satoshis_balance BIGINT DEFAULT 0;
    v_avg_price DECIMAL(16, 2) DEFAULT 0;
BEGIN
    IF (TG_OP = 'DELETE') THEN
        v_trader_id := OLD.trader_id;
    ELSE
        v_trader_id := NEW.trader_id;
    END IF;

    SELECT total_spent, satoshis_balance, avg_price
        FROM bitcoin_trader_summary 
            WHERE (trader_id = v_trader_id)
    INTO v_total_spent, v_satoshis_balance, v_avg_price;

    RAISE NOTICE '% % %', v_total_spent, v_satoshis_balance, v_avg_price;

    IF (NOT FOUND) THEN
        IF (TG_OP = 'INSERT') THEN
            INSERT INTO bitcoin_trader_summary
            SELECT NEW.trader_id, NEW.value, NEW.satoshis,
                   ROUND(NEW.value / (NEW.satoshis::DECIMAL / 100000000::DECIMAL));
            RETURN NEW;
        END IF;
    END IF;

    IF (TG_OP = 'INSERT') THEN
        IF (NEW.offer_type = 'BUY') THEN
            v_total_spent := v_total_spent + NEW.value;
            v_satoshis_balance := v_satoshis_balance + NEW.satoshis;
        ELSIF (NEW.offer_type = 'SELL') THEN
            v_total_spent := v_total_spent - NEW.value;
            v_satoshis_balance := v_satoshis_balance - NEW.satoshis;
        END IF;
    ELSIF (TG_OP = 'UPDATE') THEN
        IF (NEW.offer_type = 'BUY') THEN
            v_total_spent := (v_total_spent - OLD.value) + NEW.value;
            v_satoshis_balance := (v_satoshis_balance - OLD.satoshis) + NEW.satoshis;
        ELSIF (NEW.offer_type = 'SELL') THEN
            v_total_spent := (v_total_spent + OLD.value) - NEW.value;
            v_satoshis_balance := (v_satoshis_balance + OLD.satoshis) - NEW.satoshis;
        END IF;
    ELSIF (TG_OP = 'DELETE') THEN
        IF (OLD.offer_type = 'BUY') THEN
            v_total_spent := v_total_spent - OLD.value;
            v_satoshis_balance := v_satoshis_balance - OLD.satoshis;
        ELSIF (OLD.offer_type = 'SELL') THEN
            v_total_spent := v_total_spent + OLD.value;
            v_satoshis_balance := v_satoshis_balance + OLD.satoshis;
        END IF;
    END IF;

    IF (v_total_spent > 0) THEN
        v_avg_price := ROUND(v_total_spent / (v_satoshis_balance::DECIMAL / 100000000::DECIMAL));
    ELSE
        v_avg_price := 0;
    END IF;

    UPDATE bitcoin_trader_summary SET
        total_spent = v_total_spent,
        satoshis_balance = v_satoshis_balance,
        avg_price = v_avg_price
    WHERE
        trader_id = v_trader_id;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger vinculada a tabela bitcoin_trade.
CREATE TRIGGER bitcoin_trade_trigger
    AFTER INSERT OR UPDATE OR DELETE ON bitcoin_trade
    FOR EACH ROW EXECUTE FUNCTION process_bitcoin_trader_summary();

INSERT INTO bitcoin_trade (description, value, satoshis, last_price, trader_id, offer_type)
VALUES ('Compra de BTC', 800::DECIMAL, 2000000, 40000::DECIMAL, 1, 'BUY');

INSERT INTO bitcoin_trade (description, value, satoshis, last_price, trader_id, offer_type)
VALUES ('Compra de BTC', 225.40::DECIMAL, 354531, 63577::DECIMAL, 1, 'BUY');

INSERT INTO bitcoin_trade (description, value, satoshis, last_price, trader_id, offer_type)
VALUES ('Compra de BTC', 390::DECIMAL, 651194, 59890::DECIMAL, 1, 'BUY');

INSERT INTO bitcoin_trade (description, value, satoshis, last_price, trader_id, offer_type)
VALUES ('Venda de BTC', 188.97::DECIMAL, 269957, 70000::DECIMAL, 1, 'SELL');

DELETE FROM bitcoin_trade WHERE id = 1;
DELETE FROM bitcoin_trade WHERE id = 2;
DELETE FROM bitcoin_trade WHERE id = 3;
DELETE FROM bitcoin_trade WHERE id = 4;

-- É importante sempre fazer a conversão de inteiro em decimal/numeric através dos dois pontos,
-- outra forma de fazer é utilizando a função CAST().
UPDATE bitcoin_trade SET last_price = 100000::DECIMAL, satoshis = 390000 WHERE id = 3;
UPDATE bitcoin_trade SET value = 877::DECIMAL, satoshis = 2192500 WHERE id = 1;

SELECT * FROM bitcoin_trade;
SELECT * FROM bitcoin_trader_summary;

SELECT * FROM bitcoin_trade_id_seq;

SELECT * FROM bitcoin_trade;
SELECT * FROM bitcoin_trader_summary;    

```

5) Inserindo 10 milhões de trades novamente.
```postgresql
DO $$
    BEGIN
        FOR i IN 1..10000000 LOOP
                INSERT INTO bitcoin_trade (description, value, satoshis, last_price, trader_id, offer_type)
                VALUES ('TESTE', 500.50::DECIMAL, 200200, 250000.00::DECIMAL, 10, 'BUY');
            END LOOP;
    END;
$$;
```

6) Rodando a consulta a tabela de resumo dos trades e verificando o plano de execução.

### **Verificando a performance**
A consulta abaixo irá executar mais rápido que todas as outras, com o menor custo de execução, pois todos os valores
são processados sob-demanda pela trigger.

E claro, o único trade-off aqui é a complexidade para dar manutenção em lógica de negócio dentro de triggers, mas
dependendo do contexto talvez seja a única solução que atenda o requisito de performance. Talvez seja ainda mais
seguro mover essa function para uma stored procedure que garante um [lock](https://www.postgresql.org/docs/current/explicit-locking.html) 
no registro (FOR UPDATE) dentro de uma transação, então fica aqui um ponto a ser melhorado.

```postgresql
EXPLAIN ANALYZE
    SELECT total_spent, satoshis_balance, avg_price
        FROM bitcoin_trader_summary WHERE trader_id = 10;
```

### **Pontos importantes**
Vale lembrar que, em cenários de aplicações distribuídas que utilizam 
[middlewares](https://www.redhat.com/pt-br/topics/middleware/what-is-middleware) como
brokers de mensageria ([ActiveMQ](https://activemq.apache.org/components/artemis/), 
[Kafka](https://kafka.apache.org/)),
não é só possível mas também recomendado mover toda a lógica da trigger para consumidores de mensagens 
no lado da aplicação, dessa forma, a cada trade realizado, um evento é emitido por um producer 
e consumido de modo assíncrono por um consumer.

Além da garantia do desacoplamento, temos a garantia de que essas mensagens (ou eventos) serão persistidos no
cluster do middleware e podem ser consumidos novamente em caso de falha no lado da aplicação. Se não tivéssemos
nenhum middleware poderíamos usar o próprio Postgres para produzir e consumir essas mensagens via pub/sub usando
[LISTEN](https://www.postgresql.org/docs/current/sql-listen.html) e 
[NOTIFY](https://www.postgresql.org/docs/current/sql-notify.html) ou rodando um job que lê os últimos trades 
não processados e garante um lock para executar a lógica
de negócio e gravar o resumo do trader na tabela, o conceito é o mesmo.

No geral, triggers (gatilhos) são mais usados para detectar alterações em determinados campos de uma tabela e para
fins de auditoria, é mais recomendado que a lógica de negócio (boa parte dela) fique concentrada na aplicação e não
no banco de dados, mas novamente, vai depender do contexto e da criticidade do negócio.

> There is no "one size fits all" solution =)