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

3. Controles de Seguridad y Decisiones de DiseñoCapa / DominioControl ImplementadoImpacto en Seguridad y FinOpsAutenticación CloudWorkload Identity Federation (OIDC) con JWKS público cargado en GCP y modo executable.Elimina llaves JSON estáticas; tokens OIDC caducan en 5 min y se renuevan al vuelo.Control de AdmisiónAttribute Condition: assertion.hostname == 'vm-app-01' y audiencia restringida.Impide que otros nodos emitan tokens válidos incluso dentro del mismo Pool.Menor Privilegio IAMRol roles/storage.objectAdmin vinculado únicamente a nivel de Bucket (no de proyecto).Acota el radio de impacto (blast radius) exclusivamente al repositorio de respaldos.Aislamiento de DatosVM de BD sin salida a Internet; usuario MySQL restringido a IP 10.0.2.20 y --no-tablespaces.Elimina exposición directa a Internet y evita otorgar privilegios globales PROCESS.Seguridad en RuntimeContraseña de BD inyectada vía variable de entorno MYSQL_PWD en systemd.Evita la exposición de credenciales en argumentos de procesos (ps aux).Gobierno y FinOpsBucket Standard en us-central1 (< 5 GB) + Lifecycle Rule (Delete cuando age >= 7).Costo operativo de $0.00 USD (Always Free Tier) sin acumulación histórica.

5. Guía de Implementación Reproducible
Paso 1: Aprovisionamiento en Google Cloud Platform (gcp/setup_gcp_wif.sh)
Bash
export PROJECT_ID="<TU_PROJECT_ID>"
export PROJECT_NUMBER="<TU_PROJECT_NUMBER>"
export REGION="us-central1"
export BUCKET_NAME="harvester-wif-lab-${PROJECT_ID}"
export SA_NAME="sa-harvester-app"
export POOL_ID="harvester-onprem-pool"
export PROVIDER_ID="harvester-oidc-provider"

gcloud config set project $PROJECT_ID

# 1. Habilitar APIs necesarias
gcloud services enable \
  iam.googleapis.com \
  cloudresourcemanager.googleapis.com \
  iamcredentials.googleapis.com \
  sts.googleapis.com \
  storage.googleapis.com

# 2. Crear Bucket regional (Always Free Tier) y Service Account con menor privilegio
gcloud storage buckets create gs://${BUCKET_NAME} \
  --location=${REGION} \
  --uniform-bucket-level-access

gcloud iam service-accounts create ${SA_NAME} \
  --display-name="Harvester App Workload SA"

gcloud storage buckets add-iam-policy-binding gs://${BUCKET_NAME} \
  --member="serviceAccount:${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"

# 3. Crear Workload Identity Pool y OIDC Provider con JWKS público
gcloud iam workload-identity-pools create ${POOL_ID} \
  --location="global" \
  --display-name="Harvester On-Premise Pool"

gcloud iam workload-identity-pools providers create-oidc ${PROVIDER_ID} \
  --location="global" \
  --workload-identity-pool=${POOL_ID} \
  --display-name="Harvester Local OIDC Provider" \
  --issuer-uri="[https://idp.harvester.local](https://idp.harvester.local)" \
  --allowed-audiences="[https://iam.googleapis.com/projects/$](https://iam.googleapis.com/projects/$){PROJECT_NUMBER}/locations/global/workloadIdentityPools/${POOL_ID}/providers/${PROVIDER_ID}" \
  --attribute-mapping="google.subject=assertion.sub,attribute.host=assertion.hostname" \
  --attribute-condition="assertion.hostname == 'vm-app-01'" \
  --jwk-json-path="jwks.json"

# 4. Autorizar impersonación para el sujeto vm-app-01
gcloud iam service-accounts add-iam-policy-binding \
  "${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principal://[iam.googleapis.com/projects/$](https://iam.googleapis.com/projects/$){PROJECT_NUMBER}/locations/global/workloadIdentityPools/${POOL_ID}/subject/vm-app-01"



