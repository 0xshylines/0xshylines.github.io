---
title: "Kurumsal Yazılımlarda Yanlış Yapılandırma: Yetki Yükseltme Senaryosu (LPE)"
date: 2026-03-10 00:00:00 +0300
categories: [Araştırma, Yerel Yetki Yükseltme]
tags: [lpe, windows-security, exploitation, poc, writeup]
image:
  path: /assets/img/posts/lpe-analysis/step1.png
---

## Giriş: Bir Zafiyet Zincirinin Anatomisi

Siber güvenlik dünyasında çoğu zaman tek bir hata sistemi ele geçirmeye yetmez. Ancak birkaç küçük "ihmal" bir araya geldiğinde, en düşük yetkili kullanıcıyı bile sistemin mutlak hakimi (SYSTEM) yapabilir. Bu yazıda, kurumsal bir yazılımın kurulum ve yapılandırma süreçlerinde tespit ettiğim üç farklı zafiyetin, nasıl birleşerek kritik bir **Local Privilege Escalation (LPE)** vektörüne dönüştüğünü inceleyeceğiz.

---

## 1. İlk Kapı: Kimlik Doğrulama Mekanizmasının Atlatılması

Analiz sürecimiz, kurulumun bizden **hedef dizin** içerisinde bir `.p12` teknik sertifikası talep etmesiyle başladı. Normal şartlarda bu sertifikanın güvenilir bir otorite tarafından imzalanmış olması gerekir. Ancak yaptığımız testlerde, OpenSSL ile ürettiğimiz **self-signed** (kendinden imzalı) sahte bir sertifikanın sistem tarafından sorgusuz sualsiz kabul edildiğini gördük.

![Sertifika Talebi](/assets/img/posts/lpe-analysis/step1.png)
_Görsel 1: Kurulumun sertifika beklediği aşama._

Uygulama, sertifikanın geçerliliğini doğrulamadığı için "ilk kale" kolayca düşmüş oldu. Sahte sertifika ile kurulum başarıyla tamamlandı.

## 2. Kalıcılık ve Yüksek Yetki Analizi

Kurulum bittikten sonra sistemin arka planında neler döndüğünü incelemek için Windows Görev Zamanlayıcı'yı (Task Scheduler) kontrol ettik. Karşılaştığımız tablo oldukça ilginçti: Kurulum, **SYSTEM** yetkileriyle çalışan ve periyodik olarak tetiklenen çok sayıda zamanlanmış görev oluşturmuştu.

![Görev Zamanlayıcı Analizi](/assets/img/posts/lpe-analysis/step3.png)
_Görsel 3: SYSTEM yetkisiyle çalışan görevlerin listesi._

Bu görevlerin ortak noktası, **hedef dizin** içerisindeki `.bat` uzantılı script dosyalarını çalıştırmalarıydı.

## 3. Zayıf Dosya İzinleri: En Zayıf Halka (CWE-276)

İşte zafiyetin asıl sömürü noktası tam olarak burası. Yüksek yetkili görevlerin çalıştırdığı bu script dosyalarının bulunduğu **hedef dizin** üzerindeki güvenlik izinlerini (ACL) incelediğimizde, **"Authenticated Users"** grubuna **"Yazma"** ve **"Değiştirme"** yetkisi verildiğini tespit ettik.

![ACL İzinleri](/assets/img/posts/lpe-analysis/step2.png)
_Görsel 2: Standart kullanıcılara verilen yüksek yetki tanımlamaları._

Bu durum şu anlama geliyor: Sistemdeki herhangi bir standart kullanıcı, SYSTEM tarafından çalıştırılacak olan script'in içeriğini değiştirebilir.

## 4. Teknik Detaylar ve Runtime Exceptions

Yazılımın Java tabanlı bileşenleri, sertifika doğrulama süreçlerinde mantıksal hatalar (`ArrayIndexOutOfBoundsException`) üretmektedir. Bu durum, yazılımın girdi doğrulama (input validation) süreçlerindeki eksiklikleri ve stabilite sorunlarını gözler önüne sermektedir.

![Hata Analizi](/assets/img/posts/lpe-analysis/step4.png)
_Görsel 4: Çalışma zamanında oluşan teknik istisna detayları._

## 5. Kavram Kanıtı (PoC): SYSTEM Yetkisine Erişim

Tespit edilen zayıf dosya izinlerini sömürmek için standart bir kullanıcıyla **hedef dizin** altındaki script dosyasını manipüle ettik. Amacımız, SYSTEM yetkileriyle komut çalıştırabildiğimizi kanıtlamaktı.

![OpenSSL PoC](/assets/img/posts/lpe-analysis/step5.png)
_Görsel 5: Test amaçlı sertifika üretim süreci._

Görev tetiklendiğinde, düşük yetkili kullanıcının normalde erişemeyeceği dizinlerde SYSTEM imzalı dosyaların oluştuğunu gözlemledik.

## 6. Sonuç ve Öneriler

Yapılan analizler sonucunda; güvensiz dosya izinleri ile yüksek yetkili zamanlanmış görevlerin birleşimi, yerel bir saldırganın sistem üzerinde tam kontrol sağlamasına imkan tanımaktadır.

![Final Analizi](/assets/img/posts/lpe-analysis/step6.png)
_Görsel 6: Tüm bulguların birleştiği teknik görünüm._

### Güvenlik Tavsiyeleri

> * **ACL Sıkılaştırması:** Uygulama dizinlerindeki yazma yetkileri "Authenticated Users" grubundan kaldırılmalı, sadece gerekli servis hesaplarına kısıtlı izin verilmelidir.
> * **Sertifika Doğrulaması:** Güvenilmeyen veya kendinden imzalı sertifikalarla kurulum yapılmasına izin verilmemelidir.
> * **Güvenli Dosya Hiyerarşisi:** Kritik scriptler, sadece yönetici yetkisiyle değiştirilebilen güvenli dizinlerde tutulmalıdır.
