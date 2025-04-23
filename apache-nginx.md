# README — **Guía de Implementación Avanzada de Servidores Web**  

*Versión 1.0 — 23 abr 2025*  

---

## Tabla de Contenidos
1. [Resumen](#resumen)  
2. [Escenarios Soportados](#escenarios-soportados)  
3. [Arquitecturas de Referencia](#arquitecturas-de-referencia)  
4. [Implementación On-Premise](#implementación-on-premise)  
5. [Implementación en AWS](#implementación-en-aws)  
6. [Tuning de Rendimiento](#tuning-de-rendimiento)  
7. [Endurecimiento y Seguridad](#endurecimiento-y-seguridad)  
8. [Observabilidad & Logging](#observabilidad--logging)  
9. [CI/CD y Automatización](#cicd-y-automatización)  
10. [DR & Backup](#dr--backup)  
11. [Mantenimiento & Operación Continua](#mantenimiento--operación-continua)  
12. [Apéndice A — Snippets](#apéndice-a--snippets)  
13. [Apéndice B — Referencias](#apéndice-b--referencias)  

---

## Resumen
Esta guía describe **cómo desplegar servidores Apache HTTP Server y Nginx** tanto en centros de datos on-premise como en AWS, abarcando:

* Instalación y configuración esencial ✦  
* Ajustes avanzados de seguridad, rendimiento y logging  
* Arquitecturas Cloud nativas (S3 + CloudFront, ALB + EC2, contenedores)  
* Infraestructura como código, CI/CD y monitoreo en producción  

---

## Escenarios Soportados
| Escenario | Uso típico | SLA objetivo | Coste |
|-----------|-----------|--------------|-------|
| **On-Prem Clásico** | Intranets, sistemas legacy | 99.5 % | Hardware propio |
| **AWS Estático** (S3 + CloudFront) | Web Jamstack, landing | 99.9 % | Muy bajo |
| **AWS Dinámico** (ALB + ASG) | API, apps LAMP/Node | 99.95 % | Medio |
| **AWS Containers** (ECS/Fargate) | Micro-servicios, Blue/Green | 99.95 % | Variable |

---

## Arquitecturas de Referencia

```
            ┌──────────────┐          ┌─────────────┐
Internet ──►│ Route 53 DNS │─────────►│ CloudFront  │───► S3 (OAC)
            └──────────────┘          └─────────────┘
                                         ▲    │
                                         │    ▼
                                 AWS WAF WebACL │
                                              Logs ⇢ CloudWatch
```

```
Internet → Route 53 → AWS WAF → ALB (TLS 1.3) →  Auto Scaling Grp
                                              ↘︎ health-checks  ↘︎
                                               RDS/Aurora    ElastiCache
```

---

## Implementación On-Premise

### 1. Requisitos mínimos
* **SO**: Ubuntu 22.04 LTS / RHEL 9 (kernel ≥ 5.15)  
* **CPU**: ≥ 2 vCPU, **RAM**: ≥ 2 GiB  
* Partición `/var` separada; reloj en **UTC**  

### 2. Instalación rápida

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install apache2 -y        # o: sudo apt install nginx -y
sudo systemctl enable --now apache2
```
*Pasos fundamentales tomados del doc fuente*   

### 3. Apache — Configuración base

Archivo `/etc/apache2/sites-available/prod.conf`:

```apache
<VirtualHost *:443>
  ServerName www.ejemplo.com
  DocumentRoot /var/www/ejemplo

  # TLS
  SSLEngine on
  SSLProtocol -all +TLSv1.3 +TLSv1.2
  SSLCipherSuite HIGH:!aNULL:!MD5
  H2Direct on              # HTTP/2

  # Seguridad esencial
  TraceEnable Off          # 
  ServerSignature Off      # 
  Header always set X-Frame-Options "SAMEORIGIN"
  Header always set Content-Security-Policy "default-src 'self'"

  # Rendimiento
  <IfModule mpm_event_module>
       KeepAlive On
       MaxRequestWorkers 200     # (#CPU × 25 aprox.)
       ServerLimit       200
  </IfModule>

  ErrorLog  ${APACHE_LOG_DIR}/ejemplo_error.log
  CustomLog ${APACHE_LOG_DIR}/ejemplo_access.log combined
</VirtualHost>
```

### 4. Nginx — Configuración base

`/etc/nginx/sites-available/prod`:

```nginx
server {
  listen 443 ssl http2;
  server_name www.ejemplo.com;
  root /var/www/ejemplo;
  index index.html index.htm;

  ssl_certificate     /etc/letsencrypt/live/ejemplo/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/ejemplo/privkey.pem;
  ssl_protocols       TLSv1.3 TLSv1.2;
  server_tokens off;          # no versión

  # Rate-limit simple
  limit_req_zone $binary_remote_addr zone=req30:10m rate=30r/m;
  limit_conn_zone $binary_remote_addr zone=addr:10m;

  location / {
        limit_req zone=req30 burst=10 nodelay;
        try_files $uri $uri/ =404;
  }

  log_format json_logs escape=json
       '{ "@timestamp":"$time_iso8601", '
       '"client":"$remote_addr",'
       '"request":"$request",'
       '"status":$status }';
  access_log /var/log/nginx/access.json json_logs;
}
```

### 5. Módulos / extras recomendados

| Servidor | Módulo            | Motivo                                       |
|----------|-------------------|----------------------------------------------|
| Apache   | `mod_http2`       | HTTP/2, prioridades de petición              |
| Apache   | `mod_security`    | WAF embebido, OWASP CRS                      |
| Nginx    | `ngx_brotli`      | Compresión Brotli                            |
| Nginx    | `ngx_http_v2`     | HTTP/2 (core)                                |
| Ambos    | `logrotate`       | Gestión de logs                              |

---

## Implementación en AWS

### 1. Opción A — Sitio estático (CloudFront + S3)

| Componente | Configuración productiva |
|------------|--------------------------|
| **S3 Bucket** | • Bloqueo de público ON<br>• Cifrado SSE-S3<br>• Versión + MFA Delete |
| **CloudFront** | • HTTP/3 + QUIC<br>• Caching TTL = 86400s<br>• Compresión Brotli/Gzip<br>• OAC (Origin Access Control)<br>• Logs → S3 |
| **Seguridad** | AWS WAF rule-groups (OWASP, IP-reputation), Shield Std |
| **Observabilidad** | CloudWatch metrics + CloudWatch Logs Insights |

### 2. Opción B — Aplicación dinámica (ALB + EC2 ASG)

| Servicio | Ajustes claves |
|----------|----------------|
| **ALB** | Política TLS `ELBSecurityPolicy-TLS13-1-2-2021-06` (TLS 1.3) citeturn0search0 |
| **Auto Scaling Group** | Min = 2 instancias (AZ Múltiple), *Rolling Update* |
| **AMI** | CloudInit instala Apache/Nginx + SSM Agent |
| **EC2** | Instancias Graviton2 (t4g.medium) para coste |  
| **UserData (extracto)**| ```bash\napt update && apt -y install nginx\naws s3 cp s3://config-bucket/nginx.conf /etc/nginx/\n``` |
| **Back-ends** | RDS Multi-AZ, ElastiCache Redis, EFS *(si comparte media)* |

### 3. Opción C — Containers (ECS Fargate)

* Defina task definition con **nginx:1.25-alpine**  
* Service → Application Load Balancer (listener 443)  
* **CodePipeline** (Source → Build → Deploy Blue/Green)  

---

## Tuning de Rendimiento

| Área | Apache | Nginx |
|------|--------|-------|
| **Concurrencia** | `MPM event`, `MaxRequestWorkers` = CPU × 25 | `worker_processes auto; worker_connections 4096;` |
| **Keep-Alive** | `KeepAliveTimeout 2` | `keepalive_timeout 10s;` |
| **Micro-caching** | `CacheQuickHandler On` + `CacheEnable` | `proxy_cache_path … levels=1:2 keys_zone=micro:10m max_size=1g inactive=60m;` |
| **HTTP/3** | (experimental con `mod_h2`) | a partir de Nginx 1.25 + `quic on;` |
| **Compression** | `mod_brotli` + `mod_deflate` | `brotli on; gzip on;` |

---

## Endurecimiento y Seguridad

1. **TLS 1.2 / 1.3 únicamente**  
2. **TraceEnable Off** + **ServerSignature Off** en Apache   
3. **server_tokens off** en Nginx  
4. **CSP**, **X-Frame-Options**, **HSTS (max-age ≥ 1 año)**  
5. **Fail2Ban** para bloqueos de fuerza bruta (ssh/http)  
6. **Escáner**: Amazon Inspector / OpenVAS antes de PRD  
7. **Least-Privilege IAM / sudo**  

---

## Observabilidad & Logging

### Métricas
* **Prometheus exporters** (`apache_exporter`, `nginx_exporter`)  
* **CloudWatch Agent**: CPU, memoria, disco  
* **Custom metrics**: p95 latencia, tamaño de obj cache hit  

### Logs
* JSON logs + **Fluent Bit** → OpenSearch / CloudWatch Logs  
* **ALB** & **CloudFront** access-logs en S3 (retención ≥ 90 días)  

### Trazas
* **AWS X-Ray** para apps (SDK / Sidecar)  
* Correlación con cabecera `X-Amzn-Trace-Id` → logs  

---

## CI/CD y Automatización

| Fase | Herramientas | Puntos clave |
|------|--------------|-------------|
| **Build** | GitHub Actions → Docker | `docker build --platform linux/arm64 …` |
| **Test** | OWASP ZAP, k6 | En PR → pipeline bloqueante |
| **Deploy** | Terraform Cloud + Atlantis | PR plan output, *terraform apply* vía merge |
| **Release** | CodeDeploy Blue/Green (ASG) | Weighted target groups (10 % → 50 % → 100 %) |

---

## DR & Backup

* **Snapshots** EBS/RDS diarios (retención 35 d)  
* **Cross-Region Replication** S3 CRR + Route 53 fail-over  
* **RTO** ≤ 60 min, **RPO** ≤ 15 min  
* Runbooks en **AWS SSM Documents**  

---

## Mantenimiento & Operación Continua

| Tarea | Comando / Acción | Frecuencia |
|-------|------------------|------------|
| Verificar sintaxis | `apachectl configtest` / `nginx -t` | Cada cambio |
| Reinicio elegante | `apachectl -k graceful` | Mensual |
| Parches SO | `unattended-upgrades` / SSM Patch Manager | Semanal |
| Rotación de logs | `logrotate -f /etc/logrotate.d/*` | Diaria |
| Auditoría TLS | ssllabs.com scan | Semestral |

---

## Apéndice A — Snippets

*### Terraform — CloudFront + S3 OAC*
```hcl
resource "aws_s3_bucket" "site" {
  bucket = "ejemplo-site"
  force_destroy = true
  versioning { enabled = true }
  lifecycle_rule {
    id      = "retencion"
    enabled = true
    noncurrent_version_expiration { days = 30 }
  }
}

resource "aws_cloudfront_origin_access_control" "oac" {
  name                              = "site-oac"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
  origin_access_control_origin_type = "s3"
}

resource "aws_cloudfront_distribution" "cdn" {
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  origin {
    domain_name              = aws_s3_bucket.site.bucket_regional_domain_name
    origin_access_control_id = aws_cloudfront_origin_access_control.oac.id
  }
  default_cache_behavior {
    allowed_methods  = ["GET","HEAD"]
    cached_methods   = ["GET","HEAD"]
    viewer_protocol_policy = "redirect-to-https"
    compress = true
  }
}
```

*### Apache — ModSecurity activación rápida*
```bash
sudo apt install libapache2-mod-security2
sudo cp /usr/share/modsecurity-crs/crs-setup.conf.example /etc/modsecurity/
sudo a2enmod security2
sudo systemctl restart apache2
```

---

## Apéndice B — Referencias
* Instalación Apache & directivas (`TraceEnable`, `ServerSignature`) — Fuente PDF del ejercicio   
* AWS TLS 1.3 policies — AWS Docs citeturn0search0  
* OWASP Core Rule Set — <https://coreruleset.org/>  
* Nginx QUIC docs — <https://www.nginx.com/blog/introducing-technical-preview-nginx-support-for-http3/>  

---

**Fin del documento**. ¡Feliz despliegue!
