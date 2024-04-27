+++
title = 'B+ tree與資料庫索引的應用'
date = 2023-09-02T12:30:17+08:00
draft = false
description = '介紹BTS, B tree和B+ tree的原理以及在資料庫索引的應用'
toc = true
tags = ['Data Structure', 'Database']
categories = ['Data Structure', 'Database']
+++

## 什麼是B+ tree
Tree是資料結構或演算法一定會講到的主題之一，大學的時候最常教的Tree大概是二元搜尋樹，那B+ tree跟二元搜尋樹哪裡不一樣呢？

B+ tree的特點是，資料有序排序，而且搜尋時間穩定

也因為這樣，常用於磁碟等大資料儲存系統，還有用在許多關聯式資料庫的索引中，如MySQL、PostgreSQL、MongoDB等

## Binary search tree(BTS)
來複習一下Binary search tree的規則吧~
![BTS](./BTS.png)

- 每個節點可以有左、右子節點
- 每個節點可以有0 - 2個子節點
- 左節點 < 根節點 < 右節點
- 任意節點的左、右子樹也分別為BTS

上圖是一棵平衡的BTS。假設現在要找出19：
1. 19 < 50，往左邊找
2. 19 > 17，往右邊找
3. 19 < 23，往左邊找
在這棵樹裡面找19，硬碟總共I/O 3次
也就是說，最差的情況下，硬碟的I/O次數跟樹的高度一樣

## B tree
等一下，標題不是在講B+ tree嗎？因為了解BTS和B tree之後，可以更了解B+ tree的特點，也就明白為什麼資料庫的索引選擇使用它

講B tree之前，先明確一個名詞：order，指的是一個節點最多可以擁有的子節點數量（以下以m表示）

