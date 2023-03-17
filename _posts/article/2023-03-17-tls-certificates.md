---
layout: post
title: "PKI, сертификаты и все все все..."
date: 2023-03-17 20:28:39 +0300
categories: article
---

Цель написания данной статьи - наконец раз и навсегда агрегировать и рассмотреть все бытовые случаи использования сертификатов, ключей, электронных подписей
а самое главное - понять, как эти технологии взаимодействуют друг с другом и каждый день предоставляют нам возможность безопасно пользоваться интернетом.

Наверное каждый, кто сталкивался и пытался разобраться с такими протоколами как `https`, `tls/ssl`, `ssh`... видел на просторах интернета замечательную историю,
как [Боб и Алиса](https://ru.wikipedia.org/wiki/Алиса_и_Боб) хотят обменяться сообщениями по закрытому каналу. Поэтому повторять уже сказанное десятки раз я не буду,
мой манускрипт больше про те аспекты, которые почему-то обычно остаются за кадром.

# Виды шифрований

Итак, мы знаем, что Бобу необходимо зашифровать сообщение Алисе, чтобы [злоумышленник](https://ru.wikipedia.org/wiki/Атака_посредника) при перехвате сообщения ничего в нем не понял.
Существует 2 вида шифрований:

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-d52n{background-color:#32cb00;border-color:inherit;text-align:left;vertical-align:top}
.tg .tg-s3wk{background-color:#c0c0c0;border-color:inherit;color:#333333;text-align:left;vertical-align:top}
.tg .tg-0pky{border-color:inherit;text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-s3wk">Симметричное</th>
    <th class="tg-s3wk">Асимметричное</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-d52n">Быстро</td>
    <td class="tg-0pky">Медленно</td>
  </tr>
  <tr>
    <td class="tg-0pky">Небезопасно</td>
    <td class="tg-d52n">Безопасно</td>
  </tr>
</tbody>
</table>

