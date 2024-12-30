# WebSphere Application Server Health Check Rehberi

## 1. BAŞLANGIÇ HAZIRLIĞI VE ERİŞİMLER

### 1.1 WebSphere Admin Console Erişimi

#### Console'a Erişim
- Default URL: `https://<hostname>:9043/ibm/console`
  - Production ortamında genelde farklı port kullanılır
  - SSL sertifika uyarısı alabilirsiniz, bu normaldir

#### Erişim Yoksa Yapılacaklar
1. WAS servisinin çalıştığını kontrol edin:
```bash
# Linux/Unix için
ps -ef | grep WebSphere
# Çıktıda 'dmgr' ve 'nodeagent' processlerini görmelisiniz

# Windows için
tasklist | findstr /i "java"
# DMGR ve NodeAgent servislerini görmelisiniz
```

2. Porta erişim kontrolü:
```bash
# Telnet ile port kontrolü
telnet <hostname> 9043

# Netstat ile port dinleme kontrolü
netstat -an | grep 9043
```

3. Firewall kontrolü:
```bash
# Linux için
iptables -L -n | grep 9043

# Windows için
netsh advfirewall firewall show rule name=all | findstr "9043"
```

### 1.2 Operating System Erişimi

#### SSH Key Oluşturma ve Yetkilendirme
```bash
# SSH key oluşturma
ssh-keygen -t rsa -b 4096 -C "was_health_check"

# Public key'i servera kopyalama
ssh-copy-id -i ~/.ssh/id_rsa.pub wasadmin@<server_ip>

# Yetki kontrolü
ssh wasadmin@<server_ip> "id"
# Çıktıda wasadmin kullanıcısının gruplarını görmelisiniz
```

#### WAS Dizin Yapısı ve Önemli Lokasyonlar
```plaintext
/opt/IBM/WebSphere/AppServer/
├── bin/           # Temel komutlar
├── logs/          # Server logları
├── profiles/      # Server profilleri
│   ├── AppSrv01/  # Default profile
│   │   ├── logs/  # Profile specific loglar
│   │   └── config/ # Profile konfigürasyonları
├── java/          # JDK dizini
└── properties/    # Property dosyaları
```

### 1.3 Monitoring Tool Erişimi ve Kurulumu

#### AppDynamics Agent Kurulumu
1. Agent dosyasını indirin:
```bash
# Agent dizini oluştur
mkdir -p /opt/appdynamics/agent

# Agent'ı indir ve aç
wget <AppD_download_url> -O appd-agent.zip
unzip appd-agent.zip -d /opt/appdynamics/agent
```

2. WAS'a agent ekleyin:
```bash
# Generic JVM arguments'e eklenecek
-javaagent:/opt/appdynamics/agent/javaagent.jar
-Dappdynamics.agent.applicationName=<app_name>
-Dappdynamics.agent.tierName=<tier_name>
-Dappdynamics.agent.nodeName=<node_name>
```

## 2. VERSİYON VE SİSTEM BİLGİSİ TOPLAMA

### 2.1 WAS Versiyon Bilgisi

#### Binary Klasöründen Versiyon Öğrenme
```bash
cd /opt/IBM/WebSphere/AppServer/bin

# Detaylı versiyon bilgisi
./versionInfo.sh -maintenancePackages

# Çıktıyı anlama:
# Product Name: IBM WebSphere Application Server
# Version: 9.0.5.x
# Build Level: cf12345
# Build Date: 12/12/2023
```

#### Admin Console'dan Versiyon Kontrolü
1. Navigasyon:
```plaintext
Servers > Server Types > WebSphere application servers > [server_name] > 
Java and Process Management > Process Definition > Java Virtual Machine
```

2. Versiyon bilgisinin yeri:
- "Runtime" sekmesinde "Java Runtime Environment" altında
- "Additional Properties" altında "Java SDK Version"

### 2.2 JDK Versiyon Kontrolü

#### IBM JDK Version Check
```bash
# IBM JDK path'i
/opt/IBM/WebSphere/AppServer/java/bin/java -version

# Çıktıyı anlama:
# IBM J9 VM: Bu IBM'in JVM'i
# JRE 1.8.0: Java versiyonu
# pxa6480sr7fp10-20220401_01: Build seviyesi
```

