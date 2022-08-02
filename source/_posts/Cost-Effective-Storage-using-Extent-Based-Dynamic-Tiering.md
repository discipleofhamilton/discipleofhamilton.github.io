---
title: Cost Effective Storage using Extent Based Dynamic Tiering
date: 2022-07-26 21:50:20
tags: 'Auto Tiering Storage'
---

Here is the [paper](https://www.usenix.org/legacy/event/fast11/tech/full_papers/Guerra.pdf).

市場上雖然有許多基於SSD的multi-tier的storage解決方案，但如何配置最佳的設備給每個tier，來達到用最小的花費完成最好的performance。

本文提出的方法叫 "Extent-based Dynamic Tiering (EDT) Solution"。首先什麼是extent? 一個extent指的是一段連續的儲存空間，而一個檔案的物理大小一定是一個extent容量的整數倍；其次，EDT又是用哪些方法來完成上述的目標? 總共有兩個關鍵元件作為基礎，以下是簡述:

1. [A Configuration Adviser (EDT-CA)](#EDT-CA): 決定購買和安裝合適的混合storage device來滿足一個給定工作負載/應用場景的最低成本。
2. [A Dynamic Tier Manager (EDT-DTM)](EDT-DTM): 在當前給定的工作負載/應用場景中，EDT-DTM運行在configured storage system，通過動態地搬移extent (fixed-size portions of a volume)來將資料放置在最合適的tier上。


<!-- more -->
## Extent-based Tiering

回過頭來看EDT的原型，Extent-based Tiering。研究中表明當extent的大小越小，資料擺放的就會更高效。但會帶來metadata[^2] (元數據、後設資料)的開銷，而metadata的目的是持續追蹤extent的位置和其他統計資料，並且此開銷會隨著extent大小越小而增加。

**THE PROBLEM IS HOW TO GET AN ACCEPTABLE EXTENT SIZE?**



## Dynamic Tiering

什麼又是Dynamic Tiering? 並且它如何與Extent-based Tiering融合成EDT?

* 作用: 為了處理extent跨tier移動的時間範圍/尺度。(Dynamic Tiering is to deal with the time scale at which extents move across tiers.)



## EDT

**$EDT = EDT_{CA} + EDT_{DTM}$**

EDT能執行requests來不間斷地在storage devices間移動extent。<font color="red">通過同時採用EDT-CA和EDT-DTM來最小化成本，其中EDT-CA用來最小化購買成本(acquisition cost)，而EDT-DTM則是最小化運行成本(operating cost)</font>。

![EDT_system_arch](https://i.imgur.com/yz0ZaYl.png)


### EDT-CA

EDT-CA是一個配置建議者(Configuration Adviser)的角色，作用是給出正確的儲存設備數量在每個tier上來安裝storage system。

通過<font color="red">模擬</font>在Tier中放置的每個extent來估計滿足工作負載所需的不同Tier中的資源，進而滿足性能需求的同時最小化其生產成本。接著在每個epoch[^1]: 中重複此過程，並基於各自extents在該epoch的性能需求，將extents分派到其最低成本的tier上。此模擬結束後，EDT-CA會根據所有epoch中的每個tier的最大設備需求數量給出一個設備集(the set of devices)。



### EDT-DTM

EDT-DTM是一個動態層管理者(Dynamic Tier Manager)，運行在一個正在執行的系統，並持續管理Tier間extent的擺放。它蒐集extent level的統計數據並估計在不同tiers中extent的資源消耗，之後用於規劃與執行資料的搬移/遷移(data movement/migration)。

EDT-DTM中有幾個機制和演算法幫助它完成上述目標:

* 節流矯正機制(throttling correction mechanism): EDT-DTM實現一個阻礙矯正機制來確保滿足隨時間變化的性能需求，它會持續監控array performance，當性能節流(throttle)被偵測到，則將extent擺放到新的位置以恢復性能。
* 擺放演算法(Placement Algorithm): 目的是擺放每個extent到滿足性能需求的前提下，耗能最低(lowest-energy)的Tier上。並且通過將同一個Tier上的extents合併到少數的設備上，來降低沒有在使用的設備的功耗，進而最小化能量的使用。

以上兩個演算法都有用到資料遷移模組(Migration Module)。



### Common Component

EDT是由兩個主要角色EDT-CA和EDT-DTM所構成，其中包含各個元件，當中有兩個元件被EDT-CA和EDT-DTM共享來蒐集討計數據和計算資源消耗:

1. Data Collector
2. Resource Consumption Model



#### Data Collector

Data Collector會收到IO completion的資訊，包含傳送大小、回應時間、Logical block address(LBA)[^3]、IO產生的Volume ID和IO執行在哪個array。

Collector將每個I/O的(LBA，Volume id)pairs映射到系統中唯一的extent，並為每個extent編譯隨機I/O的數量和傳輸的字節數(bytes)。然後週期性地(在本文的實現中是每分鐘)計算每個extent的瞬時帶寬和隨機IOPS以及指數加權移動平均。除了extent的統計數據外，Collector還會聚集每個array的統計數據。它將每個I/O映射到它的array，並編譯它的IOPS和平均響應時間。 EDT-DTM使用這些量測數據來確定array上的I/O是否被節流。對於一個非常大的系統，由Data Collector收集的資料量可能非常大。若因此產生問題，可以將extent大小做得更大，以減少統計數據量。



#### Resource Consumption Model

Resource Consumption Model用來決定對該extent而言，把它放在哪個Tier最有效率。它利用該extent的統計數據來估計此extent被放置在一個給定類型(type)的設備中時的資源消耗，並根據在設備層級(device level)觀察到的容量與性能需求來分配資源。而任何工作負載的優化，如資料去重(deduplication)、壓縮(compression)和緩存(caching)等，在此model中不需要考慮，因為它們的影響會被捕捉在使用統計數據(usage statistics)中。

單一extent所消耗的資源只會有兩個維度: 容量(capacity)與性能(performance)。若一個extent的大小視為$E_{C}$；性能需求視為$E_{P}$，從前面的epochs的random IOPS rate (RIOR)和bandwidth量測而得。

在設備$D$中extent所需的容量占比為:
$$
RC(E_{C},D)=\frac{Capacity\ required\ by\ extent}{Total\ space\ in\ device}
$$
性能:
$$
RC(E_{P},D)=RIOR\times Rtime+Bandwidth\times Xtime
$$
其中各自的定義為:

* $RIOR$: random IOs一秒傳輸到一個extent的數量(IO/s)
* $Rtime$: 該設備預期的回應時間(s/IO)
* $Bandwidth$: 該設備的bandwidth規格(MB/s)
* $Xtime$: 平均傳輸時間(s/MB)

**一個extent的總體資源需求是容量使用比例和行能使用比例對最大值**:
$$
RC(E,D)=max(RC(E_{C},D), RC(E_{P},D))
$$

![EDT_lowest_cost_tier_for_ext](https://i.imgur.com/dPjZa8E.png)

### Configuration Adviser

在[EDT-CA](#EDT-CA)的章節中有概略說明它的作用，本章將詳細說明它如何提出最優配置。

首先它是上述的Data Collector和Resource Consumption Model的基礎之上建構而成，而論文提出一個輕量的啟發式方法來達到低成本的extent擺放:

1. Binning: 首先定義出$extent\ cost(E,D)=cost(D)\times RC(E,D)$，利用此條公式可以把extent擺放在最小成本且符合性能需求的Tier上，並對所有extent進行成本的計算，再將extent分割成每個Tier中的一個bin。

2. Sizing a bin: 

   * 每個bin的性能與容量資源消耗定義:

     * {% raw %}
      $$
       \left\lbrace
           \begin{aligned}
               RC_{P}=\sum RC(E_{P},D),\ \forall E \\
               RC_{C}=\sum RC(E_{C},D),\ \forall E
           \end{aligned}
       \right.
       $$
       {% endraw %}

     * 上述兩者的最大值將給定總體bin的資源需求與對此bin類型來說所需要設備四捨五入後的近似整數。

3. 此過程在每個epoch中獨立且重複地運行，為了確保在該epoch中每個Tier最小成本的設備數量。

4. 通過分配所有epoch中使用的每種類型設備的最大數量來實現最終配置。例如: 

   * epoch $t_{0}$: 2 Type $D$ devices and 1 Type $D^{\prime}$ device
   * epoch $t_{1}$: 1 Type $D$ device and 2 Type $D^{\prime}$ devices
   * final: 2 Type $D$ devices and 2 Type $D^{\prime}$ devices



### Dynamic Tier Manager

在[EDT-DTM](#EDT-DTM)章節中有簡述它的作用與內部機制，此章節將針對整體具體作用與內部機制進行詳細說明。

EDT-DTM除了有Data Collector和Resource Consumption Model之外還有3個新模組來幫助它持續優化extent的擺放:

1. [Tiering and Consolidation module](#Tiering-and-Consolidation-Algorithms)
2. [Throttling Detector/Corrector module](#Throttling-Detector-and-Corrector)
3. [Migrator module](Migrator)



#### Tiering and Consolidation Algorithms

Tiering and Consolidation (TAC)演算法在每個epoch的結尾時進行extent的擺放，目的是滿足extent的性能需求和最小化動態的系統能源。這樣的節能佈局既可以利用異構底層硬體(SSD、SAS和SATA驅動器)的優勢(即性能或每瓦特容量)，也可以在可能的情況下將數據整合到更少的設備中，並關閉未使用的設備。

與配置問題(Configuration Problem)相同的是，功率最小化布置問題(placement for power minimization)也是NP-Hard問題。為了解決功率最小化布置問題，TAC需要兩個輸入: 

1. 當前的隨機IO rate和積極運行的系統中每個extent的bandwidth
2. IO Size in bytes和在儲存系統中每個array的隨機IO rate與bandwidth capacity

利用上述兩步驟產生的新的extent布局還適應工作負載的變化。例如:

1. Tiering:

   對每個extent $E$和設備類型 $D$計算配置extent到該設備的功率/能量負荷比值(fraction power burden):
   $$
   extent\ power(E,D)=power(D)\times RC(E,D)
   $$
   通過上述公式，TAC可把extent擺放在符合其性能與最低功耗需求的Tier上，並且此步驟可以通過合併(consolidation)來降低活躍功率。此外，將extent分配給Tier是在本地與一個extent接一個extent的基礎上執行的，而與該Tier的總體性能需求或可用空間無關。

2. Consolidation: 

   分配給每個Tier的extent是通過它們的$RC$值進行排序，並使用First Fit遞減啟發式(First Fit Decreasing heuristic)將extent放置在array中，這是extent packing一種很好最逼近優解決方案的演算法。當extent已經分配到考慮中的某個Tier，並且extent超過其可用性能或Tier耗盡array中空間時，extent列表中剩餘的extent將降級到具有該extent下一個較低功耗負擔的Tier。

   為所有的Tier重複上述的打包過程，合併extents到一個Tier中最小數量的array，而已經在正確Tier上和在此epoch上會保持啟動在一個array上的extent會保留它們在前一個epoch的位置，從而節省遷移成本。而在extent布局/擺放中所有未使用的arrays會被設置為低功率的狀態，以節省電力/能量。



#### Throttling Detector and Corrector

儘管TAC機制能夠實現動態性能和功率優化，但非預期的負載和工作集(working set)的變化會突然改變extent的性能要求。追蹤性能的變化相當困難，尤其當某個extent的IO rate上升時。位於低性能Tier的extent無法表現出高的I/O rate，這將導致對extent的真實IOPS需求進行節流，人為地將其限制在一個較低的值。而Throttling Detector通過監控每分鐘每個活躍的array的平均回應時間來克服了上述的限制。

若一個array的IO平均回應時間表明在該array中出現不需要(udesirably)的high request queue，則EDT將判定該array正在限制(throttling)應用程式的真實IOPS需求並造成延遲。當偵測到節流時，由TAC驅動的待處理的(pending)資料前移動作都立即暫停，並且EDT-DTM切換到節流糾正(throttling correction)模式進行回復。

為了選擇目標array(s)，第一步先從最有可能被遷移到的Tier開始，並檢查哪個已經激活(正在被高度使用)的array可以吸收新的extent。若沒有array可以吸收，則c會考慮在該tier中沒有被使用且處於可被使用狀態的arrays；若最優的Tier無法放置該extent，則在下一個具有更高功率負荷的Tier上嘗試相同方法。



#### Migrator

Migrator用來解決TAC和Throttling algorithm發出的資料移動請求，並通過比較新舊extent的布局和識別那些extents需要搬移。雖然資料搬移可以緩解節流問題，但同時也會造成額外的IO阻塞(IO traffic)，因此必須小心處理以免影響前台(foreground)IO的性能。

Migrator通過以下幾點來達成損耗與提升之間的平衡:

1. 每個設備同時只允許進行一次資料搬移
2. 在發布搬移的請求前，Migrator只有在源(source)設備和目標(target)設備都可用時才允許請求，從而執行添加控制；若沒有允許，則該請求會重新回到佇列中，並拿取下一個請求。
3. Migrator通過分解一個extent成更小的傳輸單位，並調整(pace)傳輸請求的速度以符合可用或需要的IO rate最小值來控制自身前跟資料遷移相關的資源消耗。



## Definition

[^1]: The intervals on the order of minutes or hours that plan extent movement
[^2]: 描述其他資料資訊的資料
[^3]: It is  a common scheme used for specifying the location of blocks of data stored on computer storage devices, generally secondary storage systems such as HDD.
