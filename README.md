# NGINX Worker Settings

## Worker Process Nedir?

NGINX, yüksek performanslı bir **master-worker mimarisi** kullanır:

- **Master Process**: Konfigürasyon dosyalarını okur, worker process'leri yönetir ve sinyal işleme görevlerini üstlenir
- **Worker Process'ler**: Gerçek HTTP isteklerini işler, istemci bağlantılarını kabul eder ve yanıtları döner
- Bu mimari sayesinde NGINX, aynı anda binlerce bağlantıyı verimli bir şekilde işleyebilir

Worker process'ler birbirinden bağımsız çalışır ve bir worker'da oluşan sorun diğerlerini etkilemez.

## Temel Ayarlar

### Worker Process Sayısı

```nginx
# Ana worker process sayısını belirler
# auto: CPU çekirdek sayısına göre otomatik ayarlar
# Sayı: Manuel olarak worker process sayısını belirler
worker_processes auto;
```

### Worker Connections

```nginx
events {
    # Her worker process'in aynı anda işleyebileceği maksimum EŞZAMANLI bağlantı sayısı
    # bu değer worker_rlimit_nofile'dan küçük olmalı (eğer tanımlıysa)
    # Hesaplama: worker_connections < (worker_rlimit_nofile - 500)
    # hesaplamada 500, nginx'in kendi dosya açma işlemleri için ayrılan sayıdır
    # NOT: Bu değer toplam istek sayısını değil, aynı anda açık olan bağlantı sayısını belirler
    worker_connections 1024;
}
```

### Dosya Limiti

```nginx
# Worker process başına açık dosya limiti
# Bu değer ulimit -n değerinden küçük veya eşit olmalı
# Linux içerisinde olan limiti ezer yani aslında bu limit verilirse nginx bu limiti kullanır
# Eğer limit verilmezse container içerisindeki limit kullanılır(ulimit -n)
worker_rlimit_nofile 65535;
```

## Gelişmiş Ayarlar

### Event Model

```nginx
events {
    # Linux için en verimli I/O event model (epoll)
    use epoll;
    
    # bir worker'ın aynı anda birden fazla yeni bağlantıyı kabul edip edemeyeceği
    # on: Performans artışı sağlar, off: Daha dengeli dağılım
    multi_accept on;
    
    # worker process'lerin yük dengeleme stratejisi
    # off: Tüm worker'lar eşit şekilde bağlantı kabul eder (önerilen)
    # on: Sıralı olarak bağlantı kabul eder (eski davranış)
    accept_mutex off;
}
```

### Worker Priority

```nginx
# Worker process önceliği (-20 en yüksek, 20 en düşük)
# nginx worker process'lerinin işletim sistemi seviyesindeki önceliğini belirler. (Default: 0)
worker_priority 0;
```

### Monitoring Endpoints

```nginx
# Worker process bilgilerini gösteren endpoint
location /nginx-status {
    stub_status on;
    access_log off;
    allow 127.0.0.1;
    allow ::1;
    deny all;
}

# Test endpoint - Worker process PID ve zaman bilgisi
location /test {
    return 200 "Worker Process Test - PID: $pid \n Time: $time_local\n";
    add_header Content-Type text/plain;
}
```

## Kullanım

```bash
# Projeyi çalıştır
docker compose up -d

# Status kontrolü
#curl container içerisinden nginx-status'e istek atılırsa worker process sayısı ve bağlantı bilgileri gelir

# Worker bilgisi
curl http://localhost:8080/test
```

## Optimizasyon İpuçları

1. **Worker Process Sayısı**: `worker_processes auto` - CPU çekirdek sayısına göre otomatik ayarlar
2. **Bağlantı Hesabı**: `worker_processes × worker_connections = toplam eşzamanlı bağlantı`
3. **Dosya Limiti Hesabı**: `worker_connections < (worker_rlimit_nofile - 500)`
   - 500, nginx'in kendi dosya açma işlemleri için ayrılan dosya sayısıdır
4. **Sistem Limiti**: worker_rlimit_nofile ≤ ulimit -n (sistem limiti)

## Yaygın Hatalar

- **"too many open files"** → `worker_rlimit_nofile` artırın
- **Düşük performans** → `worker_processes auto` kullanın
- **Connection refused** → `worker_connections` artırın
