# Installation Öncesi Genel Bilgilendirme
###### Sistem Gereksinimleri
###### Donanım Gereksinimleri:
        - CPU: Minimum 4 çekirdek (Önerilen: 8 çekirdek)
        - RAM: Minimum 8 GB (Önerilen: 16 GB)
        - Disk Alanı:
                - / (root): 50 GB
                - /opt: 100 GB
                - /tmp: 10 GB
        - Swap Alanı: RAM’in 2 katı
###### İşletim Sistemi Gereksinimleri:
        - RHEL 7 veya 8, SUSE Linux Enterprise Server 12 veya 15, Windows Server 2016 veya 2019
        - Güncel kernel ve güvenlik yamaları uygulanmış olmalıdır.
###### Ağ ve Güvenlik Gereksinimleri:
        - DNS çözümleme yapılandırılmalıdır.
        - Gerekli portlar açık olmalıdır (9060, 9043, 8879, 2809).
        - Güvenlik Duvarı Ayarları: DMGR ve Node iletişimi için gerekli izinler tanımlanmalıdır.
        - SSL/TLS yapılandırması için sertifikalar hazır edilmelidir.
###### Gerekli Dosyalar
        1. IBM Installation Manager (IM): WAS yüklemeleri için gerekli araç.
        2. WAS ND Repository: Kurulum ve güncelleme paketlerini içerir.
        3. Java SDK: WAS'ın çalışması için gerekli ortam.
        4. Fix Pack: Hata düzeltmeleri ve güvenlik yamaları içerir.
  ###### Dizin Yapısı Hazırlığı
  ```bash
  mkdir -p /opt/setup/{installationManager,fixpack,wasrepo,sdk}
  
 - installationManager ->>>>>> IM kurulum dosyaları
 - fixpack ->>>>>>>>>>>>>>>>>> Güncelleme paketleri
 - wasrepo ->>>>>>>>>>>>>>>>>> Ana repository dosyaları
 - sdk ->>>>>>>>>>>>>>>>>>>>>> Java SDK dosyaları
  # Kullanıcı ve Yetkilendirme
# wasadmin grubunu oluşturma

groupadd wasadmin

# wasadmin kullanıcısını oluşturma

useradd wasadmin -g wasadmin
passwd wasadmin

# Yetki doğrulama

id wasadmin
ls -ld /opt/IBM/InstallationManager
  ```
  


---
  ###### DMGR (Deployment Manager)
   * Tüm WebSpehere ortamını yöneten merkezi yönetim sunucusu
   * Konfigürasyon değişikliklerini yönetir.
   * Admin Console'u barındırır
   * 9043(HTTPS) ve 9060(HTTP) portlarını kullanır
   * 8879 portu federation için kullanılır.
  ###### Custom Profile (Node)
   * Uygulamaların çalıştığı sunucu profilidir.
   * Her node'da bir NodeAgent bulunur.
   * NodeAgent, DMGR ile iletişimi sağlar.
   * 9080 ve 9443 portlarını kullanır.
  ###### Cluster
   * Aynı uygulamayı çalıştıran server grubudur.
   * Yük dengeleme ve yüksek erişebilirlik sağlar.
   * En az iki server içermesi önerilir.
  

  ### Kurulum Öncesi Hazırlık

  ```bash
  # Sistem gereksinim kontrolü
  df -h     # Disk alanı kontrolü ( en az 50GB gerekli)
  free -h   # RAM kontrolü (en az 8GB gerekli)
  nproc     # CPU core sayısı (en az 4 core önerilen)

  # Gerekli dizinleri oluşturalım
  mkdir -p /opt/IBM/InstallationManager  # IM için
  mkdir -p /opt/IBM/WebSphere            # WAS için
  mkdir -p /opt/IBM/IMShared             # Shared resources için
  mkdir -p /opt/setup/{installationManager,wasrepo,fixpack,sdk}  # kurulum dosyaları için
  
  # wasadmin kullanıcısı (zorunlu değil ama best practice)
  useradd -m -s /bin/bash wasadmin
  passwd wasadmin
  # Sistem limitlerini ayarla
  cat << 'EOF' >> /etc/security/limits.conf
  wasadmin soft nofile 8192
  wasadmin hard nofile 8192
  wasadmin soft nproc 8192
  wasadmin hard nproc 8192
  EOF
  
  ```

