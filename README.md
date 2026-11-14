# PostgreSQL Connection Resilience

**Database Reliability Engineering | PostgreSQL | Docker | Replication**

Projeto desenvolvido como atividade prática da disciplina **DBaaS – Alta Disponibilidade e Réplicas**, do MBA em Engenharia e Ciência de Dados.

O trabalho tem como objetivo reproduzir, diagnosticar e tratar um incidente de disponibilidade em um ambiente PostgreSQL replicado.

---

## 1. Identificação do projeto

| Informação | Descrição |
|---|---|
| Disciplina | DBaaS – Alta Disponibilidade e Réplicas |
| Atividade | Incidentes em Bancos Replicados |
| Cenário | D – Esgotamento de conexões |
| Banco de dados | PostgreSQL 16 |
| Ambiente | Docker Compose |
| Instância primária | Porta 5442 |
| Instância réplica | Porta 5443 |
| Repositório | [GitHub](https://github.com/Ludimila-Araujo/dataeng_postgresql-connection-resilience) |


## 2. Cenário do incidente

O cenário D simula o esgotamento de conexões disponíveis para um usuário da aplicação, provocando a recusa de novas conexões ao banco de dados.

Para reproduzir o incidente, foi criada a role `app` com limite de três conexões simultâneas:

```sql
CREATE ROLE app LOGIN PASSWORD 'app' CONNECTION LIMIT 3;
```

Em seguida, três sessões foram mantidas ocupadas utilizando a função `pg_sleep()`.

Ao tentar estabelecer uma quarta conexão, o PostgreSQL retornou:

```text
FATAL: too many connections for role "app"
```


## 3. Ambiente e arquitetura

O ambiente experimental utiliza duas instâncias PostgreSQL executadas em containers Docker:

| Instância | Porta | Função |
|---|---:|---|
| Primary | 5442 | Instância principal |
| Replica | 5443 | Instância de réplica |

A verificação do estado das instâncias utiliza:

```sql
SELECT pg_is_in_recovery();
```

Resultado esperado:
- Primário: `false`.
- Réplica: `true`.


## 4. Reprodução do incidente

Foram realizadas duas execuções para reproduzir e investigar o comportamento do banco.

### Execução 01: reprodução inicial

Na primeira execução, três sessões foram abertas utilizando `pg_sleep(300)`.

A quarta tentativa de conexão foi recusada, confirmando o esgotamento do limite individual da role `app`.

O diagnóstico identificou três sessões ativas em `pg_stat_activity`.

A tentativa de encerramento administrativo retornou `(0 rows)`. Como as sessões provavelmente terminaram naturalmente antes da intervenção, não foi possível comprovar a recuperação por meio desse procedimento.

### Execução 02: reprodução controlada

Na segunda execução, foi utilizado `pg_sleep(1800)` para manter as sessões ocupadas por mais tempo.

O procedimento permitiu:

1. Ocupar as três conexões disponíveis.
2. Reproduzir a recusa da quarta conexão.
3. Identificar as três sessões ativas.
4. Encerrar administrativamente as sessões.
5. Confirmar o restabelecimento de uma nova conexão.


## 5. Linha do tempo

### Execução 01

| Horário | Evento | Resultado |
|---|---|---|
| 12:43 | Tentativa de quarta conexão | Conexão recusada |
| 12:49 | Tentativa de encerramento administrativo | Retorno `(0 rows)` |
| 12:52 | Nova conexão | Conexão estabelecida após o término natural das sessões |

### Execução 02

| Horário | Evento | Resultado |
|---|---|---|
| 13:28 | Abertura das três sessões | Conexões ocupadas |
| 13:29 | Tentativa de quarta conexão | Conexão recusada |
| 13:29 | Diagnóstico com `pg_stat_activity` | Três sessões `active` |
| Após o diagnóstico | Execução de `pg_terminate_backend()` | Três sessões encerradas com sucesso |
| 13:31:17 | Nova tentativa de conexão | Conexão estabelecida |


## 6. Métricas de recuperação

### Recovery Time Objective (RTO)

O RTO representa o intervalo entre a identificação da indisponibilidade e o restabelecimento do acesso ao banco.

Na segunda execução:

- **T0:** 13:29, quarta conexão recusada.
- **T1:** 13:31:17, nova conexão estabelecida.
- **RTO observado:** aproximadamente 2 minutos.

A precisão da medição é limitada porque o horário de T0 foi registrado apenas até o minuto.

### Recovery Point Objective (RPO)

O RPO representa a quantidade de dados que poderia ser perdida em decorrência de um incidente.

Neste experimento, não foram realizadas operações de escrita ou transações de negócio durante a reprodução do esgotamento de conexões.

Não foram observadas perdas de dados persistidos. Entretanto, não foi possível medir experimentalmente a perda de dados ou determinar um RPO efetivo.

### Failover

Não foi necessário realizar promoção da réplica ou procedimento de failover. A recuperação ocorreu diretamente no primário.


## 7. Análise da causa raiz

A causa raiz identificada foi o esgotamento do limite individual de conexões configurado para a role `app`.

As três sessões simultâneas atingiram o limite estabelecido. Consequentemente, novas conexões utilizando o mesmo usuário foram recusadas pelo PostgreSQL.

A consulta à visão `pg_stat_activity` confirmou a existência das três sessões no estado `active`.

**Impacto observado:** impossibilidade de estabelecer novas conexões utilizando a role `app` enquanto as três conexões existentes ocupavam seu limite.

O experimento demonstrou o esgotamento do limite individual da role, não sendo suficiente para concluir que o limite global de conexões do servidor também tenha sido atingido.


## 8. Procedimento de recuperação

A recuperação foi realizada por meio do encerramento administrativo das sessões que ocupavam o limite da role `app`.

Comando utilizado:

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE usename = 'app';
```

Na segunda execução, foram obtidos três retornos positivos (`t`), confirmando o encerramento das sessões.

Em seguida, foi realizada uma nova tentativa de conexão:

```sql
SELECT current_user, now();
```

**Resultado:** conexão estabelecida com sucesso, comprovando o restabelecimento do acesso ao banco.

### Medidas preventivas aplicadas

Além da recuperação por encerramento administrativo das sessões, foram configuradas duas medidas preventivas na role `app`:

```sql
ALTER ROLE app SET statement_timeout = '3s';
ALTER ROLE app SET idle_session_timeout = '5s';
```

As configurações foram aplicadas a novas sessões e tiveram seus comportamentos verificados experimentalmente.

**Teste do `statement_timeout`:**

Foi executada a consulta:

```sql
SELECT pg_sleep(30);
```

O PostgreSQL cancelou a execução, retornando:

```text
ERROR: canceling statement due to statement timeout
```

**Teste do `idle_session_timeout`:**

Foi aberta uma nova sessão interativa com o usuário `app`. Após o período de inatividade, o PostgreSQL encerrou a conexão, retornando:

```text
FATAL: terminating connection due to idle-session timeout
```

Os dois testes confirmaram o funcionamento das configurações preventivas.

### 8.1 Comparação com serviço gerenciado

Em um ambiente gerenciado com Amazon RDS for PostgreSQL, o incidente de esgotamento de conexões poderia ser tratado com recursos adicionais de gerenciamento e monitoramento.

O **Amazon RDS Proxy** atua como intermediário entre a aplicação e o banco de dados, gerenciando um conjunto de conexões que podem ser reutilizadas por diferentes solicitações. Esse mecanismo reduz a necessidade de estabelecer conexões diretamente com o banco a cada requisição e permite configurar limites para o pool, tempos de espera e encerramento de conexões de clientes ociosas.

O monitoramento do ambiente também permitiria acompanhar métricas relacionadas às conexões e aos recursos do banco, possibilitando identificar comportamentos anormais e configurar alarmes para situações de risco.

Já o recurso **Multi-AZ** oferece alta disponibilidade por meio de uma instância standby em outra zona de disponibilidade, permitindo o failover automático diante de determinadas falhas de infraestrutura. Entretanto, esse mecanismo não resolve diretamente o esgotamento de conexões provocado por uma aplicação ou usuário.

Dessa forma, para o incidente reproduzido neste trabalho, o RDS Proxy seria o recurso mais diretamente relacionado à prevenção e ao gerenciamento do consumo de conexões, enquanto o Multi-AZ complementaria a arquitetura oferecendo disponibilidade diante de falhas de infraestrutura.

Assim como no ambiente experimental, o uso de um proxy não elimina a necessidade de configurar limites adequados, controlar a concorrência e monitorar o comportamento das aplicações.


## 9. Respostas às perguntas do cenário

### 9.1 Por que limitar conexões por usuário e não apenas pelo limite global?

O limite global de conexões do PostgreSQL protege os recursos gerais do servidor, mas não impede que uma única aplicação ou usuário consuma grande parte das conexões disponíveis. Assim, a definição de limites individuais por role permite isolar o consumo de recursos, evitando que uma aplicação monopolize as conexões e prejudique o acesso de outros usuários e serviços. Dessa forma, no experimento, a role `app` foi configurada com limite de três conexões simultâneas. Quando esse limite foi atingido, novas conexões utilizando a mesma role foram recusadas, enquanto o servidor PostgreSQL permaneceu em execução.

Em conclusão, os limites por usuário complementam o limite global, contribuindo para o isolamento e a disponibilidade dos serviços que compartilham o banco de dados.

### 9.2 Qual seria o papel de um pool de conexões ou proxy?

Um pool de conexões ou proxy atua como intermediário entre a aplicação e o banco de dados, gerenciando um conjunto limitado de conexões que podem ser reutilizadas por diferentes solicitações. Essa abordagem reduz a necessidade de estabelecer novas conexões diretamente com o PostgreSQL a cada solicitação, contribuindo para controlar a concorrência e evitar o consumo excessivo de conexões disponíveis.

Nesta perspectiva, ferramentas como PgBouncer e Amazon RDS Proxy oferecem mecanismos de gerenciamento e reutilização de conexões. Dessa forma, ajudam a reduzir a pressão sobre o banco de dados e a melhorar a disponibilidade dos serviços. É importante salientar que o uso de um pool ou proxy não elimina a necessidade de configurar limites de conexões, tempos de espera e políticas de controle de carga.


## 10. Evidências experimentais

As capturas de tela das execuções estão organizadas no diretório `evidencias/`, separadas conforme a execução do experimento.

### Execução 01

| Arquivo | Evidência |
|---|---|
| `01_esgotamento_conexoes.png` | Reprodução do esgotamento de conexões |
| `02_diagnostico_sessoes.png` | Identificação das sessões ativas |
| `03_tentativa_recuperacao.png` | Tentativa inicial de recuperação |

### Execução 02

| Arquivo | Evidência |
|---|---|
| `01_diagnostico_tres_sessoes_ativas.png` | Diagnóstico das três sessões ativas |
| `02_terminacao_administrativa_tres_sessoes_confirmada.png` | Encerramento administrativo das sessões |
| `03_nova_conexao_bem_sucedida.png` | Confirmação do restabelecimento da conexão |
| `04_configuracao_statement_timeout.png` | Configuração do limite de execução |
| `05_configuracao_idle_session_timeout.png` | Configuração do limite de inatividade |
| `06_teste_statement_timeout.png` | Validação do cancelamento de consulta |
| `07_teste_idle_session_timeout.png` | Validação do encerramento de sessão ociosa |

As capturas documentam a reprodução do incidente, o diagnóstico, a recuperação e a validação das medidas preventivas.