### 2.3 Sistem Topolojisi

#### Node ve Cluster Yapısını Çıkarma

1. Nodeagent Durumu:
```bash
# Tüm nodeların durumu
./serverStatus.sh -all

# Çıktıyı anlama:
# LISTENING: Node aktif
# STOPPED: Node durmuş
# STARTING: Node başlıyor
```

2. Cluster Yapısı:
```bash
# Cluster memberları
./listClusterMembers.sh -all

# Çıktının formatı:
# ClusterName(Weight:MemberName)
# Örnek: Cluster1(2:Server1,2:Server2)
```

## 3. PERFORMANS ANALİZİ VE MONITORING

### 3.1 Memory ve Heap Analizi

#### Heap Size ve Kullanımı

1. WSAdmin ile Heap Durumu:
```bash
# WSAdmin'e bağlan
./wsadmin.sh -lang jython

# Heap bilgisi al
wsadmin>print AdminControl.getAttribute(AdminControl.queryNames('WebSphere:type=JVM,*'), 'heapSize')

# Çıktıyı anlama:
# 2147483648: Maximum heap size (byte)
# Used/Free oranı için:
wsadmin>print AdminControl.getAttribute(AdminControl.queryNames('WebSphere:type=JVM,*'), 'freeMemory')
```

#### Garbage Collection Analizi

1. GC Log Analizi:
```bash
# GC log lokasyonu
cd /opt/IBM/WebSphere/AppServer/profiles/<profile_name>/logs/<server_name>

# GC pattern analizi
grep "GC cycle" native_stderr.log | tail -100

# Çıktıyı anlama:
# <GC type>: [<heap before>]->[<heap after>], <pause time>
# Örnek: GC cycle 123: [1024K]->[512K], 0.123 secs
```

2. GC Metric'leri Hesaplama:
```python
# Python script ile GC analizi
import re

def analyze_gc_log(log_file):
    gc_pattern = r'GC cycle (\d+): \[(\d+)K\]->\[(\d+)K\], (\d+\.\d+) secs'
    
    with open(log_file, 'r') as f:
        content = f.read()
        
    matches = re.finditer(gc_pattern, content)
    
    total_time = 0
    gc_count = 0
    
    for match in matches:
        gc_count += 1
        total_time += float(match.group(4))
        
    return {
        'gc_count': gc_count,
        'avg_time': total_time/gc_count if gc_count > 0 else 0
    }
```

### 3.2 Thread Analizi

#### Thread Dump Alma ve Analiz

1. Manuel Thread Dump:
```bash
# PID bulma
ps -ef | grep WebSphere

# Thread dump alma
kill -3 <pid>

# Thread dump lokasyonu
cd /opt/IBM/WebSphere/AppServer/profiles/<profile_name>/logs/<server_name>
ls -lrt javacore*.txt
```

2. Thread Dump Analizi:
```bash
# Thread state dağılımı
grep "State:" javacore*.txt | sort | uniq -c

# Deadlock kontrolü
grep -A 50 "Deadlock" javacore*.txt

# Hung thread analizi
grep -A 50 "Blocked" javacore*.txt
```

3. Thread Pattern Analizi:
```python
# Python ile thread analizi
def analyze_thread_dump(file_path):
    states = {
        'RUNNABLE': 0,
        'WAITING': 0,
        'TIMED_WAITING': 0,
        'BLOCKED': 0
    }
    
    with open(file_path, 'r') as f:
        for line in f:
            if 'State:' in line:
                for state in states:
                    if state in line:
                        states[state] += 1
                        
    return states
```

### 3.3 Connection Pool Analizi

#### JDBC Connection Pool Monitoring

