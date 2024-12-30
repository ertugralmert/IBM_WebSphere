##### Script İle
> Burada script ile cluster oluşturma, cluster optimizasyonu ve uygulama deploy script üzerinde örnek yapacağız

```bash
WebSphere Uygulama Deployment ve Cluster Yönetimi Detaylı Rehberi
1. Cluster Yapılandırması Detayları
1.1. Cluster Oluşturma ve Yönetimi

# Önce bir deployment script oluşturalım
cat << 'EOF' > /tmp/create_cluster.py
def createProductionCluster():
    try:
        # 1. Cluster oluştur
        print "Creating cluster..."
        AdminTask.createCluster('[-clusterName ProductionCluster \
            -preferLocal true]')
        
        # 2. İlk node için member oluştur (Node01)
        print "Creating first cluster member..."
        AdminTask.createClusterMember('[-clusterName ProductionCluster \
            -memberNode Node01 \
            -memberName AppServer01 \
            -memberWeight 2 \
            -genUniquePorts true]')
        
        # 3. İkinci node için member oluştur (Node02)
        print "Creating second cluster member..."
        AdminTask.createClusterMember('[-clusterName ProductionCluster \
            -memberNode Node02 \
            -memberName AppServer02 \
            -memberWeight 2 \
            -genUniquePorts true]')
        
        # 4. Session replikasyonunu ayarla
        print "Configuring session replication..."
        AdminTask.modifyReplicationDomain('[-replicationDomain \
            ProductionClusterDomain -numberOfReplicas 2]')
        
        # 5. Load balancing ayarları
        print "Configuring load balancing..."
        cluster = AdminConfig.getid('/ServerCluster:ProductionCluster/')
        AdminConfig.modify(cluster, [['preferLocal', 'true'], 
                                   ['serverIOTimeoutRetry', '3']])
        
        # 6. Kaydet
        AdminConfig.save()
        print "Cluster configuration completed successfully!"
        
    except:
        print "Error occurred during cluster creation:"
        print sys.exc_info()[0]
        print sys.exc_info()[1]
# Cluster'ı oluştur
createProductionCluster()
EOF
# Scripti çalıştır
/opt/IBM/WebSphere/AppServer/profiles/DMGR01/bin/wsadmin.sh -lang jython \
    -f /tmp/create_cluster.py -username wasadmin -password waspass123
1.2. Cluster Özellikleri ve Optimizasyonu

# Cluster optimizasyon scripti
cat << 'EOF' > /tmp/optimize_cluster.py
def optimizeCluster():
    # 1. Web Container ayarları
    webcontainer = AdminConfig.list('WebContainer')
    AdminConfig.modify(webcontainer, [['enableServletCaching', 'true'],
                                    ['disablePooling', 'false']])
    
    # 2. Thread Pool ayarları
    threadPool = AdminConfig.list('ThreadPool')
    AdminConfig.modify(threadPool, [['minimumSize', '50'],
                                  ['maximumSize', '200'],
                                  ['inactivityTimeout', '5000'],
                                  ['isGrowable', 'true']])
    
    # 3. Session yönetimi
    session = AdminConfig.list('SessionManager')
    AdminConfig.modify(session, [['enableCookies', 'true'],
                               ['enableUrlRewriting', 'false'],
                               ['enableProtocolSwitchRewriting', 'false'],
                               ['sessionPersistenceMode', 'DATABASE_OR_MEMORY']])
    
    # 4. Dynamic Cache ayarları
    cache = AdminConfig.list('DynamicCache')
    AdminConfig.modify(cache, [['enableCaching', 'true'],
                             ['cacheSize', '2000'],
                             ['enableDiskOffload', 'true']])
    
    AdminConfig.save()
optimizeCluster()
EOF
# Scripti çalıştır
/opt/IBM/WebSphere/AppServer/profiles/DMGR01/bin/wsadmin.sh -lang jython \
    -f /tmp/optimize_cluster.py -username wasadmin -password waspass123
2. Uygulama Deployment İşlemleri
2.1. WAR/EAR Hazırlığı ve Kontroller

# 1. Uygulama paketini kontrol et
jar tvf myapp.war    # WAR içeriğini listele
jar tvf myapp.ear    # EAR içeriğini listele
# 2. Deployment dizini oluştur
mkdir -p /apps/deployment/production
cp myapp.ear /apps/deployment/production/
# 3. Yetkilendirme
chown -R wasadmin:wasadmin /apps/deployment
chmod -R 755 /apps/deployment
2.2. Detaylı Deployment Script

cat << 'EOF' > /tmp/deploy_application.py
def deployProductionApp(earFile, appName, contextRoot):
    try:
        print "Starting deployment for:", appName
        
        # 1. Eski versiyonu kaldır (varsa)
        apps = AdminApp.list().split(lineSeparator)
        if appName in apps:
            print "Uninstalling existing version..."
            AdminApp.uninstall(appName)
            AdminConfig.save()
        
        # 2. Deployment options
        options = '[-appname ' + appName + ' \
                  -contextroot ' + contextRoot + ' \
                  -cluster ProductionCluster \
                  -MapModulesToServers [[".*",".*","WebSphere:cluster=ProductionCluster"]] \
                  -MapWebModToVH [[".*",".*","default_host"]] \
                  -usedefaultbindings \
                  -nouseMetaDataFromBinary \
                  -deployejb \
                  -clientMode isolated \
                  -validateinstall \
                  -nopreCompileJSPs \
                  -distributeApp \
                  -fetchKey -asyncInstall \
                  -createMBeansForResources]'
        
        # 3. Uygulamayı yükle
        print "Installing application..."
        AdminApp.install(earFile, options)
        
        # 4. Classloader ayarları
        print "Configuring classloader..."
        deploymentos = AdminConfig.list('Deployment', 
            'WebSphere:cell=*,name=' + appName + ',*')
        classloader = AdminConfig.showAttribute(deploymentos, 'classloader')
        AdminConfig.modify(classloader, [['mode', 'PARENT_LAST']])
        
        # 5. Startup behavior
        print "Setting startup behavior..."
        AdminApp.edit(appName, '[-startingWeight 2]')
        
        # 6. Session management
        print "Configuring session management..."
        AdminApp.edit(appName, '[-sessionManagement [-enableSSLTracking true \
            -enableCookies true -enableUrlRewriting false]]')
        
        # 7. Kaydet ve senkronize et
        print "Saving and synchronizing..."
        AdminConfig.save()
        
        # 8. Node'ları senkronize et
        for node in AdminControl.queryNames('type=NodeSync,*').split(lineSeparator):
            print "Synchronizing node:", node
            AdminControl.invoke(node, 'sync')
        
        # 9. Uygulamayı başlat
        print "Starting application..."
        AdminControl.invoke('WebSphere:name=' + appName + ',*', 'start')
        
        print "Deployment completed successfully!"
        
    except:
        print "Error during deployment:"
        print sys.exc_info()[0]
        print sys.exc_info()[1]
        raise
# Uygulamayı deploy et
deployProductionApp('/apps/deployment/production/myapp.ear', 
                   'ProductionApp', 
                   '/myapp')
EOF
# Scripti çalıştır
/opt/IBM/WebSphere/AppServer/profiles/DMGR01/bin/wsadmin.sh -lang jython \
    -f /tmp/deploy_application.py -username wasadmin -password waspass123
3. Deployment Sonrası Kontroller
3.1. Uygulama Durumu Kontrolü

# Status check scripti
cat << 'EOF' > /tmp/check_app_status.py
def checkApplicationStatus(appName):
    print "Checking status for application:", appName
    
    # 1. Uygulama durumu
    appManager = AdminControl.queryNames('type=ApplicationManager,*')
    running = AdminControl.queryNames('type=Application,name=' + appName + ',*')
    
    if running:
        print "Application is running"
        
        # 2. Memory kullanımı
        jvm = AdminControl.queryNames('type=JVM,*')
        heapSize = AdminControl.getAttribute(jvm, 'heapSize')
        freeMemory = AdminControl.getAttribute(jvm, 'freeMemory')
        print "Heap Size:", heapSize
        print "Free Memory:", freeMemory
        
        # 3. Session bilgileri
        sessions = AdminControl.queryNames('type=ServletSessionManager,*')
        for session in sessions.split(lineSeparator):
            if session:
                activeSessions = AdminControl.getAttribute(session, 'activeSessions')
                print "Active Sessions:", activeSessions
        
        # 4. Thread pool durumu
        threadPools = AdminControl.queryNames('type=ThreadPool,*')
        for pool in threadPools.split(lineSeparator):
            if pool:
                activeThreads = AdminControl.getAttribute(pool, 'activeThreads')
                poolSize = AdminControl.getAttribute(pool, 'size')
                print "Thread Pool Size:", poolSize
                print "Active Threads:", activeThreads
    else:
        print "Application is not running!"
# Uygulama durumunu kontrol et
checkApplicationStatus('ProductionApp')
EOF
# Scripti çalıştır
/opt/IBM/WebSphere/AppServer/profiles/DMGR01/bin/wsadmin.sh -lang jython \
    -f /tmp/check_app_status.py -username wasadmin -password waspass123

```