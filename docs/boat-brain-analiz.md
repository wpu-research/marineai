# Boat Brain (OMNI Sonar School) — Ürün Analizi ve Bizim Versiyonumuz

Tarih: 2026-10-25 · Kaynak: Instagram gönderisi (22 Eki 2026), omnisonarschool.com, Stripe ödeme sayfası, basın haberleri.

---

## 1. Ürün özeti (doğrulanmış bilgiler)

| Alan | Bilgi |
|---|---|
| Ürün | **Boat Brain AI System**, satıcı: **Boat Brain LLC** (Stripe hesabı) |
| Arkasındaki kişi | Worth Farrington, OMNI Sonar School: pelajik sportfishing sonar operatörü ve eğitmen |
| Fiyat | **8.999 $** (sistem + kurulum bileşenleri). Stripe linkinde Klarna ve Cash App de var |
| Uyumluluk | Yalnız **Furuno CSH8L (Mk2) ve CSH10** omni sonarlar |
| Lansman | Instagram, 22 Ekim 2026: "now available". Gönderi 136 beğeni ve 11 yorum aldı |
| Aynı firmanın diğer gelirleri | CSH8L / CSH10 online kursları (~2.000 $), topluluk üyeliği (50 $/ay), teknede eğitim ve turnuva sonar operatörlüğü (1.500 $/gün + masraflar) |

**Ürünün sitedeki tanımı:** NVIDIA AI bilgisayarı sonar işlemcisinin HDMI/VGA video çıkışına bağlanıyor. Ekranı gerçek zamanlı izliyor, balık izi (mark) gördüğünde sesli ve görsel uyarı veriyor.

Sitede sayılan özellikler:
1. Canlı sonar videosunda gerçek zamanlı balık tespiti
2. Sesli alarm
3. Telefon uygulamasından kontrol (iPhone). Kumanda paneli dokunmatik ekranından da kullanılabiliyor
4. Telefondan "event" oluşturma
5. Event oluşturulunca **önceki 10 dakikanın** sonar videosunu kaydetme (pre-roll ring buffer)
6. Event'leri sonradan izleme ve arşivleme (eğitim ve karşılaştırma için)
7. **NMEA2000 veri damgası**: event anındaki derinlik, su sıcaklığı, hız, konum vb.
8. AI modeli seçimi ve sistem ayarları
9. Kurulum: 12V besleme, yaklaşık 6"×6"×3" kutu. Sonar işlemcisinden kutuya bir HDMI, kutudan kumanda ekranına bir HDMI (araya girme / passthrough)

Eski taslak sayfalarda (`/test`, `/test2`) geçen yol haritası ipuçları: *"Augmented navigation processor, chart data stored locally, long-range object detection, NMEA2000 heading & GPS"*. Yani sistemi sonardan daha geniş bir "tekne beyni"ne genişletmeyi planlıyorlar.

## 2. Donanım tersine mühendisliği (ürün fotoğraflarından)

Fotoğraflardaki kutu hazır parçalardan toplanmış:

