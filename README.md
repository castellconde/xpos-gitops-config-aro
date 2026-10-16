# `xpos-gitops-config-aro` — Gobierno GitOps, Catálogo de Proveedores, KEDA y Aislamiento CDE en ARO

Repositorio de estado deseado para **OpenShift GitOps (ArgoCD)**, configuración declarativa de proveedores externos, escalamiento KEDA consciente del Circuit Breaker (**RT-02**) y políticas de red del entorno PCI-DSS (**P2**).

---

## 1. Especificación Funcional

1. **Catálogo Declarativo de Proveedores (`config/proveedores-y-bitacora.yaml`):** Permite dar de alta proveedores sobre protocolos existentes (`REST`, `SOAP`, `ISO_8583`) o ajustar tablas de mapeo de códigos canónicos, timeouts y umbrales de Circuit Breaker mediante Pull Request auditado en Git, sin recompilar el motor.
2. **Cero Secretos en Git:** El pipeline rechaza cualquier credencial en claro; todos los proveedores referencian sus llaves/certificados mediante URI de **CyberArk Conjur** (`conjur://vault/pci/...`).
3. **Escalado KEDA Health-Aware (`manifests/cde-pci-keda-and-netpol.yaml`):** Combina el lag de Kafka con el estado `closed` del Circuit Breaker para evitar escalar pods cuando el proveedor está caído y el consumidor está pausado.

---

## 2. Especificación Técnica y Diagrama de Arquitectura Hexagonal GitOps

```mermaid
flowchart LR
    subgraph Inbound_Git["Fuentes GitOps Auditadas (Inbound)"]
        PR["Pull Request en GitHub<br/>(Catálogo YAML + Manifiestos)"]
        CI["Validador Python/Schema<br/>(gitops-validate.yml)"]
    end

    subgraph Core_GitOps["Motor de Reconciliación Continua"]
        ARGO["OpenShift GitOps (ArgoCD)<br/>application-prod-cde.yaml"]
        CONJ["CyberArk Conjur Sidecar<br/>Resolución de Alias conjur://"]
    end

    subgraph Outbound_ARO["Estado Aplicado en Clúster ARO (Outbound)"]
        CM["ConfigMap Inmutable<br/>(Hot-Reload de parámetros no sensibles)"]
        KEDA["KEDA ScaledObject<br/>(Lag + CircuitBreaker State)"]
        NP["NetworkPolicy CDE PCI<br/>(Default Deny + Allowlist)"]
    end

    PR --> CI --> ARGO
    ARGO --> CM
    ARGO --> KEDA
    ARGO --> NP
    CONJ --> Outbound_ARO
```

---

## 3. Modelo de Datos de Configuración GitOps (ER Diagram)

```mermaid
erDiagram
    CATALOGO_GITOPS ||--|| POLITICA_BITACORA : "define"
    CATALOGO_GITOPS ||--|{ CONFIG_PROVEEDOR : "contiene"
    CONFIG_PROVEEDOR ||--|| POLITICA_REINTENTOS : "configura"
    CONFIG_PROVEEDOR ||--|| UMBRAL_CIRCUIT_BREAKER : "protege con"
    CONFIG_PROVEEDOR ||--|{ MAPEO_CODIGO_CANONICO : "traduce"

    POLITICA_BITACORA {
        string particion "mensual"
        int retencion_caliente_meses "6"
        int retencion_payload_dias "60"
        int retencion_total_meses "13 (>= 12 PCI)"
        int particiones_futuras "3"
    }

    CONFIG_PROVEEDOR {
        string proveedorId PK
        string protocolo "REST | SOAP | ISO_8583"
        string endpoint
        string credencial_conjur_alias "conjur://vault/pci/..."
        string estrategia_reverso
    }

    MAPEO_CODIGO_CANONICO {
        string codigo_proveedor PK
        string clase_canonica "EXITO | YA_APLICADO | ERROR_REINTENTABLE | ESTADO_DESCONOCIDO | ERROR_NEGOCIO"
    }
```

---

## 4. Diagrama de Clases de Configuración

```mermaid
classDiagram
    class CatalogoGitOps {
        +PoliticaBitacora bitacora
        +Map~String, ConfigProveedor~ proveedores
    }

    class PoliticaBitacora {
        +String particion
        +int retencionCalienteMeses
        +int retencionPayloadDias
        +int retencionTotalMeses
        +int particionesFuturas
    }

    class ConfigProveedor {
        +String protocolo
        +String endpoint
        +String credencialConjurAlias
        +TimeoutsConfig timeouts
        +ReintentosConfig reintentos
        +CircuitBreakerConfig circuitBreaker
        +String estrategiaReverso
        +Map~String, String~ tablaCodigosCanonicos
    }

    CatalogoGitOps *-- PoliticaBitacora
    CatalogoGitOps *-- ConfigProveedor
```

---

## 5. Diagrama de Secuencia de Promoción y Recarga GitOps

```mermaid
sequenceDiagram
    autonumber
    actor OPS as Integrador / SRE
    participant GH as GitHub Actions (gitops-validate)
    participant ARGO as ArgoCD (OpenShift GitOps)
    participant POD as Pods Motor Reversos (ARO)

    OPS->>GH: PR modificando tabla_codigos_canonicos o umbrales en YAML
    GH->>GH: Validar retencion_total_meses >= 12 y prefijos conjur://
    OPS->>GH: Merge PR a main
    ARGO->>GH: Detectar nuevo commit SHA
    ARGO->>POD: Sincronizar ConfigMap y recursos KEDA / NetworkPolicy
    POD->>POD: Recargar en caliente parámetros no sensibles
```

---

## 6. Códigos de Estado de Validación GitOps

| Código | Regla Validada | Acción del Pipeline |
| :--- | :--- | :--- |
| `GITOPS-VAL-00` | Esquema YAML y restricciones PCI válidas | Permite Merge y sincronización en ArgoCD |
| `GITOPS-ERR-PCI-RET` | `retencion_total_meses < 12` | Bloquea el PR por incumplimiento de PCI-DSS Req 10.7 |
| `GITOPS-ERR-SECRET` | Credencial sin prefijo `conjur://` | Bloquea el PR para impedir secretos en Git |

---

## 7. Runbook Operativo GitOps

1. **Alta de un Proveedor Nuevo con Protocolo Existente (Sin tocar código Java):**
   - Agregar el bloque del proveedor en `config/proveedores-y-bitacora.yaml` con su alias en Conjur y su `tabla_codigos_canonicos`.
   - Abrir Pull Request; una vez aprobado el check `validate-gitops`, hacer merge y verificar el estado `Synced / Healthy` en ArgoCD.
2. **Rollback Inmediato de Configuración:**
   ```bash
   git revert HEAD && git push origin main
   ```
