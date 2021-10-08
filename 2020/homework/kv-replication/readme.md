# Распределенное key-value хранилище с репликацией

В этом задании вам предстоит доработать реализацию key-value хранилища из прошлого задания, обеспечив отказоустойчивое хранение данных путем их репликации. Теперь каждая запись должна храниться не на одном узле, а на трех. Таким образом, при выходе из строя 1-2 узлов, данные не должны пропадать. При этом остается шардинг, то есть в системе из 6 узлов каждый будет хранить примерно половину данных.

Для реализации репликации будем использовать подход без лидера, используемый в [Amazon Dynamo](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf). Операции на чтение и запись выполняются с кворумами, размеры которых указывает клиент при отправке операции. Запрос на выполнение операции может быть сделан с любого узла. Далее выбирается узел-координатор, который рассылает операцию по репликам и, при получении кворума ответов, сообщает результат операции. Для операций чтения координатором может быть любой узел, а для операций записи это, как правило, одна из реплик ключа (см. статью про Dynamo).

Мы хотим сделать наше хранилище высокодоступным и уметь успешно обрабатывать запросы в присутствии отказов узлов и сети. Если из-за отказов для успешного выполнения операции не хватает живых реплик (не набирается нужный кворум), то в качестве резервных реплик должны использоваться другие доступные узлы. Если отказавшая реплика стала снова доступна, то резервная должна передать ей записанные данные. Этот подход (sloppy quorum и hinted handoff) также описан в статье про Dynamo. 

Для простоты будем считать, что отказы временные, и в случае обнаружения отказа не требуется делать перебалансировку шардов или переназначать реплики между узлами. Это требуется делать только в случае, когда администратор явно захочет добавить новый узел или вывести из системы существующий с помощью команд `JOIN` и `LEAVE`.

В силу описанного подхода, операции записи в нашей системе могут возвращать клиенту успех до того, как изменения достигнут всех реплик, а чтение может возвращать устаревшее значение. При этом система должна обеспечивать согласованность в конечном счёте - изменения должны рано или поздно оказаться на всех живых репликах. Для этого надо как минимум реализовать механизмы hinted handoff и read repair. Реализация фоновой синхронизации (anti-entropy) - по желанию.

Из-за отказов и одновременных операций записи на разных узлах системы может оказаться несколько версий значения ключа. Ваша реализация должна уметь определять, когда эти версии связаны логически (одна является потомком другой и можно оставить более свежую), а когда нет и требуется разрешение конфликта. 

Если обнаружен конфликт версий, то по-умолчанию надо хранить и возвращать клиенту все такие версии. Исключением являются ключи с префиксом `cart`. Для них надо реализовать описанную в статье про Dynamo стратегию автоматического разрешения конфликта для корзин с заказами. Значением таких ключей является строка, содержащая список товаров, разделенных запятой. При разрешении конфликта итоговое значение должно содержать все товары из обоих версий корзины.

Форматы запросов и ответов на все требуемые операции описаны в [заготовке решения](solution/node.py). В основном они повторяют интерфейс из прошлой задачи, за исключением следующих важных отличий:

- Операция `GET` дополнительно принимает размер кворума и может возвращать несколько значений (версий записи) и произвольные метаданные для них (например, векторы версий).
- Операция `PUT` дополнительно принимает размер кворума и метаданные изменяемой версии данных, а возвращает метаданные новой записанной версии.
- Операция `DELETE` отсутствует.
- Операция `LOOKUP` возвращает не один, а список узлов, вместе с их адресами.

## Тестирование

Необходимые зависимости должны быть уже установлены (см. предыдущие задания). Добавьте корень репозитория в переменную окружения PYTHONPATH и запустите тесты:

```console
$ PYTHONPATH=$ROOT_DIRECTORY_PATH python3 test.py solution
```

Если Вам нужно больше отладочной информации, добавьте флаг `-v` (показывает сообщения) или `-d` (показывает вывод вашей реализации) к тестированию. Для анализа больших логов рекомендуется перенаправлять вывод тестов в файл, добавив к команде ` &>log`, и затем искать нужную информацию, например с помощью grep.

Поднимаются несколько процессов узлов, которые коммуницируют между собой через `test_server`. Все аналогично предыдущим задачам.

Мы уже написали за Вас тесты, которы должны проходить при правильном решении. Вы можете добавлять новые тесты, по умолчанию они не будут оцениваться. Однако если вы придумаете и реализуете тест, который увеличит покрытие наших тестов, то за него можно получить бонус 2 балла. За деталями о том, как мы тестируем, обращайтесь к тестам, осознать, что там происходит -- Ваша задача.

## Оценивание

Распределение баллов по тестам:

- BasicTestCase: 1
- ReplicasCheckTestCase: 1
- StaleReplicaTestCase: 1
- ReplicasDivergenceTestCase: 1
- SloppyQuorumTestCase: 1
- PartitionedClientTestCase: 1
- PartitionedClientsTestCase: 1
- ShoppingCartTestCase: 1
- NodeJoinTestCase: 1
- NodeLeaveTestCase: 1

## Сдача

Сдача и проверка решений будет вестись через сервис Gradescope.
См. инструкцию в первой задаче.

Далее зайдите в Assignments, там вы увидите задачу KV Replication, в которую можно сдавать только zip архивы. Поместите тесты и решение в архив следующей командой:

```console
$ zip -r solution.zip test.py solution/
```

Добавьте  в папку `solution` файл `readme.md` с кратким описанием вашего решения и дополнительными комментариями, который также попадет в архив. Также, если вы добавляете свои тесты, то напишите к ним docstring - что проверяется в тесте и каким образом. **В случае отсутствия описаний решения и тестов оценка может быть снижена на 1 балл**.

Далее сдайте Ваш `solution.zip`. Дождитесь пока отработают Ваши тесты на Вашем решении (учтите, что это может занять несколько минут из-за нагрузки серверов). Если все тесты прошли успешно, то решение проставляется 2 балла и решение принимается на ручную проверку. Если какие-то тесты не прошли, то выставляется 0 баллов, и решение не принимается на ручную проверку.

Если Вы не смогли что-то реализовать, можете закомментировать некоторые тесты в [test.py](./test.py), но обязательно отразите почему Вы это сделали в readme-файле. Это сильно упростит нам проверку.

Дедлайн задачи -- __2 недели__, в Gradescope корректно проставлена дата окончания.