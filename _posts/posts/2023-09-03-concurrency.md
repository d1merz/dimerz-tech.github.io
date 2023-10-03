---
layout: post
title: "Concurrency на примере Rust Tokio"
date: 2023-09-03 20:28:39 +0300
categories: [posts, linux, rust]
---

![concurrency](../../images/posts/concurrency/epoll.png)

## Проблема
Приложениям, работающим с большими потоками данных, UI интерфейсами, сетевыми запросами необходимо постоянно ожидать 
поступление данных или событий. В этом время остальные части приложений могут быть заблокированы и ждать наступления 
тех или иных событий. 

![concurrency](../../images/posts/concurrency/wQvYdj8UNDn0Klz77JbmUw_store_header_image.png)

## Инструменты и ресурсы для решения
* 💎 **Несколько физических ядер**
* ⛓️ **Несколько потоков (threads)**
* ♻️ **Мультиплексирование**

Все вышеперечисленные методы можно и нужно комбинировать между собой.

## Concurrency vs Multithreading
![concurrency](../../images/posts/concurrency/nginx-vs-apache.png)
Поток с точки зрения ресурсов памяти и процессорного времени очень тяжелая сущность. Переключение между потоками на одном
ядре занимает много тактов, поэтому, как правило, рекомендуется создавать число потоков = числу физических ядер. 
Так работал `Apache`. Он появился очень давно, когда сервера были не такими загруженными, однако со временем появилась
[проблема 10-ти тысяч соединеий](https://en.wikipedia.org/wiki/C10k_problem). Как решать такую проблему? Масштабироваться 
за счет железа, что достаточно дорого. `Nginx` предложил свой подход, который позволил совершить прорыв. Из документации:

* **worker_processes** – The number of NGINX worker processes (the default is 1). 
  In most cases, running one worker process per CPU core works well, and we recommend setting this directive to auto to achieve that. 
  There are times when you may want to increase this number, such as when the worker processes have to do a lot of disk I/O.
* **worker_connections** – The maximum number of connections that each worker process can handle simultaneously. 
  The default is 512, but most systems have enough resources to support a larger number. 
  The appropriate setting depends on the size of the server and the nature of the traffic, and can be discovered through testing.

То есть минимальное количество соединений на одном физическом ядре равно 512. Подчеркну, <ins>минимальное</ins>. Если мы возьмем сервер
вроде Intel Xeon с 32 ядрами, то в самом худшем случае одновременно мы сможем поддерживать более 16к соединений на одной машине.
Как Nginx добился такой производительности ? Благодаря `concurrency` подходу.

### Multiplexing
В отличие от многопоточности, где процессы действительно работают одновременно и независимо друг от друга, мультиплексирование позволяет более
хитро использовать ресурсы одного ядра и создавать иллюзию одновременного выполнения.
![concurrency](../../images/posts/concurrency/6574.1623908671.jpg)

В случае с Nginx разработчики использовали модуль ядра [epoll](https://man7.org/linux/man-pages/man7/epoll.7.html). Основная идея его работы
заключается в том, чтобы попросить ядро разбудить процесс, если возникнет какое-то интересное для него событие. Сам по себе вызов 
`epoll_wait()` является блокирующим. Очень вкратце есть несколько стратегий его использования:

* **level-triggered**
* **edge-triggered**

Также есть 2 вида файловых дескрипторов:

* **blocking**
* **non-blocking**

Общая концепция вкратце:

* Вызов функций `read()\write()` заблокирует `thread` до тех пор, пока данные не поступили в `blocking fd`
* В случае `non-blocking fd` вызов функций `read()\write()` не заблокирует `thread`, а сразу же вернет control flow, 
  даже если читать/писать было нечего
* `epoll` с `level-triggered` стратегией будет каждый раз возвращать дескрипторы, если они могут быть использованы 
  (например, в них есть `data`). При этом вызов `epoll_wait` в случае, если ни на одном из интересующих дескрипторов
  нет событий (данных) заблокирует `thread`. Например, при чтении из сокета, если в сокете остались данные, `epoll` будет
  снова и снова (при каждом вызове) сообщать нам об этом. 
* `epoll` с `edge-triggered` стратегией будет разблокирован единожды по событию. Если мы не дочитали данные из сокета,
  очередной вызов `epoll_wait` будет блокировать процесс, пока не придет новая порция данных. Данный вызов можно использовать
  только с неблокирующими сокетами, чтобы сразу же вернуть состояние.

## Async/await на примере Tokio
Скоро будет...


## Полезные ссылки
* [Про Nginx](https://www.nginx.com/blog/tuning-nginx/)
* [Apache vs Nginx](https://gohost.kz/blog/hosting/chto-luchshe-nginx-vs-apache/)
* [Очень хорошо про epoll](https://habr.com/ru/articles/416669/)