---
title: Raspberry Pi ile KDE Connect 
description: Monitörsüz(headless) bir Raspberry Pi cihazda KDE Connect kurulumu
author: melihguclu
date: 2024-11-24 
categories: [Linux, Programlar]
tags: [shell, linux, raspberrypi, kdeconnect]
toc: true
lang: tr
---

## Giriş ve Amaç
Bu yazıda **monitörsüz(headless) Raspberry Pi** cihazımıza **KDE Connect** yükleyip Android eşlik uygulaması veya başka bir Linux bilgisayar aracılığıyla kontrol etmeyi öğreneceğiz.

KDE Connect tasarım itibariyle büyük oranda görsel kullanıcı arayüzüne ve dbus servislerine bağımlı. Bu nedenle kurulum sırasında birkaç ekstra ayar yapmamız gerekecek.

## Kurulum

KDE Connect yerel ağınızdaki diğer cihazlarınızla bağlantı kurabilmek için Linux'un MDNS aracı olan [avahi](/posts/avahi) programını kullanıyor. Buna ek olarak [dbus](/posts/dbus) paketini de kuruyoruz.

```bash
sudo apt update && sudo apt upgrade
sudo apt install kdeconnect avahi dbus
```
## Konfigürasyon

Kurulum sonrası çalışmaya başlayan ***kdeconnect*** servisini devre dışı bırakmamız gerekiyor.Aksi takdirde servis bir grafik arayüzü bulamadığı için hata verip kapanacak.

Bu servis dosyasını bulmak bazen göründüğü kadar kolay olmayabilir.
Bu nedenle önce
1. `systemctl --user status | grep kdeconnect`
ile servisin tam adını bulup

2. `systemctl --user disable --now <servis-adı>`
ile devre dışı bırakmamız gerekecek.

3. Ayrıca güvenlik duvarının avahi ve kdeconnect'e izin verecek şekilde ayarlanması gerekecek. [^ref1]

4. KDE Connect'i grafik arayüzü olmadan kullanabilmek için `-platform offscreen` argümanı ile başlatacağız. [^ref1]

### Yeni Servis Dosyası
Favori editörümüzü kullanarak yeni bir [***systemd***](/posts/systemd) servis dosyası oluşturalım.

`vim /home/melih/.config/systemd/user/kdeconnectd-offscreen.service`

```bash
[Unit]
Description=KDE Connect user service for headless setup

[Service]
Restart=always
Type=simple
WorkingDirectory=/home/melih

ExecStart=/usr/lib/arm-linux-gnueabihf/libexec/kdeconnectd -platform offscreen

[Install]
WantedBy=default.target
```
 
Aşağıdaki komut ile yeni servisimizi etkinleştirelim ve çalıştıralım.

`systemctl --user enable --now kdeconnectd-offscreen.service`

Servisi doğru bir şekilde yapılandırdık fakat servisin tetiklenmesi için hala SSH ya da monitör-klavye kullanarak oturum açmamız gerekiyor. 

`loginctl enable-linger` komutu KDEConnect'in sistem ile birlikte başlamasını sağlayacak ve her defasında oturum açmamız gerekmeyecek.

> Bu diğer benzer servislerinin de otomatik başlamasına sebep olabilir.
{: .prompt-warning }

## Kullanım

1. `kdeconnect-cli -l` komutu diğer KDEConnect cihazlarını ve onların ID'lerini listeliyor.

2. `kdeconnect-cli --pair -d <device-id>` komutu ise eşleştirmek için bir istek gönderiyor.

### Komut çalıştırma
KDE Connect'in komut satırı arayüzü, grafik arayüzüne kıyasla daha az özelliğe sahip. Yine de küçük bir araştırma ile **"Run Command"** özelliğinin nasıl çalıştığını anlayabiliriz.

- Run command ile ilişkili ayar dosyası **~/.config/kdeconnect/\<deviceid>/kdeconnect_runcommand/config** dizininde bulunuyor.

- Oluşturduğum basit aracı kullanarak komutları kolayca ekleyip kullanabilirsiniz. 
<https://github.com/melihguclu0/rcgenerate>

```bash
[General]
 commands="@ByteArray({\"_12345678_90ab_cdef_0123_000000000000_\":{\"command\":\"/home/melih/scripts/system_temp.sh\",\"name\":\"CPUTemp\"},\"_12345678_90ab_cdef_0123_000000000001_\":{\"command\":\"/home/melih/scripts/temp.sh\",\"name\":\"temp\"},\"_e0067ef0_902b_447a_878c_000000000002_\":{\"command\":\"/home/melih/scripts/service_stat.sh\",\"name\":\"Query-Stats\"}})"
```
Bu dosyadaki ilk bölümü ele alacak olursak:

1. *`_12345678_90ab_cdef_0123_000000000000_`*
: Bu karmaşık yazı, komutu tanımlayan benzersiz bir tanımlayıcı. Milyarlarca farklı komutumuz olmadığı sürece tanımlayıcıların aynı olmaması ve formatın sabit kalması bizim için yeterli. 

2. *`/home/melih/scripts/system_temp.sh`*
: İkinci alan komut yürütüldüğünde çalışacak program.

3. *`CPUTemp`*
: sonuncusu ise komut adı.

Bu dosyayı çalıştırmak istediğimiz  komutlara göre değiştirerek "Run Command" özelliğini kullanabiliriz.
Fakat yapılan değişiklikler ***\<device_id>*** ile belirtilen cihaz ile sınırlı kalacak. Eğer aynı komutları diğer cihazlarda da kullanmak istersek sembolik link kullanmamız gerekecek.

```bash
ln -s /home/<username>/.config/kdeconnect/<device-id>/kdeconnect_runcommand/config \
/home/<username>/.config/kdeconnect/<device2-id>/kdeconnect_runcommand/config
```

> Sembolik link oluştururken relative path**(~/relative/path)** yerine absolute path**(/home/\<username>/absolute/path)** kullanılmalı. 
{: .prompt-warning }

Artık diğer cihazlarımızdan komut çalıştırabiliriz. Örnek olarak yukarıdaki `/home/melih/scripts/system_temp.sh` komutu çalıştığında CPU paketinin sıcaklığını telefonuma bildirim olarak gönderiyor.
```bash
kdeconnect-cli -d <device-id> --ping-msg "System Temperature is: $(vcgencmd measure_temp | cut -d'=' -f2)"
```

## Uyarılar

> Çoğu kurulumda **NOPASSWD** seçeneği varsayılan olarak aktif olduğundan `sudo` benzeri yetki yükseltme programları parola istemeyecektir. Bu durumun zaafiyet oluşturmaması için parolanın aktif edilmesi önerilir. [^ref2] 
{: .prompt-danger }

## Referanslar
[^ref1]: <https://userbase.kde.org/KDEConnect#Troubleshooting>
[^ref2]: <https://www.raspberrypi.com/documentation/computers/configuration.html#secure-your-raspberry-pi>
