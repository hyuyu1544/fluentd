# fluentd

一個分送資料流的中介平台，可以預處理數據後，分送到各自的儲存點（資料庫）

## concept

在k8s中，pod送出的stdout及stderr會記錄在docker log中, fluentd可以用來搜集所有node中的docker log(自定義), 使用共同的pipeline來對log做預處理然後分流（送進不同DB,或是送出alert...等）

- 優點：log集中處理，方便維護
- 缺點：要學習其語法，debug不易


## usage
最常見的方式就是 `source` 收集日誌(e.g. docker log)，然後由串聯的 `filter` 做流式的處理，最後交給 `match` 進行分發

- Source: 所有數據來自哪裡
	- http: 以http的port 收http的訊息
	- forward： 以tcp的port 收tcp的數據包
	- 可以同時進行
- Match: 分發日誌
	- 最好精確匹配
	- 順序很重要
- Filter:對data做加工
	- 一定會有@type: 要加什麼工, e.g:
		- record_transformer: 插入新欄位
		- teams: teams的plugin
		- copy: 複製ｄａｔａ
			- 可以接多個story
	- 有多個plugin可以參考
	
匹配方式：
- a.* -> 匹配`a.b`, `a.c`; 不匹配`a`,`a.b.c`
- a.** -> 匹配 `a`,`a.b`,`a.b.c`
- {a,b} -> 匹配 `a`, `b`
- combine -> a{b,c.**} ...

## issue

1. no patterns matched
```
2021-01-21 11:01:20 +0000 [warn]: no patterns matched tag="kubernetes.....var.log.containers....-86f78646d9-pnplz_...-c158bda01f672a39bedb5a09aa76c3c14920bb08092797ea5bc4e749a24fb7e9.log"
2021-01-21 11:01:20 +0000 [warn]: no patterns matched tag="kubernetes....var.log.containers....-5889b4974f-6j687_...-736cb17924d81d05bc01860d7aebee6b19b2dcc4dfae39747f2be3595986c16e.log"
```
解：下面的match pattern 沒有跟上面給的tag match到

## Reference
- https://www.itread01.com/content/1549792272.html
