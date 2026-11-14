\# PostgreSQL Connection Resilience



\*\*Database Reliability Engineering | PostgreSQL | Docker | Replication | Incident Response\*\*



\## 1. Identificação



\- \*\*Disciplina:\*\* DBaaS – Alta Disponibilidade e Réplicas

\- \*\*Atividade:\*\* Incidentes em Bancos Replicados

\- \*\*Cenário:\*\* D – Esgotamento de conexões (negação de serviço)

\- \*\*Ambiente:\*\* PostgreSQL com Docker Compose

\- \*\*Primário:\*\* porta 5442

\- \*\*Réplica:\*\* porta 5443

\- \*\*Repositório:\*\* \[dataeng\_postgresql-connection-resilience](https://github.com/Ludimila-Araujo/dataeng\_postgresql-connection-resilience)



\## 2. Resumo do incidente



O incidente analisado corresponde ao cenário D: esgotamento de conexões, caracterizado pela recusa de novas conexões ao banco de dados em decorrência do atingimento do limite individual configurado para o usuário da aplicação.



Para reproduzir o problema, foi criada a role `app`, com limite de três conexões simultâneas (`CONNECTION LIMIT 3`). Em seguida, três sessões foram mantidas ocupadas por meio da execução da função `pg\_sleep()`.



Ao tentar estabelecer uma quarta conexão, o PostgreSQL retornou o erro:



`FATAL: too many connections for role "app"`



A primeira execução permitiu reproduzir e diagnosticar o incidente, mas não comprovou a recuperação por intervenção administrativa, pois as sessões provavelmente terminaram naturalmente antes da tentativa de encerramento.



Na segunda execução, com sessões de maior duração, foi possível identificar as três conexões ativas e encerrá-las utilizando `pg\_terminate\_backend()`. Uma nova conexão com o usuário `app` foi estabelecida com sucesso, comprovando o restabelecimento da disponibilidade.



O experimento demonstrou como limites individuais de conexão podem contribuir para a contenção de incidentes e como a identificação e o encerramento controlado de sessões podem restaurar o acesso ao banco.



\## 3. Linha do tempo



\### Execução 01: reprodução inicial



| Horário | Ação | Resultado |

|---|---|---|

| 12:43 | Tentativa de estabelecer a quarta conexão | Conexão recusada por esgotamento do limite da role `app` |

| 12:49 | Execução de `pg\_terminate\_backend()` | Retorno `(0 rows)`, sem comprovação de encerramento administrativo |

| 12:52 | Nova tentativa de conexão | Conexão estabelecida após o término natural das sessões, sem comprovação de recuperação por intervenção |



\### Execução 02: reprodução controlada e recuperação



| Horário | Ação | Resultado |

|---|---|---|

| 13:28 | Abertura das três sessões com `pg\_sleep(1800)` | Três conexões ocupadas |

| 13:29 | Tentativa de estabelecer a quarta conexão | Conexão recusada |

| 13:29 | Consulta à `pg\_stat\_activity` | Três sessões identificadas no estado `active` |

| 13:30 aproximadamente | Execução de `pg\_terminate\_backend()` | Três sessões encerradas com sucesso (`t`) |

| 13:31 | Nova tentativa de conexão | Conexão estabelecida com sucesso |





\## 4. Medições



\### Recovery Time Objective (RTO)



O RTO representa o intervalo entre a identificação da indisponibilidade e o restabelecimento do acesso ao banco de dados.



Na segunda execução, foram registrados os seguintes marcos:



\- \*\*T0:\*\* 13:29, tentativa de quarta conexão recusada.

\- \*\*T1:\*\* 13:31, nova conexão estabelecida com sucesso.

\- \*\*RTO observado:\*\* aproximadamente 2 minutos.





\### Recovery Point Objective (RPO)



O RPO representa a quantidade de dados que poderia ser perdida em decorrência de um incidente.



Neste experimento, não foram realizadas operações de escrita ou transações de negócio durante a reprodução do esgotamento de conexões. O teste concentrou-se na disponibilidade de novas conexões e no encerramento de sessões ativas.



Dessa forma, não foi possível medir experimentalmente a perda de dados ou determinar um RPO efetivo. Também não foram observadas perdas de dados persistidos durante o procedimento realizado.



\### Failover e replicação



O cenário não exigiu promoção da réplica ou procedimento de failover. A recuperação foi realizada diretamente no PostgreSQL primário, por meio do encerramento das sessões que ocupavam o limite de conexões da role `app`.





\## 5. Causa raiz



\## 5. Causa raiz



A causa raiz do incidente foi o esgotamento do limite individual de conexões configurado para a role `app` no PostgreSQL.



A role foi criada com a seguinte configuração:



`CONNECTION LIMIT 3`



Durante a reprodução, três sessões simultâneas foram mantidas ocupadas por meio da função `pg\_sleep()`. Com o limite individual atingido, uma quarta tentativa de conexão utilizando o mesmo usuário foi recusada pelo PostgreSQL, retornando:



`FATAL: too many connections for role "app"`



A consulta à visão `pg\_stat\_activity` confirmou a existência de três sessões do usuário `app` no estado `active`, demonstrando que o limite configurado havia sido atingido.



O incidente foi provocado pela combinação de um limite restritivo de conexões para o usuário da aplicação e sessões de longa duração que mantiveram os recursos de conexão ocupados.



\*\*Impacto observado:\*\* impossibilidade de estabelecer novas conexões utilizando a role `app` enquanto as três conexões existentes ocupavam seu limite.



\*\*Escopo:\*\* o experimento demonstrou o esgotamento do limite individual da role. Não foram realizadas medições que permitissem concluir que o limite global de conexões do servidor PostgreSQL também havia sido atingido.





\## 6. Correção aplicada e validação



A correção emergencial foi realizada por meio do encerramento administrativo das sessões que ocupavam o limite de conexões da role `app`.



Foi utilizado o comando:



```sql

SELECT pg\_terminate\_backend(pid)

FROM pg\_stat\_activity

WHERE usename = 'app';

```



Na segunda execução, o comando retornou três resultados positivos (t), confirmando o encerramento das três sessões identificadas.



Em seguida, foi realizada uma nova tentativa de conexão utilizando o usuário app:



```sql

SELECT current\_user, now();

```



Resultado da validação: conexão estabelecida com sucesso, confirmando o restabelecimento do acesso ao banco de dados.



A primeira execução apresentou resultado inconclusivo para a recuperação administrativa, pois o comando retornou (0 rows). A segunda execução, com sessões de maior duração, permitiu comprovar a eficácia do procedimento.



As medidas preventivas baseadas em parâmetros de timeout serão documentadas após sua configuração e validação experimental.



\## 7. Comparação com serviço gerenciado



Pendente de análise.



\## 8. Prevenção e lições aprendidas



Pendente de execução e análise.



\## 9. Respostas às questões do cenário



Pendente de desenvolvimento.

