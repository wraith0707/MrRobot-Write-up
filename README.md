# Mr. Robot CTF - Write-up

* **Platform:** TryHackMe / HackTheBox
* **Zorluk Seviyesi:** Medium
* **Makale Amacı:** Bu makinede Nmap ve Gobuster ile servis/dizin keşfi, robots.txt üzerinden özel sözlük dosyası elde etme, Hydra ile WordPress panel brute-force saldırısı gerçekleştirme, tema üzerinden Reverse Shell yükleme ve SUID yetki yükseltme adımları gerçekleştirilmiştir.

---

## 1. Keşif ve Bilgi Toplama (Reconnaissance & Enumeration)

Hedef IP adresine yönelik yapılan Nmap taraması ile açık portlar ve servisler tespit edilmiştir.

* **Komut:** `nmap -sC -sV -T4 10.113.137.81`
* **Açık Portlar:**
  * **Port 22 (tcp):** OpenSSH 8.2p1 (Ubuntu)
  * **Port 80 (tcp):** Apache httpd
  * **Port 443 (tcp):** SSL/HTTP Apache httpd

<img width="1727" height="853" alt="1" src="https://github.com/user-attachments/assets/d9094d7e-814e-4ef3-9416-7124c2ea0956" />


* **Dizin Taraması:** Gobuster aracı ile web dizinleri taranmış, `/blog`, `/wp-login.php`, `/admin` ve `/intro` gibi kritik dizinler ve dosyalar tespit edilmiştir.

<img width="1901" height="602" alt="2" src="https://github.com/user-attachments/assets/cde1f293-43ce-4d5b-98d9-9716cb1c6af4" />
<img width="1721" height="615" alt="3" src="https://github.com/user-attachments/assets/24c61d1e-012c-4353-8197-ce0a9595a237" />

* **Robots.txt Analizi:** Hedef sitenin `robots.txt` dosyası incelendiğinde `fsocity.dic` adında özel bir kelime listesi ve ilk bayrak (`key-1-of-3.txt`) dosyası olduğu görülmüştür.

<img width="806" height="196" alt="4" src="https://github.com/user-attachments/assets/4111758f-c4c2-422b-a4a3-428727fb9b4a" />

---

## 2. Sömürü (Exploitation & Brute-Force)

* **Wordlist İnceleme ve Düzenleme:** Siteden indirilen `fsocity.dic` dosyası içerisindeki tekrar eden kelimeleri temizlemek ve düzenlemek için `sort -u` komutu kullanılmıştır.

<img width="691" height="817" alt="5" src="https://github.com/user-attachments/assets/d5456d1e-7174-406d-8355-507a6857398e" />
<img width="1850" height="436" alt="6" src="https://github.com/user-attachments/assets/6f256b7a-33d3-4eac-b458-180eb90974b1" />
<img width="836" height="105" alt="7" src="https://github.com/user-attachments/assets/ef66dc8f-252e-4fb0-882d-bb65273c9d9d" />

* **Hydra ile Brute-Force:** Elde edilen düzenlenmiş kelime listesi (`fsocity.txt`) ve `elliot` kullanıcı adı kullanılarak WordPress giriş paneline Hydra ile brute-force saldırısı düzenlenmiş ve şifre başarıyla kırılmıştır.

<img width="1901" height="587" alt="8" src="https://github.com/user-attachments/assets/b8bf456c-f0a4-4f7a-9fa1-8e0daf3fd163" />

---

## 3. İlk Erişim (Foothold - Reverse Shell)

* Kırılan şifre ile WordPress admin paneline (`/wp-admin/`) giriş yapılmıştır.

<img width="1540" height="851" alt="9" src="https://github.com/user-attachments/assets/7fe030ed-cd15-4d66-adc9-cc2bfa08dac1" />

* Panel içerisinden tema dosyaları kısmına (`Appearance > Editor`) geçilmiş, kullanılacak olan `php-reverse-shell.php` kodu ilgili tema dosyasına (örneğin 404 şablonuna) yapıştırılarak güncellenmiştir.

<img width="1917" height="962" alt="10" src="https://github.com/user-attachments/assets/92e5e5cd-b557-4d75-bfce-da82aae1540c" />
<img width="114" height="137" alt="11" src="https://github.com/user-attachments/assets/4af8ea69-b6f8-46a0-a052-0a5fa41a8a41" />
<img width="1789" height="871" alt="12" src="https://github.com/user-attachments/assets/8cb17be9-39eb-45f3-a3ab-9df8d703b8e5" />

* Saldırgan makinesinde Netcat ile dinleme (listener) başlatılmıştır:
  * **Komut:** `nc -lvnp 1234`

<img width="837" height="155" alt="13" src="https://github.com/user-attachments/assets/1871bd54-93e8-428c-ae07-bc9ef56ad66a" />

* Tarayıcı üzerinden yüklenen PHP dosyasının bulunduğu URL (`/wp-content/themes/.../404.php`) tetiklenmiştir.

<img width="546" height="46" alt="14" src="https://github.com/user-attachments/assets/57a4f3db-3c05-4a03-9a26-0805d7c4981c" />

* Netcat dinleyicisine bağlantı gelmiş ve `daemon` yetkileriyle sisteme ilk erişim (shell) sağlanmıştır.

<img width="1906" height="351" alt="15" src="https://github.com/user-attachments/assets/e6676593-5616-4515-b6c2-a2f9b7d42327" />

---

## 4. Yetki Yükseltme ve Bayraklar (Privilege Escalation)

* `/home/robot` dizini altında `key-2-of-3.txt` ve `password.raw-md5` dosyaları tespit edilmiş, MD5 hash değeri okunmuştur.

<img width="1228" height="585" alt="16" src="https://github.com/user-attachments/assets/fa60bfae-0967-41d6-adef-b9d909c260d6" />

* Elde edilen MD5 hash değeri bir çevrimiçi hash kırıcı araç yardımıyla çözülmüş ve şifre elde edilmiştir.

<img width="1367" height="555" alt="17" src="https://github.com/user-attachments/assets/4b6b2dcb-3927-4c76-8173-a0366a75da7a" />

* Sistemde SUID yetkisine sahip dosyaları bulmak için `find / -perm -u=s -type f 2>/dev/null` komutu çalıştırılmış ve `/usr/local/bin/nmap` üzerinde SUID yetkisi olduğu görülmüştür.

<img width="1250" height="637" alt="18" src="https://github.com/user-attachments/assets/3353b9f6-55b0-4b13-9237-5519de19477e" />

* Eski sürüm Nmap'in interaktif modu (`nmap --interactive`) kullanılarak komut satırına düşülmüş ve `!sh` komutu ile doğrudan `root` yetkileri elde edilmiştir.

<img width="980" height="171" alt="19" src="https://github.com/user-attachments/assets/9db663a9-2260-428d-a842-784f441155c9" />

* `id` komutu ile yetkinin `root` olduğu doğrulanmış ve sistem tamamen ele geçirilmiştir.

<img width="922" height="109" alt="20" src="https://github.com/user-attachments/assets/d718a3d2-6bc0-4721-8433-10c6e3dbee00" />
