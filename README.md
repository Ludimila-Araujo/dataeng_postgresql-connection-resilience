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

## 9. Evidências experimentais

As capturas de tela e registros das execuções estão organizados no diretório `evidencias/`.

| Diretório | Conteúdo |
|---|---|
| [Exec