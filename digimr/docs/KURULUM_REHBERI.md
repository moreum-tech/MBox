# digimr - Deployment Guide / Dağıtım Kılavuzu

**Versiyon**: 1.0
**Tarih**: Şubat 2026

---

## İçindekiler / Table of Contents

1. [Sistem Gereksinimleri / System Requirements](#sistem-gereksinimleri--system-requirements)
2. [Güvenlik Yapılandırması / Security Configuration](#güvenlik-yapılandırması--security-configuration)
3. [Docker Deployment](#docker-deployment)
4. [Kubernetes Deployment](#kubernetes-deployment)
5. [Windows Server Deployment](#windows-server-deployment)
6. [Linux Server Deployment](#linux-server-deployment)
7. [Veritabanı Yapılandırması / Database Configuration](#veritabanı-yapılandırması--database-configuration)
8. [Load Balancing & Scaling](#load-balancing--scaling)
9. [Monitoring & Logging](#monitoring--logging)
10. [Backup & Recovery](#backup--recovery)
11. [SSL/TLS Yapılandırması](#ssltls-yapılandırması)

---

## Sistem Gereksinimleri / System Requirements

### Minimum Gereksinimler / Minimum Requirements

**API Servisi:**
- CPU: 2 vCPU
- RAM: 4 GB
- Disk: 20 GB SSD
- OS: Windows Server 2019+, Ubuntu 20.04+, RHEL 8+
- .NET 8.0 Runtime

**Agent (Desktop):**
- CPU: 1 vCPU / 2 cores
- RAM: 512 MB
- Disk: 100 MB
- OS: Windows 10+, Ubuntu 20.04+, macOS 11+
- .NET 8.0 Runtime

### Önerilen Gereksinimler / Recommended Requirements

**API Servisi (Production):**
- CPU: 4+ vCPU
- RAM: 8+ GB
- Disk: 100+ GB SSD
- OS: Ubuntu 22.04 LTS / Windows Server 2022
- .NET 8.0 Runtime
- Redis (caching)
- PostgreSQL / SQL Server (optional, for audit logs)

**Network:**
- 1 Gbps+ bandwidth
- Static IP address
- Firewall configured
- SSL/TLS certificate

---

## Güvenlik Yapılandırması / Security Configuration

### Production appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "DigitalSignature": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "api.example.com",
  "Kestrel": {
    "Limits": {
      "MaxRequestBodySize": 104857600,
      "KeepAliveTimeout": "00:02:00",
      "RequestHeadersTimeout": "00:00:30"
    }
  },
  "Security": {
    "MaxDocumentSizeBytes": 104857600,
    "MaxBatchDocuments": 100,
    "MaxBatchTotalSizeBytes": 524288000,
    "RequireHttps": true,
    "AllowedOrigins": [
      "https://app.example.com"
    ],
    "RequireApiKey": true,
    "ApiKeys": {
      "Enabled": true,
      "HeaderName": "Authorization",
      "Prefix": "Bearer "
    }
  },
  "RateLimit": {
    "EnableRateLimiting": true,
    "PermitLimit": 100,
    "Window": "00:01:00",
    "QueueLimit": 10
  },
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=digimr;Username=digimr_user;Password=YOUR_PASSWORD",
    "Redis": "localhost:6379,password=YOUR_REDIS_PASSWORD"
  }
}
```

### Environment Variables (Hassas Bilgiler / Sensitive Data)

```bash
# API Keys
DIGIMR_API_KEY=dmr_live_1234567890abcdef

# Database
DATABASE_CONNECTION_STRING="Host=db.example.com;Database=digimr;Username=digimr_user;Password=SECURE_PASSWORD"
REDIS_CONNECTION_STRING="redis.example.com:6379,password=REDIS_PASSWORD"

# TSA
TSA_URL=http://tsa-test.kamusm.gov.tr

# PKCS#11 Libraries
PKCS11_LIBRARY_PATH=/usr/lib/opensc-pkcs11.so

# Agent (Remote Token)
AGENT_API_KEY=agent_secure_key_123
```

### Security Headers Middleware

`Program.cs`'e ekleyin:

```csharp
// Security headers
app.Use(async (context, next) =>
{
    // Prevent clickjacking
    context.Response.Headers.Add("X-Frame-Options", "DENY");

    // Prevent MIME sniffing
    context.Response.Headers.Add("X-Content-Type-Options", "nosniff");

    // XSS protection
    context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");

    // Referrer policy
    context.Response.Headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");

    // Content Security Policy
    context.Response.Headers.Add("Content-Security-Policy",
        "default-src 'self'; " +
        "script-src 'self' 'unsafe-inline' 'unsafe-eval'; " +
        "style-src 'self' 'unsafe-inline'; " +
        "img-src 'self' data:; " +
        "font-src 'self' data:; " +
        "connect-src 'self';");

    // HSTS (for HTTPS only)
    if (context.Request.IsHttps)
    {
        context.Response.Headers.Add("Strict-Transport-Security",
            "max-age=31536000; includeSubDomains; preload");
    }

    await next();
});
```

---

## Docker Deployment

### Dockerfile (API Service)

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy project files
COPY ["src/DigitalSignature.API/DigitalSignature.API.csproj", "src/DigitalSignature.API/"]
COPY ["src/DigitalSignature.Core/DigitalSignature.Core.csproj", "src/DigitalSignature.Core/"]
COPY ["src/DigitalSignature.CAdES/DigitalSignature.CAdES.csproj", "src/DigitalSignature.CAdES/"]
COPY ["src/DigitalSignature.PAdES/DigitalSignature.PAdES.csproj", "src/DigitalSignature.PAdES/"]
COPY ["src/DigitalSignature.XAdES/DigitalSignature.XAdES.csproj", "src/DigitalSignature.XAdES/"]
COPY ["src/DigitalSignature.Xml/DigitalSignature.Xml.csproj", "src/DigitalSignature.Xml/"]
COPY ["src/DigitalSignature.Token/DigitalSignature.Token.csproj", "src/DigitalSignature.Token/"]
COPY ["src/DigitalSignature.RemoteToken/DigitalSignature.RemoteToken.csproj", "src/DigitalSignature.RemoteToken/"]
COPY ["src/DigitalSignature.Mobile/DigitalSignature.Mobile.csproj", "src/DigitalSignature.Mobile/"]

# Restore packages
RUN dotnet restore "src/DigitalSignature.API/DigitalSignature.API.csproj"

# Copy all source
COPY . .

# Build
WORKDIR "/src/src/DigitalSignature.API"
RUN dotnet build "DigitalSignature.API.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "DigitalSignature.API.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Create non-root user
RUN useradd -m -u 1000 digimr && chown -R digimr:digimr /app
USER digimr

ENTRYPOINT ["dotnet", "DigitalSignature.API.dll"]
```

### Dockerfile (Agent)

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 5555

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

COPY ["src/DigitalSignature.Agent/DigitalSignature.Agent.csproj", "src/DigitalSignature.Agent/"]
COPY ["src/DigitalSignature.Agent.Contracts/DigitalSignature.Agent.Contracts.csproj", "src/DigitalSignature.Agent.Contracts/"]
COPY ["src/DigitalSignature.Core/DigitalSignature.Core.csproj", "src/DigitalSignature.Core/"]
COPY ["src/DigitalSignature.Token/DigitalSignature.Token.csproj", "src/DigitalSignature.Token/"]

RUN dotnet restore "src/DigitalSignature.Agent/DigitalSignature.Agent.csproj"

COPY . .
WORKDIR "/src/src/DigitalSignature.Agent"
RUN dotnet build "DigitalSignature.Agent.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "DigitalSignature.Agent.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app

# Install PKCS#11 libraries
RUN apt-get update && \
    apt-get install -y opensc pcscd && \
    rm -rf /var/lib/apt/lists/*

COPY --from=publish /app/publish .

RUN useradd -m -u 1000 digimr && chown -R digimr:digimr /app
USER digimr

ENTRYPOINT ["dotnet", "DigitalSignature.Agent.dll"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: src/DigitalSignature.API/Dockerfile
    ports:
      - "8080:80"
      - "8443:443"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ASPNETCORE_URLS=https://+:443;http://+:80
      - ASPNETCORE_Kestrel__Certificates__Default__Path=/app/certs/certificate.pfx
      - ASPNETCORE_Kestrel__Certificates__Default__Password=${CERT_PASSWORD}
      - ConnectionStrings__DefaultConnection=${DATABASE_CONNECTION}
      - ConnectionStrings__Redis=${REDIS_CONNECTION}
    volumes:
      - ./certs:/app/certs:ro
      - ./logs:/app/logs
    depends_on:
      - postgres
      - redis
    restart: unless-stopped
    networks:
      - digimr-network

  agent:
    build:
      context: .
      dockerfile: src/DigitalSignature.Agent/Dockerfile
    ports:
      - "5555:5555"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - Agent__Mode=Remote
      - Agent__RequireApiKey=true
      - Agent__ApiKey=${AGENT_API_KEY}
    devices:
      - /dev/bus/usb:/dev/bus/usb  # USB token access
    privileged: true  # Required for PKCS#11
    volumes:
      - ./agent-logs:/app/logs
    restart: unless-stopped
    networks:
      - digimr-network

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=digimr
      - POSTGRES_USER=digimr_user
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    restart: unless-stopped
    networks:
      - digimr-network

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"
    restart: unless-stopped
    networks:
      - digimr-network

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - api
    restart: unless-stopped
    networks:
      - digimr-network

volumes:
  postgres-data:
  redis-data:

networks:
  digimr-network:
    driver: bridge
```

### .env (Docker Environment)

```env
# Database
POSTGRES_PASSWORD=SECURE_DB_PASSWORD
DATABASE_CONNECTION=Host=postgres;Database=digimr;Username=digimr_user;Password=SECURE_DB_PASSWORD

# Redis
REDIS_PASSWORD=SECURE_REDIS_PASSWORD
REDIS_CONNECTION=redis:6379,password=SECURE_REDIS_PASSWORD

# Certificates
CERT_PASSWORD=SECURE_CERT_PASSWORD

# Agent
AGENT_API_KEY=secure_agent_api_key_123
```

### nginx.conf

```nginx
events {
    worker_connections 1024;
}

http {
    upstream api_backend {
        server api:80;
    }

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=sign_limit:10m rate=5r/s;

    server {
        listen 80;
        server_name api.example.com;

        # Redirect HTTP to HTTPS
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name api.example.com;

        ssl_certificate /etc/nginx/certs/certificate.crt;
        ssl_certificate_key /etc/nginx/certs/certificate.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;

        # Security headers
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
        add_header X-Frame-Options "DENY" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;

        # Request size limits
        client_max_body_size 100M;

        # API endpoints
        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;

            proxy_pass http://api_backend;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Timeouts
            proxy_connect_timeout 30s;
            proxy_send_timeout 300s;
            proxy_read_timeout 300s;
        }

        # Sign endpoint (stricter rate limit)
        location /api/v1/sign {
            limit_req zone=sign_limit burst=10 nodelay;

            proxy_pass http://api_backend;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 30s;
            proxy_send_timeout 600s;
            proxy_read_timeout 600s;
        }

        # Health check
        location /health {
            access_log off;
            proxy_pass http://api_backend;
        }
    }
}
```

---

## Kubernetes Deployment

### Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: digimr
```

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: digimr-config
  namespace: digimr
data:
  appsettings.json: |
    {
      "Logging": {
        "LogLevel": {
          "Default": "Warning",
          "DigitalSignature": "Information"
        }
      },
      "Security": {
        "MaxDocumentSizeBytes": 104857600,
        "MaxBatchDocuments": 100,
        "RequireHttps": true
      }
    }
```

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: digimr-secrets
  namespace: digimr
type: Opaque
stringData:
  database-connection: "Host=postgres;Database=digimr;Username=digimr_user;Password=SECURE_PASSWORD"
  redis-connection: "redis:6379,password=REDIS_PASSWORD"
  api-key: "dmr_live_1234567890abcdef"
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: digimr-api
  namespace: digimr
spec:
  replicas: 3
  selector:
    matchLabels:
      app: digimr-api
  template:
    metadata:
      labels:
        app: digimr-api
    spec:
      containers:
      - name: api
        image: moreum/digimr-api:1.0.0
        ports:
        - containerPort: 80
          name: http
        - containerPort: 443
          name: https
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ConnectionStrings__DefaultConnection
          valueFrom:
            secretKeyRef:
              name: digimr-secrets
              key: database-connection
        - name: ConnectionStrings__Redis
          valueFrom:
            secretKeyRef:
              name: digimr-secrets
              key: redis-connection
        volumeMounts:
        - name: config
          mountPath: /app/appsettings.json
          subPath: appsettings.json
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
      volumes:
      - name: config
        configMap:
          name: digimr-config
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: digimr-api
  namespace: digimr
spec:
  selector:
    app: digimr-api
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443
  type: LoadBalancer
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: digimr-ingress
  namespace: digimr
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: digimr-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: digimr-api
            port:
              number: 80
```

### HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: digimr-api-hpa
  namespace: digimr
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: digimr-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## Windows Server Deployment

### IIS Kurulumu / IIS Setup

1. **IIS ve ASP.NET Core Hosting Bundle yükleyin:**

```powershell
# IIS yükle
Install-WindowsFeature -name Web-Server -IncludeManagementTools

# ASP.NET Core Hosting Bundle yükle
# https://dotnet.microsoft.com/download/dotnet/8.0
Invoke-WebRequest -Uri "https://download.visualstudio.microsoft.com/download/pr/.../dotnet-hosting-8.0-win.exe" -OutFile "dotnet-hosting.exe"
.\dotnet-hosting.exe /install /quiet /norestart

# IIS restart
iisreset
```

2. **Application Pool oluşturun:**

```powershell
Import-Module WebAdministration

New-WebAppPool -Name "DigiMRPool" -Force
Set-ItemProperty IIS:\AppPools\DigiMRPool -name "managedRuntimeVersion" -value ""
Set-ItemProperty IIS:\AppPools\DigiMRPool -name "enable32BitAppOnWin64" -value $false
```

3. **Web sitesi oluşturun:**

```powershell
New-Website -Name "DigiMR API" `
    -Port 443 `
    -PhysicalPath "C:\inetpub\digimr\api" `
    -ApplicationPool "DigiMRPool" `
    -Ssl

# SSL sertifikası bağlayın
$cert = Get-ChildItem -Path Cert:\LocalMachine\My | Where-Object {$_.Subject -eq "CN=api.example.com"}
New-WebBinding -Name "DigiMR API" -Protocol "https" -Port 443 -SslFlags 1
$binding = Get-WebBinding -Name "DigiMR API" -Protocol "https"
$binding.AddSslCertificate($cert.Thumbprint, "My")
```

4. **Firewall kuralları:**

```powershell
New-NetFirewallRule -DisplayName "DigiMR API HTTPS" `
    -Direction Inbound `
    -Action Allow `
    -Protocol TCP `
    -LocalPort 443
```

### Windows Service Olarak Çalıştırma

```powershell
# Servis oluştur
New-Service -Name "DigiMRAPI" `
    -BinaryPathName "C:\Program Files\DigiMR\DigitalSignature.API.exe" `
    -DisplayName "digimr API Service" `
    -StartupType Automatic `
    -Description "Digital Signature API Service"

# Servisi başlat
Start-Service -Name "DigiMRAPI"

# Servis durumu
Get-Service -Name "DigiMRAPI"
```

---

## Linux Server Deployment

### SystemD Service

`/etc/systemd/system/digimr-api.service`:

```ini
[Unit]
Description=digimr API Service
After=network.target

[Service]
Type=notify
User=digimr
Group=digimr
WorkingDirectory=/opt/digimr/api
ExecStart=/opt/digimr/api/DigitalSignature.API
Restart=on-failure
RestartSec=10
KillMode=mixed
KillSignal=SIGINT
TimeoutStopSec=90
SyslogIdentifier=digimr-api

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/digimr/api/logs

# Environment
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false
EnvironmentFile=/etc/digimr/api.env

[Install]
WantedBy=multi-user.target
```

`/etc/digimr/api.env`:

```env
ASPNETCORE_URLS=http://0.0.0.0:5000
ConnectionStrings__DefaultConnection=Host=localhost;Database=digimr;Username=digimr_user;Password=SECURE_PASSWORD
ConnectionStrings__Redis=localhost:6379,password=REDIS_PASSWORD
```

### Kurulum Komutları

```bash
# Kullanıcı oluştur
sudo useradd -r -m -d /opt/digimr -s /bin/bash digimr

# Uygulama dosyalarını kopyala
sudo mkdir -p /opt/digimr/api
sudo cp -r ./publish/* /opt/digimr/api/
sudo chown -R digimr:digimr /opt/digimr

# Servis dosyasını kopyala
sudo cp digimr-api.service /etc/systemd/system/
sudo chmod 644 /etc/systemd/system/digimr-api.service

# Servisi etkinleştir ve başlat
sudo systemctl daemon-reload
sudo systemctl enable digimr-api
sudo systemctl start digimr-api

# Durum kontrol
sudo systemctl status digimr-api

# Logları izle
sudo journalctl -u digimr-api -f
```

---

## Monitoring & Logging

### Prometheus Metrics

`Program.cs`:

```csharp
using Prometheus;

var builder = WebApplication.CreateBuilder(args);

// Prometheus metrics
builder.Services.AddSingleton<IMetricsFactory>(Metrics.DefaultFactory);

var app = builder.Build();

// Metrics endpoint
app.MapMetrics();

// Custom metrics
var signatureCounter = Metrics.CreateCounter("digimr_signatures_total", "Total signatures created", new CounterConfiguration
{
    LabelNames = new[] { "format", "level", "status" }
});

var signatureDuration = Metrics.CreateHistogram("digimr_signature_duration_seconds", "Signature operation duration", new HistogramConfiguration
{
    LabelNames = new[] { "format" }
});
```

### Serilog Configuration

```json
{
  "Serilog": {
    "Using": ["Serilog.Sinks.Console", "Serilog.Sinks.File", "Serilog.Sinks.Elasticsearch"],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft.AspNetCore": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:yyyy-MM-dd HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "/app/logs/digimr-.log",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30,
          "fileSizeLimitBytes": 104857600,
          "rollOnFileSizeLimit": true
        }
      },
      {
        "Name": "Elasticsearch",
        "Args": {
          "nodeUris": "http://elasticsearch:9200",
          "indexFormat": "digimr-logs-{0:yyyy.MM.dd}",
          "autoRegisterTemplate": true
        }
      }
    ],
    "Enrich": ["FromLogContext", "WithMachineName", "WithThreadId"]
  }
}
```

### Health Checks

```csharp
builder.Services.AddHealthChecks()
    .AddNpgSql(connectionString, name: "postgres")
    .AddRedis(redisConnectionString, name: "redis")
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddCheck("pkcs11", () => {
        // Check PKCS#11 library availability
        return File.Exists("/usr/lib/opensc-pkcs11.so")
            ? HealthCheckResult.Healthy()
            : HealthCheckResult.Degraded("PKCS#11 library not found");
    });

app.MapHealthChecks("/health");
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Name == "self"
});
```

---

## Backup & Recovery

### Database Backup (PostgreSQL)

```bash
#!/bin/bash
# /opt/digimr/backup.sh

BACKUP_DIR="/var/backups/digimr"
DATE=$(date +%Y%m%d_%H%M%S)
DB_NAME="digimr"
DB_USER="digimr_user"

mkdir -p $BACKUP_DIR

# Database backup
pg_dump -U $DB_USER -d $DB_NAME -F c -f $BACKUP_DIR/db_$DATE.backup

# Redis backup
redis-cli --rdb $BACKUP_DIR/redis_$DATE.rdb

# Keep last 30 days
find $BACKUP_DIR -name "*.backup" -mtime +30 -delete
find $BACKUP_DIR -name "*.rdb" -mtime +30 -delete

# Upload to S3 (optional)
aws s3 cp $BACKUP_DIR/db_$DATE.backup s3://digimr-backups/
```

### Cron Job

```bash
# Daily backup at 2 AM
0 2 * * * /opt/digimr/backup.sh >> /var/log/digimr-backup.log 2>&1
```

---

## SSL/TLS Yapılandırması

### Let's Encrypt (Certbot)

```bash
# Certbot kurulumu
sudo apt-get install certbot python3-certbot-nginx

# Sertifika alma
sudo certbot --nginx -d api.example.com

# Otomatik yenileme
sudo certbot renew --dry-run
```

### Özel Sertifika / Custom Certificate

```bash
# PFX oluşturma
openssl pkcs12 -export -out certificate.pfx \
    -inkey private.key \
    -in certificate.crt \
    -certfile ca-bundle.crt
```

`appsettings.json`:

```json
{
  "Kestrel": {
    "Endpoints": {
      "Https": {
        "Url": "https://0.0.0.0:443",
        "Certificate": {
          "Path": "/app/certs/certificate.pfx",
          "Password": "YOUR_CERT_PASSWORD"
        }
      }
    }
  }
}
```

---

## Destek / Support

- GitHub: https://github.com/moreum-tech/digimr
- Email: support@moreum.tech
- Enterprise: enterprise@moreum.tech

---

## Lisans / License

Copyright © 2026 Moreum Tech - MIT License
