+++
title = 'Test & Test Double'
date = 2022-05-18T15:42:38+08:00
draft = false
description = '分享一下個人理解的Test種類，還有Test Double, Mock等等'
toc = true
tags = ['Test']
categories = ['Test']
+++

## 測試的目的
不知道大家有沒有那種，隨著專案功能越來越多，一不小心就把舊的功能改壞的經歷？

或是手動重複做相同的測試，花費大量時間呢？

測試可以幫助減少這些問題，除此之外還有：
1. 測試結果是否符合預期
2. 提供一個範例：假如這個功能很複雜，查看測試可以快速了解這個功能的用處
3. 提供類似規格書的功能：查看test就可以大概知曉這個class開發了哪些功能
4. 方便重構或優化：重構以後還可以保證功能沒有壞掉

## 測試種類
**事先聲明！** 我覺得有些專有名詞的定義不是很清晰，有些人的觀點甚至有點衝突，所以這篇都是我查閱許多資料和教學，**基於個人理解**寫出來的

測試的種類有很多，但這篇只會講我接觸過的三種
- Unit test：專注測試一個class
- Feature test：測試一個功能，確保符合驗收標準
- Browser test：瀏覽器模擬使用者操作

Feature Test相較Unit Test更注重是否符合需求，Unit Test比較能定位問題在哪裡

打個比方，電腦無法開機有很多原因，一個一個單元測試每個東西才能定位原因

