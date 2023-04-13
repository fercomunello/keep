# PL/PGSQL - Exemplos

Scripts e exemplos dedicados para fins de estudo do banco de dados Postgres.

### Populando tabela com registros
```plpgsql
CREATE TABLE instrutor (
    id SERIAL PRIMARY KEY,
    nome VARCHAR(255) NOT NULL,
    salario DECIMAL(10, 2)
);

INSERT INTO instrutor(nome, salario) VALUES ('Instrutor 1', 100);
INSERT INTO instrutor(nome, salario) VALUES ('Instrutor 2', 200);
INSERT INTO instrutor(nome, salario) VALUES ('Instrutor 3', 300);
INSERT INTO instrutor(nome, salario) VALUES ('Instrutor 4', 400);
INSERT INTO instrutor(nome, salario) VALUES ('Instrutor 5', 500);

SELECT * FROM instrutor;
```

### Criando functions
Funções que recebem uma entrada, processam e devolvem uma saída.

````plpgsql
CREATE FUNCTION dobro_do_salario(instrutor) RETURNS DECIMAL AS $$
	SELECT $1.salario * 2 AS dobro;
$$ LANGUAGE SQL;

SELECT instrutor.nome, dobro_do_salario(instrutor.*) AS desejo FROM instrutor;

CREATE OR REPLACE FUNCTION cria_instrutor_falso() RETURNS instrutor AS $$
	SELECT 22::INTEGER AS id, 'Nome falso' AS nome, 200::DECIMAL AS salario;
$$ LANGUAGE SQL;

SELECT instrutor.nome, dobro_do_salario(instrutor.*) AS desejo FROM instrutor;

SELECT cria_instrutor_falso();
SELECT * FROM cria_instrutor_falso();

DROP FUNCTION instrutores_bem_pagos;

CREATE FUNCTION instrutores_bem_pagos(valor_salario DECIMAL) RETURNS instrutor AS $$
	SELECT * FROM instrutor WHERE salario > valor_salario;
$$ LANGUAGE SQL;

DROP FUNCTION instrutores_bem_pagos;

CREATE FUNCTION instrutores_bem_pagos(valor_salario DECIMAL) RETURNS TABLE (id INTEGER, nome VARCHAR, salario DECIMAL) AS $$
	SELECT * FROM instrutor WHERE salario > valor_salario;
$$ LANGUAGE SQL;

DROP FUNCTION instrutores_bem_pagos;

CREATE FUNCTION instrutores_bem_pagos(valor_salario DECIMAL) RETURNS SETOF instrutor AS $$
	SELECT * FROM instrutor WHERE salario > valor_salario;
$$ LANGUAGE SQL;

DROP FUNCTION instrutores_bem_pagos;

CREATE FUNCTION instrutores_bem_pagos(valor_salario DECIMAL) RETURNS SETOF record AS $$
	SELECT * FROM instrutor WHERE salario > valor_salario;
$$ LANGUAGE SQL;

SELECT * FROM instrutores_bem_pagos(300);
````

### Typed Functions
````plpgsql
CREATE TYPE dois_valores AS (soma INTEGER, produto INTEGER);

CREATE FUNCTION soma_e_produto (
	IN numero_1 INTEGER, IN numero_2 INTEGER
) RETURNS dois_valores AS $$
	SELECT numero_1 + numero_2 AS soma, numero_1 * numero_2 AS produto;
$$ LANGUAGE SQL;

SELECT * FROM soma_e_produto(3, 3);
DROP FUNCTION instrutores_bem_pagos;

CREATE FUNCTION instrutores_bem_pagos(
	valor_salario DECIMAL, OUT nome VARCHAR, OUT salario DECIMAL) RETURNS SETOF record AS $$
	SELECT nome, salario FROM instrutor WHERE salario > valor_salario;
$$ LANGUAGE SQL;

SELECT * FROM instrutores_bem_pagos(300);
````

### PL/PGSQL
````plpgsql
DROP FUNCTION data_atual;

