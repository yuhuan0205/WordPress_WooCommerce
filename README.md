# WordPress_WooCommerce

## DB schema 分析

### Post

在 WordPress 中，文章、頁面跟附件都屬於 Post 只是類型跟所用到的欄位有所不同。

這邊針對幾個比較重要的欄位作分析跟討論

* ID： 唯一識別，流水號
* post_author：作者，關連到 user 表的 ID，但並非為 FK，所以當 post_author 並未在 user 表中出現，會顯示為無作者
* post_content：內容，是為一 html 結構
* post_status：文章狀態，有已發布、草稿、繼承(版本異動)、排程中等等
* comment_status：是否開放留言，文章可以，頁面跟附件不行
* post_modified：更新時間，有任何更新都會異動此欄位
* post_parent：狀態為繼承的 post 會有此欄位，可以追溯其 root
* guid：route，進入文章頁面的路徑
* post_type：文章類型，文章、頁面、版本紀錄跟附件等等
* post_mime_type：附件的類型，圖片、音檔等等
* comment_count：留言數

index：

* post_author
* post_name
* post_parent
* type_status_date

根據以上的 schema 可以發現，Post Table 裡面放了很多東西，基本上可以被單獨用 route 存取的頁面都算是 Post，除此之外 Post 的版本紀錄也放在裡面。

優點：

* 抽象化文章、頁面跟附件，使其在 wordpress 框架內可以很好擴充

缺點：

* 每個類性都可能有用不到的欄位，使得開發者容易混淆
* 版本紀錄跟一般的 Post 放在一起使得 Post 更新頻繁時會不斷新增資料，同時這些資料也不常被存取，應當放在不同的資料表內
* 題外話，這個版本功能只提供版本紀錄用，並沒有防止同時修改造成的版本衝突

### Order

訂單主要分成以下 3 個表 Order, Order_Item, Order_Item_Meta

Order 主要用於存放概覽，包括訂單狀態、幣值、購買者等等，比較普通。Order_Item 則是訂單內所購買的某項商品跟訂單之間的關聯。詳細的資訊放在 Order_Item_Meta 中，Order_Item_Meta 結構比較特殊，因此展開來說

* order_item_id：訂單項目 ID，關聯到 Order_Item table 
* meta_id：唯一識別 id，流水號
* meta_key：設定值的 key
* meta_value：設定值的 value

優點：

* Order_Item_Meta 可以動態新增新的 KV，而不用調整 table schema
* 設計上整個 Order 流程中所用到的表算是符合正規化的設計

缺點：

* Order_Item_Meta 冗餘資料很多，有點類似 NoSQL 的設計
* Order_Item_Meta 相同 order item id 的資料可能會散落到不同的 page(sql) 上，而導致 disk 檢索時沒辦法連續存取在同個區塊導致效能低落

### User

User table 就是很基本的註冊時必填的資料，pass 是 hash 過的 password。而存放用戶自訂設定(樣式、語言、介紹、個人網頁等)的地方則是放在 usermeta table 中，這個表的結構比較特殊。

* user_id：用戶 ID，關聯到 user table 但不是 fk
* umeta_id：唯一識別 id，流水號
* meta_key：設定值的 key
* meta_value：設定值的 value

優點：

* usermeta 可以動態新增新的 KV，而不用調整 table schema

缺點：

* usermeta 冗餘資料很多，有點類似 NoSQL 的設計
* usermeta 相同user id 的資料可能會散落到不同的 page(sql) 上，而導致 disk 檢索時沒辦法連續存取在同個區塊導致效能低落

### Session

是 woocommerce 用於紀錄使用者狀態的資料，主要功能有購物車資訊、用戶資訊跟通知等暫時性的資料。

* session_id： 流水號
* session_key：若為登入用戶，會儲存用戶 id；若為訪客，則是儲存一暫時性的 token(由瀏覽器記住)
* session_value：一個 php 序列化字串，保存使用者的狀態
* session_expiry：過期時間

優點：

* 使用符合框架的序列化字串達到高效解析
* 使用 userid 作為 key 跨裝置可以共享同一個 session

缺點：

* 不易閱讀
* 若整個系統改用其他語言重寫時，php 序列化字串會比較沒有彈性，使用 json 取代之可能會更加有彈性

## How To Scale Up

假設此電商系統未來會面臨龐大流量，甚至有可能會轉為 C2C 的形式讓使用者自行上架、管理商品，此時這個架構就會面臨巨大的挑戰。以下是我試想的幾個挑戰：

### 雙 11 或熱門商品上架引起的瞬發巨大流量

在此情境下會有幾個議題：

1. 訂單高併發問題，此問題分成兩個面向
   1. 水平擴展議題
      * 由於整個系統是一單體架構，使得難以針對特定功能做水平擴展，因此依據功能拆分成多個微服務分開部署，可以最有效率的水平擴展。以上述情境來說，把 Order 相關功能獨立出來並在預期會有巨大流量的時間點做水平擴展
   2. 資料庫效能問題
      * 根據此系統，新增一個訂單至少需要涉及 3 個 table 的操作，這還不包括檢查庫存的操作。而為了保證此流程的安全性，是必得使用交易來處理，因此減少此一流程涉及的 table 數量不僅能降低死鎖發生的機率還能提高效能。
      * 可以將 Order_Item_Meta 整併進 Order_Item，雖然降低了欄位變動的靈活性，但減少了很多 insert 操作
      * Order_Item 改以 OrderId + OrderItemId 作為 PK 並移除 OrderId 的 index，減少在 insert 時多維護一組 index 的效能犧牲