---
## Installation Manager Kurulumu
Burada komut arayüzü ile kurulum yapılacak. Komutu çalıştırdıktan sonra kurulum aşamalarını takip edip kurulumu tamamlayabilirsiniz.
```bash
# IM kurulum dosyasına izin ver
chmod +x /opt/setup/installationManager/installc
# IM kurulumunu başlat
/opt/setup/installationManager/installc -acceptLicense \
  -installationDirectory /opt/IBM/InstallationManager \
  -dataLocation /opt/IBM/IMShared \
  -log /tmp/IM_install.log
# Yetkilendirmeleri düzenle
chown -R wasadmin:wasadmin /opt/IBM/InstallationManager
chmod -R 755 /opt/IBM/InstallationManager

 # /opt/IBM : IBM ürünleri için standart dizindir
 # /opt/IBM/IMShared: Birden fazla IM olduğunda paylaşalılan kaynaklar için
 ```
## WAS ND Kurulumu
 - Repository kontrolü yapıyoruz
 - Kurulacak paketleri listeliyoruz
  ```bash
  # Mevcut paketleri listele
  /opt/IBM/InstallationManager/eclipse/tools/imcl listAvailablePackages \
    -repositories /opt/setup/wasrepo/
   ```
- WAS ND Kurulum Komutu:
Kurulum komutu aşağıdaki gibidir:
  ```bash
  # Base WAS ND ve SDK kurulumu
  /opt/IBM/InstallationManager/eclipse/tools/imcl install \
    com.ibm.websphere.ND.v90_9.0.5001.20190828_0616 \
    com.ibm.java.jdk.v8_8.0.8035.20241125_0150 \
    -repositories /opt/setup/wasrepo/,/opt/setup/sdk/ \
    -installationDirectory /opt/IBM/WebSphere/AppServer \
    -sharedResourcesDirectory /opt/IBM/IMShared \
    -acceptLicense \
    -showProgress \
    -log /tmp/install.log
  # Fix pack kurulumu
  /opt/IBM/InstallationManager/eclipse/tools/imcl install \
    com.ibm.websphere.ND.v90_9.0.5022.20241118_0055 \
    -repositories /opt/setup/fixpack/ \
    -installationDirectory /opt/IBM/WebSphere/AppServer \
    -acceptLicense \
    -showProgress \
    -log /tmp/update.log

   ```


## Profil Oluşturma
#### DMGR Profili

###### Önemli Parametreler:
- **-profileName** : Profil adı  
- **-profilePath** : Profilin dizin yolu  
- **-templatePath** : Şablon dizini  
- **-cellName** : Hücre adı, tüm WAS ortamını temsil eder  
- **-nodeName** : DMGR node'unun benzersiz adı  
- **-hostName** : Sunucu hostname  
- **-enableAdminSecurity** : Güvenliği etkinleştirir  
- **-adminUserName** : Yönetici kullanıcı adı  
- **-adminPassword** : Yönetici şifresi  

`
/opt/IBM/WebSphere/AppServer/bin/managerprofiles.sh -create -profileName DMGR01 -profilePath /opt/IBM/WebSphere/AppServer/profiles/DMGR01 -templatePath /opt/IBM/WebSphere/AppServer/profileTemplates/management -cellName Cell01 -nodeName CellManager01 -hostName dmgr.example.com -enableAdminSecurity true -adminUserName wasadmin -adminPassword waspass123 `

#### Custom Profil (Farklı Sunucuda)
```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh -create -profileName Custom01 -profilePath /opt/IBM/WebSphere/AppServer/profiles/Custom01 -templatePath /opt/IBM/WebSphere/AppServer/profileTemplates/managed -nodeName Node01 -hostName appserver1.example.com -federateLater true 
```
- burada profilename, profilepath ,hostname gibi tanımları siz belirleyebilirsiniz.

###### Federate Etme
> Önemli Notlar:
        • Federation öncesi DMGR’ın çalıştığından emin olun:
