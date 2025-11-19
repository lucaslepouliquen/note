# Troubleshooting DNS et Configuration Application Gateway Azure

## Contexte

Ce document résume les problèmes rencontrés lors de la mise en place d'un Application Gateway Azure pointant vers un service Kubernetes (AKS) et les solutions apportées.

---

## 1. Problème Initial : Erreur DNS

### Symptômes

Lors de l'accès à `https://midpoint-poc.promod.net`, deux erreurs sont apparues :

1. **Erreur PowerShell (curl)** :
```
Le nom distant n'a pas pu être résolu: 'midpoint-poc-mag-dev.aks-internal'
```

2. **Erreur Navigateur** :
```
DNS_PROBE_FINISHED_NXDOMAIN
Désolé, impossible d'accéder à cette page.
```

### Diagnostic

Ces erreurs indiquent deux problèmes distincts :

#### Problème A : Résolution DNS publique
- Le domaine `midpoint-poc.promod.net` n'est **pas résolvable publiquement**
- Il n'existe pas d'enregistrement DNS pour ce domaine dans la zone DNS publique
- Les clients (navigateurs) ne peuvent donc pas atteindre l'Application Gateway

#### Problème B : Résolution DNS interne
- L'Application Gateway essaie de résoudre `midpoint-poc-mag-dev.aks-internal`
- Ce FQDN n'existe pas ou n'est pas accessible depuis le subnet de l'Application Gateway
- Le backend pool de l'AppGW ne peut pas router le trafic vers le cluster AKS

---

## 2. Architecture de la Solution

```
Internet
    │
    ▼
[DNS Public: midpoint-poc.promod.net]
    │ (résolution vers IP publique)
    ▼
[Application Gateway]
    │ (subnet VNet)
    │
    ├─ Frontend: IP publique + Listener HTTPS
    │
    ├─ Routing Rule: PRIVATE-Front-Midpoint
    │
    ├─ Rewrite Rules: Modification des headers
    │
    └─ Backend Pool: PRIVATE-Aks-Mag
        │ (résolution DNS interne)
        ▼
    [AKS Cluster]
        └─ Service: midpoint-poc-mag-dev
           └─ Pods
```

---

## 3. Solutions Mises en Place

### Solution 1 : Configurer le DNS Public

Le domaine doit pointer vers l'IP publique de l'Application Gateway.

#### Si DNS géré dans Azure DNS

```bash
# Récupérer l'IP publique de l'Application Gateway
az network public-ip show \
  --name <nom-ip-publique-appgw> \
  --resource-group <resource-group> \
  --query ipAddress -o tsv

# Créer l'enregistrement DNS
az network dns record-set a add-record \
  --resource-group <resource-group> \
  --zone-name promod.net \
  --record-set-name midpoint-poc \
  --ipv4-address <IP-PUBLIQUE-APPGW>
```

#### Si DNS géré en externe (OVH, CloudFlare, etc.)

Créer un enregistrement **A** :
- **Nom** : `midpoint-poc`
- **Type** : `A`
- **Valeur** : `<IP publique de l'Application Gateway>`
- **TTL** : `300` (5 minutes) ou selon vos besoins

**Vérification** :
```bash
nslookup midpoint-poc.promod.net
# Doit retourner l'IP publique de l'AppGW

dig midpoint-poc.promod.net
# Doit retourner un enregistrement A
```

---

### Solution 2 : Configurer le Backend Pool Correctement

Le backend pool doit pouvoir résoudre et atteindre le service Kubernetes.

#### Option A : Utiliser le FQDN Kubernetes interne

Si l'Application Gateway est dans le même VNet que l'AKS (ou en peering) :

**Terraform - Backend Address Pool**
```hcl
backend_address_pool {
  name  = "PRIVATE-Aks-Mag"
  fqdns = ["midpoint-poc-mag-dev.default.svc.cluster.local"]
}
```

