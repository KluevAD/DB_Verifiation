# Настройка верификации отказоустойчивости БД.

***

Ниже представлена сравнительная таблица, которая показывает разницу настроек параметра "synchronous_commit" в файле "postgresql.conf".

|**Значение synchronous_commit**|**Гарантированная локальная фиксация**|**Гарантированная фиксация на ведомом после сбоя PG**|**Гарантированная фиксация на ведомом после сбоя ОС**|**Согласованность запросов на ведомом**|
|:---:|:---:|:---:|:---:|:---:|
|remote_apply|+|+|+|+|
|on|+|+|+||
|remote_write|+|+|||
|local|+||||
|off|||||

Исходя из этих данных можно предугадать лучший и худший исходы:

1. Лучший - **remote_apply**

Со значением remote_apply фиксирование завершается только после получения ответов от текущих синхронных ведомых серверов, говорящих, что они получили запись о фиксировании транзакции, сохранили её в надёжном хранилище, а также применили транзакцию, так что она стала видна для запросов на этих серверах. С таким вариантом задержка при фиксировании оказывается больше, так как необходимо дожидаться воспроизведения WAL.

2. Худший - **off**

Со значением off фиксирования нет.

***

# Использованный скрипт верификации отказоустойчивости

***

```python
import psycopg2
import os
from psycopg2 import Error

def ping(ip):
    response = os.system("ping " + ip)
    return response

def close_connection(Master_ip):
    os.system(f"ssh postgres@{Master_ip} 'ifconfig eth0 down && sleep 10 && ifconfig eth0 down'")

def comparison(Master_ip, Replica_ip):
    while ping(Master_ip) == 0:
        connection = psycopg2.connect(dbname='basket', user='postgres', password='12345', host = Master_ip )
        cur = connection.cursor()
        cur.execute('SELECT COUNT(*) FROM basket;')
        rec_count = cur.fetchone()

        connection2 = psycopg2.connect(dbname='basket', user='postgres', password='12345', host = Replica_ip  )
        cur2 = connection2.cursor()
        cur2.execute('SELECT COUNT(*) FROM basket;')
        rec_count_repl = cur2.fetchone()

        print(f"Number of records <basket> table on Master: {rec_count[0]}\n Number of records <basket> table on Replica: {rec_count_repl[0]}")
        if rec_count[0] == rec_count_repl[0]:
            print("Same number of records!")
            exit(0)
        if rec_count[0] != rec_count_repl[0]:
            print(f"Mismatch number of records!: {(rec_count[0]-rec_count_repl[0])}")
            exit(0)

async def verification(Master_ip, Replica_ip):
    connection = psycopg2.connect(dbname='basket', user='postgres', password='12345', host = Master_ip )
    try:
        i = 0
        while i < 100000:
            async with connection.transaction():
                await connection.execute(f"INSERT INTO basket (Name, Phone) VALUES ('{i}','{i}');")
            if i == 1000:
                close_connection()
            i = i + 1
    except:
        pass

def main():

    Master_ip = '192.168.200.100'
    Replica_ip = '192.168.200.101'
    
    while( True ):
        verification(Master_ip, Replica_ip)

if __name__ == "__main__":
    main()
```

***