### Unit Test 跟Feature Test 同樣重要
以測試覆蓋率來講，好像Unit Test寫完就好了？但Feature Test一樣很重要，舉個搞笑的例子，之前我在Laracast[這篇貼文](https://laracasts.com/discuss/channels/testing/feature-vs-unit)看到一個gif

![烘手機和垃圾桶](./image-1.gif)

烘手機和垃圾桶的單元測試都沒問題，組合在一起就出事（吹葉機？）

所以我們需要功能測試確保各個部件組合出來的功能正常

## Unit Test

### 測試的結構
我通常會用[Given-When-Then](https://en.wikipedia.org/wiki/Given-When-Then)的結構寫測試

Given表示前情提要，When表示做了什麼事，Then表示預期的結果

舉一個屬於feature test的例子：
- Given：我是會員
- When：進入首頁
- Then：可以在文章下方看到喜歡、收藏按鈕

### Test Double
之前我總是搞不清楚這是什麼，在這裡分享我自己的理解

Double 是測試的某種替代品，我們不用關心裡面怎麼實作，它是

![假的](https://memeprod.sgp1.digitaloceanspaces.com/user-wtf/1571558429260.jpg)

Double常用於第三方的API，例如丟檔案到S3、寄信等，或是這個API回應時間很長的情況，或是這個API需要付錢等等

當然這種寄信功能我們確實要測試正不正常，但不需要每次都測試，因此可以用Double代替

來看一個PHP Test的例子：

這是一個News class，建立一則新聞的時候，會使用Downloader去S3下載一張圖片作為封面圖
```php
// News.php
namespace App;

class News
{
    protected $coverImage = null;

    public function __construct(ImageDownloader $downloader)
    {
        $this->downloader = $downloader;
    }

    public function create()
    {
        $this->coverImage = $this->downloader->downloadCoverImage();
    }

    public function hasCoverImage()
    {
        return !empty($this->coverImage);
    }
}

// ImageDownloader.php
namespace App;

class ImageDownloader
{
    public function downloadCoverImage()
    {
        // Download a random image.
    }

    public function downloadIcon()
    {
        // Download a icon.
    }
}
```

再來看測試，遵循剛才說的Given-When-Then方法：
- Given：建立ImageDownloader和News class
- When：建立一篇新聞
- Then：斷言這篇新聞有封面圖
```php
// NewsTest.php
namespace Tests;

use App\News;
use App\ImageDownloader;
use PHPUnit\Framework\TestCase;

class NewsTest extends TestCase
{
    /** @test */
    public function download_a_image_when_news_created()
    {
        $downloader = new ImageDownloader;
        $news = new News($downloader);
        $news->create();
        $this->assertTrue($news->hasCoverImage());
    }
}
```

#### 用Dummy/Fake改寫
上面提到Downloader會去S3下載圖片，但我又不希望每個新聞測試都真的下載一張圖

現在用Dummy，或者說Fake來改寫，建立一個假的class取代原本的，下載圖片的function永遠只會回傳fake cover
```php
namespace Tests;

class FakeImageDownloader
{
    public function downloadCoverImage()
    {
        return 'fake_cover.jpg';
    }

    public function downloadIcon()
    {
        // Download a icon.
        return 'fake_icon.jpg';
    }
}
```
回頭修改測試
Given的部分，用建立fake物件取代原本的，然後建立News object
```php
namespace Tests;

use App\News;
use PHPUnit\Framework\TestCase;

class NewsTest extends TestCase
{
    /** @test */
    public function download_a_image_when_news_created()
    {
        $downloader = new FakeImageDownloader;
        $news = new News($downloader);
        $news->create();
        $this->assertTrue($news->hasCoverImage());
    }
}
```

#### Stub & Mock
每次給一個Test寫一個class，多出新檔案還是挺費事了，可以用Stub或Mock代替

##### Stub
用createMock建立一個stub，然後設定downloadCoverImage這個function會return stub cover jpg
```php
class NewsTest extends TestCase
{
    /** @test */
    public function download_a_image_when_news_created()
    {
        // stub
        $downloader = $this->createMock(ImageDownloader::class);
        $downloader->method('downloadCoverImage')->willReturn('stub_cover.jpg');
        // downloadIcon根本不會用到，但多寫以下這行也不會噴錯。stub不關心你呼叫了指定的method幾次，甚至不關心是否呼叫過
        $downloader->method('downloadIcon')->willReturn('stub_icon.jpg');
        $news = new News($downloader);
        $news->create();
        $this->assertTrue($news->hasCoverImage());
    }
}
```

##### Mock
Mock一樣也是用createMock建立，然後預期downloadCoverImage這個function會被呼叫一次，然後return mock cover jpg
```php
class NewsTest extends TestCase
{
    /** @test */
    public function download_a_image_when_news_created()
    {
        // mock
        $downloader = $this->createMock(ImageDownloader::class);
        $downloader->expects($this->once())->method('downloadCoverImage')->willReturn('mock_cover.jpg');
        // 以下這行會噴錯，因為預期(expects)執行downloadIcon method一次，但實際上沒有
        $downloader->expects($this->once())->method('downloadIcon')->willReturn('mock_icon.jpg');
        $news = new News($downloader);
        $news->create();
        $this->assertTrue($news->hasCoverImage());
    }
}
```

##### Stub 和Mock 的區別？
建立新聞的時候不會使用downloadIcon這個function

Stub不關心執行指定的method幾次，甚至不關心是否執行過，因此stub加上下面這行也不會噴錯

而mock，因為mock會預期(expects)執行downloadIcon method一次，但實際上沒有，就會噴錯

我個人認為Mock用在需要嚴謹測試這個被mock的class的時候才需要用到，否則只是一個小改動就要改一堆mock

## Browser Test
這裡我用Python Test當範例，因為用Python操作Selenium真的很方便((誠懇

首先我指定使用chrome，然後打開我們的首頁，接下來尋找導航欄的「新聞」，我這裡用css selector尋找元素

之後將由標懸停在這個元素上

最後斷言會在頁面上看到「台股盤勢」四個字
```python
import unittest
from selenium import webdriver
from selenium.webdriver import ActionChains
from selenium.webdriver.common.by import By

class HomeTest(unittest.TestCase):
    def setUp(self):
        self.driver = webdriver.Chrome()

    def test_search_in_python_org(driver):
        driver = self.driver
        driver.get("https://www.cnyes.com/")
        elem = driver.find_element(
            by=By.CSS_SELECTOR,
            value='nav ul li a[data-global-ga-label="新聞"]>span'
        )
        ActionChains(driver).move_to_element(elem).perform()
        self.assertIn("台股盤勢", driver.page_source)

    def tearDown(self):
        self.driver.close()
```


## 參考
- [PHP Testing Jargon](https://laracasts.com/series/php-testing-jargon)