1. Aktif Connection Sayısı:
```sql
-- DB2 için detailed connection analizi
WITH CONNECTIONS AS (
    SELECT APPLICATION_NAME,
           CLIENT_WRKSTNNAME,
           CLIENT_USERID,
           COUNT(*) as CONN_COUNT,
           MAX(ELAPSED_TIME_SEC) as MAX_TIME
    FROM TABLE(MON_GET_CONNECTION(cast(NULL as bigint),-2))
    GROUP BY APPLICATION_NAME, CLIENT_WRKSTNNAME, CLIENT_USERID
)
SELECT *
FROM CONNECTIONS
WHERE MAX_TIME > 300
ORDER BY CONN_COUNT DESC;

-- Çıktıyı anlama:
-- APPLICATION_NAME: WAS application adı
-- CONN_COUNT: Bu app için toplam connection
-- MAX_TIME: En uzun connection süresi
```

2. Connection Pool Settings Kontrolü:
```bash
# WSAdmin ile pool ayarları
wsadmin>print AdminConfig.show('DataSource')

# Önemli parametreler:
# connectionTimeout: Connection alma timeout
# maxConnections: Maximum connection sayısı
# minConnections: Minimum connection sayısı
```

## 4. PROBLEM ANALİZİ VE ÇÖZÜM

### 4.1 High CPU Problemi Analizi

#### CPU Kullanan Thread'leri Bulma

1. Top ile Thread Analizi:
```bash
# CPU consuming threadler
top -H -p <pid>

# Çıktıyı anlama:
#  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
# 1234 wasadmin  20   0   82.3g  23.0g  20.4g R 99.9 72.0   2587:23 java

# Thread ID'yi hex'e çevirme
printf "%x\n" 1234
# Çıktı: 4d2
```

2. Thread Stack Trace Analizi:
```bash
# Thread dump'ta hex ID arama
grep -A 50 "4d2" javacore*.txt

# Stack trace'i anlama:
# at java.lang.Thread.sleep -> Thread uyuyor
# at java.net.SocketInputStream.read -> Network I/O
# at java.lang.Object.wait -> Object lock bekliyor
```

### 4.2 Memory Leak Analizi

#### Heap Dump Alma ve Analiz

1. Heap Dump Alma:
```bash
# IBM JDK için heap dump
./jmap -dump:format=b,file=heap_$(date +%Y%m%d_%H%M%S).bin <pid>

# Heap dump size kontrolü
ls -lh heap_*.bin

# Heap dump'ı sıkıştırma
tar czf heap_$(date +%Y%m%d_%H%M%S).tar.gz heap_*.bin
```

2. IBM HeapAnalyzer Kullanımı:
```bash
# HeapAnalyzer başlatma
./heapAnalyzer.sh

# Analiz adımları:
1. File > Open Heap Dump
2. Leak Suspects > Analyze
3. Object References > Show Paths to GC Root
```

## 5. MONITORING VE ALERTING

### 5.1 Custom Monitoring Script

```python
#!/usr/bin/python3

import subprocess
import re
import time
import smtplib
from email.message import EmailMessage

class WASMonitor:
    def __init__(self):
        self.thresholds = {
            'cpu': 75,
            'memory': 85,
            'gc_frequency': 20
        }
    
    def check_cpu(self, pid):
        cmd = f"ps -p {pid} -o %cpu"
        result = subprocess.run(cmd.split(), capture_output=True, text=True)
        cpu_usage = float(result.stdout.split('\n')[1])
        return cpu_usage > self.thresholds['cpu']
    
    def check_memory(self, pid):
        cmd = f"ps -p {pid} -o %mem"
        result = subprocess.run(cmd.split(), capture_output=True, text=True)
        mem_usage = float(result.stdout.split('\n')[1])
        return mem_usage > self.thresholds['memory']
    
    def analyze_gc_log(self, log_file):
        gc_count = 0
        with open(log_file, 'r') as f:
            gc_count = len(re.findall('GC cycle', f.read()))
        return gc_count > self.thresholds['gc_frequency']
    
    def send_alert(self, message):
        msg = EmailMessage()
        msg.set_content(message)
        msg['Subject'] = 'WAS Health Check Alert'
        msg['From'] = "monitoring@company.com"
        msg['To'] = "admin@company.com"
        
        s = smtplib.SMTP('localhost')
        s.send_message(msg)
        s.quit()

    def run(self):
        while True:
            try:
                # Check all metrics
                if self.check_cpu('pid'):
                    self.send_alert('High CPU Usage Alert')
                
                if self.check_memory('pid'):
                    self.send_alert('High Memory Usage Alert')
                
                if self.analyze_gc_log('/path/to/gc.log'):
                    self.send_alert('High GC Frequency Alert')
                
                time.sleep(300)  # Check every 5 minutes
                
            except Exception as e:
                self.send_alert(f'Monitoring Error: {str(e)}')

if __name__ == '__main__':
    monitor = WASMonitor()
    monitor.run()
```

