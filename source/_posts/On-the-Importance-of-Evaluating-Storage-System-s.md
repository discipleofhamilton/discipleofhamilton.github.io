---
title: On the Importance of Evaluating Storage System's $Cost
date: 2022-08-03 14:45:03
tags: 'Auto Tiering Storage'
---

概說:

此篇論文提出一個從**成本/金錢**層面對儲存系統進行評估的方案，評估對象包含性能等方面。

貨幣成本應該在儲存系統的**預期壽命內**進行評估，並考慮設備的磨損(wear-out)和更換。

相關領域研究中，很少有人探討Tiering和Caching在混合儲存系統中的利弊。本文為促進研究，開發了一套冊被映射(device-mapper)目標程式給Linux Kernel，目的是結合SSD和HDD。此套App想在SSD上起到2個作用:

1. 將SSD作為Cache，並把髒資料(dirty data)異步回寫到HDD中
2. 將SSD作為主要儲存設備(Tier)
3. 其他: 其餘目標還有管理冷熱資料在SSD和HDD之間的通用策略

在儲存系統的案例中，作者觀察業界真實使用情況與自身經驗，總結了以下情況:

1. 發現在某些工作負載/使用情況(workload)，純SSD的方案會在短期內帶來高成本，但長期使用下成本會低得多。
2. 還有一些情況，用SSD作為Cache比在Tier上作為熱資料的主要儲存設備的成本要低得多。**但剩餘的工作負載皆與之相反。**

<!-- more -->

## Cost Model

成本: 包含能源和電力成本，以及設備更換本等等。

成本模型(Cost Model): 由13條公式計算得出一個數值，並用此數值做為參考來比較不同情況下成本的高低。其中將以下幾點參數/數值化後，再依照定義的公式進行計算:

1. 儲存設備的數量(the number of storage device)
2. 未來成本估計的時間因素($\alpha$)
3. 購買成本: 單位容量的價格$\times$總容量
4. 能源和電力價格
5. Endurance Cost: 每台儲存設備的價格$\times$該台設備的折損率
6. 定義SSD與HDD的磨損程度與使用期限
7. 固定提供服務的成本(Service Cost): 主電源、冷氣等等

13條公式:

1. $1\leq i \leq n,(n:the\ number\ of\ storage\ devices)$

2. $1 \leq \alpha, (time\ factor,\ default=1)$

3. $Cost=Purchase+TCO$

4. $Purchase=\sum_{i=1}^{n}{C_{dev_{i}}}$

5. $C_{dev_{i}}=Normalized\ Price_{dev_{i}}\times Capacity_{dev_{i}}$

6. $TCO=\alpha \times (C_{energy}+C_{power}+C_{endurance})+C_{service}$

7. $C_{energy}=Lookup_{LIPA}(Amount_{energy})$

8. $C_{power}=Lookup_{LIPA}(Amount_{power})$

9. $C_{endurance}=\sum_{i=1}^{n}{C_{dev_{i}}}$

10. $C_{endurance_{i}}=C_{dev_{i}}\times \frac{dev_{i}\ wearout}{Limit_{i}}$

11. {% raw %}
    $
    \left\lbrace
        \begin{aligned}
        	writes\ if\ dev_{i}=SSD, \\
        	\#startstop\ if\ dev_{i}=HDD 
        \end{aligned}
        \right.
    $
    {% endraw %}

12. {% raw %}
    $
    Limit_{i}\ wearout=
          \left\lbrace
            \begin{aligned}
                Limit_{writes}\ if\ dev_{i}=SSD, \\
                Limit_{cycles}\ if\ dev_{i}=HDD 
              \end{aligned}
          \right.
    $
    {% endraw %}
    
13. $C_{ser}=fixed\ estimation$

公式12和13看得出來，本文對SSD的耐用性及使用極限的設置主要是看寫入的次數，當然，讀取依然會影響其耐用性，所以本文對其進行了簡單的轉換。例如:由讀取引起的寫入就記作$reads/10$；而HDD則是看start-stop-cycle作為主要引響因素，並忽略其他。作者基於當時查到的工業規格，SSD能持續寫入36.5TB的資料，而HDD則是能完成300,000 spin up/down cycles。



## System

本文為了將評估與測試簡單化，因此只用一顆SSD和一個HDD組成兩個自己完成的系統，一個是Caching；另一個是Tiering，評估時再加上基於Linux DM "Linear" target的Tiering系統。兩個自己完成的系統支援多個可配置參數:

1. Extent Size (ES)
2. Promotion/Pre-fetching Threshold (PT): Access counts before promoted/fetched.
3. Maximum Concurrent Migration Limit (MCML)

此外，以上兩系統有以下四點不同:

