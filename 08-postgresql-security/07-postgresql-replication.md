# Репликация PostgreSQL

## Streaming Replication

`
# Primary
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB

# Standby
primary_conninfo = 'host=primary port=5432 user=replica password=secret'
`

## Logical Replication

`sql
CREATE PUBLICATION mypub FOR TABLE users;
CREATE SUBSCRIPTION mysub CONNECTION 'host=primary port=5432 user=replica password=secret' PUBLICATION mypub;
`

## Безопасность

| Практика | Описание |
|----------|----------|
| Отдельный пользователь | Для репликации |
| SSL | Обязательно |
| Ограничение сети | Только standby |
| Monitoring | Лаг репликации |

---
_Статья создана на основе анализа материалов Habr по PostgreSQL_
