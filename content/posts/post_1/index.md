---
title: "非前端的掙扎：如何在雲端平台架設個人網站 - AWS Route53, S3, Hugo"
date: 2022-10-28T22:22:20+08:00
draft: false
featuredImagePreview: "post_1_cover.png"
---
{{< figure src="post_1_cover.png" >}}

&nbsp;

### 前言

&nbsp;

在現在不管是升學、求職、或著是品牌經營，已經有很多人會選擇用個人網站來呈現自己（ 尤其是在設計或著前端領域需要作品集展現）。
個人網站不只能夠拿來展示作品集，也可以用來作個人部落格來分享文章。

最初由於是求職需求，自己一直有著想要試著架設個人網站的想法，所以一直在想如何在滿足個人技能發展方向為前提的情況來架設個人網站。
衡量了個人的時間能力以及需求後列出了下列幾點：

- 由於個人並不是前端工程師，前端相關的技能也不硬，最好是能選擇一個現成的框架工具來使用。(光用CSS)
- 希望能點到雲端平台整合相關的經驗，選擇將網站運營在雲端平台上。
- 個人網站用途：部落格文章撰寫，求職導向。

&nbsp;

在選擇框架工具上，市面上已經有非常多功能強大的架站工具了 ( Wordpress, Wix, Strikingly等 )，這些工具基本上整合了從版面配置到 SEO 所有運營網站必要的功能，
能滿足從一般個人到商業上的需求，不過相對地越是要求客製化的功能則大多數都是收費項目，在追求高靈活度以及個人需求考量，並沒有考量使用這些
產品，而是決定去找一個靜態網頁生成框架(Static site generator)，用最低限度的前端經驗來去實踐版面實作。

參考選項：Jekyll, **Hugo**, 11ty, Next.js, Plain HTML & Boostrap

