# Kredi Kartı Müşteri Churn Analizi

> Python ile Veri Bilimi Dersi — Dönem Sonu Projesi  
> Bursa Uludağ Üniversitesi | 2025–2026

\---

## Proje Başlığı

**Kredi Kartı Müşteri Churn Tahmini: Maliyet Odaklı Makine Öğrenmesi ve Finansal Simülasyon**

\---

## Öğrenci Bilgileri

Metin Mutlu<br>
Öğrenci Numarası: 132230127<br>
Ders: Python ile Veri Bilimi

\---

## Projenin Amacı

Bu çalışmada, bir bankanın kredi kartı portföyünden derlenen **10.127 müşteriye** ait demografik ve finansal veriler kullanılarak hangi müşterilerin kartı bırakmaya (churn) meyilli olduğu makine öğrenmesi yöntemleriyle tahmin edilmektedir.

Çalışmanın standart churn tahmin yaklaşımlarından ayrıştığı iki temel nokta:

1. **Data Fusion:** Tek bir hazır CSV yerine iki farklı kaynak (Kaggle müşteri verisi + TCMB enflasyon verisi) anlamlı biçimde birleştirilmiş; bu harmonlama iş mantığına dayalı yeni değişkenler türetmek için kullanılmıştır.
2. **Finansal Simülasyon:** Modelin başarısı yalnızca teknik metriklerle (AUC, F1) değil, doğrudan bir ROI senaryosuyla kanıtlanmıştır. Standart %50 tahmin eşiğinin neden finansal olarak yetersiz kaldığı matematiksel hesaplama ile gösterilmiştir.

**Temel araştırma sorusu:** *Hangi tahmin eşiği değerinde elde tutma kampanyasının net karı maksimum olur ve standart 0.50 eşiği finansal açıdan ne kadar kayıba yol açmaktadır?*

\---

## Veri Kaynakları

### 1\. BankChurners.csv (Ana Veri Seti)

