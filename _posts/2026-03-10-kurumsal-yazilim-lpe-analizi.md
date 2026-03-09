---
title: "Kurumsal Yazılımlarda Yanlış Yapılandırma: Yetki Yükseltme Senaryosu (LPE)"
date: 2026-03-10 00:00:00 +0300
categories: [Research, Local Privilege Escalation]
tags: [lpe, windows-security, exploitation, poc, writeup]
image:
  path: /assets/img/posts/lpe-analysis/step0.png
---

## Giriş: Bir Zafiyet Zincirinin Anatomisi

Yazıma hoş geldiniz.. 
Siber güvenlik dünyasında çoğu zaman tek bir hata sistemi ele geçirmeye yetmez. Ancak birkaç küçük "ihmal" bir araya geldiğinde, en düşük yetkili kullanıcıyı bile sistemin mutlak hakimi (SYSTEM) yapabilir. Bu yazıda, gerçek bir kurumsal  yazılımın kurulum ve yapılandırma süreçlerinde tespit ettiğim üç farklı zafiyetin, nasıl birleşerek kritik bir **Local Privilege Escalation (LPE)** vektörüne dönüştüğünü inceleyeceğiz.

---

## 1. Kimlik Doğrulama Mekanizmasının Atlatılması

Analiz sürecimiz, kurulumun bizden **hedef dizin** içerisinde bir `.p12` teknik sertifikası talep etmesiyle başladı. Normal şartlarda bu sertifikanın güvenilir bir otorite tarafından imzalanmış olması gerekir. Ancak yaptığımız testlerde, OpenSSL ile ürettiğimiz **self-signed** (kendinden imzalı) sahte bir sertifikanın sistem tarafından sorgusuz sualsiz kabul edildiğini gördük.

![Sertifika Talebi](/assets/img/posts/lpe-analysis/step1.png)
_Görsel 1: Kurulumun sertifika beklediği aşama._


## 2. Sahte Sertifika Üretimi

Kurulum sihirbazı, meşru bir kimlik doğrulaması için hedef dizininde geçerli bir teknik sertifika (.p12) dosyası talep ediyordu. Bu noktada, yazılımın sertifika güven zincirini (Chain of Trust) doğrulayıp doğrulamadığını test etmek amacıyla bir "Counterfeit Identity" (Sahte Kimlik) saldırısı kurgulandı.

![Sahte sertifika üretimi](/assets/img/posts/lpe-analysis/step2.png)
_Görsel 2: OpenSSL üzerinden sahte sertifika üretim ve dışa aktarım süreci._

## 3. Sertifikanın Yerleştirilmesi ve Kurulumun Tetiklenmesi (CWE-276)

Yazılımın en büyük güvenlik açığı burada ortaya çıktı, eksik Sertifika Doğrulaması (CWE-295). Kurulum mekanizması, sunulan sertifikanın güvenilir bir sertifika otoritesi (CA) tarafından imzalanıp imzalanmadığını kontrol etmedi. Kendi ürettiğimiz bu "sahte" sertifika ve belirlediğimiz parola sisteme girildiğinde, kurulum sihirbazı kimliğimizi doğrulanmış kabul ederek bir sonraki aşamaya geçti.

Bu noktada kritik olan, yazılımın dosya bütünlüğü veya imza doğrulaması yapıp yapmayacağıydı. Kurulum ekranına geri dönüp "Devam Et" komutunu verdiğimizde, yazılımın herhangi bir "Güvenilir Sertifika Otoritesi" (CA) kontrolü yapmadan sahte sertifikayı doğrudan kabul ettiğini gözlemledik.

![Sahte sertifika](/assets/img/posts/lpe-analysis/step3.png)
_Görsel 3: Sahte sertifikanın kabul edilmesiyle birlikte kurulum sürecinin başarıyla tetiklenmesi._


## 4. Arka Kapıları Aramak — Görev Zamanlayıcı Analizi

Kurulum sihirbazı işini bitirip sahneden çekildiğinde, bir sızma testi uzmanı için asıl "eğlence" yeni başlar: Post-Exploitation. Bu aşamada ilk kural şudur: Sisteme ne ekildi, nerede kök saldı ve en önemlisi; bu kökler hangi yetkiyle besleniyor?