這邊最後選擇用 [Hugo](https://gohugo.io/) 來做個人網頁。( Hugo 是以 Go 編寫的靜態網頁生成器，官方網站上就有許多豐富的[版面主題](https://themes.gohugo.io/)可以套用。)

&nbsp;

在架設平台上則選擇了自己較為熟悉的 AWS，這邊架站所需要用到的功能基本上全都選擇用 AWS 上所提供的服務 (沒錯我懶XD )：


- 域名註冊 (Domain name registration) : **Route53**.
  花費方面一年平均 **US$18** 左右，覺得太貴可以換其他域名註冊商 (推薦 [Namecheap](https://www.namecheap.com/))

- 網站部署 (Site deploy & host) : **S3 bucket**.
  AWS S3 bucket 算是個蠻萬用的功能，除了一般拿來做存儲之外，還可以拿來直接部署靜態網頁內容(HTML, Javascript 和一些 CSS)
  不需要另外架設網頁伺服器。不過 S3 能部署的網頁也僅止於靜態網頁，若前端需要一些太過複雜的處理或是學術研究及商業用途，
  還是直接用最王道的方式找台伺服器或虛擬機來部署比較洽當。

- CDN (Content Delivery Network) : **CloudFront**.
  用個CDN不只能保護網站免於直接遭到 DDoS 還可以提升網站載入速度，對效能以及安全性來說都是加分的。

- TLS/SSL : **Certificate Manager**.
  如果讓拜訪網站的人看到瀏覽器上顯示著 Not secure 也是會有害觀感。為了呈現專業度以及證明自己重視資安還是用個憑證比較好。

&nbsp;

除了上述基本的需求功能外，想到自己之後有可能會因為經常發文到網站上，需要經常性添加新檔案並重新上傳至S3做部署，這邊則索性將
網站專案放到 Github 上做管控並利用 Github Action，在每次 Push 檔案到 master 分支時自動執行 Hugo 的靜態網站內容生成指令，
同時把生成後的內容直接上傳至 S3 來做部署。(一個非常簡單的土炮CD pipeline)

&nbsp;
### 域名註冊

這邊只簡單介紹用 AWS Route53 註冊域名及設置 Hosted Zone。

前往 AWS console 的 Route53 主頁選擇 Get Started，畫面會列出主要的服務功能，若要註冊新域名則選 Register a domain.

{{< figure src="post_1_1.png" >}}

&nbsp;

之後變可以輸入自己想要且還沒有被註冊的域名及網域，需注意的是不同網域的註冊及更新費用也會有所不同。之後變跟隨畫面指示一步步完成註冊手續。
域名註冊後，Route53 會自動為新註冊的域名啟用 DNS，同時並創立一個名稱跟域名相同的公開 hosted zone．
{{< figure src="post_1_2.png" >}}
Route53 會指派四個 name servers 到這個新創立的 hosted zone，這邊要稍微注意的是剛註冊好的域名需要等一段時間才會有效。

&nbsp;

### S3 Bucket 設置
這邊為了同時應付從 *http://\<domain\>.com* 及 *http://www.\<domain>\.com* 來的請求，需要創建兩個 S3 buckets : 
- **www.\<domain\>.com** : 用來將從 http://www.\<domain\>.com 的請求導向實際有存放網頁的 \<domain\>.com bucket。
  {{< figure src="post_1_3.png" >}}
  Bucket 名稱和域名相同，並選擇自己偏好的 Region（ 基本上離自己的客群所在的地區近一點速度會比較快 ）。
  這邊有個需要小心的地方，預設上 S3 bucket 會阻擋所有從外部來的請求，不過因為在這裡是要拿來設置網頁，如果阻擋的話網頁就沒人看得了，所以這邊需要把
  **Block all public access** 的勾選給取消，取消後下面會有個需要勾選的 Disclaimer。
  因為在網頁上的東西所有人都可以訪問，除非自己採取特別的訪問限制方法，否則不要把非公開的資料內容上傳到 bucket。

  &nbsp;

  Bucket 創建之後，點選它到 Properties 的分頁，在最下方有個 **Static website hosting** 的區塊點擊 **Edit** 來設定讓訪問該 bucket 的所有
  請求都重新導向到 **\<domain\>.com** bucket。
  {{< figure src="post_1_4.png" >}}
  
  &nbsp;

  為了讓所有請求都能夠訪問，還需要編輯 Bucket policy 的存取權限，這邊用 *PublicReadGetObject* 並指定剛創建的 bucket。
   {{< figure src="post_1_5.png" >}}

  設定完成後，可以在 Bucket title 下方看到有個紅色的標籤標示著 **Publicly accessible**。

  &nbsp;

- **\<domain\>.com** : 實際用來存放靜態網頁的 bucket。(所有 html, css 及圖像內容等。)
  創建流程和基本和上面相同，差別在於創建後在 **Static website hosting** 的設定，因為是要用來存放實際網頁內容如 html, css 等，在
  Hosting type 的地方選擇 **Host a static website** 並指定預設根目錄的頁面與404 error 時導向的失敗頁面。
  {{< figure src="post_1_6.png" >}}
  
  之後可以上傳任意 html 檔 ( index.html, 404.html ) 來驗證設定是否無誤。上述設定完成後可以試著點擊 *Bucket website endpoint* 下面的 URL，
  如果設定都正確的話可以看到上傳的 index.html 頁面。(同理若試著進入不存在的目錄則會顯示 404.html 頁面)

&nbsp;

### 建立域名連結

  有了註冊的域名和 S3 bucket 之後，下一步便是建立域名和 bucket 之間的連結。這邊需要透過 Route53 來建立域名和子域名所用的 alias record，
  這個 alias record 會直接對應到 S3 bucket 的 IP 位置端末，讓所有訪問該域名或子域名的請求全部導向對應的 S3 bucket。

  在 Route53 的 Hosted zone 下，如果是透過 Route53註冊的域名應該會有對應域名名稱的 Host Zone 出現，在對應的域名下點擊 **Create record** 來建立
  相對應要給 S3 bucket 使用的 alias record。
  第一次不熟悉 Route53 的設定介面的話可以選右上角的 *Switch to wizard* 介面來做設定。

  &nbsp;

  Routing Policy 上選擇 *Simple routing*，畫面會來 Simple routing records to add to [hosted zone]，點擊 **Define simple record**，
  會跳出一個小視窗：

* Record name : 若要設定給域名用的則直接留白就好，子域名的話就填 www.

* Record type : 選擇 *A - Routes traffic to an IPv4 address and some AWS resources.*

* Value/Route traffic to : 因為要直接對應到 S3 bucket，*Choose endpoint* 的地方選 **Alias to S3 website endpoint**，Region 的欄位選擇 bucket 所在的地域，如果域名有對上的話會 bucket endpoint 會直接出現在下拉選項內。

* Evaluate target health 的選項可以勾掉，按下 **Define simple record**。
  
&nbsp;
  
  {{< figure src="post_1_7.png" >}}

  同理，子域名 (wwww.xxxxx.com) 也用同樣的方式建立 alias record。
  
  &nbsp;

  Alias record 加上之後過了一段時間就可以直接去瀏覽器輸入註冊的域名或子域名來看看是否能進到 S3 bucket。
  到這一步，基本上網頁的架設就已經大致完成了。(單就可以讓外部用戶瀏覽的程度來說XD)

&nbsp;

### 網站部署

這邊根據個人需求，只要是靜態網頁的內容物都可以上傳到 S3 bucket 上。若一樣是選擇用 Hugo 的話，選定某個喜歡的主題生成一些貼文內容後執行 `hugo` command 便會自動生成所有靜態檔案，之後再將生成的檔案全部丟到 S3 bucket 上就行了。（ 要注意不要上傳錯丟到 redirect 用的 bucket）


&nbsp;

### CDN 及 TLS/SSL 設定

這裡應該算是相對麻煩的步驟，但透過 AWS CloudFront 來設置 SSL 不只增加網站的可靠性，同時也可以經由 CDN 來加速網站的載入速度。
在發這篇文的時間點，AWS的官方文件有說明：
```
To use an ACM certificate with CloudFront, make sure you request (or import) the certificate 
in the US East (N. Virginia) Region (us-east-1).
```

去到 AWS Certificate Manager (以下簡稱 ACM) 直接點 **Request a certificate**，憑證類別選 *Request a public certificate*。
把需要申請憑證的域名列出來後，剩下的則看個人需求做調整。
{{< figure src="post_1_8.png" >}}

&nbsp;

申請送出後變等待 Status 欄位會顯示 *Pending validation*，由於上面我選擇用 DNS validation，因此要去把送申請域名的 CNAME record 加到 Route53，
點擊 Pending 的申請後進到頁面後會看到畫面上有個 **Create records in Route53**，AWS 很貼心地準備了這個按鈕讓我們不用手動一個個加到 Hosted zone。
{{< figure src="post_1_9.png" >}}

&nbsp;

按下去之後到 Hosted zone 的頁面就可以看到新增的 CNAME record。之後等一段時間就可以看到域名認證的 Status 欄位轉變成 *Issued* 就算驗證成功了。
{{< figure src="post_1_10.png" >}}

下一步便是去到 CloudFront 去創建要 Distribution，這邊先來創建網域用的，在主要頁面點擊 **Create distribution**，在 Origin domain 的地方選擇
上面創建的 S3 bucket 的 endpoint URL，因為上面我們有啟用 static website hosting 的功能，畫面會跳出一個提示框框表示：
```
This S3 bucket has static web hosting enabled. If you plan to use this distribution as a website, 
we recommend using the S3 website endpoint rather than the bucket endpoint.
```
這邊按下 **Use website endpoint**。下方在 Default cache behavior 的地方，因為要改用 SSL，在 Viewer protocol policy 的地方點選 *Redirect HTTP to HTTPS*，再往下滑到 Alternate domain name (CNAME) - optional，這邊加一下要對應的域名，同時在 Custom SSL certificate - optional 的地方選擇剛才透過
ACM 申請到的憑證。
{{< figure src="post_1_11.png" >}}

同理也為子域名 (www.xxxxx.com) 用上述相同的方式創建一個 distribution。創建之後給他一點時間去做配置。

&nbsp;

有了 CloudFront distribution 後，最後一步就是把 distribution 自動生成的域名加到 Route53 的 Hosted zone。
把 distribution 的域名複製起來後到 Hosted zone 內找到剛才上面所創建的 A Record，原本在創建時是指定直接指向到 S3 bucket，這邊改成指向到 CloudFront 後在下面的欄位貼上剛才複製起來的 distribution 域名。（如果有對應到的話會很貼心地自動出現在下拉選單給你點）

{{< figure src="post_1_12.png" height="80" width="100">}}

&nbsp;

一樣子域名的 A Record 也改成重新指向到對應的 CloudFront distribution。
(在前面的步驟中，若 Hosted zone 內給域名用的 A Record 沒有自動生成的話也沒關係，直接在這邊新創一個就行。 )

最後把瀏覽器的 Cache 清空等待一段時間後重新輸入註冊的域名後應該就可以看到 URL 左側不再是 `Not Secure` 了。

{{< figure src="post_1_13.png" height="80" width="100">}}

&nbsp;

### Github Action