> **Note** : Remplacer `default` par le namespace réel du service

**Vérifier le FQDN du service** :
```bash
kubectl get svc midpoint-poc-mag-dev -n <namespace>
kubectl get svc midpoint-poc-mag-dev -n <namespace> -o jsonpath='{.metadata.name}.{.metadata.namespace}.svc.cluster.local'
```

#### Option B : Utiliser l'IP du Service (Internal LoadBalancer)

Si le service Kubernetes expose une IP interne via un LoadBalancer :

```bash
# Récupérer l'IP interne du service
kubectl get svc midpoint-poc-mag-dev -n <namespace> -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

**Terraform - Backend Address Pool**
```hcl
backend_address_pool {
  name         = "PRIVATE-Aks-Mag"
  ip_addresses = ["10.x.x.x"]  # IP du service K8s
}
```

#### Option C : Utiliser un Ingress Controller (Recommandé)

Déployer un Ingress Controller (NGINX, Traefik) dans AKS avec un service de type LoadBalancer interne :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: http
  - port: 443
    targetPort: https
  selector:
    app: ingress-nginx
```

Puis pointer le backend pool vers l'IP du LoadBalancer de l'Ingress Controller.

---

### Solution 3 : Configurer les Backend HTTP Settings

Le backend HTTP settings définit comment l'Application Gateway communique avec le backend.

**Terraform Configuration**
```hcl
backend_http_settings {
  name                  = "PRIVATE-Front-Midpoint"
  cookie_based_affinity = "Disabled"
  port                  = 80  # ou 443 si HTTPS
  protocol              = "Http"  # ou "Https"
  request_timeout       = 30
  
  # IMPORTANT : Définir le Host header à envoyer au backend
  pick_host_name_from_backend_address = false
  host_name = "midpoint-poc-mag-dev.default.svc.cluster.local"
  
  # Si votre backend nécessite un probe personnalisé
  probe_name = "midpoint-health-probe"
}

# Health Probe (optionnel mais recommandé)
probe {
  name                = "midpoint-health-probe"
  protocol            = "Http"
  path                = "/health"  # Endpoint de health check
  interval            = 30
  timeout             = 30
  unhealthy_threshold = 3
  
  pick_host_name_from_backend_http_settings = true
  
  match {
    status_code = ["200-399"]
  }
}
```

**Points importants** :

1. **`pick_host_name_from_backend_address`**
   - `false` : Utilise `host_name` défini manuellement
   - `true` : Utilise le FQDN/IP du backend pool comme Host header

2. **`host_name`**
   - Doit correspondre au hostname que le backend s'attend à recevoir
   - Pour Kubernetes : généralement `service-name.namespace.svc.cluster.local`

3. **Health Probe**
   - Permet à l'AppGW de vérifier que le backend est disponible
   - Évite d'envoyer du trafic vers des backends down

---

### Solution 4 : Configurer les Rewrite Rules

Les rewrite rules permettent de modifier les headers HTTP entre le client et le backend.

#### Configuration Actuelle (à corriger)

```hcl
# ❌ INCORRECT - Utilise response_header_configuration
rewrite_rule_set {
  name = "Midpoint"
  
  rewrite_rule {
    name          = "Midpoint"
    rule_sequence = 100
    
    response_header_configuration {  # ❌ ERREUR
      header_name  = "X-Forwarded-Proto"
      header_value = "https"
    }
    
    response_header_configuration {  # ❌ ERREUR
      header_name  = "X-Forwarded-Host"
      header_value = "midpoint-poc.promod.net"
    }
  }
}
```

#### Configuration Correcte