### 5.2 Log Aggregation ve Analiz

#### ELK Stack Kurulumu ve Konfigürasyonu

1. Filebeat Konfig

## 6. DETAYLI LOG ANALİZİ VE MONITORING

### 6.1 Sistemik Log Analizi

#### SystemOut.log Analizi
```bash
# Önemli hata patternleri ve anlamları:
grep "WSVR0605W" SystemOut.log
# WSVR0605W -> Thread hang durumu

grep "WSVR0606W" SystemOut.log
# WSVR0606W -> Thread stall durumu

grep "WSVR0661W" SystemOut.log
# WSVR0661W -> WebContainer thread pool dolu

grep "WSVR0537W" SystemOut.log
# WSVR0537W -> JDBC connection pool timeout

# Son 1 saatteki hataları analiz et
find . -name "SystemOut.log" -mmin -60 -exec grep "^E" {} \;
```

#### Error Pattern Analizi Script'i
```python
#!/usr/bin/python3

def analyze_error_patterns(log_file):
    error_patterns = {
        'WSVR0605W': 'Thread Hang',
        'WSVR0606W': 'Thread Stall',
        'WSVR0661W': 'ThreadPool Full',
        'WSVR0537W': 'Connection Timeout',
        'OutOfMemoryError': 'Memory Issue',
        'Deadlock': 'Thread Deadlock'
    }
    
    error_counts = {pattern: 0 for pattern in error_patterns}
    
    with open(log_file, 'r') as f:
        for line in f:
            for pattern in error_patterns:
                if pattern in line:
                    error_counts[pattern] += 1
                    
    return error_counts

# Kullanım
errors = analyze_error_patterns('/path/to/SystemOut.log')
for pattern, count in errors.items():
    if count > 0:
        print(f"{pattern}: {count} occurrences")
```

### 6.2 Performance Monitoring Script'leri

#### JVM Memory Monitoring
```python
#!/usr/bin/python3
import subprocess
import json
import time

class JVMMonitor:
    def __init__(self, pid):
        self.pid = pid
        
    def get_heap_usage(self):
        cmd = f"jstat -gc {self.pid}"
        result = subprocess.run(cmd.split(), capture_output=True, text=True)
        
        # Çıktıyı parse et
        values = result.stdout.split('\n')[1].split()
        
        heap_info = {
            'young_gen_used': float(values[2]),
            'young_gen_total': float(values[1]) + float(values[2]),
            'old_gen_used': float(values[4]),
            'old_gen_total': float(values[3]) + float(values[4]),
            'gc_count': int(values[6]),
            'gc_time': float(values[7])
        }
        
        return heap_info
    
    def monitor(self, interval=60):
        while True:
            try:
                heap_info = self.get_heap_usage()
                
                # JSON formatında kaydet
                with open(f'heap_stats_{time.strftime("%Y%m%d")}.json', 'a') as f:
                    json.dump({
                        'timestamp': time.time(),
                        'metrics': heap_info
                    }, f)
                    f.write('\n')
                    
                time.sleep(interval)
                
            except Exception as e:
                print(f"Error: {str(e)}")
                time.sleep(interval)

# Kullanım
monitor = JVMMonitor('<pid>')
monitor.monitor()
```