/opt/IBM/WebSphere/AppServer/profiles/DMGR01/bin/startManager.sh
        • Federation sonrası node agent başlatılmalıdır:
/opt/IBM/WebSphere/AppServer/profiles/Custom01/bin/startNode.sh
        • Federation Hata Analizi:
                ○ Log Kontrolü:
tail -f /opt/IBM/WebSphere/AppServer/profiles/Custom01/logs/addNode.log
        • Port Çakışması:
netstat -tuln | grep 8879
        • Yetki Hataları:
id wasadmin
        • Zaman Aşımı Sorunları:
ping "dmgr-host"

```bash
/opt/IBM/WebSphere/AppServer/profiles/Custom01/bin/addNode.sh dmgr.example.com 8879 -username wasadmin -password waspass123 
```
- Burada wasusername ve password siz kendiniz girmeniz gerekiyor.
- hostname doğru yazmalısınız.
- DİKKKAAATT!!!! hostname tanımı yapsanız bile /etc/hosts kısma iki sunucuda hostname'leri eklemeniz gerekiyor ki gerekli iletişim sağlansın. Sunucularında hostnamelerini öğrenmek için $hostname veya $hostname -f komutlarını kullanabilirsiniz.

#### DMGR Sunucusunda Custom Profil Oluşturma
```bash
/opt/IBM/WebSphere/AppServer/bin/managerprofiles.sh -create -profileName Custom02 -profilePath /opt/IBM/WebSphere/AppServer/profiles/Custom02 -templatePath /opt/IBM/WebSphere/AppServer/profileTemplates/managed -nodeName Node02 -hostName dmgr.example.com -federateLater true 
```
- Burada da nodename, profilename siz belirleyebilirisniz.
- Hostname kısmına $hostname komutundan çıkan hostname yazmanı gerekiyor.

###### Federate Etme
- Aynı sunucuda oldukları için hostname kısmına localhost yazabliriz.
> Önemli Notlar:
        • Federation öncesi DMGR’ın çalıştığından emin olun:
/opt/IBM/WebSphere/AppServer/profiles/DMGR01/bin/startManager.sh
        • Federation sonrası node agent başlatılmalıdır:
/opt/IBM/WebSphere/AppServer/profiles/Custom01/bin/startNode.sh
        • Federation Hata Analizi:
                ○ Log Kontrolü:
tail -f /opt/IBM/WebSphere/AppServer/profiles/Custom01/logs/addNode.log
        • Port Çakışması:
netstat -tuln | grep 8879
        • Yetki Hataları:
id wasadmin
        • Zaman Aşımı Sorunları:
ping "dmgr-host"
```bash
/opt/IBM/WebSphere/AppServer/profiles/Custom02/bin/addNode.sh localhost 8879 -username wasadmin -password waspass123
```

### Cluster Yapılandırılması
* Yük Dengeleme:
  - Gelen istekleri sunucular arasında dağıtır
  - Performans artışı sağlar
  - Kaynak kullanımını optimize eder.
* Yüksek Erişibilirlik
  - Bir sunucu çökerse diğeri devam eder.
  - Kesintisiz hizmet sağlar
  - Bakım için sunucular sırayla kapatılabilir.
  
> Cluster oluşturma kısmını browser (GUI) ile daha rahat yapabilirsiniz. 
> wsadmin.sh -> WebSphere için yönetim aracı (administration scripting tool)

