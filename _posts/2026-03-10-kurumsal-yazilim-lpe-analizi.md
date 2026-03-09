---
title: "Kurumsal Yazılımlarda Yanlış Yapılandırma: Yetki Yükseltme Senaryosu (LPE)"
date: 2026-03-10 00:00:00 +0300
categories: [Araştırma, Yerel Yetki Yükseltme]
tags: [lpe, windows-security, exploitation, poc, writeup]
image:
  path: /assets/img/posts/lpe-analysis/step1.png
---

## Giriş: Bir Zafiyet Zincirinin Anatomisi

Siber güvenlik dünyasında çoğu zaman tek bir hata sistemi ele geçirmeye yetmez. Ancak birkaç küçük "ihmal" bir araya geldiğinde, en düşük yetkili kullanıcıyı bile sistemin mutlak hakimi (SYSTEM) yapabilir. Bu yazıda, kurumsal bir yazılımın (TDFA) kurulum ve yapılandırma süreçlerinde tespit ettiğim üç farklı zafiyetin, nasıl birleşerek kritik bir **Local Privilege Escalation (LPE)** vektörüne dönüştüğünü inceleyeceğiz.

## 1. İlk Kapı: Kimlik Doğrulama Mekanizmasının Atlatılması (CWE-295)

Analiz sürecimiz, kurulumun bizden `C:\MASTER` dizininde bir `.p12` teknik sertifikası talep etmesiyle başladı. Normal şartlarda bu sertifikanın güvenilir bir otorite tarafından imzalanmış olması gerekir. Ancak yaptığımız testlerde, OpenSSL ile ürettiğimiz **self-signed (kendinden imzalı)** sahte bir sertifikanın sistem tarafından sorgusuz sualsiz kabul edildiğini gördük.

![Sertifika Talebi](/assets/img/posts/lpe-analysis/step1.png)
_Görsel 1: Kurulumun sertifika beklediği o an._

Uygulama, sertifikanın geçerliliğini doğrulamadığı için "ilk kale" kolayca düşmüş oldu. Sahte sertifika ile kurulum başarıyla tamamlandı.

## 2. Kalıcılık ve Yüksek Yetki Analizi

Kurulum bittikten sonra sistemin arka planında neler döndüğünü incelemek için Windows Görev Zamanlayıcı'yı (Task Scheduler) kontrol ettik. Karşılaştığımız tablo oldukça ilginçti: Kurulum, **SYSTEM** yetkileriyle çalışan ve periyodik olarak tetiklenen çok sayıda zamanlanmış görev oluşturmuştu.

![Görev Zamanlayıcı Analizi](/assets/img/posts/lpe-analysis/step3.png)
_Görsel 3: SYSTEM yetkisiyle çalışan görevlerin listesi._

Bu görevlerin ortak noktası, `C:\tdfa\appli\` dizinindeki `.bat` uzantılı script dosyalarını çalıştırmalarıydı.

## 3. Zayıf Dosya İzinleri: En Zayıf Halka (CWE-276)

İşte zafiyetin "vuruş" noktası tam olarak burası. Yüksek yetkili görevlerin çalıştırdığı bu `.bat` dosyalarının bulunduğu klasörün güvenlik izinlerini (ACL) incelediğimizde, **"Authenticated Users"** grubuna **"Yazma"** ve **"Değiştirme"** yetkisi verildiğini gördük.

![ACL İzinleri](/assets/img/posts/lpe-analysis/step2.png)
_Görsel 2: Standart kullanıcılara verilen "Full Control" benzeri yetkiler._

Bu durum şu anlama geliyor: Sistemdeki en düşük yetkili kullanıcı bile, SYSTEM tarafından çalıştırılacak olan script'in içeriğini değiştirebilir.

## 4. Kavram Kanıtı (PoC): SYSTEM Yetkisine Merhaba

Zayıf dosya izinlerini sömürmek için standart bir kullanıcıyla `Batch_TDFA.bat` dosyasını açıp içine basit bir komut ekledik. Amacımız, SYSTEM yetkileriyle dosya yazabildiğimizi kanıtlamaktı.

![PoC Denemesi](/assets/img/posts/lpe-analysis/step4.png)
_Görsel 4: Script dosyasının manipüle edilmesi._

Görev tetiklendiğinde, düşük yetkili kullanıcının erişemeyeceği dizinlerde SYSTEM imzalı dosyaların oluştuğunu gözlemledik. Bu, saldırganın sistem üzerinde tam kontrol elde edebileceğinin en net kanıtıdır.

## Etki Değerlendirmesi

Bu zafiyet zinciri sömürüldüğünde, saldırgan:
* Tüm kurumsal verileri çalabilir veya şifreleyebilir (Ransomware riski).
* Sistemi bir sıçrama tahtası olarak kullanarak ağda **Lateral Movement** (Yanal Hareket) yapabilir.
* Kalıcı arka kapılar (Backdoor) yerleştirerek erişimini sabitleyebilir.

## Sonuç ve Öneriler

Kurumsal yazılımların kurulum süreçleri, sadece işlevsellik değil, güvenlik perspektifiyle de denetlenmelidir. Bu vaka özelinde;
1.  **En Az Yetki Prensibi:** Uygulama dizinlerinde yazma yetkisi sadece yönetici gruplarıyla sınırlandırılmalıdır.
2.  **Sertifika Doğrulaması:** Güvenilmeyen sertifikalarla kurulum yapılmasına izin verilmemelidir.
3.  **Güvenli Klasör Hiyerarşisi:** Kritik scriptler, standart kullanıcıların erişemeyeceği `Program Files` gibi korumalı dizinlerde tutulmalıdır.

---
*Bu analiz Oğuzhan KARASU tarafından Secunnix bünyesinde gerçekleştirilmiştir.*
