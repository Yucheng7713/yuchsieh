---
title: "非前端的掙扎：如何在雲端平台架設個人網站 - AWS Route53, S3, Hugo"
date: 2022-10-28T22:22:20+08:00
draft: false
featuredImagePreview: "/images/post_1_cover.png"
---

{{< figure src="/images/post_1_cover.png" >}}
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

{{< figure src="/images/post_1_1.png" >}}

&nbsp;

之後變可以輸入自己想要且還沒有被註冊的域名及網域，需注意的是不同網域的註冊及更新費用也會有所不同。之後變跟隨畫面指示一步步完成註冊手續。
域名註冊後，Route53 會自動為新註冊的域名啟用 DNS，同時並創立一個名稱跟域名相同的公開 hosted zone．
{{< figure src="/images/post_1_2.png" >}}
Route53 會指派四個 name servers 到這個新創立的 hosted zone，這邊要稍微注意的是剛註冊好的域名需要等一段時間才會有效。



### S3 Bucket 設置
這邊為了同時應付從 *http://\<domain\>.com* 及 *http://www.\<domain>\.com* 來的請求，需要創建兩個 S3 buckets : 
- **www.\<domain\>.com** : 用來將從 http://www.\<domain\>.com 的請求導向實際有存放網頁的 \<domain\>.com bucket。
  {{< figure src="/images/post_1_3.png" >}}
  Bucket 名稱和域名相同，並選擇自己偏好的 Region（ 基本上離自己的客群所在的地區近一點速度會比較快 ）。
  這邊有個需要小心的地方，預設上 S3 bucket 會阻擋所有從外部來的請求，不過因為在這裡是要拿來設置網頁，如果阻擋的話網頁就沒人看得了，所以這邊需要把
  **Block all public access** 的勾選給取消，取消後下面會有個需要勾選的 Disclaimer。
  因為在網頁上的東西所有人都可以訪問，除非自己採取特別的訪問限制方法，否則不要把非公開的資料內容上傳到 bucket。

  &nbsp;

  Bucket 創建之後，點選它到 Properties 的分頁，在最下方有個 **Static website hosting** 的區塊點擊 **Edit** 來設定讓訪問該 bucket 的所有
  請求都重新導向到 **\<domain\>.com** bucket。
  {{< figure src="/images/post_1_4.png" >}}
  
  &nbsp;

- **\<domain\>.com** : 實際用來存放靜態網頁的 bucket。(所有 html, css 及圖像內容等。)

  

### 網站部署


### CDN 及 TLS/SSL 設定

### Github Action