```bash
/opt/IBM/WebSphere/AppServer/profiles/DMGR01/bin/wsadmin.sh -lang jython -c
"""
# Cluster oluştur
AdminTask.createCluster('[-clusterName AppCluster01 -preferLocal true]')
# Node01 için cluster member
AdminTask.createClusterMember('[-clusterName AppCluster01 -memberNode Node01 -memberName AppServer01]')
# Node02 için cluster member
AdminTask.createClusterMember('[-clusterName AppCluster01 -memberNode Node02 -memberName Appserver02]')
#Değişikleri kaydet
AdminConfig.save()
"""
#cd /opt/IBM/WebSphere/AppServer/profiles/DMGR01/bin/ ->> bu dizinde çalışmanız gerekir.
```
### Deployment Yöntemi
Bu kısmı GUI ile halletmek daha kolay ve güvenli ancak bilgi amaçlı gerekli kodları paylaşmak istiyorum.
```bash
cd /opt/IBM/WebSphere/AppServer/profiles/DMGR01/bin/
/opt/IBM/WebSphere/AppServer/profiles/DMGR01/bin/wsadmin.sh -lang jython -c """
# Uygulama kurulumu
AdminApp.install('/path/to/myapp.ear',
  '[ -appname MyApp 
     -contextroot /myapp 
     -cluster AppCluster01 
     -MapModulesToServers [[".*",".*","WebSphere:cluster=AppCluster01"]] 
     -MapWebModToVH [[".*",".*","default_host"]] ]')
AdminConfig.save()
"""
# Parametrelerin anlamı:
# -appname: Uygulama adı
# -contextroot: URL path
# -cluster: Hangi cluster'a deploy edileceği
# -MapModulesToServers: Modül-server eşleştirmesi
# -MapWebModToVH: Virtual host eşleştirmesi
```

---
## JVM ve Performance Ayarları
###### JVM Ayarı
- Memory Yönetimi:
  - Heap size uygulamanın ihtiyacına göre ayarlanmalı
  - Çok küçük = OutOfMemoryError
  - Çok Büyük = Gereksiz kaynak kullanımı
- Garbage Collection:
  - Doğru GC algoritması seçilmeli
  - Pause time vs Throughput dengesi
  - Monitoring ile optimize edilmeli
```bash
# JVM ayarları
/opt/IBM/WebSphere/AppServer/profiles/DMGR01/bin/wsadmin.sh -lang jython -c """
server = AdminConfig.getid('/Node:Node01/Server:AppServer01/')
jvm = AdminConfig.list('JavaVirtualMachine', server)
# Memory ayarları
AdminConfig.modify(jvm, [
    ['initialHeapSize', 2048],     # Başlangıç heap boyutu (MB)
    ['maximumHeapSize', 4096],     # Maksimum heap boyutu (MB)
    ['genericJvmArguments', 
     '-XX:+UseG1GC                 # G1 Garbage Collector
      -XX:MaxGCPauseMillis=200     # Maksimum GC duraklaması
      -XX:ParallelGCThreads=8      # Parallel GC thread sayısı
      -XX:+HeapDumpOnOutOfMemoryError # OOM durumunda heap dump
      -XX:HeapDumpPath=/logs/heapdumps/'] # Heap dump dizini
])
AdminConfig.save()
"""

# PMI (Performance Monitoring Infrastructure)
/opt/IBM/WebSphere/AppServer/profiles/DMGR01/bin/wsadmin.sh -lang jython -c """
# PMI service'i aktifleştir
pmiService = AdminConfig.list('PMIService')
AdminConfig.modify(pmiService, [
    ['enable', 'true'],
    ['statisticSet', 'all']        # Tüm metrikleri topla
])
# Önemli bileşenleri monitör et
components = ['webAppModule',       # Web uygulamaları
             'connectionPool',      # Database bağlantıları
             'jvmRuntimeModule']    # JVM metrikleri
for comp in components:
    pmiModule = AdminConfig.list('PMIModule')
    AdminConfig.modify(pmiModule, [
        ['enable', 'true'],
        ['contextInfo', 'true']
    ])
AdminConfig.save()
"""

```

#### Not 
- Deployment Hakkında
  - Her zaman cluster'a deploy edin
  - PARENT_LAST classloader mode kullanabilirsiniz
  - Deployment öncesi yedek almanız gerekir
  - Uygulama loglarını ayrı dizine yönlendirin
  - Health check scriptleri hazırlayın
  - Monitoring ayarlarını yapılandırın
  - Resource gereksinimleri belirleyin
- Cluster Hakkında
  - En az iki node kullanın
  - Session replikasyonunu aktif edin
  - Load balancing stratejisi önemli
  - Failover gibi senoryolar belirleyebilirsiniz
  - Memory ve thread pool ayarlarını kontrol edin
  - Regular health check yapabilirsiniz
  - Performance metriklerini takip edin









  

