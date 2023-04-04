---
layout: post
title: "Best time to buy and sell stock"
date: 2023-03-28 20:28:39 +0300
categories: leetcode, dp   
---

Всем привет! Недавно решал серию интересных задач на динамику. Интересными мне эти задачи показались, 
так как решение очередной задачи вытекает из предыдущей, а самая последняя (и сложная) объединяет в себе 3 предыдущих и решается в одну строчку :slightly_smiling_face:.

## Best time to buy and sell stock 1
[Условие](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)

Эта задача относится к разделу `easy` и очевидное решение напрашивается само с собой - нужно найти 2 таких дня, 
чтобы разница между стоимостью акций в эти дни была максимальна (в положительную сторону, конечно), при этом минимальное 
значение (цена покупки) находилось левее, чем максимум (цена продажи). Так как задача не решается за квадратичное время
(см. ограничение $$1<=prices.length<=2*10^5$$), это дает нам сразу подсказку, что решать нужно за один проход (или как максимум за $$nlog(n)$$). 
Когда я вижу один проход и есть еще некоторая последовательность, я сразу думаю в сторону динамики. Давайте попытаемся определить, что может являться состоянием в этой задаче.
Очевидно, состояния здесь 2: либо у нас есть акция (и мы можем ее продать), либо нет (и мы можем ее купить). Сразу обозначим соответственно: True, False. 
Осталось придумать мемоизацию (переход) из состояния в состояние на основе уже посчитанных значений.
Я считаю это самым сложным в динамическом программировании и обычно начинаю рассматривать какие-то мелкие или крайние случаи. Давайте рассмотрим ситуацию, когда у нас есть 2 дня:
`[1, 10]`. В данном случае, очевидно, что купив по цене 1 и продав при цене 10 мы заработаем, поэтому наши состояния и их значения для каждого из дней будут описываться как:

`d[0][True] = -1` - купили в 0й день и заработали -1
`d[1][False] = -1 + 10 = 9` - продали во второй день, получили прибыль 9