#### Thread Pool Monitoring
```python
#!/usr/bin/python3
import subprocess
import json
import time

class ThreadPoolMonitor:
    def __init__(self):
        self.wsadmin_path = "/opt/IBM/WebSphere/AppServer/bin/wsadmin.sh"
        
    def get_pool_stats(self):
        # wsadmin script
        script = """
print AdminControl.getAttribute('WebSphere:name=WebContainer,*', 'poolSize')
print AdminControl.getAttribute('WebSphere:name=WebContainer,*', 'activeCount')
"""
        
        with open('monitor.py', 'w') as f:
            f.write(script)
            
        cmd = f"{self.wsadmin_path} -f monitor.py -lang jython"
        result = subprocess.run(cmd.split(), capture_output=True, text=True)
        
        lines = result.stdout.strip().split('\n')
        return {
            'pool_size': int(lines[0]),
            'active_threads': int(lines[1])
        }
        
    def monitor(self, interval=60):
        while True:
            try:
                stats = self.get_pool_stats()
                
                with open(f'thread_stats_{time.strftime("%Y%m%d")}.json', 'a') as f:
                    json.dump({
                        'timestamp': time.time(),
                        'metrics': stats
                    }, f)
                    f.write('\n')
                    
                time.sleep(interval)
                
            except Exception as e:
                print(f"Error: {str(e)}")
                time.sleep(interval)

# Kullanım
monitor = ThreadPoolMonitor()
monitor.monitor()
```

### 6.3 Advanced Troubleshooting Teknikleri

#### Hung Thread Analizi
```bash
# Thread dump sequence alma
for i in {1..3}; do
    echo "Taking dump $i at $(date)"
    kill -3 <pid>
    sleep 30
done

# Thread dump karşılaştırma script'i
#!/bin/bash

DUMP1=$1
DUMP2=$2

# Thread ID'leri çıkar
grep "Thread-" $DUMP1 | sort > threads1.txt
grep "Thread-" $DUMP2 | sort > threads2.txt

# Aynı durumda kalan thread'leri bul
diff threads1.txt threads2.txt | grep "^<" | cut -d" " -f2
```

#### Memory Leak Investigation
```python
#!/usr/bin/python3

class HeapAnalyzer:
    def __init__(self, heap_dump):
        self.heap_dump = heap_dump
        
    def find_large_objects(self):
        cmd = f"jmap -histo:live {self.heap_dump}"
        result = subprocess.run(cmd.split(), capture_output=True, text=True)
        
        objects = []
        for line in result.stdout.split('\n')[2:]:  # Skip header
            if not line:
                continue
            parts = line.split()
            objects.append({
                'count': int(parts[1]),
                'bytes': int(parts[2]),
                'class': parts[3]
            })
            
        return sorted(objects, key=lambda x: x['bytes'], reverse=True)[:10]
        
    def analyze_growth(self, gc_logs):
        with open(gc_logs, 'r') as f:
            content = f.read()
            
        # Parse GC logs for heap growth pattern
        pattern = r'\[PSYoungGen: (\d+)K->(\d+)K\((\d+)K\)\]'
        matches = re.finditer(pattern, content)
        
        growth = []
        for match in matches:
            before = int(match.group(1))
            after = int(match.group(2))
            total = int(match.group(3))
            
            growth.append({
                'before': before,
                'after': after,
                'total': total,
                'growth_rate': (after - before) / before if before > 0 else 0
            })
            
        return growth

# Kullanım
analyzer = HeapAnalyzer('heap.bin')
large_objects = analyzer.find_large_objects()
growth_pattern = analyzer.analyze_growth('gc.log')
```

### 6.4 Performance Testing ve Analiz

#### Load Test Monitoring
```python
#!/usr/bin/python3

class LoadTestMonitor:
    def __init__(self):
        self.metrics = {
            'response_times': [],
            'error_counts': {},
            'thread_usage': [],
            'heap_usage': []
        }
        
    def collect_metrics(self):
        # Response time collection
        cmd = "curl -w '%{time_total}\n' -o /dev/null -s http://your-app-url"
        result = subprocess.run(cmd.split(), capture_output=True, text=True)
        self.metrics['response_times'].append(float(result.stdout))
        
        # Error count from logs
        with open('SystemOut.log', 'r') as f:
            errors = len(re.findall('ERROR', f.read()))
            self.metrics['error_counts'][time.time()] = errors
            
        # Thread usage
        cmd = f"ps -p <pid> -L -o pcpu,pid,tid"
        result = subprocess.run(cmd.split(), capture_output=True, text=True)
        thread_count = len(result.stdout.split('\n')) - 2  # Remove header and empty line
        self.metrics['thread_usage'].append(thread_count)
        
    def generate_report(self):
        report = {
            'avg_response_time': sum(self.metrics['response_times']) / len(self.metrics['response_times']),
            'max_response_time': max(self.metrics['response_times']),
            'error_rate': sum(self.metrics['error_counts'].values()) / len(self.metrics['error_counts']),
            'avg_thread_usage': sum(self.metrics['thread_usage']) / len(self.metrics['thread_usage'])
        }
        
        return report

# Kullanım
monitor = LoadTestMonitor()
while True:
    monitor.collect_metrics()
    time.sleep(5)  # 5 saniye aralıklarla ölç
```

