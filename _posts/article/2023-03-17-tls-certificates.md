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
.tg .tg-0lax{text-align:left;vertical-align:top}
.tg .tg-ut5e{background-color:#c0c0c0;border-color:inherit;color:#333333;text-align:center;vertical-align:top}
.tg .tg-0pky{border-color:inherit;text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0lax"></th>
    <th class="tg-ut5e">Симметричное</th>
    <th class="tg-ut5e">Асимметричное</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0lax">Скорость, нагрузка на CPU</td>
    <td class="tg-d52n">Быстро (короткий ключ)</td>
    <td class="tg-0pky">Медленно (длинный ключ)</td>
  </tr>
  <tr>
    <td class="tg-0lax"> Секьюрность</td>
    <td class="tg-0pky">Небезопасно (нужно обмениваться общим ключом)</td>
    <td class="tg-d52n">Безопасно (обмен происходит только открытым ключом)</td>
  </tr>
  <tr>
    <td class="tg-0lax">Алгоритмы</td>
    <td class="tg-0lax">DES, RC4, 3DES, AES, ChaCha20</td>
    <td class="tg-0lax">DSA, RSA, Diffie-Helman, ECDSA, ECDH</td>
  </tr>
</tbody>
</table>

Как видно, кажется, что универсального решения нет, но на самом деле есть. И называется оно гибридным шифрованием. Это когда мы генерируем общий (быстрый) ключ, а передаем его секьюрно
с помощью закрытого и открытого ключа. Получается, что общий ключ не подвержен перехвату (грубо), а последующее шифрование им данных происходит быстро.

Давайте сразу проверим это на практике, разыграем реальный сценарий и попробуем разные схемы шифрования и дешифрования.
### Шифрование симметричным ключом
Пусть Алисе нужно попросить Боба перевести ей деньги на банковский счет. Для этого она хочет послать сообщение: "Боб, переведи мне деньги на счет 7777 7777 7777 7777". Формируем такой файл:
```bash
echo "Боб, переведи мне деньги на счет 7777 7777 7777 7777" > message
```

Теперь нам нужно выбрать алгоритм шифрования и общий ключ. Допустим Алиса и Боб договорились, что будут использовать `AES-256` с ключом `BOBALICE`. 
Алиса шифрует сообщение с помощью ее любимой и самой распространенной утилиты-библиотеки [openssl](https://en.wikipedia.org/wiki/OpenSSL):
```bash
openssl aes-256-cbc -a -in message -out message.enc -pass pass:BOBALICE
```

Я намеренно выбрал самый простой способ использования openssl с наименьшим количеством настроек. Итак, рассмотрим флаги:
* `aes-256-cbc` - выбранный нами алгоритм шифрования
* `-a` означает, что нужно использовать формат выходного файла base64 (это позволит нам открыть его в текстовом редакторе)
* `-in` - наше сообщение, которое мы подаем на вход
* `-out` - base64-encoded зашифрованный файл
* `-pass pass:` - наш общий секретный ключ

Итак, посмотрим что у нас получилось:
```bash
cat message.enc
U2FsdGVkX1879r+wdwV5VqsrW40pSUmoSt9vWRCIdKRQBa+KterWsEi/Yzf2qf04
5/a+KyphdV723/l9fVEooyl34nAftACIxLzZvfWAyByfu3qhB2jLk1GbBeSMs39z
```

Теперь мы можем отправить этот файлик Бобу, зная, что кроме него никто не знает наш общий секретный ключ. Боб расшифровывает файл:

```bash
openssl aes-256-cbc -d -a -in message.enc -out message.dec -pass pass:BOBALICE
```
* `aes-256-cbc` - выбранный нами алгоритм шифрования
* `-a` означает, что нужно использовать формат входного файла base64
* `-in` - зашифрованное сообщение, которое мы подаем на вход
* `-out` - дешифрованное сообщение
* `-pass pass:` - наш общий секретный ключ

Давайте убедимся, что сообщение расшифровано успешно:
```bash
cat message.dec
Боб, переведи мне деньги на счет 7777 7777 7777 7777
```

Вот так просто мы реализовали достаточно секьюрную передачу данных между Алисой и Бобом. Тут конечно стоит сказать, что на самом деле фраза `BOBALICE` не является ключом сама по себе.
На основании ее openssl строит этот самый ключ, а сделано это для нашего с вами удобства, чтобы мы не думали о генерации 256 символов (на самом деле генерация сложно подбираемого ключа это 
тема для огромной статьи). Но суть от этого не меняется - `openssl` за нас строит такой ключ, базируясь на придуманном нами пароле.

### Гибридное шифрование
Все это время в общении Боба и Алисы отсутствовал один важный персонаж - злоумышленник Ева:
![alt text](../../images/leetcode/tls-certificates/image1.png)

Когда Алиса сообщила Бобу их общий ключ, Ева подслушала его и смогла расшифровать сообщение. Узнав об этом, Алиса и Боб решили действовать хитрее, 
применив ассиметричное шифрование для шифра их общего ключа. Для этого они решили использовать самый популярный алгоритм `RSA`. Для начала уже Боб генерирует пару ключей:

```bash
openssl genrsa -out bob.private 2048
openssl rsa -in bob.private -outform PEM -pubout -out bob.public
```

Здесь мы сначала генерируем 2048-битный приватный ключ (который кроме Боба знать никто не может), а затем на его основе генерируется публичный ключ (который можно показывать всем). 
`-outform PEM` указывает на хранение ключа в PEM формате, о чем мы поговорим в разделе про сертификаты. Посмотрим, что у нас получилось:

```bash
cat bob.private
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAu0LVuZC0Kld9izrNX6H++gFzbc6b9j3cefnZqsqmwuHcgIuO
o0TNxcf9nfRmETMFG1WzXbSXgSh/cjOAkNdYDc1aC3sGTrW8aP6KVN8q6hzryyY4
rmNFGpkt+fJLIYIqQtkt9+A1l01bpRhphfMVFodAvRnBbDyZ0y2XiqLkmPdQlG+s
f+UtOEb5QYdKrzl0bSIVWLoBqUnvn6Xy8R+zUzVYSwHqMenKH1wG72xnnZavb2HW
qO2+AJXomdrANBkCILfqyM7J+20lIeAOUvc1ljtmb0t7AVo0wNlic1vDiJhomJ7m
lRBi3x+mOjg19LIKX8+aPWal1BCyT2t8KFh0lwIDAQABAoIBAC6cl0iGbWIwvu4+
J8rLLwWaANpvWED16wBtobGymcneoJ1an5miUboOauG7aQCU+4tvTKNhhWi/5OFP
Uow7Hg7pc6Vx7QMAl7Jm0ZYIqsz1f/fkPq2hxhy1zkzt86eWl+Bl4NxXcd+EgPBa
YEHYXQJrjkDDvXKsrvly2srGezJbyDAhVvi6dGl30IYRSroWXrSgzrVsZKW45yl2
T/o1OBhwKxsiY22Cj7Zl30Z7mrneudlBEQKw3bgLmhIBXxluS0UMcTntLfJQuZky
5NcgJN4pNeFeISkFDs6/o5qB2j7E9P+NMdFIqQYYNu4LwZvGqWNjIoL8fRQek7S0
LxXhW+kCgYEA3W7gqD4fMsM6bOec9oRbaiHBprjvW9i4ESuBErzudcqU2NeMP7Le
8buWi8Ii4CzJvnVIsGi5uXPYobklcemR8R2nxK7JzaWX+LOa0ENGHcuDJ33tMaW4
ejLjIwbkprRddy2uAP9h13b0d78PI+J7O1ufxYaTBx2S0HPmCVPDlSsCgYEA2H5X
ZiYZkqIttSurw5TzVsAz+/eruzvetCfCPQUUMBcVew1ebnjcB5BEgWbFyHMFmPSH
T2bTW63Anh8Qt0fKdPiVrSJYmZpSd8YoxlLHlMQkowYVOjI26L35ud+5JxUeHl7G
oTfsnwruy2Qw2BO/bexHNw9x7hQbHPv96U/DwEUCgYEAqlb0LZh0SYmTdVmLDrxs
0i0bmlSjlSUXG+Y6iZ6bVjO74z+VXR8NkCm9Os/iIk9x9rikbUhyru4mFOqnmoSS
R0iH2a6YAUHEsYYArvtBggQfZY7im5GfYymoPza7SF5xlA+ozo2mx6hzkXaBgBpm
sBfgxYOrsQqX+RFfQYeVajcCgYBL5SbAkbS/WgmKDNA/VXVlQSY4O+BBWEC/iXyn
paMpJL6MD+i987K3GBVcA+psVh5tO7Q/2r1X03bGN//FMO7ByO+37WWAG8Czao2a
0SRUe9qYKCbC6x2ixA7K4apoEBgp73HUzOUv8EY8mVgiN/IRGBbzL3/yRawnohOm
eicWvQKBgEKQ4XPW3mmyvTVf9h72c8CygTq3mZiBhlczxTDvZdf37EznI1E0LRGH
OiwjKO4NOy14TlwUtKijWczJtVJRPqoOYiTMVUB4OkKiYVfvRF7u7IS3r4WeuNcY
Ei/VuLLS8nAKpITuQ5FJOaASFuDbIzZsqZ7fl26YxRIh/uidEJML
-----END RSA PRIVATE KEY-----
```
Это тот самый секретный ключ, доступ к которому злоумышленник может получить, разве что при физическом доступе к машине Боба, в рамках нашей задаче будем считать, что это невозможно.

```bash
cat bob.public
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAu0LVuZC0Kld9izrNX6H+
+gFzbc6b9j3cefnZqsqmwuHcgIuOo0TNxcf9nfRmETMFG1WzXbSXgSh/cjOAkNdY
Dc1aC3sGTrW8aP6KVN8q6hzryyY4rmNFGpkt+fJLIYIqQtkt9+A1l01bpRhphfMV
FodAvRnBbDyZ0y2XiqLkmPdQlG+sf+UtOEb5QYdKrzl0bSIVWLoBqUnvn6Xy8R+z
UzVYSwHqMenKH1wG72xnnZavb2HWqO2+AJXomdrANBkCILfqyM7J+20lIeAOUvc1
ljtmb0t7AVo0wNlic1vDiJhomJ7mlRBi3x+mOjg19LIKX8+aPWal1BCyT2t8KFh0
lwIDAQAB
-----END PUBLIC KEY-----
```
А вот это уже открытый для всех желающих публичный ключ Боба. Теперь Боб спокойно пересылает его Алисе. И даже если Ева перехватит этот ключ, толку от него никакого.
Алиса же на своей стороне шифрует этим ключем их общий пароль BOBALICE:
```bash
echo "BOBALICE" > password
openssl rsautl -encrypt -inkey bob.public -pubin -in password -out password.enc
```
* `rsautl` - утилита библиотеки openssl для шифровки дешифровки
* `-encrypt` говорит о том, что необходимо зашифровать файл
* `-inkey` - публичный ключ Боба
* `-pubin -in` - файл для шифрования
* `-out` - зашифрованный файл с общим паролем

Теперь Алиса спокойно отправляет по открытой сети файл password.enc и даже если Ева перехватит это сообщение, то расшифровать она его не сможет, потому что закрытый ключ находится на машине Боба.
Боб же, получив сообщение от Алисы, расшифровывает его с помощью своего приватного ключа:
```bash
openssl rsautl -decrypt -inkey bob.private -in password.enc
BOBALICE
```
А дальше задача общения между Бобом и Алисой сводится к уже решенной симметричным шифрованием. На самом деле, Алиса могла бы сгенерировать свою пару ключей и они могли бы общаться
посредством ассиметричного шифра, но как мы уже сказали ранее это более процессорно-затратно. Получается, что для Евы потерян смысл слушать канал между Бобом и Алисой и кажется задача
безопасной передачи данных решена, однако нет...

## ЭЦП
На самом деле, всем известна проблема отсутствия аутентификации при таком подходе. Такое может произойти, если злоумышленник узнает о планах Боба и Алисы
пообщаться и напишет Алисе первым, представившись Бобом. Тогда Алиса передаст ему свой открытый ключ, и у них начнется обмен сообщениями :heart_eyes:.
Для этого Алиса должна как то понимать, что пишет ей именно Боб. Для этого была придумана электронная цифровая подпись.

Электронная подпись появилась благодаря внезапному свойству ассиметричных ключей быть симметричными :clown_face:. То есть то, <ins> что шифрует открытый ключ, может быть расшифровано закрытым, 
а то что шифрует закрытый ключ может быть расшифровано открытым </ins>. 
