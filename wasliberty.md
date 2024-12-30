# WebSphere Application Server Liberty Kapsamlı Kılavuzu

## İçindekiler
1. [Giriş ve Genel Bakış](#giriş-ve-genel-bakış)
2. [WAS Liberty Kurulumu](#was-liberty-kurulumu)
3. [IDE Entegrasyonları](#ide-entegrasyonları)
4. [Temel Konfigürasyon](#temel-konfigürasyon)
5. [Deployment İşlemleri](#deployment-işlemleri)
6. [Spring Boot Uygulamaları](#spring-boot-uygulamaları)
7. [Monitoring ve Health Check](#monitoring-ve-health-check)
8. [Troubleshooting](#troubleshooting)

## Giriş ve Genel Bakış

WebSphere Application Server Liberty (WAS Liberty), IBM tarafından geliştirilen hafif, esnek ve hızlı bir Java EE uygulama sunucusudur. Geleneksel WAS'a göre daha hafif ve modern bir alternatif sunar.

### Temel Özellikler
- Hızlı başlatma süresi
- Düşük bellek kullanımı
- Modüler yapı
- Docker desteği
- Spring Boot uyumluluğu
- Mikroservis mimarisi desteği

## WAS Liberty Kurulumu

### Önkoşullar
- JDK 8 veya üzeri
- Minimum 2GB RAM
- 1GB boş disk alanı

### Standalone Kurulum Adımları

1. IBM websitesinden WAS Liberty'nin son sürümünü indirin:
```bash
wget https://public.dhe.ibm.com/ibmdl/export/pub/software/websphere/wasdev/downloads/wlp/23.0.0.12/wlp-kernel-23.0.0.12.zip
```

2. İndirilen ZIP dosyasını çıkartın:
```bash
unzip wlp-kernel-23.0.0.12.zip -d /opt/ibm/
```

3. Çevre değişkenlerini ayarlayın:
```bash
export LIBERTY_HOME=/opt/ibm/wlp
export PATH=$LIBERTY_HOME/bin:$PATH
```

4. Kurulumu doğrulayın:
```bash
server version
```

### Docker ile Kurulum

```dockerfile
FROM ibmcom/websphere-liberty:latest

COPY server.xml /config/
COPY application.war /config/dropins/

EXPOSE 9080 9443
```

## IDE Entegrasyonları

### Eclipse ile Entegrasyon

1. IBM WebSphere Developer Tools (WDT)Pluginini Kurun:
   - Help > Eclipse Marketplace
   - "WebSphere" araması yapın
   - "IBM WebSphere Application Server Liberty Developer Tools" kurulumunu yapın

2. Server View'da Yeni Server Ekleme:
   - Window > Show View > Servers
   - New > Server
   - IBM > WebSphere Application Server Liberty seçin
   - Liberty runtime yolunu gösterin

3. Proje Oluşturma:
   - File > New > Dynamic Web Project
   - Target Runtime olarak Liberty'yi seçin
   - Finish'e tıklayın

### IntelliJ IDEA ile Entegrasyon

1. Ultimate Edition'da Liberty Support Eklentisini Kurun:
   - File > Settings > Plugins
   - Marketplace'de "Liberty" arayın
   - "WebSphere Liberty" eklentisini kurun

2. Application Server Konfigürasyonu:
   - Run > Edit Configurations
   - + > WebSphere Liberty Server
   - Liberty home dizinini gösterin
   - Server.xml yolunu belirtin

3. Run/Debug Konfigürasyonu:
```xml
<server description="Liberty Server">
    <featureManager>
        <feature>webProfile-8.0</feature>
        <feature>localConnector-1.0</feature>
    </featureManager>
    <httpEndpoint host="localhost" 
                  httpPort="9080"
                  httpsPort="9443" 
                  id="defaultHttpEndpoint"/>
</server>
```

## Temel Konfigürasyon

### server.xml Yapılandırması

```xml
<?xml version="1.0" encoding="UTF-8"?>
<server description="Liberty Server">
    
    <!-- Enable features -->
    <featureManager>
        <feature>servlet-4.0</feature>
        <feature>jsp-2.3</feature>
        <feature>jaxrs-2.1</feature>
        <feature>cdi-2.0</feature>
        <feature>jpa-2.2</feature>
    </featureManager>

    <!-- HTTP endpoint -->
    <httpEndpoint id="defaultHttpEndpoint"
                  host="*"
                  httpPort="9080"
                  httpsPort="9443" />

    <!-- Application configuration -->
    <application location="myapp.war" 
                 name="MyApplication" 
                 context-root="/myapp" />
                 
    <!-- JDBC datasource -->
    <dataSource jndiName="jdbc/myDB">
        <properties serverName="localhost"
                   portNumber="5432"
                   databaseName="mydb"
                   user="admin"
                   password="password" />
    </dataSource>
</server>
```

### Özel Konfigürasyon Dosyaları

1. bootstrap.properties:
```properties
# Server özellikleri
com.ibm.ws.logging.trace.specification=*=info
default.http.port=9080
default.https.port=9443
```

2. jvm.options:
```
-Xms256m
-Xmx512m
-XX:+UseG1GC
-Dcom.ibm.ws.logging.message.file=messages.log
```

## Deployment İşlemleri

### WAR Dosyası Deployment

1. Dropins Klasörü ile Deploy:
```bash
cp myapp.war ${LIBERTY_HOME}/usr/servers/defaultServer/dropins/
```

2. Command Line ile Deploy:
```bash
server package defaultServer --include=usr
```

3. Maven ile Deploy:
```xml
<plugin>
    <groupId>io.openliberty.tools</groupId>
    <artifactId>liberty-maven-plugin</artifactId>
    <version>3.7.1</version>
    <configuration>
        <serverName>defaultServer</serverName>
        <include>usr</include>
        <bootstrapProperties>
            <default.http.port>9080</default.http.port>
            <default.https.port>9443</default.https.port>
        </bootstrapProperties>
    </configuration>
</plugin>
```

### JAR Deployment

Spring Boot uygulamaları için:

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <executable>true</executable>
    </configuration>
</plugin>
```

## Spring Boot Uygulamaları

### Spring Boot ile Liberty Entegrasyonu

1. pom.xml Konfigürasyonu:
```xml
<dependency>
    <groupId>io.openliberty.features</groupId>
    <artifactId>springBoot-2.0</artifactId>
    <version>23.0.0.12</version>
    <type>esa</type>
</dependency>
```

2. Application.properties:
```properties
server.port=9080
spring.application.name=myapp
management.endpoints.web.exposure.include=*
```

3. Liberty Features:
```xml
<featureManager>
    <feature>springBoot-2.0</feature>
    <feature>servlet-4.0</feature>
</featureManager>
```

### Spring Boot Actuator Entegrasyonu

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

## Monitoring ve Health Check

### Health Check Endpoints

1. Liberty Admin Center Konfigürasyonu:
```xml
<featureManager>
    <feature>adminCenter-1.0</feature>
</featureManager>

<basicRegistry id="basic">
    <user name="admin" password="admin123"/>
</basicRegistry>

<administrator-role>
    <user>admin</user>
</administrator-role>
```

2. Custom Health Check:
```java
@Health
@ApplicationScoped
public class ServiceHealth implements HealthCheck {
    @Override
    public HealthCheckResponse call() {
        HealthCheckResponseBuilder builder = HealthCheckResponse.named("service");
        
        // Sağlık kontrolü mantığı
        if (checkServiceHealth()) {
            builder.up();
        } else {
            builder.down()
                   .withData("error", "Service unreachable");
        }
        
        return builder.build();
    }
}
```

### Metrics Monitoring

1. Prometheus Integration:
```xml
<featureManager>
    <feature>mpMetrics-2.3</feature>
</featureManager>

<mpMetrics authentication="false"/>
```

2. Custom Metrics:
```java
@Counted(name = "endpoint_hits", 
         absolute = true,
         description = "Endpoint hit count")
@GET
@Path("/data")
public Response getData() {
    // İş mantığı
    return Response.ok().build();
}
```

## Troubleshooting

### Yaygın Hatalar ve Çözümleri

1. JVM Heap Sorunları:
```bash
# Heap dump alma
server dump defaultServer --include=heap

# Memory analizi
jmap -heap <pid>
```

2. Log Analizi:
```bash
# Trace açma
-Dcom.ibm.ws.logging.trace.specification=*=finest

# Log dosyası inceleme
tail -f ${LIBERTY_HOME}/usr/servers/defaultServer/logs/trace.log
```

3. Performance Tuning:
```xml
<!-- Thread pool ayarları -->
<executor name="LargeThreadPool"
          id="default"
          coreThreads="20"
          maxThreads="100"
          keepAlive="60s"/>

<!-- Connection pool ayarları -->
<connectionManager maxPoolSize="100"
                  minPoolSize="10"
                  agedTimeout="30m"/>
```

### Debug Mode

1. Eclipse'te Debug:
   - Server'ı debug modunda başlatın
   - Break pointler ekleyin
   - Debug perspektifini kullanın

2. Remote Debug:
```bash
server debug defaultServer
```

3. JVM Debug Parametreleri:
```
-Xdebug
-Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=7777
```

Bu dokümantasyon, WAS Liberty'nin temel kurulum ve kullanımından ileri seviye yapılandırma ve troubleshooting konularına kadar geniş bir yelpazede bilgi sağlamaktadır. Her bölüm pratik örnekler ve kod parçacıkları ile desteklenmiştir.