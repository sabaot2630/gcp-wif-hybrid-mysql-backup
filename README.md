# Respaldo Automatizado Zero-Trust de Bases de Datos On-Premise hacia Google Cloud Storage (sin Llaves JSON Estáticas)

![GCP](https://img.shields.io/badge/Google_Cloud-Workload_Identity_Federation-4285F4?style=for-the-badge&logo=google-cloud&logoColor=white)
![Security](https://img.shields.io/badge/Security-Zero--Trust_%7C_No_Static_Keys-1B365D?style=for-the-badge)
![Infrastructure](https://img.shields.io/badge/On--Premise-Harvester_HCI_v1.7.1-2C5282?style=for-the-badge)
![FinOps](https://img.shields.io/badge/FinOps_Cost-$0.00_Always_Free-2E7D32?style=for-the-badge)

## 1.Problema de Ingeniería

Las organizaciones que operan infraestructura híbrida enfrentan un reto crítico al integrar cargas de trabajo locales con la nube pública: distribuir archivos de credenciales estáticas (llaves JSON de Service Accounts) en servidores on-premise introduce un riesgo elevado de exfiltración y compromete el cumplimiento de auditorías de seguridad. Adicionalmente, muchos entornos remotos operan detrás de enlaces **CGNAT** sin direcciones IP públicas fijas ni puertos entrantes abiertos.

Este proyecto resuelve ambos desafíos mediante una arquitectura híbrida **Zero-Trust** que extrae volcados comprimidos de una base de datos aislada en **Harvester HCI** y los deposita en **Google Cloud Storage** utilizando **Workload Identity Federation (OIDC/STS)** en modo `executable`, generando tokens efímeros bajo demanda y manteniendo un costo operativo de **$0.00 USD**.

---

## 2. Arquitectura Híbrida

```text
+-----------------------------------------------------------------------------------+
|                        INFRAESTRUCTURA ON-PREMISE (HARVESTER HCI)                 |
|                                                                                   |
|  +-----------------------------+          +------------------------------------+  |
|  |   vm-db-private (VLAN Priv) |          |     vm-app-01 (Bridge / Worker)    |  |
|  |         10.0.2.30           |          |             10.0.2.20              |  |
|  |                             |  MySQL   |                                    |  |
|  |  - MySQL 8.0                |<---------|  - systemd timer (02:00 UTC)       |  |
|  |  - Sin salida a Internet    |  Puerto  |  - mysqldump (--no-tablespaces)    |  |
|  |  - Usuario backup_user      |   3306   |  - get_wif_token.py (JWT 5 min)    |  |
|  |    restringido a 10.0.2.20  |          |  - private.pem (RSA-2048 local)    |  |
|  +-----------------------------+          +------------------+-----------------+  |
+--------------------------------------------------------------|--------------------+
                                                               |
                                     Tráfico HTTPS Saliente    | (1) Intercambio JWT -> STS
                                     (Compatible con CGNAT)    | (2) Impersonación IAM
                                                               v
+-----------------------------------------------------------------------------------+
|                          GOOGLE CLOUD PLATFORM (GCP)                              |
|                                                                                   |
|  +-----------------------------------+    +------------------------------------+  |
|  |   Security Token Service (STS)    |    |     IAM Credentials & Service      |  |
|  |   & Workload Identity Pool        |--->|     Account (sa-harvester-app)     |  |
|  |                                   |    |                                    |  |
|  |  - Valida firma vs jwks.json      |    |  - Emite OAuth2 Token (1 hr)       |  |
|  |  - Condición: hostname==vm-app-01 |    |  - Rol: roles/storage.objectAdmin  |  |
|  +-----------------------------------+    +------------------+-----------------+  |
|                                                              |                    |
|                                                              v (3) Upload TLS     |
|                                           +------------------------------------+  |
|                                           |      Cloud Storage Bucket          |  |
|                                           |      (Región: us-central1)         |  |
|                                           |                                    |  |
|                                           |  - Uniform Bucket-Level Access     |  |
|                                           |  - Soft Delete (7 días)            |  |
|                                           |  - Lifecycle Rule: Delete > 7 días |  |
|                                           +------------------------------------+  |
+-----------------------------------------------------------------------------------+