А может, ли быть такое, чтобы мы не купили, а потом продали? Нет! Поэтому у нас вырисовывается правило, что в состояние False (то есть продажи) мы можем перейти только из состояния True.
При этом, по условию задачи, мы можем купить и продать только 1 раз. Поэтому если мы совершили покупку, то еще раз мы ее совершить не можем. Тогда в состояние True мы можем перейти только
из состояния False. Немного расширим наш кейс:
`[1, 10, 13]`
И запишем всевозможные значения:

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-dztg{background-color:#c0c0c0;color:#c0c0c0;text-align:center;vertical-align:top}
.tg .tg-vxga{background-color:#ffffff;text-align:center;vertical-align:middle}
.tg .tg-p1mt{background-color:#c0c0c0;color:#000000;text-align:center;vertical-align:middle}
.tg .tg-6qw1{background-color:#c0c0c0;text-align:center;vertical-align:top}
.tg .tg-nrix{text-align:center;vertical-align:middle}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-dztg"></th>
    <th class="tg-6qw1">Day 0</th>
    <th class="tg-6qw1">Day 1</th>
    <th class="tg-6qw1">Day 2</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-p1mt">True</td>
    <td class="tg-nrix">-1 (купили)</td>
    <td class="tg-vxga">-1(купили в Day 0)<br>-10 (купили в&nbsp;&nbsp;Day 1)</td>
    <td class="tg-nrix">-1(купили в Day 0)<br>-10 (купили в Day 1)<br>-13 (купили в Day2)</td>
  </tr>
  <tr>
    <td class="tg-p1mt">False</td>
    <td class="tg-nrix">0 (ждем)</td>
    <td class="tg-nrix"><span style="font-weight:400;font-style:normal;text-decoration:none">0 (ждем)</span><br>+9(продали)</td>
    <td class="tg-nrix"><span style="font-weight:400;font-style:normal;text-decoration:none">0 (ждем)</span><br>+9(продали в Day 1)<br>+13(продали в Day2)<br></td>
  </tr>
</tbody>
</table>

Таблица показывает нам, что для каждого дня возможно несколько состояний в зависимости от предыдущих действий. Причем чем больше последовательность, тем больше состояний
генерируется. Нам нужна оптимизация, и интуиция должна подсказывать, что для очередного дня мы должны выбирать наиболее выгодный результат из всех возможных. То есть в нашем случае 
максимум из всех потенциальных значений. Тогда правило перехода будет такое:
* В состояние True можно попасть только их состояния False
* В состояние False можно попасть только из состояния True
* Мы должны выбирать оптимальное действие (купить/продать или ждать), которое приводит к максимальной прибыли

Формально:
`d[i][True] = max(d[i-1][False] - price[i], d[i-1][True])`
`d[i][False] = max(d[i-1][True] + price[i], d[i-1][False])`

Теперь взглянем на нашу таблицу отбросив неоптимальные случаи:
<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-dztg{background-color:#c0c0c0;color:#c0c0c0;text-align:center;vertical-align:top}
.tg .tg-vxga{background-color:#ffffff;text-align:center;vertical-align:middle}
.tg .tg-p1mt{background-color:#c0c0c0;color:#000000;text-align:center;vertical-align:middle}
.tg .tg-6qw1{background-color:#c0c0c0;text-align:center;vertical-align:top}
.tg .tg-nrix{text-align:center;vertical-align:middle}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-dztg"></th>
    <th class="tg-6qw1">Day 0</th>
    <th class="tg-6qw1">Day 1</th>
    <th class="tg-6qw1">Day 2</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-p1mt">True</td>
    <td class="tg-nrix">-1 (купили)</td>
    <td class="tg-vxga">-1(купили в Day 0)</td>
    <td class="tg-nrix">-1(купили в Day 0)</td>
  </tr>
  <tr>
    <td class="tg-p1mt">False</td>
    <td class="tg-nrix">0 (ждем)</td>
    <td class="tg-nrix">+9(продали)</td>
    <td class="tg-nrix">+13(продали в Day2)</td>
  </tr>
</tbody>
</table>

На мой взгляд, это очень красивая задача, которая показывает всю суть динамики - сначала сгенерировать всевозможные варианты, а потом выбрать оптимальное значение внутри каждого состояния.
Ну и ответом конечно будет являться максимум из всех возможных значений. Мое решение:

```python
class Solution:
    def maxProfit(self, prices: List[int]) -> int:
        d = [{True:0, False: 0}] * len(prices)
        d[0][True] = -prices[0] // начальное значение
        d[0][False] = 0 // начальное значение
        for i in range(1, len(prices)):
            d[i][True] = max(d[i-1][False] - prices[i], d[i-1][True])
            d[i][False] = max(d[i-1][True] + prices[i], d[i-1][False])
        max(d, key=lambda x: max(x))
```

## Best time to buy and sell stock 2
[Условие](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii/)

Данная задача расширяет предыдущую - теперь мы можем покупать и продавать несколько раз. Но все также мы не можем иметь на руках более одной акции. Концептуально данная задача
ничем не отличается от предыдущей, у нас все также сохраняются состояния, и переходы между ними поддаются той же логике. Давайте возьмем пример из условия `[7,1,5,3,6,4]`. 
Применим к нему наш алгоритм:

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-dztg{background-color:#c0c0c0;color:#c0c0c0;text-align:center;vertical-align:top}
.tg .tg-vxga{background-color:#ffffff;text-align:center;vertical-align:middle}
.tg .tg-baqh{text-align:center;vertical-align:top}
.tg .tg-p1mt{background-color:#c0c0c0;color:#000000;text-align:center;vertical-align:middle}
.tg .tg-6qw1{background-color:#c0c0c0;text-align:center;vertical-align:top}
.tg .tg-y6fn{background-color:#c0c0c0;text-align:left;vertical-align:top}
.tg .tg-nrix{text-align:center;vertical-align:middle}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-dztg"></th>
    <th class="tg-6qw1">Day 0</th>
    <th class="tg-6qw1">Day 1</th>
    <th class="tg-6qw1"><span style="font-weight:400;font-style:normal;text-decoration:none">Day 2</span></th>
    <th class="tg-y6fn"><span style="font-weight:400;font-style:normal;text-decoration:none">Day 3</span></th>
    <th class="tg-y6fn"><span style="font-weight:400;font-style:normal;text-decoration:none">Day 4</span></th>
    <th class="tg-y6fn"><span style="font-weight:400;font-style:normal;text-decoration:none">Day 5</span></th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-p1mt">True</td>
    <td class="tg-nrix">-7</td>
    <td class="tg-vxga">-1</td>
    <td class="tg-nrix">-1</td>
    <td class="tg-baqh">1</td>
    <td class="tg-baqh">1</td>
    <td class="tg-baqh">3</td>
  </tr>
  <tr>
    <td class="tg-p1mt">False</td>
    <td class="tg-nrix">0</td>
    <td class="tg-nrix">0</td>
    <td class="tg-nrix">4</td>
    <td class="tg-baqh">4</td>
    <td class="tg-baqh">7</td>
    <td class="tg-baqh">7</td>
  </tr>
</tbody>
</table>

Как видим, с каждым новым днем, мы можем отмечать рост прибыли, а значит максимум от продаж мы получим в самый последний день. Это и есть решение данной задачи:

```python
class Solution:
    def maxProfit(self, prices: List[int]) -> int:
        d = [{True: 0, False: 0} for _ in range(len(prices))]
        d[0][True] = -prices[0]
        d[0][False] = 0
        for i in range(1, len(prices)):
            d[i][True] = max(d[i - 1][False] - prices[i], d[i - 1][True])
            d[i][False] = max(d[i - 1][True] + prices[i], d[i - 1][False])
        return max(d[-1][True], d[-1][False])
```

## Best time to buy and sell stock 3
[Условие](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iii/)

В третьей задаче вводится ограничение на количество транзакций - их можно сделать максимум 2. Самое очевидное, что пришло лично мне в голову это ввести новые состояния:
* 0 - ничего не купили
* 1 - купили
* 2 - продали
* 3 - купили 2й раз
* 4 - продали 2й раз

Очевидно как изменятся наши правила:
* Из состояния 0 можно перейти только в состояние 1, либо остаться в 0
* Из состояния 1 можно перейти только в состояние 2, либо остаться в 1
* Из состояния 2 можно перейти только в состояние 3, либо остаться в 2
* Из состояния 3 можно перейти только в состояние 4, либо остаться в 3

Ответом будет опять же является ответ, полученный в последний день (так как прибыль накапливается аналогично прошлой задаче):

```python
class Solution:
    def maxProfit(self, prices: List[int]) -> int:
        # Задаем какие-то маленькие значения для 5 состояний, которые точно меньше любой отрицательной прибыли
        d = [[-1000000000]*5 for _ in range(len(prices))]
        d[0][0] = 0
        d[0][1] = -prices[0]
        for i in range(1, len(prices)):
            d[i][0] = 0 # ничего не купили/не продали на текущий момент
            d[i][1] = max(d[i - 1][0] - prices[i], d[i - 1][1]) # купили первый раз
            d[i][2] = max(d[i - 1][1] + prices[i], d[i - 1][2]) # продали первый раз
            d[i][3] = max(d[i - 1][2] - prices[i], d[i - 1][3]) # купили второй раз
            d[i][4] = max(d[i - 1][3] + prices[i], d[i - 1][4]) # продали второй раз
        return max(d[-1])
```

## Best time to buy and sell stock 4
[Условие](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iv/)

Ну и наконец в последней задаче нам предлагается вместо 2х возможных транзакций выполнять `k`. И если внимательно посмотреть на задачу 3, 
то можно увидеть некоторую общую формулу, которая красиво выливается в решение текущей задачи:
```python
d[i][j] = max(d[i - 1][j-1] - prices[i], d[i - 1][j]) if j % 2 != 0 else max(d[i - 1][j - 1] + prices[i], d[i - 1][j])
```

В этом и есть магия динамики, так как порой простое и рациональное решение можно получить лишь спустя нескольких более грубых и сложных логических выкладок.

Прилагаю свое решение:

```python
class Solution:
    def maxProfit(self, k: int, prices: List[int]) -> int:
        d = [[-100000]*(2*k+1) for _ in range(len(prices))]
        d[0][0] = 0
        d[0][1] = -prices[0]
        for i in range(1, len(prices)):
            d[i][0] = 0
            for j in range(1, 2*k + 1):
                d[i][j] = max(d[i - 1][j-1] - prices[i], d[i - 1][j]) if j % 2 != 0 else max(d[i - 1][j - 1] + prices[i], d[i - 1][j])
        return max(d[-1])
```