CREATE OR REPLACE FUNCTION data_atual() RETURNS TIMESTAMP AS $$
	BEGIN
		RETURN NOW()::TIMESTAMP;
	END;
$$ LANGUAGE plpgsql;

SELECT data_atual();

CREATE OR REPLACE FUNCTION primeira_pl() RETURNS INTEGER AS $$
	DECLARE
		primeira_variavel INTEGER DEFAULT 3;
	BEGIN
		primeira_variavel := primeira_variavel * 2;
		
		BEGIN
			primeira_variavel := 7;
		END;
		
		RETURN primeira_variavel;
	END;
$$ LANGUAGE plpgsql;

SELECT primeira_pl();


CREATE OR REPLACE FUNCTION cria_instrutor_falso() RETURNS instrutor AS $$
	DECLARE
		retorno instrutor;
	BEGIN
		--RETURN ROW(22::INTEGER, 'Nome falso', 200::DECIMAL)::instrutor;
		SELECT 22::INTEGER, 'Nome falso', 200::DECIMAL INTO retorno;
		
		RETURN retorno;
	END;
$$ LANGUAGE plpgsql;

SELECT * FROM cria_instrutor_falso();

DROP FUNCTION instrutores_bem_pagos;
CREATE FUNCTION instrutores_bem_pagos(valor_salario DECIMAL) RETURNS SETOF instrutor AS $$
	BEGIN
		RETURN QUERY SELECT * FROM instrutor WHERE salario > valor_salario;
	END;
$$ LANGUAGE plpgsql;

SELECT * FROM instrutores_bem_pagos(300);

DROP FUNCTION soma_e_produto;
CREATE FUNCTION soma_e_produto (IN numero_1 INTEGER, IN numero_2 INTEGER) RETURNS dois_valores AS $$
	BEGIN
		RETURN ROW(numero_1 + numero_2, numero_1 * numero_2)::dois_valores;
	END;
$$ LANGUAGE plpgsql;

SELECT * FROM soma_e_produto(3, 9);
````

### PL/PGSQL - Condicionais
````plpgsql
CREATE FUNCTION salario_ok(instrutor instrutor) RETURNS VARCHAR AS $$
	BEGIN
		IF (instrutor.salario > 200) THEN
			RETURN 'Salarío está ok.';
		ELSE
			RETURN 'Salário pode aumentar.';
		END IF;
	END;
$$ LANGUAGE plpgsql;

SELECT instrutor.nome, salario_ok(instrutor.*) FROM instrutor;

CREATE OR REPLACE FUNCTION salario_ok_2(id_instrutor INTEGER) RETURNS VARCHAR AS $$
	DECLARE
		var_instrutor instrutor;
	BEGIN
		SELECT * FROM instrutor WHERE instrutor.id = id_instrutor INTO var_instrutor;
		
		IF (var_instrutor.salario > 200) THEN
			RETURN 'Salarío está ok.';
		ELSE
			RETURN 'Salário pode aumentar.';
		END IF;
	END;
$$ LANGUAGE plpgsql;

SELECT instrutor.nome, salario_ok_2(instrutor.id) FROM instrutor;

CREATE OR REPLACE FUNCTION salario_ok_3(instrutor instrutor) RETURNS VARCHAR AS $$
	BEGIN
		/*IF (instrutor.salario > 300) THEN
			RETURN 'Salarío está ok.';
		ELSEIF (instrutor.salario = 300) THEN
			RETURN 'Salário pode aumentar.';
		ELSE 
			RETURN 'Salário está defasado.';
		END IF;*/
		
		/*CASE
			WHEN instrutor.salario = 100 THEN
				RETURN 'Salário muito baixo.'
			WHEN instrutor.salario = 200 THEN
				RETURN 'Salário baixo.'
			WHEN instrutor.salario = 300 THEN
				RETURN 'Salário ok.'
			ELSE 
				RETURN 'Salário ótimo.'
		END CASE;*/
		
		CASE instrutor.salario
			WHEN 100 THEN
				RETURN 'Salário muito baixo.';
			WHEN 200 THEN
				RETURN 'Salário baixo.';
			WHEN 300 THEN
				RETURN 'Salário ok.';
			ELSE 
				RETURN 'Salário ótimo.';
		END CASE;
	END;
