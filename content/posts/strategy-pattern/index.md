+++
title = 'Strategy Pattern 策略模式'
date = 2026-03-19T07:11:22+08:00
draft = false
description = '筆記 PHP 策略模式的實作方式'
toc = true
tags = ['PHP', 'Design Pattern']
categories = ['PHP', 'Design Pattern']
+++
## 簡介
公司專案因為商業邏輯較複雜，常會遇到有多種方式處理同一件事的場景。例如網購場景，對主流程而言都要付款和配送，但付款有信用卡、LINE pay 等方式，配送有宅配或超取等選項\
這時就可以考慮使用策略模式：
1. 從外部來看，只需要知道流程，不需要知道內部不同的策略如何實作，避免在主流程中有大量 if else 和重複的 code
2. 每個策略有各自的 class，切分後邏輯更清晰不混雜，有新的策略就新增一個 class，特定策略需要更改只需修改它的 class，不會動到其他策略及主流程
3. 避免多個人同時改動相同幾個 class，最後解衝突解到頭痛

如上述的網購場景，以下爲 PHP 實作範例，有信用卡和 LINE pay 兩種付款方式，都可以退款，並且退款還有一些共用的後續操作

另外，個人習慣策略模式配合簡單工廠建立指定的 class。並且若有共用邏輯可改用 abstract 而不是 interface
```php
enum PayType: int {
    case CARD = 1;
    case LINE = 2;
}

abstract class PaymentService
{
    public abstract function pay();
    public abstract function refund();
    public function handleRefund()
    {
        $this->refund();
        echo "進行其他共用操作\n";
    }
}

class Card extends PaymentService
{
    public function pay()
    {
        echo "信用卡 付款\n";
    }

    public function refund()
    {
        echo "信用卡 退款\n";
    }
}

class Line extends PaymentService
{
    public function pay()
    {
        echo "LINE PAY 付款\n";
    }

    public function refund()
    {
        echo "LINE PAY 退款\n";
    }
}

class PaymentFactory
{
    public static function create(PayType $payType): PaymentService
    {
        return match ($payType) {
            PayType::CARD => new Card(),
            PayType::LINE => new Line(),
            default => throw new \UnexpectedValueException('無效的付款方式'),
        };
    }
}

$request = ['pay_type' => 1];
$paymentService = PaymentFactory::create(PayType::from($request['pay_type']));
$paymentService->pay();
$paymentService->handleRefund();

// output:
// 信用卡 付款
// 信用卡 退款
// 進行其他共用操作
```