下面是B tree的規則&特點：
![B tree](https://cdn.programiz.com/sites/tutorial2program/files/b-tree.png)

- 每個節點的子節點數量
  - 2 ≦ root ≦ m
  - ⌈m / 2⌉ ≦ internal ≦ m
- key數量 = m - 1
- 一個節點的key升序排序
- 所有葉節點出現在同一個水平上
- Self-balancing

說明：

看看上圖的樹，假設m = 3，那所有節點最多有2個key

拿中間的30|33節點來說，這個節點有2個key(30, 33)，3個子節點(25|28, 31, 35) ，並且升序排序30<33
而且葉節點在同一個水平

### Self-balancing
中文是「自平衡」，指的是在對樹執行插入和刪除操作時，自動保持高度盡可能小

也因為這樣，B tree相較二元樹的優勢是，在相同資料數量的情況，高度更低

高度更低，代表硬碟I/O的次數更少

舉個例子，在上圖的樹中尋找25：
1. 20 < 25 < 40，往中間找
2. 25 < 30，往左邊找
3. 找到25

這棵樹總共有16個key，可以想像，相同的16筆資料畫成二元樹，搜尋要花多久時間

### B tree的插入操作
這段是筆記，只想看原理的人可以跳過

例子一：
![B tree的插入範例1](./b%20tree-1.jpg)

設m = 4，就是一個節點最多3個key

左半部分我們看到，這裡有一個根節點，10 20 30。現在我要加入一個40

但是一個節點最多3個key，已經滿了。所以從中間打斷，把20提出來當根，10和30分別為左右節點，再把40放進去

例子二：
![B tree的插入範例2](./b%20tree-2.jpg)

一樣設m = 4

左半部分我們看到一棵樹，現在我要加一個70進去

但又不能放到右節點，因為已經滿了

這時把右節點從中間打斷，50提上去在根節點，然後把40和60變成子節點，這時就有三個子節點了，再把70放進去

### B tree的刪除操作
例子一：
![B tree的刪除範例1](./b%20tree%20刪除-1.jpg)
當刪除key後會違反節點最小key數量時，從左兄弟節點（或右兄弟節點）挪用key到父節點，再挪用父節點的key到子節點

例子二：
![B tree的刪除範例2](./b%20tree%20刪除-2.jpg)
當刪除key後會違反節點最小key數量，又無法挪用左右兄弟節點時，挪用父節點的key與左兄弟節點（或右兄弟節點）合併

例子三：
![B tree的刪除範例3](./b%20tree%20刪除-3.jpg)
當刪除key後會違反最多可擁有子節點數量時，將中序遍歷的前一個或後一個key合併
註：這棵樹的中序遍歷為10 20 30 40 50 60

例子四：
![B tree的刪除範例4](./b%20tree%20刪除-4.jpg)
當無法挪用子節點的key，也無法挪用兄弟節點的key時，將父節點與兄弟節點合併，子節點合併

## B+ tree
![B+ tree](https://cdn.programiz.com/sites/tutorial2program/files/search-tree.png)
終於講到B+ tree了！前面搞懂B tree之後，B+ tree更容易理解，它們的區別在
- 每個葉節點以Singly linked list 或 Doubly linked list 鏈結在一起
- 中間節點的key也會出現在葉節點
- B tree的資料存在每個節點，B+ tree的資料存在葉節點


從這些規則可以得出，每個節點的key大小長這樣：
![B+ tree的節點大小](./b+%20tree%20節點大小.jpg)

### B+ tree的優勢
B tree和B+ tree的區別之一在於，B tree的資料存在每個節點，B+ tree的資料存在葉節點

B tree
![B tree的資料存在每個節點](https://cdn.programiz.com/sites/tutorial2program/files/B-tree.png)

B+ tree
![B+ tree的資料存在葉節點](https://cdn.programiz.com/sites/tutorial2program/files/B+tree.png)

只有葉節點有資料，因此相同的大小的節點，B+ tree的key可以塞更多。也就是說同樣的資料量，B+ tree的高度更矮

B tree最好的情況是找到根節點，最壞的情況是找到葉節點，相較之下B+ tree更穩定

另一個優勢是，B+ tree的範圍查找更方便

前面有提到，B+ tree的每個葉節點以Singly linked list 或 Doubly linked list 鏈結在一起。正是如此範圍查找更方便

B tree
![B tree](./b%20tree.jpg)

B+ tree
![B+ tree](https://cdn.programiz.com/sites/tutorial2program/files/search-tree.png)

舉個例子，上面兩棵樹中我想找20 ~ 40的資料

B tree用中序遍歷查找，順序是20 15 25 30 35 40

B+ tree因為有link，所以找到20以後開始往兄弟找，順序是20 25 30 35 40，比B tree快很多！

### B+ tree的插入操作
這段是筆記，只想看原理的人可以跳過
![B+ tree的插入範例](./B+%20tree插入.jpg)

### B+ tree的刪除操作
範例一：
![B+ tree的刪除範例1](./B+%20tree刪除-1.jpg)
當刪除key後會違反節點最小key數量時，挪用兄弟節點的key作為新的子節點，並將中間key變成父節點

範例二：
![B+ tree的刪除範例2](./B+%20tree刪除-1.jpg)
當刪除key也是中間節點，將中序遍歷的前一個或後一個key作為父節點

註：這棵樹的中序遍歷為10 20 30 40

範例三：
![B+ tree的刪除範例3](./B+%20tree刪除-3.jpg)
當刪除key是父節點的父節點，用中序遍歷的後一個key代替

註：這棵樹的中序遍歷為10 20 30 40 50

## 總結三種tree的比較
|     | BTS  | B tree | B+ tree |
|  ----  | ----  | ---- | ---- |
| 優點  | 在記憶體內搜尋速度快 | 硬碟I/O次數少 | 相較B tree，硬碟I/O次數少，範圍查詢快 |
| 缺點  | 應用在硬碟會有大量I/O | 插入、刪除節點速度慢，且範圍查詢使用中序遍歷 | 插入、刪除節點速度慢，且佔用空間大 |
| 應用  |Unix kernel管理virtual memory areas | 資料庫、檔案系統 | DBMS索引 |

## B+ tree在資料庫索引的應用

### MySQL InnoDB的Index
MySQL InnoDB採用了B+ tree來處理兩種索引，分別是：
- Clustered index：一個table只有一個clustered index，也就是PK。每個key都是PK，葉節點儲存這個record的資料
- Secondary index：除了PK其他索引都是secondary index。key是索引的欄位，葉節點儲存PK的資料

![MySQL InnoDB的Index](./MySQL%20InnoDB.jpg)
從上圖看到，當要搜尋name = 'Tom'的資料時，先從Secondary index找出Tom的PK為93。再用這個PK 93從Clustered index取得該record的資料

### MySQL MyISAM的Index
與InnoDB不同的是，在MyISAM，無論Clustered index還是Secondary index，葉節點都是儲存record資料的**位址**
![MySQL MyISAM的Index](./MySQL%20MyISAM.jpg)

題外話：MySQL 8.0以後，系統table都改用InnoDB了。官方也建議沒有特殊需求的話，都用InnoDB

### MySQL的Covering index
假設有這樣一張資料表有下面幾個欄位：
- id(PK)
- name
- age
- grade年級
- class班級
然後建立一個table_idx1是由grade和class組成

上面說到，除了PK其他索引都是secondary index，那這個table的secondary index的key就是由grade和class組成，葉節點資料就會包含id

當我用這個SQL statement查詢的時候：
```sql
SELECT id, grade, class WHERE grade = 'xxx' AND class = 'yyy'
```

使用這個secondary index就能得到所有需要的資料id, grade, class，我就不用再找clustered index了，這就是covering index

### PostgreSQL的B tree Index
PostgreSQL的索引稱為B tree index，沒有明確說他們使用b tree還是b + tree，但看文件描述感覺應該是b + tree，也可以看看這篇SO的討論[B+ tree or B-tree](https://stackoverflow.com/questions/25004505/b-tree-or-b-tree)

在PG，所有B tree的index都是secondary index

而index tuple儲存指向record資料(heap tuple)的位置TID(block, offset)

![PostgreSQL的B tree Index](https://www.interdb.jp/pg/pgsql01/fig-1-06.png)

上圖右側的資料表，他的index是col，當沿著索引找到col = Queen的資料時，看到record資料位於第7個page的第2個itemId指向的tuple，那我們就可以去這裡查找

### PostgreSQL的Index-only scan
先講解一下什麼是visible。一個record的visible對不同transcation來說是不一樣的
![Visible](https://miro.medium.com/v2/resize:fit:828/format:webp/1*j4E-CkOqXJHc7xzH6ZnQmw.png)

舉個例子，上圖中，當我A transaction取得整個資料表的資料，假設是5筆資料，這時B transaction就插入了一筆資料並且commit，之後A transaction再次取得整個資料表會拿到幾筆資料呢？

答案是5筆。因為當transaction開始會對讀取到的資料snap shot，之後讀取都是讀這個snap shot。所以對A transaction來說，第六筆資料就是invisible。當然這要視資料庫的isolation level隔離級別而定

回看PG的index
![PostgreSQL的Index-only scan](https://www.interdb.jp/pg/pgsql07/fig-7-07.png)

在PostgreSQL，資料的visibility都是存在heap tuple而不是index tuple裡面。所以就像前面提到的，找完index都要根據block和offset找heap tuple。這時配合visibility map就可以依情況避免。1表示這個page的所有tuple都是visible，可以不用查找heap tuple

第二種方法是建立covering index
```sql
CREATE INDEX tab_x_y ON tab(x) INCLUDE (y);
```

建立一個索引是x欄位，但又包含y欄位，這樣我只要找x和y欄位時，就不用再去找heap tuple了

這個方法我還沒用過，有興趣可以參考COSCUP 2023 的一期影片[PostgreSQL Index 的一些探討](https://www.youtube.com/watch?v=nWFBL7jVWPc)

## 參考
- [Tree](https://web.ntnu.edu.tw/~algo/Tree.html)
- [學習手記：2018清華大學DB/AI Bootcamp — II — B-Tree Indexing](https://medium.com/hallblazzar-開發者日誌/學習手記-2018清華大學db-ai-bootcamp-ii-b-tree-indexing-648a096e1598)
- [Introduction of B-Tree](https://www.geeksforgeeks.org/introduction-of-b-tree-2/)
- [漫画：什么是 B- 树？](https://mp.weixin.qq.com/s/raIWLUM1kdbYvz0lTWPTvw)
- [B-tree](https://www.programiz.com/dsa/b-tree)
- [B+ Trees internal nodes](https://stackoverflow.com/questions/70636295/b-trees-internal-nodes)
- [Heap Only Tuple and Index-Only Scans](https://www.interdb.jp/pg/pgsql07.html)
- [B+ tree or B-tree](https://stackoverflow.com/questions/25004505/b-tree-or-b-tree)
- [B+ Tree : Search, Insert and Delete operations](https://iq.opengenus.org/b-tree-search-insert-delete-operations/)
- [對於 MySQL Repeatable Read Isolation 常見的三個誤解](https://medium.com/@chester.yw.chu/%E5%B0%8D%E6%96%BC-mysql-repeatable-read-isolation-%E5%B8%B8%E8%A6%8B%E7%9A%84%E4%B8%89%E5%80%8B%E8%AA%A4%E8%A7%A3-7a9afbac65af)

