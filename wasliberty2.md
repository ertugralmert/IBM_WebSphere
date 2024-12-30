# WebSphere Application Server Liberty - Detaylı Kılavuz

## İçindekiler
1. [Giriş ve Genel Bakış](#giriş-ve-genel-bakış)
2. [WAS Liberty Kurulumu](#was-liberty-kurulumu)
3. [IDE Entegrasyonları](#ide-entegrasyonları)
4. [Container Ortamları](#container-ortamları)
5. [WAS ND'den Liberty'ye Geçiş](#was-ndden-libertyye-geçiş)
6. [Deployment ve Konfigürasyon](#deployment-ve-konfigürasyon)
7. [Log Yönetimi ve Analizi](#log-yönetimi-ve-analizi)
8. [Performance Tuning](#performance-tuning)
9. [Security Yapılandırması](#security-yapılandırması)
10. [Troubleshooting](#troubleshooting)

## Container Ortamları

### Docker Detaylı Konfigürasyon

1. Dockerfile Örneği:
```dockerfile
# Base image seçimi
FROM ibmcom/websphere-liberty:kernel-java11-openj9-ubi

# Liberty özellikleri ekleme
RUN features.sh \
    --acceptLicense \
    webProfile-8.0 \
    microProfile-3.3 \
    javaee-8.0

# Server konfigürasyonu kopyalama
COPY --chown=1001:0 server.xml /config/
COPY --chown=1001:0 jvm.options /config/
COPY --chown=1001:0 bootstrap.properties /config/

# Uygulama deployment
COPY --chown=1001:0 myapp.war /config/dropins/

# Health check tanımlama
HEALTHCHECK --interval=5m --timeout=3s \
  CMD curl -f http://localhost:9080/health/ || exit 1

# Port açma
EXPOSE 9080 9443

# Environment variables
ENV LOG_DIR=/logs \
    WLP_OUTPUT_DIR=/opt/ibm/wlp/output

# Volume tanımlama
VOLUME ["/logs", "/opt/ibm/wlp/output"]
```

2. Multi-stage Build Örneği:
```dockerfile
# Build stage
FROM maven:3.8-openjdk-11 as builder
WORKDIR /build
COPY pom.xml .
COPY src src
RUN mvn clean package

# Runtime stage
FROM ibmcom/websphere-liberty:kernel-java11-openj9-ubi
COPY --from=builder /build/target/*.war /config/dropins/
COPY --chown=1001:0 server.xml /config/
```

### Kubernetes Deployment

1. Deployment YAML:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: liberty-app
  namespace: my-apps
spec:
  replicas: 3
  selector:
    matchLabels:
      app: liberty-app
  template:
    metadata:
      labels:
        app: liberty-app
    spec:
      containers:
      - name: liberty
        image: my-registry/liberty-app:1.0
        ports:
        - containerPort: 9080
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        readinessProbe:
          httpGet:
            path: /health
            port: 9080
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 9080
          initialDelaySeconds: 60
          periodSeconds: 10
        volumeMounts:
        - name: liberty-logs
          mountPath: /logs
      volumes:
      - name: liberty-logs
        persistentVolumeClaim:
          claimName: liberty-logs-pvc
```

2. Service YAML:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: liberty-service
spec:
  selector:
    app: liberty-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9080
  type: LoadBalancer
```

### OpenShift Deployment

1. Template YAML:
```yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: liberty-template
parameters:
- name: APP_NAME
  required: true
  value: liberty-app
objects:
- apiVersion: apps.openshift.io/v1
  kind: DeploymentConfig
  metadata:
    name: ${APP_NAME}
  spec:
    replicas: 2
    strategy:
      type: Rolling
    template:
      metadata:
        labels:
          app: ${APP_NAME}
      spec:
        containers:
        - name: ${APP_NAME}
          image: ${APP_NAME}:latest
          ports:
          - containerPort: 9080
```

2. Route Yapılandırması:
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: liberty-route
spec:
  to:
    kind: Service
    name: liberty-service
  tls:
    termination: edge
```

## WAS ND'den Liberty'ye Geçiş

### Migration Analizi

1. Migration Toolkit Kullanımı:
```bash
# Migration Toolkit kurulum
wget https://public.dhe.ibm.com/ibmdl/export/pub/software/websphere/wasdev/downloads/wamt/2.0/wamt-2.0.zip
unzip wamt-2.0.zip

# Analiz çalıştırma
./wamt analyze --sourceAppServer=/path/to/was/profile --targetAppServer=liberty
```

2. WebSphere Configuration Migration Tool:
```bash
# Tool çalıştırma
java -jar wmct.jar --sourceProfile=/path/to/was/profile --targetLiberty=/path/to/liberty
```

### Uygulama Dönüşümü

1. EJB Dönüşümleri:
```java
// Eski WAS EJB
@Stateless
@RolesAllowed("Admin")
public class UserServiceBean implements UserService {
    @Resource(name="jdbc/myDB")
    private DataSource dataSource;
}

// Liberty versiyonu
@ApplicationScoped
@Authenticated
public class UserService {
    @Inject
    @Named("myDB")
    private DataSource dataSource;
}
```

2. Web Services Dönüşümü:
```xml
<!-- server.xml Web Services Feature -->
<featureManager>
    <feature>jaxws-2.2</feature>
    <feature>jaxrs-2.1</feature>
</featureManager>
```

### JNDI Kaynaklarının Taşınması

1. Datasource Dönüşümü:
```xml
<!-- WAS ND JDBC -->
<jdbcProvider>
    <properties.oracle/>
</jdbcProvider>
<dataSource jndiName="jdbc/myDB">
    <jdbcDriver libraryRef="OracleLib"/>
    <properties.oracle URL="jdbc:oracle:thin:@//host:port/service"/>
</dataSource>

<!-- Liberty JDBC -->
<dataSource jndiName="jdbc/myDB">
    <jdbcDriver libraryRef="OracleLib"/>
    <properties.oracle databaseName="MYDB"
                      serverName="host"
                      portNumber="1521"/>
</dataSource>
```

## Log Yönetimi ve Analizi

### Log Yapılandırması

1. server.xml Log Konfigürasyonu:
```xml
<logging traceSpecification="*=info:com.myapp.*=fine"
         maxFileSize="20"
         maxFiles="10"
         consoleLogLevel="INFO"/>

<basicRegistry id="basic" realm="BasicRealm">
    <user name="admin" password="admin123"/>
</basicRegistry>
```

2. logback.xml Konfigürasyonu:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${server.output.dir}/logs/myapp.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${server.output.dir}/logs/myapp-%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

### Log Analizi Araçları

1. Log Parser Script:
```python
import re
import pandas as pd

def parse_liberty_logs(log_file):
    pattern = r'(\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z)\s+(\w+)\s+(\w+)\s+(.*)'
    logs = []
    
    with open(log_file, 'r') as f:
        for line in f:
            match = re.match(pattern, line)
            if match:
                timestamp, level, component, message = match.groups()
                logs.append({
                    'timestamp': timestamp,
                    'level': level,
                    'component': component,
                    'message': message
                })
    
    return pd.DataFrame(logs)
```

2. ELK Stack Integration:
```yaml
# Filebeat configuration
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /logs/messages.log
    - /logs/trace.log
  fields:
    type: websphere-liberty

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "liberty-logs-%{+yyyy.MM.dd}"
```

### Performance Monitoring

1. JMX Monitoring:
```xml
<featureManager>
    <feature>monitor-1.0</feature>
</featureManager>

<monitor filter="*"/>
```

2. Prometheus & Grafana Integration:
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'liberty'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['liberty-host:9080']
```

## Security Yapılandırması

### SSL/TLS Konfigürasyonu

1. Keystore Oluşturma:
```bash
# Keystore oluşturma
keytool -genkey -alias liberty -keyalg RSA -keystore liberty.keystore

# Sertifika export
keytool -export -alias liberty -file liberty.cer -keystore liberty.keystore
```

2. server.xml SSL Konfigürasyonu:
```xml
<keyStore id="defaultKeyStore" 
          location="liberty.keystore"
          type="JKS" 
          password="mypassword"/>

<ssl id="defaultSSLConfig" 
     keyStoreRef="defaultKeyStore" 
     trustStoreRef="defaultTrustStore"/>

<httpEndpoint httpsPort="9443" 
              httpPort="9080">
    <tcpOptions soReuseAddr="true"/>
    <sslOptions sslRef="defaultSSLConfig"/>
</httpEndpoint>
```

### LDAP Entegrasyonu

```xml
<ldapRegistry id="ldap" 
              realm="SampleLdapRealm" 
              host="ldap.example.com" 
              port="389" 
              baseDN="dc=example,dc=com"
              bindDN="cn=admin,dc=example,dc=com" 
              bindPassword="password">
    <group name="AdminGroup" filter="(&amp;(objectclass=groupofnames)(cn=admins))"/>
</ldapRegistry>
```

## Performance Tuning

### JVM Optimizasyonu

1. jvm.options:
```
# Heap size
-Xms1024m
-Xmx2048m

# GC options
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:ParallelGCThreads=8

# Large pages
-XX:+UseLargePages
-XX:LargePageSizeInBytes=2m

# JIT compiler
-XX:+TieredCompilation
-XX:ReservedCodeCacheSize=240m
```

2. Thread Pool Ayarları:
```xml
<executor name="DefaultExecutor"
          id="default"
          coreThreads="40"
          maxThreads="100"
          keepAlive="60s"
          stealPolicy="STRICT"
          rejectedWorkPolicy="CALLER_RUNS"/>
```

### Connection Pool Optimizasyonu

```xml
<connectionManager id="DefaultCM"
                  maxPoolSize="100"
                  minPoolSize="10"
                  agedTimeout="30m"
                  connectionTimeout="10s"
                  maxIdleTime="30m"
                  purgePolicy="ValidateAllConnections"
                  reapTime="3m"/>
```

## Troubleshooting

### Memory Leak Analizi

1. Heap Dump Analizi:
```bash
# Heap dump alma
server dump defaultServer --include=heap

# Eclipse Memory Analyzer ile analiz
jmap -dump:format=b,file=heap.bin <pid>
```

2. Thread Dump Analizi:
```bash
# Thread dump alma
kill -3 <pid>

# Thread dump analizi
jstack -l <pid> > thread_dump.txt
```

### Performance Sorunları

1. Response Time Analizi:
```xml
<!-- server.xml -->
<monitoring filter="*"
           traceSpecification="*=info:com.ibm.ws.webcontainer*=all"/>
```

2. Database Query Analizi:
```xml
<dataSource jndiName="jdbc/myDB" statementCacheSize="60">
    <connectionManager maxPoolSize="50" minPoolSize="10"/>
    <properties.db2.jcc databaseName="MYDB"
                        currentSchema="MYSCHEMA"
                        enableSeamlessFailover="1"
                        enableClientAffinitiesList="1"/>
</dataSource>
```

Bu genişletilmiş dokümantasyon, özellikle container ortamları, migration süreçleri ve detaylı troubleshooting konularında daha fazla bilgi içermektedir. Her bölüm pratik örnekler ve kod parçacıkları ile desteklenmiştir.


# WebSphere Application Server Liberty - Detaylı Kılavuz (Devam)

## CI/CD Pipeline Entegrasyonu

### Jenkins Pipeline

1. Jenkinsfile Örneği:
```groovy
pipeline {
    agent any
    
    environment {
        LIBERTY_HOME = '/opt/ibm/wlp'
        DOCKER_REGISTRY = 'my-registry.com'
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('Liberty Server Package') {
            steps {
                sh '''
                    ${LIBERTY_HOME}/bin/server package defaultServer \
                    --include=usr \
                    --archive=./target/liberty-server.zip
                '''
            }
        }
        
        stage('Docker Build & Push') {
            steps {
                script {
                    docker.build("${DOCKER_REGISTRY}/liberty-app:${BUILD_NUMBER}")
                    docker.withRegistry('https://${DOCKER_REGISTRY}', 'registry-credentials') {
                        docker.image("${DOCKER_REGISTRY}/liberty-app:${BUILD_NUMBER}").push()
                    }
                }
            }
        }
        
        stage('Deploy to K8s') {
            steps {
                sh '''
                    kubectl apply -f k8s/deployment.yaml
                    kubectl set image deployment/liberty-app \
                    liberty-container=${DOCKER_REGISTRY}/liberty-app:${BUILD_NUMBER}
                '''
            }
        }
    }
}
```

### GitLab CI/CD

```yaml
variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"
  LIBERTY_IMAGE: ${CI_REGISTRY_IMAGE}/liberty-app

stages:
  - build
  - test
  - package
  - deploy

build:
  stage: build
  script:
    - mvn clean package
  artifacts:
    paths:
      - target/*.war

test:
  stage: test
  script:
    - mvn verify

package-liberty:
  stage: package
  script:
    - docker build -t ${LIBERTY_IMAGE}:${CI_COMMIT_SHA} .
    - docker push ${LIBERTY_IMAGE}:${CI_COMMIT_SHA}

deploy-k8s:
  stage: deploy
  script:
    - kubectl set image deployment/liberty-app liberty=${LIBERTY_IMAGE}:${CI_COMMIT_SHA}
```

## Cloud Native Özellikler

### MicroProfile Konfigurasyon

1. ConfigSource Tanımlama:
```java
@ApplicationScoped
public class CustomConfigSource implements ConfigSource {
    private Map<String, String> properties = new HashMap<>();
    
    @PostConstruct
    public void init() {
        // Özel konfigürasyon yükleme
        properties.put("app.feature.enabled", "true");
    }
    
    @Override
    public Map<String, String> getProperties() {
        return properties;
    }
    
    @Override
    public String getValue(String key) {
        return properties.get(key);
    }
    
    @Override
    public String getName() {
        return "CustomConfig";
    }
}
```

2. Config Injection:
```java
@Inject
@ConfigProperty(name = "app.feature.enabled", defaultValue = "false")
private boolean featureEnabled;
```

### Service Discovery

1. Consul Entegrasyonu:
```xml
<server>
    <featureManager>
        <feature>mpConfig-2.0</feature>
        <feature>servicediscovery-1.0</feature>
    </featureManager>
    
    <consul host="consul-server" port="8500"/>
    
    <serviceDiscovery>
        <dynamicService name="${app.name}"
                       endpoint="/health"
                       ttl="30s"/>
    </serviceDiscovery>
</server>
```

2. Kubernetes Service Discovery:
```java
@ApplicationScoped
public class ServiceDiscovery {
    @Inject
    @ConfigProperty(name = "k8s.namespace")
    private String namespace;
    
    public String discoverService(String serviceName) {
        return String.format("%s.%s.svc.cluster.local", serviceName, namespace);
    }
}
```

## İleri Seviye Monitoring

### Prometheus JMX Exporter

1. jmx_exporter Konfigürasyonu:
```yaml
---
startDelaySeconds: 0
ssl: false
lowercaseOutputName: true
lowercaseOutputLabelNames: true

rules:
- pattern: ".*"
  name: liberty_$1_total
  
whitelistObjectNames:
  - WebSphere:*
  - java.lang:*
```

2. Liberty Server Konfigürasyonu:
```xml
<featureManager>
    <feature>mpMetrics-2.3</feature>
    <feature>monitor-1.0</feature>
</featureManager>

<mpMetrics authentication="false"/>

<jmxConnection>
    <consoleConnector host="*" port="9443"/>
</jmxConnection>
```

### ELK Stack ile Log Analizi

1. Logstash Pipeline:
```ruby
input {
  beats {
    port => 5044
  }
}

filter {
  if [type] == "liberty" {
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:message}" }
    }
    
    date {
      match => [ "timestamp", "ISO8601" ]
      target => "@timestamp"
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "liberty-logs-%{+YYYY.MM.dd}"
  }
}
```

2. Kibana Dashboard:
```json
{
  "title": "Liberty Monitoring Dashboard",
  "panels": [
    {
      "type": "visualization",
      "source": {
        "aggs": [
          {
            "enabled": true,
            "id": "1",
            "params": {
              "field": "level.keyword"
            },
            "schema": "segment",
            "type": "terms"
          }
        ],
        "params": {
          "addTooltip": true,
          "addLegend": true,
          "type": "pie"
        },
        "title": "Log Levels Distribution"
      }
    }
  ]
}
```

### APM (Application Performance Monitoring)

1. OpenTelemetry Integration:
```xml
<featureManager>
    <feature>mpOpenTracing-2.0</feature>
</featureManager>

<openTracing>
    <jaeger host="jaeger-collector"
            port="14268"
            samplerType="const"
            samplerParam="1"/>
</openTracing>
```

2. Trace Instrumentation:
```java
@Traced
@Path("/api")
public class MyResource {
    @Inject
    private Tracer tracer;
    
    @GET
    @Path("/data")
    public Response getData() {
        Span span = tracer.buildSpan("getData").start();
        try {
            // İş mantığı
            return Response.ok().build();
        } finally {
            span.finish();
        }
    }
}
```

## High Availability ve Clustering

### Session Replikasyonu

1. JCache Konfigürasyonu:
```xml
<featureManager>
    <feature>sessionCache-1.0</feature>
</featureManager>

<httpSessionCache libraryRef="hazelcast"/>
<library id="hazelcast">
    <fileset dir="${shared.resource.dir}/hazelcast"/>
</library>
```

2. Hazelcast Konfigürasyonu:
```xml
<hazelcast>
    <network>
        <join>
            <multicast enabled="false"/>
            <tcp-ip enabled="true">
                <member>liberty-node1</member>
                <member>liberty-node2</member>
            </tcp-ip>
        </join>
    </network>
    <map name="sessions">
        <backup-count>1</backup-count>
        <max-idle-seconds>3600</max-idle-seconds>
    </map>
</hazelcast>
```

### Load Balancing

1. Apache HTTP Server Konfigürasyonu:
```apache
<Proxy balancer://liberty_cluster>
    BalancerMember http://liberty1:9080 route=node1
    BalancerMember http://liberty2:9080 route=node2
    
    ProxySet stickysession=JSESSIONID
</Proxy>

ProxyPass /myapp balancer://liberty_cluster/myapp
ProxyPassReverse /myapp balancer://liberty_cluster/myapp
```

2. Kubernetes Ingress:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: liberty-ingress
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "JSESSIONID"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: liberty-service
            port:
              number: 80
```

## Liberty Tools ve Yardımcı Uygulamalar

### Liberty Developer Tools

1. Eclipse Plugin Özellikleri:
- Server oluşturma ve yönetme
- Incremental publish
- Debug modunda çalıştırma
- Hot deployment
- Resource monitoring

2. IntelliJ IDEA Integration:
- Liberty runtime yönetimi
- Server.xml editörü
- Deployment descriptor validasyonu
- Debug desteği

### Migration Toolkit

1. Application Scanner:
```bash
# Uygulama tarama
migration-toolkit scan --sourceAppServer=websphere \
                      --targetAppServer=liberty \
                      --sourceAppLocation=/path/to/ear \
                      --targetAppLocation=/path/to/output
```

2. Configuration Scanner:
```bash
# Konfigürasyon tarama
migration-toolkit scanConfig --sourceAppServer=websphere \
                           --targetAppServer=liberty \
                           --sourceProfile=/path/to/profile
```

### Performance Tuning Tools

1. Liberty Server Dump:
```bash
server dump defaultServer --include=thread,heap,system
```

2. Request Timing:
```xml
<logging traceSpecification="*=info:RequestTiming=all"
         maxFileSize="20"
         maxFiles="10"/>
```