### 6.5 Capacity Planning Araçları

#### Resource Usage Prediction
```python
#!/usr/bin/python3
import pandas as pd
from sklearn.linear_model import LinearRegression
import numpy as np

class CapacityPlanner:
    def __init__(self):
        self.model = LinearRegression()
        
    def load_metrics(self, file_path):
        # JSON formatında metrik dosyasını oku
        df = pd.read_json(file_path, lines=True)
        return df
        
    def predict_resource_usage(self, metrics_file, days_ahead=30):
        df = self.load_metrics(metrics_file)
        
        # Trend analizi
        df['timestamp'] = pd.to_datetime(df['timestamp'], unit='s')
        df.set_index('timestamp', inplace=True)
        
        # Günlük ortalamalar
        daily_avg = df.resample('D').mean()
        
        # Trend tahmini
        X = np.arange(len(daily_avg)).reshape(-1, 1)
        y = daily_avg['metrics.heap_used'].values
        
        self.model.fit(X, y)
        
        # Gelecek tahminleri
        future_days = np.arange(len(daily_avg), len(daily_avg) + days_ahead).reshape(-1, 1)
        predictions = self.model.predict(future_days)
        
        return predictions
        
    def generate_capacity_report(self, metrics_file):
        current_usage = self.load_metrics(metrics_file).tail(1)['metrics.heap_used'].values[0]
        predictions = self.predict_resource_usage(metrics_file)
        
        report = {
            'current_usage': current_usage,
            'predicted_30_days': predictions[-1],
            'growth_rate': (predictions[-1] - current_usage) / current_usage * 100,
            'days_until_capacity': self.calculate_days_until_capacity(predictions)
        }
        
        return report
        
    def calculate_days_until_capacity(self, predictions, capacity_threshold=0.85):
        days = 0
        for pred in predictions:
            if pred / self.total_capacity > capacity_threshold:
                return days
            days += 1
        return None

# Kullanım
planner = CapacityPlanner()
report = planner.generate_capacity_report('metrics.json')
print(f"Predicted capacity usage in 30 days: {report['predicted_30_days']:.2f}%")
print(f"Growth rate: {report['growth_rate']:.2f}%")
print(f"Days until capacity threshold: {report['days_until_capacity']}")
```

Bu eklemeler ile birlikte WAS Health Check sürecini daha detaylı ve programatik olarak yönetebilirsiniz. Her bir script'in amacı ve kullanımı açıklanmıştır. Bu araçları kendi ortamınıza göre özelleştirerek kullanabilirsiniz.

Ayrıca, bu monitoring ve analiz araçlarını cron job'lar ile otomatize edebilir ve alerting sistemine entegre edebilirsiniz. Örnek bir crontab yapılandırması:

```bash
# Her 5 dakikada bir JVM monitoring
*/5 * * * * /path/to/jvm_monitor.py

# Her saat başı thread pool monitoring
0 * * * * /path/to/thread_monitor.py

# Her gün gece yarısı capacity planning
0 0 * * * /path/to/capacity_planner.py
```

Bu monitoring ve analiz araçlarını kullanırken dikkat edilmesi gereken noktalar:
1. Script'leri production ortamında çalıştırmadan önce test edin
2. Resource usage'ı (CPU, memory) kontrol edin
3. Error handling mekanizmalarını implemente edin
4. Monitoring verilerini düzenli olarak archive'leyin
5. Alert threshold'larını ortamınıza göre ayarlayın

