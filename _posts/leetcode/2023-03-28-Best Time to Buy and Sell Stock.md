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

На самом деле, это очень красивая задача, которая показывает всю красоту динамики - сначала сгенерировать всевозможные варианты, а потом выбрать оптимальное значение внутри каждого состояния.
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