| Parça | Fotoğraftaki kanıt | Tahmini birim maliyet |
|---|---|---|
| **Seeed Studio reComputer Super** (Jetson Orin Nano Super / Orin NX Super) | Ön panelde "reComputer Super" etiketi, 2× RJ45, USB 3.2, DC 12–19V girişi, fanlı soğutucu | ~600 $ (Orin Nano 8GB) – ~1.000–2.400 $ (Orin NX 16GB; bayiye göre çok değişiyor) |
| **USB HDMI capture kartı (loop-out'lu)** | Sağdaki gümüş kutu: ses giriş/çıkış jakları, HDMI giriş ve çıkış | 30–150 $ |
| **Harici USB SSD** (mavi; büyük ihtimalle Samsung T7) | Sol taraf, USB kablolu | 80–150 $ |
| Wi-Fi/BT modülü ve 2 anten | Üstte iki çubuk anten | 20–40 $ |
| Montaj braketi (yeşil, Seeed'in standart braketi), kablolar | — | 20 $ |
| NMEA2000 arayüzü (USB/CAN gateway, ör. Actisense NGX-1 / Yacht Devices) | Fotoğrafta görünmüyor, büyük ihtimalle kutuda ayrı geliyor | 150–300 $ |
| VGA→HDMI çevirici (CSH8'in eski çıkışı için) | Sitede "HDMI/VGA" yazıyor | 20–40 $ |

**Tahmini BOM: ~1.000–3.000 $. Satış fiyatı 8.999 $ → brüt marj %65–85.** Asıl değer donanımda değil; eğitilmiş model, sektörden gelen güvenilirlik ("sonar okulu") ve destek hizmetinde.

Zayıf noktalar:
- **Marine-grade değil.** Fanlı, açık portlu bir geliştirici kiti. IP koruması, tuz/nem dayanımı, 12/24V tekne şebekesine karşı geçici gerilim (load-dump) koruması yok. 24V teknelerde ayrıca DC-DC dönüştürücü gerekiyor.
- Görüntü **ekran yakalamaya** dayanıyor. Ham sonar verisi yok; model, Furuno'nun renk paletine, menü katmanlarına ve ölçek ayarlarına bağımlı.
- Marka bağımlılığı: yalnız Furuno. Gönderinin altındaki ilk yorum da bu: *"Does it work with Simrad SY50?"*

## 3. Yazılım mimarisi (çıkarım)

```
Sonar işlemcisi ──HDMI/VGA──▶ Capture (loop-out) ──HDMI──▶ Kumanda ekranı
                                   │ USB3 (V4L2)
                                   ▼
                         Jetson (reComputer Super)
   ┌───────────────────────────────────────────────────────────────┐
   │ GStreamer capture → ROI kırpma (sonar dairesi)                 │
   │   ├─▶ YOLO tabanlı detektör (TensorRT)  — sınıflar: fish, boat  │
   │   │      └─▶ tracker + alarm kuralları → ses / bildirim         │
   │   └─▶ NVENC H.264/265 segment kaydı → 10 dk ring buffer → SSD   │
   │ NMEA2000 okuyucu (CAN/USB gateway) → event'e veri damgası       │
   │ Yerel web sunucusu / API  ◀── Wi-Fi ──▶ iPhone uygulaması      │
   └───────────────────────────────────────────────────────────────┘
```

Ürün görsellerinde "fish" (kırmızı) ve "boat" (yeşil) sınıf etiketli bounding box'lar var. Bu da YOLO tarzı bir nesne tespit modeli olduğunu gösteriyor. Not: Instagram gönderisi "AI content" etiketli, yani pazarlama görselleri kısmen kompozit olabilir. Tespit doğruluğuna dair yayınlanmış bir veri yok.

## 4. Rekabet

| Ürün | Bilgi |
|---|---|
| **Kongsberg × Viam "Sonar AI"** | Simrad **SY50 / SY60** omni sonarlar için. 24 Mart 2026'da duyuruldu. "World's first AI sonar operator for omnidirectional sonars" diye tanıtılıyor. Balık tespiti, ekrana işaretçi, sesli uyarı ve geçmiş kaydı var. İnternetsiz çalışıyor, bulut yükleme ve paylaşım için **abonelik** gerekiyor. Üretici entegrasyonu olduğu için ham veriye erişimi var |
| **Boat Brain** | Furuno CSH8/CSH10 için harici kutu. Tek seferlik 8.999 $ |
| Akademik/kamu çalışmaları | DIDSON/ARIS akustik kameralarda YOLO, NREL Fish Detection AI modelleri vb. Sportfishing ürünü değil ama yöntem ve veri seti referansı olarak işe yarar |

**Pazar boşluğu:** Furuno'nun kendi AI çözümü yok. Diğer markalar (Furuno FSV-25/FSV-75, CSH5L, Koden, Sonic, Wesmar, JRC) ve ticari balıkçılık tarafı tamamen boş.

Hedef pazar sinyali: CSH10 yaklaşık **84.000 $ + kurulum** fiyatlı bir ürün. 9.000 $'lık bir eklenti bu müşteri için sistemin ~%10'u ediyor. Fiyat hassasiyeti düşük.

## 5. Bizim ürünümüz — öneri

### 5.1 Konumlandırma
**Markadan bağımsız "AI sonar operatörü".** Video çıkışı olan her sonar ve iskandille çalışacak. Boat Brain'in özelliklerinin hepsi olacak, üstüne şu farklar:

1. **Çoklu marka:** Furuno CSH5L/CSH8L/CSH10/FSV serisi, Simrad SY50 ve Koden/Sonic. Her marka ve palet için ayrı model veya domain-adaptation.
2. **Gerçek marine donanım:** Fansız, IP65+ kasa, 9–36V geniş girişli izole besleme, M12/su geçirmez konnektörler, CE/EMC (IEC 60945) hedefi.
3. **Tracking ve analitik:** Sadece "balık var" demek yerine hedefin yönünü, hızını, tekneye mesafesini ve kerterizini ekranın ölçeğini OCR ile okuyarak metrik olarak çıkarmak. Yem sürüsü / tekil büyük balık / tekne izi / dip ayrımı.
4. **Ekran üstü overlay:** Passthrough görüntüye kutu ve iz çizip kumanda ekranına geri basmak (Jetson HDMI çıkışı üzerinden).
5. **Ticari balıkçılık (Türkiye fırsatı):** Karadeniz ve Marmara gırgır filoları hamsi, palamut ve istavrit avında Furuno FSV/CSH sonarları yoğun kullanıyor. Sportfishing'e ek olarak yerli ve büyük bir pazar. Burada sürü büyüklüğü ve biyokütle tahmini ile av günlüğü özellikleri değerli olur.
6. **Veri platformu:** Opsiyonel bulut senkronu, filo panosu, sezon/bölge ısı haritaları ve model güncellemeleri (OTA). Gelir modeli: donanım + yıllık abonelik.

### 5.2 Teknik yığın (MVP)
- **Donanım (prototip):** Jetson Orin Nano Super 8GB (reComputer veya carrier board), USB3 HDMI capture (loop-out), NVMe SSD, Orin'in dahili CAN'ı + transceiver ile doğrudan NMEA2000 (gateway maliyetini düşürür). Alternatif: Yacht Devices / Actisense USB gateway.
- **Capture/encode:** GStreamer + V4L2, NVENC H.265, 1–2 dk'lık segmentler halinde ring buffer (10–30 dk pre-roll).
- **Model:** YOLO11/YOLOv8 (n/s), TensorRT FP16/INT8, ROI'ye kırpılmış 640px giriş, ByteTrack. Alarm kuralı: güven eşiği + N kare kalıcılık + mesafe/sektör filtresi + kendi dümen suyu maskesi.
- **Ekran parametreleri:** OCR (PaddleOCR / Tesseract) ile menzil, tilt ve gain değerlerini okuyup piksel→metre dönüşümü.
- **NMEA2000:** `canboat`/`python-can` veya Signal K server. PGN'ler: konum (129025/129029), COG/SOG (129026), derinlik (128267), su sıcaklığı (130312/130316), heading (127250).
- **Uygulama:** Tekne Wi-Fi AP'si üzerinde yerel web sunucusu + **PWA** (iOS ve Android tek kod). Native push internet istediği için yerelde WebSocket + ses kullanılır. Gerekirse sonradan React Native.
- **Veri ve etiketleme:** CVAT/Label Studio. Model-destekli ön etiketleme ve aktif öğrenme döngüsü.

### 5.3 Yol haritası
| Faz | Süre | Çıktı |
|---|---|---|
| 0. Veri toplama | 4–8 hafta | 3–5 partner teknede (sportfishing + gırgır) capture + NMEA kaydı yapan "kara kutu"lar. Hedef: yüzlerce saat sonar videosu |
| 1. Etiketleme ve model v0 | 4–6 hafta | Furuno CSH paleti için fish/bait/boat/wake sınıfları. Offline doğruluk metrikleri (mAP, saat başına yanlış alarm) |
| 2. MVP kutu | 6–8 hafta | Canlı tespit, alarm, 10 dk pre-roll event, NMEA damgası, PWA |
| 3. Saha pilotu | 1 sezon | Yanlış alarm oranı < X/saat, kaçırılan hedef analizi, kaptan geri bildirimi |
| 4. Ürünleşme | — | Marine kasa, CE/EMC, çoklu marka modelleri, overlay, bulut panosu |

### 5.4 Fiyat önerisi
- Sportfishing (ABD/Akdeniz): 5.000–7.000 $ donanım + 500–1.000 $/yıl model güncelleme ve bulut. Boat Brain'in 8.999 $'ının altında ve marine-grade.
- Ticari (Türkiye gırgır): daha düşük donanım fiyatı + sezonluk abonelik. Filo bazlı satış.

## 6. Riskler ve açık sorular

| Risk | Not / önlem |
|---|---|
| **Veri** en büyük engel | Etiketli sonar videosu yok. Partner tekneler ve sonar operatörleriyle (Boat Brain'in avantajı da buydu) iş birliği şart |
| Ekran-yakalama kırılganlığı | Palet, menü, range ve split-screen değişimleri. ROI tespiti + augmentasyon + marka başına model |
| Yanlış alarm yorgunluğu | Kaptan iki günde alarmı kapatır. Kalıcılık/kural katmanı ve hassasiyet ayarı kullanıcıya açık olmalı |
| Video çıkışı formatları | CSH8L Mk2 ve CSH10 işlemci çıkışlarının formatı ve çözünürlüğü (DVI-D/VGA/HDMI, SXGA vb.) kurulum kılavuzundan **doğrulanmalı**. Bu araştırmada teyit edilemedi. HDCP olmadığı varsayılıyor, test edilmeli |
| Marka/hukuk | "Furuno uyumlu" ifadesi serbest, ama logo ve marka kullanımına dikkat. Kongsberg/Viam ve Boat Brain için patent (FTO) taraması yapılmalı |
| Marine sertifikasyon | IEC 60945 (EMC, çevresel), kumanda yeri pusula güvenli mesafesi |
| Rekabet | Furuno kendi AI'ını çıkarabilir. Kongsberg ham veriye erişimiyle önde. Bizim farkımız markadan bağımsızlık ve ticari pazar olmalı |

## 7. Sonraki adımlar (öneri)
1. CSH8L/CSH10/FSV-25 kurulum kılavuzlarını edinip video çıkış özelliklerini netleştirmek.
2. 1 adet reComputer Super + HDMI capture ile masa üstü prototip kurmak. YouTube'daki ve partnerlerden gelen sonar videolarıyla capture → ROI → kayıt hattını ayağa kaldırmak.
3. Türkiye'de 2–3 gırgır kaptanı ve 2–3 sportfishing teknesiyle veri toplama anlaşması yapmak.
4. Patent ve FTO ön taraması.

---

### Kaynaklar
- Instagram gönderisi: https://www.instagram.com/p/Dey41ItGySB/
- Ürün sayfası: https://omnisonarschool.com/products/
- Ödeme (8.999 $): https://buy.stripe.com/5kQ8wPc0ZgOp9xuf5qbfO02
- Hizmetler: https://omnisonarschool.com/services/ · Kurslar: https://omnisonarschool.com/courses/
- Furuno CSH10: https://www.furuno.com/en/products/sonar/CSH-10
- CSH10 fiyat (BoatUS): https://boatus.geico.com/expert-advice/expert-advice-archive/2025/february/furuno-omni-csh10-sonar
- CSH10 özellikleri (National Fisherman): https://www.nationalfisherman.com/boats-gear/a-new-way-to-see-with-furuno-s-omni-sonar
- CSH8L Mk2: https://boattest.com/furuno-CSH8LMK2-Omni-Sonar
- Kongsberg/Viam Sonar AI: https://www.viam.com/press-releases/kongsberg-launches-sonar-ai · https://www.kongsberg.com/sy60/ · https://www.worldfishing.net/fishing-technology/kongsberg-expands-ai-sonar-partnership/1508825.article
- reComputer Super J4012: https://openelab.com/products/seeed-studio-recomputer-super-j4012
- NREL Fish Detection AI: https://catalog.data.gov/dataset/fish-detection-ai-optic-and-sonar-trained-object-detection-models