```hcl
# ✅ CORRECT - Utilise request_header_configuration
rewrite_rule_set {
  name = "Midpoint"
  
  rewrite_rule {
    name          = "Midpoint"
    rule_sequence = 100
    
    # Headers envoyés AU BACKEND (request)
    request_header_configuration {
      header_name  = "X-Forwarded-Proto"
      header_value = "https"
    }
    
    request_header_configuration {
      header_name  = "X-Forwarded-Host"
      header_value = "midpoint-poc.promod.net"
    }
    
    # Optionnel : Préserver l'host original pour les logs
    request_header_configuration {
      header_name  = "X-Original-Host"
      header_value = "{var_host}"
    }
  }
}
```

#### Différence Importante

| Configuration | Usage | Direction |
|--------------|-------|-----------|
| `request_header_configuration` | Modifie les headers **envoyés au backend** | Client → AppGW → **Backend** |
| `response_header_configuration` | Modifie les headers **renvoyés au client** | Backend → AppGW → **Client** |

**Pour `X-Forwarded-*` headers** : Toujours utiliser `request_header_configuration` car ces headers informent le backend sur la requête originale du client.

#### Association avec la Routing Rule

```hcl
request_routing_rule {
  name                       = "PRIVATE-Front-Midpoint"
  rule_type                  = "Basic"
  http_listener_name         = "PRIVATE-Front-Midpoint"
  backend_address_pool_name  = "PRIVATE-Aks-Mag"
  backend_http_settings_name = "PRIVATE-Front-Midpoint"
  priority                   = 490
  rewrite_rule_set_name      = "Midpoint"  # ✅ Association ici
}
```

---

## 4. Configuration Complète en Terraform

Voici une configuration complète et fonctionnelle :

```hcl
resource "azurerm_application_gateway" "main" {
  name                = "appgw-midpoint"
  resource_group_name = var.resource_group_name
  location            = var.location

  sku {
    name     = "Standard_v2"
    tier     = "Standard_v2"
    capacity = 2
  }

  gateway_ip_configuration {
    name      = "appgw-ip-config"
    subnet_id = var.appgw_subnet_id
  }

  frontend_port {
    name = "frontend-port-https"
    port = 443
  }

  frontend_port {
    name = "frontend-port-http"
    port = 80
  }

  frontend_ip_configuration {
    name                 = "frontend-ip-public"
    public_ip_address_id = var.public_ip_id
  }

  # Backend Pool
  backend_address_pool {
    name  = "PRIVATE-Aks-Mag"
    fqdns = ["midpoint-poc-mag-dev.default.svc.cluster.local"]
    # OU
    # ip_addresses = ["10.x.x.x"]
  }

  # Backend HTTP Settings
  backend_http_settings {
    name                  = "PRIVATE-Front-Midpoint"
    cookie_based_affinity = "Disabled"
    port                  = 80
    protocol              = "Http"
    request_timeout       = 30
    
    pick_host_name_from_backend_address = false
    host_name = "midpoint-poc-mag-dev.default.svc.cluster.local"
    
    probe_name = "midpoint-health-probe"
  }

  # Health Probe
  probe {
    name                = "midpoint-health-probe"
    protocol            = "Http"
    path                = "/"
    interval            = 30
    timeout             = 30
    unhealthy_threshold = 3
    
    pick_host_name_from_backend_http_settings = true
    
    match {
      status_code = ["200-399"]
    }
  }

  # HTTP Listener
  http_listener {
    name                           = "PRIVATE-Front-Midpoint"
    frontend_ip_configuration_name = "frontend-ip-public"
    frontend_port_name             = "frontend-port-https"
    protocol                       = "Https"
    ssl_certificate_name           = "midpoint-ssl-cert"
    host_name                      = "midpoint-poc.promod.net"
  }

  # SSL Certificate
  ssl_certificate {
    name     = "midpoint-ssl-cert"
    data     = filebase64("path/to/cert.pfx")
    password = var.ssl_cert_password
  }

  # Rewrite Rules
  rewrite_rule_set {
    name = "Midpoint"
    
    rewrite_rule {
      name          = "Midpoint"
      rule_sequence = 100
      
      request_header_configuration {
        header_name  = "X-Forwarded-Proto"
        header_value = "https"
      }
      
      request_header_configuration {
        header_name  = "X-Forwarded-Host"
        header_value = "midpoint-poc.promod.net"
      }
      
      request_header_configuration {
        header_name  = "X-Original-Host"
        header_value = "{var_host}"
      }
    }
  }

  # Routing Rule
  request_routing_rule {
    name                       = "PRIVATE-Front-Midpoint"
    rule_type                  = "Basic"
    http_listener_name         = "PRIVATE-Front-Midpoint"
    backend_address_pool_name  = "PRIVATE-Aks-Mag"
    backend_http_settings_name = "PRIVATE-Front-Midpoint"
    priority                   = 490
    rewrite_rule_set_name      = "Midpoint"
  }
}
```

