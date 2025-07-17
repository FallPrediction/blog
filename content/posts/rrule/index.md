+++
title = '簡單分享 PHP RRule 套件處理重複事件'
date = 2025-07-12T22:44:52+08:00
draft = false
description = '玩過月曆相關功能的人應該都遇過類似的需求，重複事件，例如週會、每月公休日等，有沒有通用的格式可以表示呢？簡單分享 RRule'
toc = true
tags = ['RRule', 'RFC 5545', 'PHP']
categories = ['RRule', 'RFC 5545', 'PHP']
+++

# 前言
玩過月曆相關功能的人應該都遇過類似的需求，重複事件，例如週會、每月公休日等，有沒有通用的格式可以表示呢？其實有的，而且很多月曆相關的套件，例如 full calendar，都支援這格式

日曆數據交換標準 iCalendar（[RFC 5545](https://www.rfc-editor.org/rfc/rfc5545.html)）中的 3.3.10.  Recurrence Rule

它可以用特定格式的字串表示這些重複事件，例如：
```
每個禮拜日
RRULE:FREQ=MONTHLY;BYDAY=SU

每個月第 1 3 5 個禮拜五
RRULE:FREQ=MONTHLY;BYDAY=1FR,3FR,5FR

每兩個禮拜六
RRULE:INTERVAL=2;FREQ=WEEKLY;BYDAY=SA

每週六日共 10 次
RRULE:FREQ=MONTHLY;BYDAY=SA,SU;COUNT=10

每年 12/24 23:58:40（也就是畢律斯鐘樓平安夜敲鐘的日子！）
FREQ=YEARLY;BYMONTHDAY=24;BYHOUR=23;BYMINUTE=58;BYSECOND=40
```

另外需要設定開始時間 `DTSTART`，有三種表示方法
1. DTSTART:19970714T133000 本地時間
2. DTSTART:19970714T173000Z UTC 時間
3. DTSTART;TZID=America/New_York:19970714T133000 指定時區

除此之外也可以設定結束時間 `UNTIL`，格式跟 `DTSTART` 一樣

更多例子可以看 [章節 3.8.5.3](https://www.rfc-editor.org/rfc/rfc5545.html#section-3.8.5.3)

# RRULE for PHP
許多語言都有好用的 RRule 套件可以使用，本文要介紹的是 [php-rrule](https://github.com/rlanvin/php-rrule)

假設我開一間早餐店，每週一休息，那我的公休日可以設為
```php
$rrule = new RRule([
    'FREQ' => 'weekly',
    'BYDAY' => 'MO',
]);
echo $rrule; // FREQ=WEEKLY;BYDAY=MO
```

現在我想取得未來 5 個休息日
```php
$occurrences = $rrule->getOccurrences(5);
foreach ($occurrences as $occurrence) {
    // 每個 $occurrence 都是 DateTime 物件
    echo $occurrence->format('r'), "\n";
}
```

output:
```
Mon, 07 Jul 2025 10:18:36 +0000
Mon, 14 Jul 2025 10:18:36 +0000
Mon, 21 Jul 2025 10:18:36 +0000
Mon, 28 Jul 2025 10:18:36 +0000
Mon, 04 Aug 2025 10:18:36 +0000
```

## 設定 DTSTART
指定 `DTSTART`，如果沒指定，會以現在時間代替
```php
$rrule = new RRule([
    'FREQ' => 'weekly',
    'BYDAY' => 'MO',
    'DTSTART' => '20250801T090000',
]);
```

## 加上指定日期
如果我有一天要去看表演，需要臨時休息呢？
我們可以使用 `RSet` class，包含了 `RRule`、指定日期 `RDATE`、排除規則 `EXRULE` 和排除日期 `EXDATE`，然後用前述的方式取得休息日

`RDATE` 表示方式如下
```
指定日期，8/11, 12, 15
RDATE;VALUE=DATE:20250811,20250812,20250815
```

接下來建立 `RSet`，並加入 `RRule` 和 `RDate`
```php
$rset = new RSet();
$rset->addRRule([
    'FREQ' => 'weekly',
    'BYDAY' => 'MO',
    'DTSTART' => '20250801T090000',
]);
$rset->addDate('20250811T090000');
$rset->addDate('20250812T090000');
$rset->addDate('20250815T090000');
$occurrences = $rset->getOccurrences(5);
foreach ($occurrences as $occurrence) {
    echo $occurrence->format('r'), "\n";
}
```

output:
```
Mon, 04 Aug 2025 09:00:00 +0000
Mon, 11 Aug 2025 09:00:00 +0000
Tue, 12 Aug 2025 09:00:00 +0000
Fri, 15 Aug 2025 09:00:00 +0000
Mon, 18 Aug 2025 09:00:00 +0000
```
可以看到 8/11 本來就是週一公休日，即使再加上了指定日期，取得的結果中 8/11 依然只有一筆，不會有重複

需要注意的是，每條 `$occurrence` 都是 DateTime 物件，所以如果 `RRule` 和 `RDate` 是不同時間，即使日期相同也會出現兩條項目
```php
$rset = new RSet();
$rset->addRRule([
    'FREQ' => 'weekly',
    'BYDAY' => 'MO',
    'DTSTART' => '20250801T090000',
]);
$rset->addDate('20250811');
$occurrences = $rset->getOccurrences(5);
foreach ($occurrences as $occurrence) {
    echo $occurrence->format('r'), "\n";
}
```

output:
```
Mon, 04 Aug 2025 09:00:00 +0000
Mon, 11 Aug 2025 00:00:00 +0000 <- 8/11 00:00
Mon, 11 Aug 2025 09:00:00 +0000 <- 8/11 09:00
Mon, 18 Aug 2025 09:00:00 +0000
Mon, 25 Aug 2025 09:00:00 +0000
```

## 排除重複項和指定日期
使用 `addExRrule()` 和 `addExDate()` 增加排除規則，格式增加規則相同
假設我們週一和週二公休，8/8 父親節多休一天，但 8/11 多上一天班，並且九月公休日只有週一
```php
$rset = new RSet();
$rset->addRRule([
    'FREQ' => 'weekly',
    'BYDAY' => 'MO,TU',
    'DTSTART' => '20250801',
]);
$rset->addDate('20250808');
$rset->addExDate('20250811');
$rset->addExRule([
    'FREQ' => 'weekly',
    'BYDAY' => 'TU',
    'DTSTART' => '20250901',
    'UNTIL' => '20250930',
]);
$occurrences = $rset->getOccurrences(15);
foreach ($occurrences as $occurrence) {
    echo $occurrence->format('r'), "\n";
}
```

output:
```
Mon, 04 Aug 2025 00:00:00 +0000
Tue, 05 Aug 2025 00:00:00 +0000
Fri, 08 Aug 2025 00:00:00 +0000
Tue, 12 Aug 2025 00:00:00 +0000
Mon, 18 Aug 2025 00:00:00 +0000
Tue, 19 Aug 2025 00:00:00 +0000
Mon, 25 Aug 2025 00:00:00 +0000
Tue, 26 Aug 2025 00:00:00 +0000
Mon, 01 Sep 2025 00:00:00 +0000
Mon, 08 Sep 2025 00:00:00 +0000
Mon, 15 Sep 2025 00:00:00 +0000
Mon, 22 Sep 2025 00:00:00 +0000
Mon, 29 Sep 2025 00:00:00 +0000
Mon, 06 Oct 2025 00:00:00 +0000
Tue, 07 Oct 2025 00:00:00 +0000
```

## 其他功能
檢查某日期是否在 RSet 內
```php
$rset->occursAt(new \DateTime('20250808')); // true
```

取得特定日期以後的事件
```php
$occurrences = $rset->getOccurrencesAfter(new \DateTime('20251001'), true, 5);
```

## Bonus
取得 2026/02/16 開始未來 365 天內的第一個開工日
```php
function getOpenDate(RSet $rset, \DateTime $start): \DateTime|null
{
    $end = (clone($start))->modify('+366 days');
    $period = new \DatePeriod($start, new \DateInterval('P1D'), $end);
    foreach (iterator_to_array($period) as $date) {
        if (!$rset->occursAt($date)) {
            return $date;
        }
    }
    return null;
}

$firstOpenDate = getOpenDate($rset, new \DateTime('20260216'));
if (is_null($firstOpenDate)) {
    echo '找不到開工日';
}
echo $firstOpenDate->format('r'), "\n";
```

# 參考
- [RFC 5545: Internet Calendaring and Scheduling Core Object Specification (iCalendar)](https://www.rfc-editor.org/rfc/rfc5545.html)
- [rlanvin/php-rrule: Lightweight and fast recurring dates library for PHP (RFC 5545)](https://github.com/rlanvin/php-rrule)
