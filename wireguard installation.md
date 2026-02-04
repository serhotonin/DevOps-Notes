


# 🛡️ AWS WireGuard VPN Kurulum Rehberi (DevSecOps)

> **Bağlam**
>
> **Proje:** DevOps  
> **Rol:** DevOps  
> **Amaç:** AWS üzerindeki sunuculara güvenli erişim için WireGuard VPN kurulumu ve konfigürasyon analizi  
> **Araç:** `angristan/wireguard-install` scripti

---

## 1. Hazırlık ve Sunucu Kurulumu (AWS)

İlk adımda AWS üzerinden VPN sunucumuzu ayağa kaldırıyoruz.

- **Instance Tipi:** Free Tier (`t2.micro` veya `t3.micro` yeterli)
- **OS:** Ubuntu Server (stabil, dokümantasyonu güçlü)

### Network Güvenliği (ÖNEMLİ)

- Sunucuyu oluştururken **Public IP** seçeneğini **Disable etme**  
  (Aksi halde VPN’e bağlanamazsın)
- **Security Group** tarafında:
  - Gereksiz **tüm portları kapat**
  - Sadece:
    - **SSH:** `22/TCP`
    - **WireGuard:** Seçeceğin **UDP portu**

---

### 📝 Kendime Not – Network Çalış

Bu süreçte network temelleri kritik. Özellikle şunlara bak:

- **Subnet Hesaplama (CIDR):**
  - `/24`, `/16` nedir?
  - Kaç IP üretir?
- **Public vs Private IP:**
  - AWS’de trafik nasıl akar?
  - NAT nerede devreye girer?

---

## 2. Kurulum Adımları (`angristan` Scripti)

Scripti indirip çalıştır:

```bash
curl -O https://raw.githubusercontent.com/angristan/wireguard-install/master/wireguard-install.sh
chmod +x wireguard-install.sh
./wireguard-install.sh
````

Karşına çıkacak sorular ve doğru cevaplar 👇

---

### A. Public IP Adresi

* Script, VPN client’larının sunucuyu bulabilmesi için public IP ister.
* AWS’nin verdiği public IP’yi otomatik algılar.

**Aksiyon:**
Enter → geç (doğruysa)

---

### B. Interface Seçimi

* **Soru:** Public interface hangisi?
* **Cevap:**

  ```text
  eth0
  ```

AWS EC2’de varsayılan network interface’tir.

---

### C. WireGuard Interface Name

* **Soru:** Arayüz adı?
* **Cevap:**

  ```text
  wg0
  ```

---

### D. WireGuard Internal IPv4 (VPN Subneti)

VPN tüneli içinde kullanılacak private IP bloğu.

* Çakışma olmaması için rastgele bir subnet seç
* RFC1918 aralığında olmalı

**Örnek:**

```text
10.19.11.0
```

---

### E. IPv6

* Script otomatik ayarlar

**Aksiyon:**
Enter → geç

---

### F. Port Seçimi

* Script random bir UDP port üretir
  Örn:

  ```text
  51820
  52394
  ```

⚠️ **ÇOK ÖNEMLİ**

Bu portu **AWS Security Group**’ta **UDP olarak açmayı unutma**.

---

### G. DNS Sunucuları

Client VPN’e bağlandığında kullanılacak DNS’ler.

**Önerilen:**

```text
1.1.1.1
1.0.0.1
```

Cloudflare DNS → hızlı + güvenli.

---

## 3. Client Config (Split Tunneling Ayarı)

Burada kritik bir karar var.

Biz VPN’i:

* ❌ Tüm internet trafiğini yönlendirmek için
* ✅ Sadece **iç ağdaki sunuculara erişmek için** kullanacağız

### AllowedIPs Ayarı

#### ❌ Yapılmaması Gereken

```ini
AllowedIPs = 0.0.0.0/0
```

Bu ayar:

> “Her şeyi VPN’den geçir” demektir.

---

#### ✅ Yapılması Gereken

Sadece VPN subnetini veya hedef sunucuları yaz:

```ini
AllowedIPs = 10.19.11.0/24
```

Bu ne yapar?

* Sadece `10.19.11.x` adreslerine giden trafik VPN tüneline girer
* YouTube, Instagram, Google vs **lokal internetten çıkar**

---

## 4. Konfigürasyon Analizi – Under the Hood

Kurulum bitti ama asıl önemli kısım burası 👀

---

### Config Dosyaları

* **Konum:**

  ```text
  /etc/wireguard/
  ```
* **Server Config:**

  ```text
  wg0.conf
  ```

---

### Protokol: UDP

WireGuard **UDP** kullanır.

Neden?

* TCP gibi “paket ulaştı mı?” kontrolü yok
* Daha az overhead
* Daha düşük latency

> Mantık: **Fire and forget**

---

### Iptables & Routing (FORWARDING)

`wg0.conf` içindeki şu satırlara dikkat:

```ini
PostUp=...
PostDown=...
```

Bunlar **iptables** kurallarıdır.

---

#### INPUT vs FORWARD

* **INPUT**

  * Sunucunun *kendisine* gelen trafik
* **FORWARD**

  * Sunucu üzerinden *başka bir yere* giden trafik
    (Router davranışı)

---

#### Bizim Senaryoda Ne Değişti?

Normalde script şu kuralı ekler:

```bash
POSTROUTING -o eth0 -j MASQUERADE
```

Bu ne yapar?

* VPN client’larının **internete çıkmasını** sağlar (NAT)

📌 **Not:**
Notlarında *“POSTROUTING silinecek”* demişsin.

**Sebep:**

* Client’ların bu sunucu üzerinden internete çıkmasını istemiyorsan
* Bu **MASQUERADE** kuralını kaldırırsın

Sonuç:

* VPN sadece **iç ağ erişimi** sağlar
* Sunucu bir gateway değil, kontrollü erişim noktası olur

---

### Anahtarlar (Keys)

Mantık SSH ile aynı.

* **Private Key**

  * Gizli
  * Kimseyle paylaşılmaz
* **Public Key**

  * Karşı tarafa verilir

---

#### Kontrol Noktaları

* Server tarafı:

  ```bash
  cat /etc/wireguard/wg0.conf
  ```

  `[Peer]` altında client public key’leri görürsün

* Client dosyası:

  ```bash
  /root/wg0-client-serhotonin.conf
  ```

  İçerir:

  * Client private key
  * Server public key

---

## 5. Güvenlik Duvarı (UFW) Ayarları

Amaç:

* VPN ağını izole etmek
* Yanlış config sızarsa internetin sömürülmesini engellemek

---

### 1. VPN’den Genele Çıkışı Engelle

```bash
ufw deny from 10.19.11.0/24 to 0.0.0.0/0
```

---

### 2. VPN İçi İletişime İzin Ver

```bash
ufw allow from 10.19.11.0/24 to 10.19.11.0/24
```

**Mantık:**

1. Önce: “Hiçbir yere gidemezsin”
2. Sonra: “Ama kendi subnetin serbest”

⚠️ **Kural sırası önemlidir**

---

## 6. Servis Yönetimi (wg-quick vs systemd)

VPN’i nasıl yönetiyoruz?

---

### Manuel (Test / Debug)

```bash
wg-quick up wg0
wg-quick down wg0
```

* Sunucu reboot olursa VPN kapanır

---

### Otomatik (Production / DevSecOps)

```bash
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

* `enable` → Boot’ta otomatik başlar
* `start` → Şu an çalıştırır

---

### ✅ Kontrol İpucu

Client tarafında:

```bash
wg
```

Şunları görüyorsan her şey tamamdır:

* `transfer rx/tx`
* `latest handshake`