---

## 5. Commandes de Diagnostic

### Vérifier la résolution DNS publique

```bash
# Depuis votre machine
nslookup midpoint-poc.promod.net
dig midpoint-poc.promod.net

# Vérifier la propagation DNS mondiale
# https://www.whatsmydns.net/#A/midpoint-poc.promod.net
```

### Vérifier le service Kubernetes

```bash
# Lister les services
kubectl get svc -n <namespace>

# Détails du service
kubectl describe svc midpoint-poc-mag-dev -n <namespace>

# Tester depuis un pod dans le cluster
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- sh
curl http://midpoint-poc-mag-dev.default.svc.cluster.local
```

### Vérifier l'Application Gateway

```bash
# Backend Health
az network application-gateway show-backend-health \
  --name appgw-midpoint \
  --resource-group <rg> \
  --output table

# Configuration des backend pools
az network application-gateway address-pool show \
  --gateway-name appgw-midpoint \
  --resource-group <rg> \
  --name PRIVATE-Aks-Mag

# HTTP Settings
az network application-gateway http-settings show \
  --gateway-name appgw-midpoint \
  --resource-group <rg> \
  --name PRIVATE-Front-Midpoint

# Rewrite Rules
az network application-gateway rewrite-rule show \
  --gateway-name appgw-midpoint \
  --resource-group <rg> \
  --rule-set-name Midpoint \
  --rule-name Midpoint
```

### Tester la connectivité réseau

```bash
# Depuis l'Application Gateway subnet (via une VM dans ce subnet)
# Test de résolution DNS
nslookup midpoint-poc-mag-dev.default.svc.cluster.local

# Test de connectivité
curl -H "Host: midpoint-poc-mag-dev.default.svc.cluster.local" http://10.x.x.x
```

---

## 6. Variables Terraform Disponibles pour Rewrite Rules

Lors de la configuration de rewrite rules, vous pouvez utiliser ces variables :

| Variable | Description | Exemple |
|----------|-------------|---------|
| `{var_uri_path}` | Chemin de l'URI | `/api/users` |
| `{var_uri_path_1}` | Premier groupe de capture regex | Si pattern `^/api/(.*)` → `users` |
| `{var_uri_path_2}` | Deuxième groupe de capture | Si pattern `^/api/([^/]+)/(.*)` |
| `{var_query_string}` | Query string complète | `?page=1&size=10` |
| `{var_host}` | Hostname de la requête | `midpoint-poc.promod.net` |
| `{var_client_ip}` | IP du client | `203.0.113.42` |
| `{var_server_port}` | Port du serveur | `443` |
| `{var_request_id}` | ID unique de la requête | GUID généré |
| `{http_req_<Header>}` | Header de requête spécifique | `{http_req_User-Agent}` |
| `{http_resp_<Header>}` | Header de réponse spécifique | `{http_resp_Content-Type}` |

---

## 7. Checklist de Vérification

### ✅ DNS Public

- [ ] Enregistrement A créé pour `midpoint-poc.promod.net`
- [ ] Enregistrement pointe vers l'IP publique de l'AppGW
- [ ] DNS propagé (vérifier avec nslookup/dig)
- [ ] TTL approprié configuré