$$ LANGUAGE plpgsql;

SELECT instrutor.nome, salario_ok_3(instrutor.*) FROM instrutor;
````

### PL/PGSQL - Loops
````plpgsql
DROP FUNCTION tabuada;

CREATE FUNCTION tabuada(numero INTEGER) RETURNS SETOF VARCHAR AS $$
	DECLARE
		multiplicador INTEGER DEFAULT 1;
	BEGIN
		LOOP
			--RETURN NEXT numero || ' x ' || multiplicador || ' = ' || (numero * multiplicador);
			RETURN NEXT CONCAT(numero, ' x ', multiplicador, ' = ', numero * multiplicador);
			multiplicador := multiplicador + 1;
			EXIT WHEN multiplicador = 10;
		END LOOP;
	END;
$$ LANGUAGE plpgsql;

SELECT tabuada(9);


CREATE FUNCTION tabuada_ex_2(numero INTEGER) RETURNS SETOF VARCHAR AS $$
	DECLARE
		multiplicador INTEGER DEFAULT 1;
	BEGIN
		WHILE multiplicador < 10 LOOP
			RETURN NEXT CONCAT(numero, ' x ', multiplicador, ' = ', numero * multiplicador);
			multiplicador := multiplicador + 1;
		END LOOP;
	END;
$$ LANGUAGE plpgsql;

SELECT tabuada_ex_2(9);

CREATE FUNCTION tabuada_ex_3(numero INTEGER) RETURNS SETOF VARCHAR AS $$
	DECLARE
		multiplicador INTEGER DEFAULT 1;
	BEGIN
		FOR multiplicador IN 1..9 LOOP
			RETURN NEXT CONCAT(numero, ' x ', multiplicador, ' = ', numero * multiplicador);
		END LOOP;
	END;
$$ LANGUAGE plpgsql;

SELECT tabuada_ex_3(9);

DROP FUNCTION instrutor_com_salario;
CREATE FUNCTION instrutor_com_salario(OUT nome VARCHAR, OUT salario_ok VARCHAR) RETURNS SETOF record AS $$
	DECLARE
		instrutor instrutor;
	BEGIN
		FOR instrutor IN SELECT * FROM instrutor LOOP
			nome := instrutor.nome;
			salario_ok := salario_ok_3(instrutor);
			RETURN NEXT;
		END LOOP;
	END;
$$ LANGUAGE plpgsql;

SELECT * FROM instrutor_com_salario();
````

### Exemplo avançado

Neste exemplo foi utilizado transactions (commit/rollback), functions, cursors, triggers e logs.

````plpgsql
BEGIN
	CREATE TABLE aluno (
		id SERIAL PRIMARY KEY,
		primeiro_nome VARCHAR(255) NOT NULL,
		ultimo_nome VARCHAR(255) NOT NULL,
		data_nascimento DATE NOT NULL
	);

	CREATE TABLE categoria (
		id SERIAL PRIMARY KEY,
		nome VARCHAR(255) NOT NULL UNIQUE
	);

	CREATE TABLE curso (
		id SERIAL PRIMARY KEY,
		nome VARCHAR(255) NOT NULL,
		categoria_id INTEGER NOT NULL REFERENCES categoria(id)
	);

	CREATE TABLE aluno_curso (
		aluno_id INTEGER NOT NULL REFERENCES aluno(id),
		curso_id INTEGER NOT NULL REFERENCES curso(id),
		PRIMARY KEY (aluno_id, curso_id)
	);
COMMIT;

CREATE FUNCTION cria_curso(nome_curso VARCHAR, nome_categoria VARCHAR) RETURNS void AS $$
	DECLARE
		v_categoria_id INTEGER;
	BEGIN
		SELECT id FROM categoria WHERE nome = nome_categoria INTO v_categoria_id;
		
		IF NOT FOUND THEN
			INSERT INTO categoria (nome) VALUES (nome_categoria) RETURNING id INTO v_categoria_id;
		END IF;
		
		INSERT INTO curso (nome, categoria_id) VALUES (nome_curso, v_categoria_id);
	END;