1. Capacity
2. Management Unit
3. Data Movement
4. Read/Write Policy



## Evaluation

設備:

1. 兩台相同的Lenovo ThinkCenter電腦上進行測試，每一台上面搭載4GB的RAM和Intel Core-2 Quad 2.66GHz CPU。
2. 儲存設備(Storage): Intel SSDSA2CW300G3 300GB SSD & Seagate ST32000641AS 2TB HDD，且Linux 3.5.0的kernel運行在獨立的SATA驅動上。

作者將每電腦連接至一台WattsUP Pro ES的直列式功率計算器來測量能源與電力的損耗。



 工作負載 / 應用場景:

1. Web-search trace from UMass Trace Repository
2. FIU's online trace 

作者發現在Filebench的File Server的應用場景(工作負載)的各項測試結果(評估結果)的趨勢與Web-search雷同，因而省略此場景。



Storage System:

1. Tiering: 作者自己實現的Tiering方案
2. Caching: 作者實現的Caching方案
3. Mylinear: 一種Tiering的方案，其方法來自Linux DM "Linear" target

上述不同的儲存系統中，2種Tiering的方案的SSD的容量採用總容量的1/4，並且為了在相同設備情況下對結果進行比較，因此Caching的方案的SSD也是相同的備至；不同的是在Caching系統上，SSD作為Cache不能算在總容量中，所以實際上的總容量都為HDD所組成，也就是Caching system的HDD要比Tiering system的HDD多出1/4的容量。

$\alpha$ as the time factor to project future cost extimates (i.e., run the same amount of workload multiple times). 



### Web Search

#### Throughput  Metric

* 當PT (Promotion/Pre-fetching Threshold) = 4, 16時，Caching的throughput比Tiering高4~9%
* 當PT= 64時，Caching的throughtput和Tiering差不多
* SSD hit ratio: Caching - 81~98%, Tiering - 85~98%, Mylinear - 8%
* SSD hit ratio和資料搬移(data movement)會影響混合儲存系統的throughput

Web-search應用場景的讀取(Read)遠多於寫入(Write)，因此回寫(Write-back)的開銷小。此外，Tiering系統中的SSD，在事先就會包含冷熱資料，此情況會增加對總體throughput的開銷。

* 回寫(Write-back): 指的是在Caching Storage System中，把資料從Cache(SSD)上寫回Storage Device(HDD)的一種機制。write-back 是將資料量儲存到一定的量之後，會依據同區塊的資料一次整批寫回去.裡面有提到 dirty ，他是在記憶體裡面 cache 的一個 bit 用來指示這筆資料已經被 CPU 修改過但是尚未回寫到儲存裝置中。

##### Conclusion

若Tiering系統將SSD(Tier0)作為主要的儲存設備，但其中(預先)包含冷資料，則Caching系統總體的throughtput會比Tiering高。



#### Cost

* 長期: 
  * 在time factor = 100000，也就是平均使用2.1年(範圍從0.2~7.7年)左右，Caching系統的總成本會比Tiering系統節省8 ~ 20 %。而在長期使用下，Caching系統更省成本的原因，作者歸納成以下兩點:
    1. 在Caching系統中，SSD沒有額外的主要IO，但Tiering系統有。因為SSD在Tiering系統中通常扮演最高Tier(Tier0)的角色。因此會有時常搬運的問題。
    2. 從長期使用的角度來看，SSD的耐用度降低的速度，會影響更換的頻率，若更換頻率較高，就會提高更換成本。
* 短期: 在time factor = 1時，Caching系統的成本比Tiering系統**略高**，因其需要擴充HDD的容量(上述Storage System的說明中)。

**還未了解如何從time factor轉換成使用年分**



### FIU's Online

#### Throughput  Metric

* 寫入比讀取多，表示Caching系統的回寫開銷可能會造成throughput瓶頸。
* 隨著ES (Extent Size)的變化，Caching系統的throughput會比Tiering系統**低**58 ~ 82%。
* SSD hit ratio: Caching 92 ~ 99%, Tiering 98 ~ 99%, Mylinear 84%。
* 當ES = 4k時，Caching系統的throughput降低了82%，由於它有更多的回寫IO。



#### Cost

* 長期: 
  * 在time factor = 100000，也就是平均使用3.3年(範圍從0.7~9.8年)左右，Caching系統的總成本比Tiering系統高5 ~ 23%。因為在Caching系統中，SSD沒有額外的主要IO，但有更多回寫IO，因此比Tiering系統更快消耗SSD的耐用度，造成長期使用下的設備更換成本。
* 短期: 在time factor = 1時，Caching系統的總成本會比Tiering系統**略高**，原因與Web-search中短期的原因相同。