### ✅ Application Gateway

- [ ] Backend pool configuré avec le bon FQDN ou IP
- [ ] Backend HTTP settings avec le bon `host_name`
- [ ] Health probe configuré et fonctionnel
- [ ] Listener HTTPS avec le bon hostname
- [ ] Certificat SSL valide configuré
- [ ] Rewrite rules utilisent `request_header_configuration` (pas response)
- [ ] Routing rule associe tous les composants

### ✅ Réseau Azure

- [ ] Subnet de l'AppGW peut communiquer avec le VNet AKS
- [ ] NSG autorise le trafic nécessaire
- [ ] Pas de firewall bloquant entre AppGW et AKS
- [ ] Private DNS Zone (si utilisée) correctement configurée

### ✅ Kubernetes

- [ ] Service existe et est accessible
- [ ] Service expose le bon port
- [ ] Pods backend sont healthy
- [ ] Ingress controller (si utilisé) fonctionne
- [ ] Network policies n'empêchent pas la communication

---

## 8. Bonnes Pratiques

### Sécurité

1. **Utiliser HTTPS uniquement**
   ```hcl
   # Redirection HTTP → HTTPS
   request_routing_rule {
     name               = "http-to-https-redirect"
     rule_type          = "Basic"
     http_listener_name = "http-listener"
     redirect_configuration_name = "https-redirect"
     priority           = 500
   }
   
   redirect_configuration {
     name                 = "https-redirect"
     redirect_type        = "Permanent"
     target_listener_name = "https-listener"
     include_path         = true
     include_query_string = true
   }
   ```

2. **Ajouter des headers de sécurité**
   ```hcl
   rewrite_rule {
     name          = "security-headers"
     rule_sequence = 50
     
     response_header_configuration {
       header_name  = "Strict-Transport-Security"
       header_value = "max-age=31536000; includeSubDomains"
     }
     
     response_header_configuration {
       header_name  = "X-Content-Type-Options"
       header_value = "nosniff"
     }
     
     response_header_configuration {
       header_name  = "X-Frame-Options"
       header_value = "DENY"
     }
   }
   ```

3. **WAF (Web Application Firewall)**
   ```hcl
   sku {
     name     = "WAF_v2"
     tier     = "WAF_v2"
     capacity = 2
   }
   
   waf_configuration {
     enabled          = true
     firewall_mode    = "Prevention"
     rule_set_type    = "OWASP"
     rule_set_version = "3.2"
   }
   ```

### Performance

1. **Connection Draining**
   ```hcl
   backend_http_settings {
     # ...
     connection_draining {
       enabled           = true
       drain_timeout_sec = 60
     }
   }
   ```

2. **Autoscaling**
   ```hcl
   autoscale_configuration {
     min_capacity = 2
     max_capacity = 10
   }
   ```

3. **Cookie-based affinity** (si nécessaire)
   ```hcl
   backend_http_settings {
     # ...
     cookie_based_affinity = "Enabled"
     affinity_cookie_name  = "ApplicationGatewayAffinity"
   }
   ```

### Monitoring

1. **Activer les logs de diagnostic**
   ```hcl
   resource "azurerm_monitor_diagnostic_setting" "appgw" {
     name                       = "appgw-diagnostics"
     target_resource_id         = azurerm_application_gateway.main.id
     log_analytics_workspace_id = var.log_analytics_workspace_id
     
     enabled_log {
       category = "ApplicationGatewayAccessLog"
     }
     
     enabled_log {
       category = "ApplicationGatewayPerformanceLog"
     }
     
     enabled_log {
       category = "ApplicationGatewayFirewallLog"
     }
     
     metric {
       category = "AllMetrics"
     }
   }
   ```

