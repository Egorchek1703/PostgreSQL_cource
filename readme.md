***Конфигурирование***  
  
**==================> База <==================**  
  
Основной конфигурационный файл - **postgresql.conf**, находящийся по пути /etc/postgreesql/<версия_postgresql>/<название_кластера>/postgresql.conf  
```
    cd /etc/postgreesql/16/main/
```  
Для того чтобы сервер учитывал изменённую конфигурацию, её необходимо перечитать. Для этого на хосте выполняем:  
```
    pg_ctlcluster 16 main reload
```  
Т.е. обращаемся к утилите pg_ctl через pg_ctlcluster и передаем ей версию для которой выполняем данное перечитывание и название кластера.  
Также мы можем перечитать конфигурацию выполнив запрос к БД:  
```
    SELECT pg_reload_conf();
```  
*Стоит помнить, что обычного перечитывания конфигурации не всегда достаточно для вступления некоторых изменений в силу. Может потребоваться перезапуск сервера*  
  
Чтобы узнать путь до основного конфигурационного файла мы также можем выполнить в psql:  
```
    SHOW config_file;
```  
  
Файлы можно читать с помощью функции pg_read_file():  
```
    SELECT pg_read_file('/etc/postgresql/16/main/postgresql.conf');
```  
  
Для работы с конфигурационными файлами из psql существует специальное представление pg_file_settings, содержащее в себе актуальные данные находящиеся в конфигурационном файле:  
```
    SELECT * FROM pg_file_settings;
```  
В данной таблице содержаться все разкомментированные строки конфигурационных файлов:  
    - sourcefile = файл из которого представлены строки  
    - sourceline = сама строка  
    - name = название параметра  
    - setting = значение параметра  
    - applied = значение с логическим типом данных, подтверждающее возможность применения нового значения для параметра в случае его изменения без перезапуска сервера  
Также существует еще одно представление pg_settings, которое содержит все конфигурационные параметры для сервера в данный момент:  
```
    SELECT * FROM pg_settings
    WHERE name = 'work_mem'
    \gx
```  
В данной таблице нам нужно обратить внимание на определенные поля:  
    - name = название параметра  
    - setting = значение параметра  
    - unit = единица измерения параметра  
    - boot_val = значение по умолчанию  
    - reset_val = значение которое будет установлено при выполнении команды RESET  
    - sourcefile = файл источник текущего значения параметра  
    - pending_restart = логическое значение, означающее нужен ли перезапуск сервера после пременения параметра  
    - context:  
        = user (любой пользователь может изменить для своего сеанса)  
        = internal (никто не может изменить)  
        = postmaster (значение можно поменять только перезапустив сервер)  
        = sighup (значение можно поменять перечитав конфигурацию)  
        = superuser (суперпользователь может изменить для своего сеанса)  
  
Как можно заметить, параметр work_mem был выбран в качестве примера не просто так. Он действительно важен, т.к. определяет максимально возможное кол-во памяти затрачиваемое сервером PostgreSQL, на операции сортировки, хэширования и других временных операций, например, соединение временных таблиц.  
Мы можем переназначать данный параметр (поле context в pg_settings равно user). Для корректно назначения данного параметра необходимо либо поменять конфигурацию на в файле хосте вручную, либо выполнить команду:  
```
    echo work_mem=12MB | sudo tee -a /etc/postgresql/16/main/postgresql.conf
```  
*Также важно понимать, что если конфигурационный парамерт задан несколько раз в файле, то применено будет последнее записанное значение (самое нижнее)*  
  
Помимо вышеуказанного, существует третий важный конфигурационный файл - **postgersql.auto.conf**. Он читается интепретатором PostgreSQL после файла postgresql.conf, а значит параметры, указанные в нем, имеют приоритет выше чем параметры из postgresql.conf  
*Данный файл всегда находится в каталоге кластера базы данных.*  
  
Также данный файл изменяется с помощью команд вида "**ALTER SYSTEM ...**", т.е. действительно полезен в случаях когда у нас нет доступа к корневому каталогу, но изменить параметры конфигурации необходимо.  
Для того чтобы изменить параметр, нам необходимо использовать следующий синтаксис:  
```
    ALTER SYSTEM
    SET парамерт TO значение
```
Для сброса параметра к значению по умолчанию:  
```
    ALTER SYSTEM RESET парамерт
```
Для сброса всех параметров:  
```
    ALTER SYSTEM RESET ALL
```  
  
*Параметр work_mem должен иметь единицу измерения (unit) только из указанного перечня (регистр важен) : "B", "kB", **"MB"**, "GB", "TB"*  
  
После изменения параметра work_mem в файле postgresql.auto.conf также необходимо перечитывать конфигурацию.  
  
Чтобы получить значение параметра мы можем использовать либо ключевое слово SHOW + название_парамерта в SQL запросе, либо функцию current_setting('название_параметра'). Для именения значения параметра мы можем использовать либо ключевое слово SET + название_парамерта TO 'его_значение', в SQL запросе, либо функцию set_config('название_параметра', 'новое_значение', true\false). Третий парамерт в данном случае устанавливает должно ли данное новое значение парамерта быть применено, только в текущей транзакции. Т.е. если третий параметр = false, значит значение будет применено до конца всего сеанса.  
```
    SHOW work_mem;
    SELECT current_setting('work_mem');

    SET work_mem TO '4MB';
    SELECT set_config('work_mem', '4MB', false);
```  
*Если мы хотим установить значение параметра в рамках текущей транзакции с помощью "SET ... TO ...", то нам необходимо после слова SET дописать ключевое слово **LOCAL**.*
  
Мы также можем задавать пользовательские параметры. Данные парамерты должны в названии содержать точку - это отличительный символ для того чтобы сервер понимал, что данный парамерт пользовательский. Такие параметры могут бытб полезны в роли глобальной переменной.  
  
**Журнал событий PostgreSQL находится в каталоге /var/log/postgresql/**  