Sistemin derinliklerine inmek için ilk durağımız Windows Görev Zamanlayıcısı (Task Scheduler) oldu. Amacımız, uygulamanın sistemde bıraktığı kalıcılık (persistence) izlerini sürmekti.

![Zamanlanmış görev analizi](/assets/img/posts/lpe-analysis/step4.png)
_Görsel 4: SYSTEM ayrıcalıklarıyla periyodik olarak tetiklenen zamanlanmış görevlerin listesi._

Yapılan incelemede yazılımın sistem her açıldığında veya belirli aralıklarla kendini tekrar eden görevler zinciri oluşturduğunu gördük.

## 5. Zayıf Klasör İzinleri

Önceki adımda, sistemin en tepesindeki yetkiyle (SYSTEM) çalışan bir mekanizmanın varlığını kanıtlamıştık. Ancak siber güvenlikte "yetki" tek başına bir zafiyet değildir; asıl tehlike, bu yüksek yetkinin kontrolsüz bir alana dokunmasıdır. Bu noktada kritik bir soru sorduk: "SYSTEM yetkisiyle tetiklenen bu batch.bat dosyasının bulunduğu dizine, standart bir kullanıcı olarak müdahale edebilir miyiz?"

Bu sorunun cevabını almak için hedef dizin üzerindeki Erişim Kontrol Listesi (ACL) analizine başladık. Amacımız, buraya zararlı bir exploit veya "malicious" bir kod parçası bırakıp bırakamayacağımızı tespit etmekti.

![ACL Kontrolü](/assets/img/posts/lpe-analysis/step5.png)
_Görsel 5: Standart kullanıcılara verilen "Full Control" benzeri tehlikeli yetki tanımlamaları._

Karşılaştığımız tabloda sistemdeki herhangi bir standart (düşük yetkili) kullanıcıyı kapsayan bu gruba, kritik script dosyalarının bulunduğu dizin üzerinde "Yazma (Write)" ve "Değiştirme (Modify)" yetkileri verilmişti.

## 6. Final Perde — SYSTEM Yetkisiyle Komut Çalıştırma

Saldırı zincirimizin tüm halkaları artık birleşti: Sahte sertifika ile kurulumu başlattık, SYSTEM yetkisiyle çalışan kalıcılık mekanizmalarını bulduk ve bu mekanizmaların dokunduğu dizindeki zayıf izinleri (ACL) tespit ettik. Şimdi ise teorideki bu zafiyeti, pratikte bir Yerel Yetki Yükseltme (LPE) zaferine dönüştürme vakti.

Bu aşamada, standart (düşük yetkili) kullanıcı hesabımızla hedef dizin içerisindeki kritik .bat dosyasını manipüle ettik. Orijinal içeriği, sistem üzerinde yetki seviyemizi kanıtlayacak bir komut dizisiyle değiştirdik.

![Final Analizi](/assets/img/posts/lpe-analysis/step6.png)
_Görsel 6: Manipüle edilen script'in SYSTEM yetkileriyle tetiklendiği ve yetki yükseltmenin kanıtlandığı final aşaması._

Bu durum, sadece bir yapılandırma hatasının değil, tüm sistemin kontrolünün ele geçirilebileceği kritik bir güvenlik açığının kanıtıdır. :) 

## Peki, bu analizden ne öğrenmeliyiz?
Gördüğümüz üzere, "küçük" görünen bir yanlış yapılandırma, sistemin mutlak hakimiyetini (SYSTEM) saldırgana altın tepside sunabiliyor. Siber güvenlik, sadece devasa saldırılara karşı durmak değil; her bir izin satırını, her bir sertifika doğrulamasını ve her bir zamanlanmış görevi titizlikle incelemektir.

### Güvenlik Tavsiyeleri

> * **ACL Sıkılaştırması:** Uygulama dizinlerindeki yazma yetkileri "Authenticated Users" grubundan kaldırılmalı, sadece gerekli servis hesaplarına kısıtlı izin verilmelidir.
> * **Sertifika Doğrulaması:** Güvenilmeyen veya kendinden imzalı sertifikalarla kurulum yapılmasına izin verilmemelidir.
> * **Güvenli Dosya Hiyerarşisi:** Kritik scriptler, sadece yönetici yetkisiyle değiştirilebilen güvenli dizinlerde tutulmalıdır.



