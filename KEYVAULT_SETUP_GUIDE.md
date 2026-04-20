# Guia de Configuração Azure Key Vault - Wedigitek

## Aplicações Preparadas: 15

- we-api
- billing
- data-api
- devices-api
- notifications-service
- payments
- quality-control
- reports
- surveys
- we-admin
- we-api-integrations-service
- we-loyalty-service
- we-wallet
- wefood-delivery-notifier
- wefood-realtime-server

## Arquivos Sensíveis Necessários: 9

### we-api - we-api-jwt-rs256-key

- **Nome no Key Vault**: `we-api-jwt-rs256-key`
- **Caminho na VM**: `certs/jwtRS256.key`
- **Montado em**: `/we-api/certs/jwtRS256.key`
- **Comando**:
```bash
az keyvault secret set --vault-name kv-wedigitek-prd \\
  --name we-api-jwt-rs256-key \\
  --file certs/jwtRS256.key \\
  --encoding utf-8
```

### we-api - we-api-jwt-rs256-key-pub

- **Nome no Key Vault**: `we-api-jwt-rs256-key-pub`
- **Caminho na VM**: `certs/jwtRS256.key.pub`
- **Montado em**: `/we-api/certs/jwtRS256.key.pub`
- **Comando**:
```bash
az keyvault secret set --vault-name kv-wedigitek-prd \\
  --name we-api-jwt-rs256-key-pub \\
  --file certs/jwtRS256.key.pub \\
  --encoding utf-8
```

### we-api - we-api-firebase-menugrsa-json

- **Nome no Key Vault**: `we-api-firebase-menugrsa-json`
- **Caminho na VM**: `certs/firebase/menugrsa.json`
- **Montado em**: `/we-api/certs/menugrsa.json`
- **Comando**:
```bash
az keyvault secret set --vault-name kv-wedigitek-prd \\
  --name we-api-firebase-menugrsa-json \\
  --file certs/firebase/menugrsa.json \\
  --encoding utf-8
```

### we-api - we-api-firebase-deli-2f288-json

- **Nome no Key Vault**: `we-api-firebase-deli-2f288-json`
- **Caminho na VM**: `certs/firebase/deli-2f288.json`
- **Montado em**: `/we-api/certs/deli-2f288.json`
- **Comando**:
```bash
az keyvault secret set --vault-name kv-wedigitek-prd \\
  --name we-api-firebase-deli-2f288-json \\
  --file certs/firebase/deli-2f288.json \\
  --encoding utf-8
```

### we-api - we-api-firebase-sodexo-54882-json

- **Nome no Key Vault**: `we-api-firebase-sodexo-54882-json`
- **Caminho na VM**: `certs/firebase/sodexo-54882.json`
- **Montado em**: `/we-api/certs/sodexo-54882.json`
- **Comando**:
```bash
az keyvault secret set --vault-name kv-wedigitek-prd \\
  --name we-api-firebase-sodexo-54882-json \\
  --file certs/firebase/sodexo-54882.json \\
  --encoding utf-8
```

### we-api - we-api-ticket-public-cert

- **Nome no Key Vault**: `we-api-ticket-public-cert`
- **Caminho na VM**: `certs/ticket-public-cert.cer`
- **Montado em**: `/we-api/certs/ticket-public-cert.cer`
- **Comando**:
```bash
az keyvault secret set --vault-name kv-wedigitek-prd \\
  --name we-api-ticket-public-cert \\
  --file certs/ticket-public-cert.cer \\
  --encoding utf-8
```

### we-api - we-api-api-keys-json

- **Nome no Key Vault**: `we-api-api-keys-json`
- **Caminho na VM**: `certs/api-keys.json`
- **Montado em**: `/we-api/certs/api-keys.json`
- **Comando**:
```bash
az keyvault secret set --vault-name kv-wedigitek-prd \\
  --name we-api-api-keys-json \\
  --file certs/api-keys.json \\
  --encoding utf-8
```

### we-api - we-api-socialauth-apple-key-p8

- **Nome no Key Vault**: `we-api-socialauth-apple-key-p8`
- **Caminho na VM**: `certs/AuthKey_9984Z5GVCJ.p8`
- **Montado em**: `/we-api/certs/AuthKey_9984Z5GVCJ.p8`
- **Comando**:
```bash
az keyvault secret set --vault-name kv-wedigitek-prd \\
  --name we-api-socialauth-apple-key-p8 \\
  --file certs/AuthKey_9984Z5GVCJ.p8 \\
  --encoding utf-8
```

### billing - billing-at-certificate-pfx

- **Nome no Key Vault**: `billing-at-certificate-pfx`
- **Caminho na VM**: `certificados/at-certificate.pfx`
- **Montado em**: `/billing/certs/at-certificate.pfx`
- **Comando**:
```bash
az keyvault secret set --vault-name kv-wedigitek-prd \\
  --name billing-at-certificate-pfx \\
  --file certificados/at-certificate.pfx \\
  --encoding utf-8
```

