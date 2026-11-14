\# Registro de comandos e execução



\*\*Projeto:\*\* PostgreSQL Connection Resilience  

\*\*Cenário:\*\* D – Esgotamento de conexões  

\*\*Ambiente:\*\* PostgreSQL com Docker Compose



\## 1. Preparação e validação do ambiente



| Horário | Comando | Objetivo | Resultado |

|---|---|---|---|

| A preencher | A preencher | Validar os containers | A preencher |

| A preencher | A preencher | Validar o estado do primário e da réplica | A preencher |



\## 2. Reprodução do incidente



| Horário | Comando | Objetivo | Resultado |

|---|---|---|---|

| A preencher | A preencher | Criar o usuário da aplicação com limite de conexões | A preencher |

| A preencher | A preencher | Ocupar a primeira conexão | A preencher |

| A preencher | A preencher | Ocupar a segunda conexão | A preencher |

| A preencher | A preencher | Ocupar a terceira conexão | A preencher |

| A preencher | A preencher | Tentar estabelecer a quarta conexão | A preencher |



\## 3. Diagnóstico



| Horário | Comando | Objetivo | Resultado |

|---|---|---|---|

| A preencher | A preencher | Consultar as sessões da aplicação em `pg\_stat\_activity` | A preencher |



\## 4. Correção e validação



| Horário | Comando | Objetivo | Resultado |

|---|---|---|---|

| A preencher | A preencher | Encerrar as sessões responsáveis pelo esgotamento | A preencher |

| A preencher | A preencher | Confirmar o restabelecimento de conexões | A preencher |



\## 5. Prevenção



| Horário | Comando | Objetivo | Resultado |

|---|---|---|---|

| A preencher | A preencher | Configurar `statement\_timeout` | A preencher |

| A preencher | A preencher | Configurar `idle\_session\_timeout` | A preencher |

| A preencher | A preencher | Testar os limites em uma nova sessão | A preencher |



\---



\*\*Observação:\*\* os horários e resultados deverão corresponder à execução real. Nenhum resultado será presumido.