2. **Configurer des alertes**
   ```hcl
   resource "azurerm_monitor_metric_alert" "unhealthy_backend" {
     name                = "appgw-unhealthy-backend"
     resource_group_name = var.resource_group_name
     scopes              = [azurerm_application_gateway.main.id]
     description         = "Alert when backend is unhealthy"
     
     criteria {
       metric_namespace = "Microsoft.Network/applicationGateways"
       metric_name      = "UnhealthyHostCount"
       aggregation      = "Average"
       operator         = "GreaterThan"
       threshold        = 0
     }
     
     action {
       action_group_id = var.action_group_id
     }
   }
   ```

---

## 9. Dépannage Avancé

### Problème : Backend toujours Unhealthy

**Causes possibles** :
1. Health probe mal configuré (mauvais path, port, protocole)
2. Backend ne répond pas aux health checks
3. Timeout trop court
4. Certificat SSL invalide (si HTTPS)

**Solution** :
```hcl
probe {
  name                = "detailed-probe"
  protocol            = "Http"
  path                = "/health"  # Vérifier que ce endpoint existe
  interval            = 30
  timeout             = 30
  unhealthy_threshold = 3
  
  pick_host_name_from_backend_http_settings = true
  
  match {
    status_code = ["200-399"]
    body        = "healthy"  # Optionnel : vérifier le contenu
  }
}
```

### Problème : 502 Bad Gateway

**Causes possibles** :
1. Backend inaccessible depuis l'AppGW
2. Backend retourne une erreur 5xx
3. Timeout dépassé
4. Host header incorrect

**Diagnostic** :
```bash
# Vérifier les logs
az monitor diagnostic-settings show \
  --resource <appgw-resource-id>

# Vérifier le backend health
az network application-gateway show-backend-health \
  --name <appgw-name> \
  --resource-group <rg> \
  --output table
```

### Problème : SSL/TLS Errors

**Causes possibles** :
1. Certificat expiré ou invalide
2. Mauvaise chaîne de certificats
3. Certificat ne correspond pas au hostname

**Vérification** :
```bash
# Tester le certificat
openssl s_client -connect midpoint-poc.promod.net:443 -servername midpoint-poc.promod.net

# Vérifier la date d'expiration
openssl x509 -in cert.pem -noout -dates
```

---

## 10. Ressources Complémentaires

### Documentation Officielle

- [Azure Application Gateway - Documentation](https://docs.microsoft.com/azure/application-gateway/)
- [Rewrite HTTP headers and URL](https://docs.microsoft.com/azure/application-gateway/rewrite-http-headers-url)
- [Backend health diagnostics](https://docs.microsoft.com/azure/application-gateway/application-gateway-backend-health-troubleshooting)
- [Terraform azurerm_application_gateway](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/application_gateway)

### Outils

- [DNS Checker](https://www.whatsmydns.net/) - Vérifier la propagation DNS
- [SSL Labs](https://www.ssllabs.com/ssltest/) - Tester la configuration SSL
- [Azure Network Watcher](https://azure.microsoft.com/services/network-watcher/) - Diagnostics réseau avancés

---

## Résumé des Commandes Utiles

```bash
# DNS
nslookup midpoint-poc.promod.net
dig midpoint-poc.promod.net +short

# Kubernetes
kubectl get svc -n <namespace>
kubectl describe svc <service-name> -n <namespace>
kubectl get endpoints <service-name> -n <namespace>

# Application Gateway
az network application-gateway show-backend-health \
  --name <appgw> --resource-group <rg>

az network application-gateway list-request-headers \
  --name <appgw> --resource-group <rg>

# Terraform
terraform plan
terraform apply
terraform show

# Test de connectivité
curl -v https://midpoint-poc.promod.net
curl -H "Host: midpoint-poc.promod.net" http://<appgw-ip>
```

---

**Date de création** : 19 novembre 2024  
**Auteur** : Lucas - DevOps Engineer  
**Contexte** : Troubleshooting infrastructure Promod - AKS + Application Gateway
