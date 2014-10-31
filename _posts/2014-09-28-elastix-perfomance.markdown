---
layout: post
title:  "Прозводительность elastix"
date:   2014-09-28
description: "Не так давно столкнулся с проблемами производительности в elastix, после внедрения автоматической \"звонилки\". \"Звонилка\" работает через AMI: посылается несколько звонков, соединяет оператора с первым ответившим абонентом и открыдывает всех не ответивших.<br><br>
Проблема проявлялась в резком скачке потребления процессора на сервере, грузились все 24 ядра под 100%. В этот момент связь работала с дикими перебоями."
---
Не так давно столкнулся с проблемами производительности в elastix, после внедрения скриптов, которые через AMI инициализируют вызовы.

Когда на сервере количество одновременных звонков превышало 20 штук, резко подскакивало потребления процессора на сервере: грузились все 24 ядра под 100%. В этот момент связь работала с дикими перебоями.

**График загрузки процессора до начала работ и после:**
![график до](/img/elastix_chart.png)
*Работы были закончены 4 июня. Скачки IO wait с 30 мая по 1 июня, связаны с перекодировкой записей.*

**Все данные справедливы для elastix версии 2.4.0**

### 1-й этап: перенос записей разговоров

Проблема с резкими скачками потребления CPU были от части связана с большим количеством файлов в папке
*/var/spool/asterisk/monitor*, где elastix хранит записи. 
Там было около миллиона файлов.

После каждого звонка, выполняется команда из файла *extensions_override_elastix.conf*: 

`exten => s,n(defaultmixmondir),System(test -e ${MIXMON_CALLFILENAME})`

Которая по маске сканирует всю директорию с записями.

В сети был найден скрипт, который проходит по всем файлам, передикодирует их в mp3 и переносит их в папку */home/monitor*.

Данный скрипт, лучше всего запускать в момент наименьшей нагрузки! Мы это делаем ночью.

{% highlight bash %}
#!/bin/bash
recorddir="${1:-/var/spool/asterisk/monitor}"
cd $recorddir;
for file in *.wav ; do
mp3=$(basename "$file" .wav).mp3;
nice lame -b 16 -m m -q 7-resample "$file" "$mp3";
#touch –reference "$file" "$mp3";
chown asterisk.asterisk "$mp3";
chmod 444 "$mp3";
mv "$mp3" /home/monitor;
rm -f "$file";
done
{% endhighlight %}

После чего в папке с записями хранятся только записи за сегодняшний день. У нас их обычно не больше 1-2 тысяч.

### 2-й этап: прослушивание записей

После переноса и передикодирования записей, они будут недоступны для прослушивания через модуль monitoring. Чтобы это поправить нужно создать новый виртуальный хост в apache:

*/etc/httpd/conf.d/cdr_static.conf*
{% highlight apache %}
Timeout 300

User asterisk
Group asterisk

Alias /cdr_static /var/spool/asterisk/monitor
Alias /cdr_static_mp3 /home/monitor

<Directory /var/spool/asterisk/monitor/>
   Allow from All
   RewriteEngine On

   RewriteBase /var/spool/asterisk/monitor/
   RewriteCond %{REQUEST_FILENAME} -f [OR]
   RewriteCond %{REQUEST_FILENAME} -d
   RewriteRule (.*) - [S=1]

   RewriteCond %{REQUEST_FILENAME} !-f
   RewriteRule ^(.*).wav$ /cdr_static_mp3/$1.mp3 [L]

</Directory>
{% endhighlight %}

Данный хост, сначала проверяет наличие файла в старой директории, а если там файла нет, то переадресует в новую с изменением расширения на mp3.

Так же пришлось изменить файл */var/www/html/modules/monitoring/index.php*:

в метод downloadFile нужно добавить:
 
{% highlight php %}
header( 'Location: /cdr_static/'.$namefile, true, 301 );
return true;
{% endhighlight %}

сразу после:

{% highlight php %}
$file = basename(str_replace('audio:', '', $filebyUid['userfield']));
$path = $path_record.$file;
if ($file == 'deleted' || !is_file($path)) {
    // Specified file has been deleted
    Header('HTTP/1.1 404 Not Found');
    die("<b>404 "._tr("no_file")." </b>");
}
{% endhighlight %}

Так же необходимо изменить выше строку 
{% highlight php %}
if ($file == 'deleted' || !is_file($path)) {
{% endhighlight %}
на
{% highlight php %}
if ($file == 'deleted') {
{% endhighlight %}

После этих изменений, при запросе файла, он будет переадресован в новый виртуальных хост.

### 3-й этап: hangup.agi

Если у вас нет скриптов в папке */usr/local/elastix/addons_scripts*, либо этой папки нет вообще, 
смело в файле */etc/asterisk/extensions_override_elastix.conf* комментируйте строку:

```
exten => s,n(theend),AGI(hangup.agi)
```

Это необходимо, чтобы убрать лишние запросы к AGI при каждом окончании разговора.

### 4-й этап: asterisk logs

У меня после внедрения fail2ban логи писались сразу в несколько мест.

Куда пишутся логи, можно найти в файле */etc/asterisk/logger.conf*.

Отключайте всё лишнее, оставив только:

* console (настраивает уровень логирования в консоль asterisk)
* messages (имя файла, который находится в директории astlogdir, обычно это /var/log/asterisk). 

{% highlight bash %}
console => notice,warning,error
messages => notice,warning,error
{% endhighlight %}

### 5-й этап: CDR.

По умолчанию asterisk в сборке elastix складывает CDR-записи не только в mysql, а ещё и в файлы:

* /var/log/asterisk/master.db - sqlite БД
* /var/log/asterisk/cdr-csv/Master.csv - csv таблица

Постепенно эти файлы начинают очень сильно разрастаться в размерах.

Отключаем лишнее:
  
1. Комментируем все строки в файле */etc/asterisk/cdr_sqlite3_custom.conf*
2. Комментируем в файле */etc/asterisk/cdr.conf* строки:

{% highlight bash %}
[csv]
usegmtime=yes    ; log date/time in GMT.  Default is "no"
loguniqueid=yes  ; log uniqueid.  Default is "no"
loguserfield=yes ; log user field.  Default is "no"
accountlogs=yes  ; create separate log file for each account code. Default is "yes"
{% endhighlight %}

### 6-й этап (по желанию): flash operator panel.

В elastix есть панель, для просмотра активных звонков и соединений.

Если вы ей не пользуетесь, не стоит и начинать, если пользуетесь... сами виноваты.

В качестве сервера выступает файл `/var/www/html/panel/op_server.pl`, который после долгой работы выжирает всю память.
При чём не важно, открыта у вас эта консоль или нет.

Убивание процесса `op_server.pl` не помогает. Отключить эту поделку можно следующим способом: 

в файле `/var/www/html/panel/safe_opserver`, закомментируйте строки:
{% highlight bash %}
while true; do
$FOPWEBROOT/op_server.pl
sleep 4
done
{% endhighlight %}

### Грустные итоги

Потребление процессора серьёзно снизилось, но в конечном итоге всё равно пришлось перевести отдела продаж на чистый asterisk. Причиной такому переходу послужили необъяснимые тормоза astretisk при большом количестве одновременных запросов на осуществление вызова через AMI.
