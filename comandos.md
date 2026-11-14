# Registro de comandos e execução

**Projeto:** PostgreSQL Connection Resilience  
**Cenário:** D – Esgotamento de conexões  
**Ambiente:** PostgreSQL com Docker Compose  
**Diretório do ambiente:** `C:\Users\ludim\lab1`


## 1. Preparação e validação do ambiente

### 1.1 Verificação dos containers

```cmd
cd /d C:\Users\ludim\lab1
docker compose ps
```

**Resultado:** containers do PostgreSQL primário e da réplica em execução.

- Primário: porta `5442`.
- Réplica: porta `5443`.

### 1.2 Verificação do estado dos servidores

No primário:

```cmd
docker compose exec primary psql -U postgres -c "SELECT pg_is_in_recovery();"
```

**Resultado:** `false`, confirmando que o servidor atua como primário.

Na réplica:

```cmd
docker compose exec replica psql -U postgres -c "SELECT pg_is_in_recovery();"
```

**Resultado:** `true`, confirmando que o servidor está em recuperação, atuando como réplica.


## 2. Reprodução do incidente

### 2.1 Criação do usuário da aplicação

Executado no primário:

```cmd
docker compose exec primary psql -U postgres -c "CREATE ROLE app LOGIN PASSWORD 'app' CONNECTION LIMIT 3;"
```

**Resultado:** usuário `app` criado com limite de três conexões simultâneas.

### 2.2 Primeira execução

Foram abertas três sessões, em terminais separados:

```cmd
docker compose exec primary psql -U app -d postgres -c "SELECT pg_sleep(300);"
```

**Resultado:** três sessões simultâneas permaneceram ativas.

**Horário da tentativa de quarta conexão:** 12:43.

Comando utilizado:

```cmd
docker compose exec primary psql -U app -d postgres -c "SELECT current_user;"
```

**Resultado:** conexão recusada com a mensagem:

```text
FATAL: too many connections for role "app"
```

### 2.3 Segunda execução controlada

Após a primeira execução, foi realizada uma nova reprodução, com tempo de espera ampliado:

```cmd
docker compose exec primary psql -U app -d postgres -c "SELECT pg_sleep(1800);"
```

O comando foi executado em três terminais separados.

**Horário da tentativa de quarta conexão:** 13:29.

Comando utilizado:

```cmd
docker compose exec primary psql -U app -d postgres -c "SELECT current_user;"
```

**Resultado:** conexão recusada com a mensagem:

```text
FATAL: too many connections for role "app"
```


## 3. Diagnóstico

Consulta executada no primário:

```cmd
docker compose exec primary psql -U postgres -c "SELECT pid, usename, state, now() - query_start AS tempo FROM pg_stat_activity WHERE usename = 'app';"
```

**Resultado da primeira execução:** três sessões do usuário `app` identificadas no estado `active`.

**Resultado da segunda execução:** três sessões do usuário `app` identificadas no estado `active`, executando `pg_sleep`.


## 4. Correção e validação

### 4.1 Primeira execução

Foi realizada uma tentativa de encerramento das sessões:

```cmd
docker compose exec primary psql -U postgres -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE usename = 'app';"
```

**Resultado:** retorno `(0 rows)`. As sessões já haviam terminado naturalmente antes da execução do comando. Portanto, essa tentativa não comprovou a recuperação por encerramento administrativo.

Posteriormente, uma nova conexão foi estabelecida com sucesso:

```cmd
docker compose exec primary psql -U app -d postgres -c "SELECT current_user, now();"
```

### 4.2 Segunda execução controlada

O encerramento administrativo foi executado novamente, utilizando o usuário administrador:

```cmd
docker compose exec primary psql -U postgres -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE usename = 'app';"
```

**Resultado:** três sessões encerradas com sucesso (`true`).

### 4.3 Validação da recuperação

Após o encerramento das sessões, foi realizada uma nova conexão:

```cmd
docker compose exec primary psql -U app -d postgres -c "SELECT current_user, now();"
```

**Resultado:** conexão estabelecida com sucesso. O usuário `app` voltou a acessar o banco.


## 5. Prevenção

Após a recuperação do serviço, foram aplicadas configurações para limitar o tempo de execução de consultas e o tempo de permanência de sessões ociosas do usuário `app`.

### 5.1 Configuração do statement_timeout

Comando executado no primário:

```cmd
docker compose exec primary psql -U postgres -c "ALTER ROLE app SET statement_timeout = '3s';"
```

**Resultado:** `ALTER ROLE`.

A configuração estabelece o limite de três segundos para a execução de comandos SQL nas novas sessões do usuário `app`.

### 5.2 Configuração do idle_session_timeout

Comando executado no primário:

```cmd
docker compose exec primary psql -U postgres -c "ALTER ROLE app SET idle_session_timeout = '5s';"
```

**Resultado:** `ALTER ROLE`.

A configuração estabelece o encerramento de sessões que permaneçam ociosas por mais de cinco segundos.

### 5.3 Validação do statement_timeout

Foi aberta uma nova sessão do usuário `app` e executada uma consulta com duração superior ao limite configurado:

```cmd
docker compose exec primary psql -U app -d postgres -c "SELECT pg_sleep(30);"
```

**Resultado observado:**

```text
ERROR: canceling statement due to statement timeout
```

A consulta foi cancelada pelo PostgreSQL devido ao limite de tempo de execução configurado.

### 5.4 Validação do idle_session_timeout

Foi iniciada uma nova sessão interativa do usuário `app`:

```cmd
docker compose exec primary psql -U app -d postgres
```

Dentro da sessão, foi executada a consulta:

```sql
SELECT current_user, now();
```

Em seguida, a conexão foi mantida ociosa por mais de cinco segundos.

**Resultado observado:**

```text
FATAL: terminating connection due to idle-session timeout
server closed the connection unexpectedly
```

O encerramento da conexão confirmou a aplicação do limite de tempo para sessões ociosas. Após o encerramento da sessão original, o terminal tentou estabelecer uma nova conexão automaticamente.

**Conclusão:** as duas medidas preventivas foram configuradas e validadas em novas sessões do usuário `app`.

## 6. Observações sobre as medições

- **RTO:** na segunda execução, aproximadamente 2 minutos entre a recusa da quarta conexão, às 13:29, e o restabelecimento do acesso, às 13:31.
- **RPO:** não houve perda de dados persistidos observada durante o experimento. Como não foram realizadas operações de escrita durante a reprodução, o RPO efetivo não foi medido.
- Os horários registrados correspondem aos dados observados durante as execuções.

---

**Observação:** este documento registra os comandos e resultados efetivamente observados durante a reprodução do incidente, a recuperação do serviço e a aplicação das medidas preventivas.