$$ LANGUAGE plpgsql;

SELECT * FROM cria_curso('PHP', 'Programação');
SELECT * FROM cria_curso('Java', 'Programação');
SELECT * FROM cria_curso('Postgres', 'Data Science');

SELECT * FROM categoria;
SELECT * FROM curso;

/**
* Inserir instrutores com salários
* Se o salário for maior do que a média, salvar log.
* Salvar outro log dizendo que fulano recebe mais do que X% da grade de instrutores.
*/

CREATE TABLE log_instrutores (
	id SERIAL PRIMARY KEY,
	informacao VARCHAR(255),
	momento_criacao TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE OR REPLACE FUNCTION cria_instrutor (nome_instrutor VARCHAR, salario_instrutor DECIMAL) RETURNS void AS $$
	DECLARE
		id_instrutor_inserido INTEGER;
		media_salarial DECIMAL;
		instrutores_recebem_menos INTEGER DEFAULT 0;
		total_instrutores INTEGER DEFAULT 0;
		salario DECIMAL;
		percentual DECIMAL;
	BEGIN
		INSERT INTO instrutor (nome, salario) VALUES (nome_instrutor, salario_instrutor) 
		RETURNING id INTO id_instrutor_inserido;
		
		SELECT AVG(instrutor.salario) FROM instrutor WHERE id <> id_instrutor_inserido INTO media_salarial;
		
		IF (salario_instrutor > media_salarial) THEN
			INSERT INTO log_instrutores (informacao) VALUES (nome_instrutor || ' recebe acima da média');
		END IF;
		
		FOR salario IN SELECT instrutor.salario FROM instrutor WHERE id <> id_instrutor_inserido LOOP
			total_instrutores := total_instrutores + 1;
			
			IF (salario_instrutor > salario) THEN
				instrutores_recebem_menos := instrutores_recebem_menos + 1;
			END IF;
		END LOOP;
		
		percentual = (instrutores_recebem_menos::DECIMAL / total_instrutores::DECIMAL) * 100;
		
		INSERT INTO log_instrutores (informacao) VALUES (
			nome_instrutor || ' recebe mais do que ' || trunc(percentual, 2) || '% da grade de instrutores');
	END;
$$ LANGUAGE plpgsql;

SELECT * FROM instrutor;
SELECT * FROM log_instrutores;
SELECT cria_instrutor('Fulana de tal', 1000);
SELECT cria_instrutor('Outra instrutora', 400);


DELETE FROM instrutor;
DELETE FROM log_instrutores;

INSERT INTO instrutor(nome, salario) VALUES ('Vinicius Dias', 100);
INSERT INTO instrutor(nome, salario) VALUES ('Diogo Mascarenhas', 200);
INSERT INTO instrutor(nome, salario) VALUES ('Nico Steppat', 300);
INSERT INTO instrutor(nome, salario) VALUES ('Juliana', 400);
INSERT INTO instrutor(nome, salario) VALUES ('Priscila', 500);

CREATE OR REPLACE FUNCTION log_instrutor() RETURNS TRIGGER AS $$
	DECLARE
		id_instrutor_inserido INTEGER;
		media_salarial DECIMAL;
		instrutores_recebem_menos INTEGER DEFAULT 0;
		total_instrutores INTEGER DEFAULT 0;
		salario DECIMAL;
		percentual DECIMAL(5, 2);
	BEGIN
		SELECT AVG(instrutor.salario) FROM instrutor WHERE id <> NEW.id INTO media_salarial;
		
		IF (NEW.salario > media_salarial) THEN
			INSERT INTO log_instrutores (informacao) VALUES (NEW.nome || ' recebe acima da média');
		END IF;
		
		FOR salario IN SELECT instrutor.salario FROM instrutor WHERE id <> NEW.id LOOP
			total_instrutores := total_instrutores + 1;
			
			IF (NEW.salario > salario) THEN
				instrutores_recebem_menos := instrutores_recebem_menos + 1;
			END IF;
		END LOOP;
		
		percentual = (instrutores_recebem_menos::DECIMAL / total_instrutores::DECIMAL) * 100;
		
		INSERT INTO log_instrutores (informacao) VALUES (
			NEW.nome || ' recebe mais do que ' || trunc(percentual, 2) || '% da grade de instrutores');
			
		RETURN NULL;
	END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_log_instrutores AFTER INSERT ON instrutor
FOR EACH ROW EXECUTE FUNCTION log_instrutor();

INSERT INTO instrutor(nome, salario) VALUES ('Pedro', 1000000);



DROP FUNCTION cria_instrutor;
DROP TRIGGER trigger_log_instrutores ON instrutor;

CREATE OR REPLACE FUNCTION cria_instrutor() RETURNS TRIGGER AS $$
	DECLARE
		media_salarial DECIMAL;
		instrutores_recebem_menos INTEGER DEFAULT 0;
		total_instrutores INTEGER DEFAULT 0;
		salario DECIMAL;
		percentual DECIMAL(5, 2);
		logs_inseridos INTEGER;
	BEGIN
		SELECT AVG(instrutor.salario) FROM instrutor WHERE instrutor.id <> NEW.id INTO media_salarial;
		
		IF (NEW.salario > media_salarial) THEN
			INSERT INTO log_instrutores (informacao) VALUES (NEW.nome || ' recebe acima da média');
			/*GET DIAGNOSTICS logs_inseridos = ROW_COUNT;
			IF  (logs_inseridos > 1) THEN
				RAISE EXCEPTION 'Não é possível inserir mais de um log ao mesmo tempo.';
			END IF;*/
		END IF;

		FOR salario IN SELECT instrutor.salario FROM instrutor WHERE instrutor.id <> NEW.id LOOP
			total_instrutores := total_instrutores + 1;
			-- Debug
			RAISE NOTICE 'Salário inserido: % Salário do instrutor existente: %', NEW.salario, salario;
			IF (NEW.salario > salario) THEN
				instrutores_recebem_menos := instrutores_recebem_menos + 1;
			END IF;
		END LOOP;

		percentual := instrutores_recebem_menos::DECIMAL / total_instrutores::DECIMAL * 100;
		ASSERT percentual < 100::DECIMAL, 'Instrutores novos não podem receber mais do que todos os antigos.';

		INSERT INTO log_instrutores (informacao) VALUES (
			NEW.nome || ' recebe mais do que ' || trunc(percentual, 2) || '% da grade de instrutores');
			
		RETURN NEW;
	END;
$$ LANGUAGE plpgsql;

DROP TRIGGER cria_log_instrutores ON instrutor;

CREATE TRIGGER cria_log_instrutores 
	BEFORE INSERT ON instrutor
	FOR EACH ROW 
	EXECUTE FUNCTION cria_instrutor();

SELECT * FROM instrutor;
SELECT * FROM log_instrutores;

DELETE FROM log_instrutores;

INSERT INTO instrutor (nome, salario) VALUES ('Pessoa 1', 1000);
INSERT INTO instrutor (nome, salario) VALUES ('Pessoa 2', 1600);
INSERT INTO instrutor (nome, salario) VALUES ('Pessoa 3', 444);
INSERT INTO instrutor (nome, salario) VALUES ('Pessoa 4', 600);

BEGIN
INSERT INTO instrutor (nome, salario) VALUES ('Pessoa 5', 980);
ROLLBACK;

INSERT INTO instrutor (nome, salario) VALUES ('Pessoa 5', 3000);
INSERT INTO instrutor (nome, salario) VALUES ('Pessoa 6', 1000000);

INSERT INTO instrutor (nome, salario) VALUES ('TESTE', 10000000);

DELETE FROM log_instrutores;

CREATE OR REPLACE FUNCTION instrutores_internos(id_instrutor INTEGER) RETURNS refcursor AS $$
	DECLARE
		refcursor_salarios refcursor;
		-- Exemplo 2: cursor_salarios CURSOR FOR 
		-- SELECT instrutor.salario FROM instrutor WHERE id <> id_instrutor AND salario > 0;
		
		--salario DECIMAL;
	BEGIN
		-- Exemplo 1: RETURN QUERY SELECT instrutor.salario FROM instrutor WHERE id <> id_instrutor AND salario > 0;
		-- SCROLL: permite voltar para registros anteriores
		-- NO SCROLL: query normal, só vai para frente
		
		-- Exemplo 3:
		OPEN refcursor_salarios FOR SELECT instrutor.salario 
									FROM instrutor 
									WHERE instrutor.id <> id_instrutor 
									AND instrutor.salario > 0;
									
		-- Movendo o cursor e armazenando o registro em mem.
		/*FETCH LAST FROM refcursor_salarios INTO salario;
		FETCH NEXT FROM refcursor_salarios INTO salario;
		FETCH PRIOR FROM refcursor_salarios INTO salario;
		FETCH FIRST FROM refcursor_salarios INTO salario;*/
		
		-- Só movendo o cursor
		/*MOVE LAST FROM refcursor_salarios;
		MOVE NEXT FROM refcursor_salarios;
		MOVE PRIOR FROM refcursor_salarios;
		MOVE FIRST FROM refcursor_salarios INTO salario;*/

		RETURN refcursor_salarios;
	END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION cria_instrutor() RETURNS TRIGGER AS $$
	DECLARE
		media_salarial DECIMAL;
		instrutores_recebem_menos INTEGER DEFAULT 0;
		total_instrutores INTEGER DEFAULT 0;
		salario DECIMAL;
		percentual DECIMAL(5, 2);
		logs_inseridos INTEGER;
		refcursor_salarios refcursor;
	BEGIN
		SELECT AVG(instrutor.salario) FROM instrutor WHERE instrutor.id <> NEW.id INTO media_salarial;
		
		IF (NEW.salario > media_salarial) THEN
			INSERT INTO log_instrutores (informacao) VALUES (NEW.nome || ' recebe acima da média');
		END IF;

		SELECT instrutores_internos(NEW.id) INTO refcursor_salarios;
		LOOP
			FETCH refcursor_salarios INTO salario;
			EXIT WHEN NOT FOUND;
			
			total_instrutores := total_instrutores + 1;
			IF (NEW.salario > salario) THEN
				instrutores_recebem_menos := instrutores_recebem_menos + 1;
			END IF;
		END LOOP;
		
		CLOSE refcursor_salarios;

		percentual := instrutores_recebem_menos::DECIMAL / total_instrutores::DECIMAL * 100;
		ASSERT percentual < 100::DECIMAL, 'Instrutores novos não podem receber mais do que todos os antigos.';

		INSERT INTO log_instrutores (informacao) VALUES (
			NEW.nome || ' recebe mais do que ' || trunc(percentual, 2) || '% da grade de instrutores');
			
		RETURN NEW;
	END;
$$ LANGUAGE plpgsql;

INSERT INTO instrutor (nome, salario) VALUES ('João', 15000000);

SELECT * FROM instrutor;
SELECT * FROM log_instrutores;
````

### Rodando uma procedure avulsa
````plpgsql
DO $$
	DECLARE
		refcursor_salarios refcursor;
		salario DECIMAL;
		total_instrutores INTEGER DEFAULT 0;
		instrutores_recebem_menos INTEGER DEFAULT 0;
		percentual DECIMAL;
	BEGIN
		SELECT instrutores_internos(37) INTO refcursor_salarios;
		LOOP
			FETCH refcursor_salarios INTO salario;
			EXIT WHEN NOT FOUND;
			
			total_instrutores := total_instrutores + 1;
			IF (600::DECIMAL > salario) THEN
				instrutores_recebem_menos := instrutores_recebem_menos + 1;
			END IF;
		END LOOP;
		
		CLOSE refcursor_salarios;
		
		percentual := instrutores_recebem_menos::DECIMAL / total_instrutores::DECIMAL * 100;

		RAISE NOTICE 'Percentual: % %%', percentual;
	END;
$$;
````