Kaggle platformunda [Sakshi Goyal](https://www.kaggle.com/datasets/sakshigoyal7/credit-card-customers) tarafından yayımlanan bu veri seti, 10.127 müşteriye ait 23 değişkeni içermektedir.

* **Demografik:** Yaş, cinsiyet, medeni durum, öğrenim, gelir kategorisi
* **Hesap:** Kart türü, bankadaki ay sayısı, kredi limiti, dönen bakiye
* **Davranışsal:** İşlem sayısı/tutarı, inaktif ay sayısı, kullanım oranı
* **Hedef değişken:** `Attrition\_Flag` — Existing Customer / Attrited Customer
* **Churn oranı:** %16.1 (sınıf dengesizliği mevcut)

> BankChurners.csv ve enflasyon.csv repoya eklenmiştir. Alternatif erişim için Drive linki:  
> https://drive.google.com/drive/folders/1urzrx-bEbJb-aHDbIK91vUf4Y8VdDKOd

### 2\. enflasyon.csv (TCMB Verileri)

TCMB yıllık TÜFE enflasyon oranları ve politika faizinden elle derlenen 6 satırlık veri seti (2016–2021). Data Fusion'ın temel girdisidir.

\---

## Veri Harmanlama (Data Fusion)

İki veri kaynağının birleştirilmesi şu mekanizma ile gerçekleştirilmiştir:

```
katilim\_yili = 2021 − round(Months\_on\_book / 12)
```

Bu yıl değeri TCMB enflasyon tablosuyla LEFT JOIN ile eşleştirilmiş; her müşteriye bankaya katıldığı dönemin enflasyon oranı atanmıştır.

**Gerekçe:** 2018'de %20 enflasyon yaşandığında nominal harcama artışı, gerçek satın alma gücü artışını yansıtmamaktadır. Enflasyona göre düzeltilmiş harcama değişkeni bu harmanlamanın doğrudan çıktısıdır.

\---

## İş Mantığına Dayalı Özellik Mühendisliği

|Değişken|Formül|İş Mantığı|
|-|-|-|
|`musteri\_yasam\_degeri`|`Total\_Trans\_Amt × (Months\_on\_book / 12)`|Kümülatif müşteri değeri (CLV Proxy)|
|`reel\_harcama\_degisimi`|`Total\_Amt\_Chng\_Q4\_Q1 / (1 + enflasyon / 100)`|**Data Fusion çıktısı:** Reel satın alma gücü değişimi|
|`aktiflik\_skoru`|`Trans\_Ct × (1 − inaktif\_oran) / (sikayet + 1)`|RFM benzeri bağlılık endeksi|

> SHAP analizinde üç değişken de ilk 5 önemli özellik arasına girmiştir.

\---

## Model ve Sonuçlar

**Algoritma:** Random Forest Classifier (`class\_weight='balanced'`, `n\_estimators=200`, `max\_depth=15`)

|Metrik|Değer|
|-|-|
|AUC-ROC (Test Seti)|**0.9856**|
|5-Fold CV AUC|**0.9272 ± 0.001**|
|Precision / Recall / F1 (Churn)|**0.85 / 0.85 / 0.85**|

\---

## Finansal Simülasyon — Optimal Eşik Belirleme

### Maliyet Matrisi

|Durum|Net Etki|
|-|-|
|TP — Doğru churn tahmini, kampanya gönderildi|**+950 TL**|
|FP — Yanlış alarm, boşa promosyon|**−100 TL**|
|FN — Kaçırılan churner, müşteri gitti|**−1.500 TL**|
|TN — Doğru kalan, hareketsiz|**0 TL**|

FP / FN maliyet oranı **15:1** → Bir churner'ı kaçırmak, 15 kişiye yanlış alarm vermekten daha pahalıdır.

### Eşik Optimizasyonu Sonuçları

|Senaryo|Eşik|TP|FP|FN|Net Kâr|
|-|-|-|-|-|-|
|Standart uygulama|0.50|278|47|47|184.000 TL|
|**OPTIMAL (bu çalışma)**|**0.20**|**321**|**208**|**4**|**276.150 TL**|
|Fark|—|+43|+161|−43|**+92.150 TL**|

> \*\*Sonuç:\*\* 0.20 eşiği ile 325 gerçek churner'dan 321'i (%98.8) tespit edilmiştir. Standart uygulamaya kıyasla \*\*+92.150 TL\*\* ek finansal değer üretilmiştir.

\---

## Açıklanabilir YZ (SHAP) — En Etkili 5 Değişken

|Sıra|Değişken|Etki Yönü|İş Yorumu|
|-|-|-|-|
|1|`Total\_Trans\_Ct`|Düşük → Churn ↑|45 işlem altındaki müşteriler kritik risk grubu|
|2|`Total\_Trans\_Amt`|Düşük → Churn ↑|Düşük harcama hacmi ikinci en güçlü sinyal|
|3|`aktiflik\_skoru` ⭐|Düşük → Churn ↑|**Yeni feature** — bağlılık düşünce risk yükseliyor|
|4|`Total\_Revolving\_Bal`|Düşük → Churn ↑|Kredi kullanmayan müşteri daha az bağlı|
|5|`Avg\_Utilization\_Ratio`|Düşük → Churn ↑|Limitini az kullanan müşteri daha az engaged|

> ⭐ Bu değişken projede türetilmiştir; ham veride mevcut değildir.

\---

## Dosya Yapısı

```
kredi-karti-churn-analizi/
├── churn\_analysis.ipynb      # Ana analiz notebook (32 hücre)
├── kredi\_karti\_yonetici\_ozeti.pdf        # Yönetici özeti raporu
├── README.md
├── requirements.txt
├── data/
│   ├── enflasyon.csv         # TCMB 2016-2021 TÜFE verileri (6 satır)
│   └── BankChurners.csv      # Ana veri seti — Kaggle (10.127 müşteri)
└── gorseller/
    ├── eda\_gorsel.png
    ├── korelasyon.png
    ├── feature\_engineering.png
    ├── roc\_curve.png
    ├── shap\_importance.png
    ├── shap\_beeswarm.png
    ├── finansal\_simulasyon.png
    └── confusion\_matrix.png
```

\---

## Kullanılan Kütüphaneler

```
pandas, numpy, matplotlib, seaborn, scikit-learn, shap
```

\---

## Not

BankChurners.csv (1.4 MB) GitHub'ın dosya boyutu limitinin çok altında olduğundan doğrudan repoya eklenmiştir. Yine de Drive linki yukarıda verilmiştir.

\---

## Hakkında

Python ile Veri Bilimi dersi dönem sonu projesi: bankacılık sektöründe maliyet odaklı churn tahmini ve finansal simülasyon.

