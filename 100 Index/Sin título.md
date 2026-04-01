# DBF to PostgreSQL Data Schema Mapping

### 1) ARTICULO
* **Real DBF Primary Key:** `TCAJON`
* **FKs:** No outgoing strong FKs
* **Expected Cardinality:** N/A
* **Can FK be Null:** N/A
* **Delete/Update Rule:** N/A
* **Tentative Code Fields:** None relevant

### 2) BENEFICI
* **Real DBF Primary Key:** No strong unique PK in DBF
* **Recommended PostgreSQL PK:** Surrogate ID or composite based on business logic
* **FKs:**
    * `BENEFICI.CLIENT` -> `CLIENTES.CLIENT`
    * `BENEFICI.CLIAUX` -> `CLIENTES.CLIENT`
    * `BENEFICI.SUCURSAL` -> `SUCURSAL.SUCURSAL`
    * `BENEFICI.(SUCURSAL, REMITO)` -> `SERVICIO.(SUCURSAL, REMITO)`
* **Expected Cardinality:**
    * `CLIENT`: N:1
    * `CLIAUX`: N:1
    * `SUCURSAL`: N:1
    * `(SUCURSAL, REMITO)`: N:1
* **Can FK be Null:**
    * `CLIENT`: No
    * `CLIAUX`: Yes
    * `SUCURSAL`: No
    * `(SUCURSAL, REMITO)`: No
* **Delete/Update Rule:**
    * `CLIENT`: ON DELETE RESTRICT, ON UPDATE CASCADE
    * `CLIAUX`: ON DELETE SET NULL, ON UPDATE CASCADE
    * `SUCURSAL`: ON DELETE RESTRICT, ON UPDATE CASCADE
    * `(SUCURSAL, REMITO)`: ON DELETE RESTRICT, ON UPDATE CASCADE
* **Tentative Code Fields:** `CODCBT` (possible link to `TIPOCOMP.CODCBT`, not 100% confirmed)

### 3) CBTELEC
* **Real DBF Primary Key:** No clear PK (Integration table)
* **FKs:** `CBTELEC.REMITOSERV` -> `SERVICIO.REMITO`
* **Expected Cardinality:** N:1
* **Can FK be Null:** Yes, according to integration flow
* **Delete/Update Rule:** ON DELETE RESTRICT, ON UPDATE CASCADE
* **Tentative Code Fields:** `TIPCOMPROB` (Possible AFIP/TIPOCOMP mapping), `CODPRESTAC`, `CODPRESTAA` (External catalogs)

### 4) CLIENTES
* **Real DBF Primary Key:** `CLIENT`
* **FKs:** No outgoing strong FKs
* **Expected Cardinality:** N/A
* **Can FK be Null:** N/A
* **Delete/Update Rule:** N/A
* **Tentative Code Fields:** `TIPDOC`, `CONIVA`, `CODAUX` (Flag/Auxiliary code)

### 5) COMPRAS
* **Real DBF Primary Key:** No strong unique PK in DBF
* **Recommended PostgreSQL PK:** `(CODCBT, NROCBT)` or surrogate ID
* **FKs:**
    * `COMPRAS.PROVEE` -> `PROVEDOR.PROVEE`
    * `COMPRAS.CODCBT` -> `TIPOCOMP.CODCBT`
* **Expected Cardinality:**
    * `PROVEE`: N:1
    * `CODCBT`: N:1
* **Can FK be Null:**
    * `PROVEE`: No
    * `CODCBT`: No
* **Delete/Update Rule:** ON DELETE RESTRICT, ON UPDATE CASCADE
* **Tentative Code Fields:** `NROCUI` (Fiscal data, not FK)

### 6) CONCEPTO
* **Real DBF Primary Key:** `CONCEP`
* **FKs:** No outgoing strong FKs
* **Expected Cardinality:** N/A
* **Can FK be Null:** N/A
* **Delete/Update Rule:** N/A
* **Tentative Code Fields:** `SIGNO`, `TIPO` (Usage flag)

### 8) CONVTA
* **Real DBF Primary Key:** `CODVTA`
* **FKs:** No outgoing strong FKs
* **Expected Cardinality:** N/A
* **Can FK be Null:** N/A
* **Delete/Update Rule:** N/A
* **Tentative Code Fields:** `DESVTA` (Catalog)

### 9) CUENTAS
* **Real DBF Primary Key:** No strong unique PK in DBF
* **Recommended PostgreSQL PK:** `(CLIENT, REMITO, COMPRO)`
* **FKs:**
    * `CUENTAS.CLIENT` -> `CLIENTES.CLIENT`
    * `CUENTAS.CONCEP` -> `CONCEPTO.CONCEP`
    * `CUENTAS.REMITO` -> `SERVICIO.REMITO`
* **Expected Cardinality:**
    * `CLIENT`: N:1
    * `CONCEP`: N:1
    * `REMITO`: N:1
* **Can FK be Null:**
    * `CLIENT`: No
    * `CONCEP`: No
    * `REMITO`: Yes (Some movements might not originate from Service)
* **Delete/Update Rule:**
    * `CLIENT`: ON DELETE RESTRICT, ON UPDATE CASCADE
    * `CONCEP`: ON DELETE RESTRICT, ON UPDATE CASCADE
    * `REMITO`: ON DELETE RESTRICT, ON UPDATE CASCADE
* **Tentative Code Fields:** `COMPRO` (Internal number, possible transactional link but not a strong FK)

### 10) EMPRESA
* **Real DBF Primary Key:** `CODEMP` (Usually a single record)
* **FKs:** No outgoing strong FKs
* **Expected Cardinality:** N/A
* **Can FK be Null:** N/A
* **Delete/Update Rule:** N/A
* **Tentative Code Fields:** `SUCUR`, `PTOVTA`

### 11) FACPRT
* **Real DBF Primary Key:** No clear strong PK
* **Recommended PostgreSQL PK:** Surrogate ID
* **FKs:** No outgoing strong FKs
* **Expected Cardinality:** N/A
* **Can FK be Null:** N/A
* **Delete/Update Rule:** N/A
* **Tentative Code Fields:** `DETALLE` (Logical print field code)

### 12) ITEMPEDI
* **Real DBF Primary Key:** `(PEDIDO, ITEM)`
* **FKs:**
    * `ITEMPEDI.PEDIDO` -> `PEDIDOS.PEDIDO`
    * `ITEMPEDI.TCAJON` -> `ARTICULO.TCAJON`
* **Expected Cardinality:**
    * `PEDIDO`: N:1
    * `TCAJON`: N:1
* **Can FK be Null:**
    * `PEDIDO`: No
    * `TCAJON`: No
* **Delete/Update Rule:**
    * `PEDIDO`: ON DELETE CASCADE, ON UPDATE CASCADE
    * `TCAJON`: ON DELETE RESTRICT, ON UPDATE CASCADE
* **Tentative Code Fields:** `COMPRO` (Possible link to purchase/delivery note, not strong FK)

### 13) MOVIMIEN
* **Real DBF Primary Key:** No strong unique PK in DBF
* **Recommended PostgreSQL PK:** Surrogate ID
* **FKs:**
    * `MOVIMIEN.TCAJON` -> `ARTICULO.TCAJON`
    * `MOVIMIEN.CONCEP` -> `CONCEPTO.CONCEP`
* **Expected Cardinality:**
    * `TCAJON`: N:1
    * `CONCEP`: N:1
* **Can FK be Null:**
    * `TCAJON`: No
    * `CONCEP`: No
* **Delete/Update Rule:** ON DELETE RESTRICT, ON UPDATE CASCADE
* **Tentative Code Fields:** `REMITO` (Appears to reference `SERVICIO.REMITO`)

### 14) PEDIDOS
* **Real DBF Primary Key:** `PEDIDO`
* **FKs:** `PEDIDOS.PROVEE` -> `PROVEDOR.PROVEE`
* **Expected Cardinality:** N:1
* **Can FK be Null:** No
* **Delete/Update Rule:** ON DELETE RESTRICT, ON UPDATE CASCADE
* **Tentative Code Fields:** `FORMAP` (Payment method catalog, no clear table)

### 15) PROVEDOR
* **Real DBF Primary Key:** `PROVEE`
* **FKs:** No outgoing strong FKs
* **Expected Cardinality:** N/A
* **Can FK be Null:** N/A
* **Delete/Update Rule:** N/A
* **Tentative Code Fields:** `TIPDOC`, `CONIVA`, `IMPCUE`

### 16) REMITOS
* **Real DBF Primary Key:** No strong unique PK in DBF
* **Recommended PostgreSQL PK:** Surrogate ID or `(PROVEE, NROCBT, FECCBT)`
* **FKs:** `REMITOS.PROVEE` -> `PROVEDOR.PROVEE`
* **Expected Cardinality:** N:1
* **Can FK be Null:** No
* **Delete/Update Rule:** ON DELETE RESTRICT, ON UPDATE CASCADE
* **Tentative Code Fields:** `NROCBT` (May correlate with `COMPRAS`, not a strong FK)

### 17) SERVICIO
* **Real DBF Primary Key:** `REMITO` (Main index)
* **Recommended PostgreSQL PK:** `(SUCURSAL, REMITO)`
* **FKs:**
    * `SERVICIO.CLIENT` -> `CLIENTES.CLIENT`
    * `SERVICIO.TCAJON` -> `ARTICULO.TCAJON`
    * `SERVICIO.SUCURSAL` -> `SUCURSAL.SUCURSAL`
* **Expected Cardinality:**
    * `CLIENT`: N:1
    * `TCAJON`: N:1
    * `SUCURSAL`: N:1
* **Can FK be Null:**
    * `CLIENT`: No
    * `TCAJON`: Yes
    * `SUCURSAL`: No
* **Delete/Update Rule:**
    * `CLIENT`: ON DELETE RESTRICT, ON UPDATE CASCADE
    * `TCAJON`: ON DELETE SET NULL, ON UPDATE CASCADE
    * `SUCURSAL`: ON DELETE RESTRICT, ON UPDATE CASCADE
* **Tentative Code Fields:** `CODCEM`, `CODRCV`, `TDCTIT`, `TIPDOC` (Implicit catalogs)

### 18) SUCURSAL
* **Real DBF Primary Key:** `SUCURSAL`
* **FKs:** No outgoing strong FKs
* **Expected Cardinality:** N/A
* **Can FK be Null:** N/A
* **Delete/Update Rule:** N/A
* **Tentative Code Fields:** None

### 19) TEXTOS
* **Real DBF Primary Key:** `CODTXT`
* **FKs:**
    * `TEXTOS.CPTASO` -> `CONCEPTO.CONCEP`
    * `TEXTOS.CODASO` -> `TEXTOS.CODTXT`
* **Expected Cardinality:**
    * `CPTASO`: N:1
    * `CODASO`: N:1 (Self-reference)
* **Can FK be Null:**
    * `CPTASO`: Yes
    * `CODASO`: Yes
* **Delete/Update Rule:**
    * `CPTASO`: ON DELETE SET NULL, ON UPDATE CASCADE
    * `CODASO`: ON DELETE SET NULL, ON UPDATE CASCADE
* **Tentative Code Fields:** `INDNOM`, `INDDOC`, `INDFAL`, `INDINH`, `INDCEM`, `INDBEN`, `INDPRE`, `RELTIT`, `SOLCPR` (Formatting flags)

### 20) TIPOCOMP
* **Real DBF Primary Key:** `CODCBT`
* **FKs:** No outgoing strong FKs
* **Expected Cardinality:** N/A
* **Can FK be Null:** N/A
* **Delete/Update Rule:** N/A
* **Tentative Code Fields:** `CDAFIP`, `CODCAL`, `INFOARCA`

### 21) VENTAS
* **Real DBF Primary Key:** `(CODCBT, NROCBT)`
* **FKs:**
    * `VENTAS.CLIENT` -> `CLIENTES.CLIENT`
    * `VENTAS.CODCBT` -> `TIPOCOMP.CODCBT`
    * `VENTAS.REMITO` -> `SERVICIO.REMITO`
    * `VENTAS.CODTXT` -> `TEXTOS.CODTXT`
* **Expected Cardinality:**
    * `CLIENT`: N:1
    * `CODCBT`: N:1
    * `REMITO`: N:1
    * `CODTXT`: N:1
* **Can FK be Null:**
    * `CLIENT`: No
    * `CODCBT`: No
    * `REMITO`: Yes
    * `CODTXT`: Yes
* **Delete/Update Rule:**
    * `CLIENT`: ON DELETE RESTRICT, ON UPDATE CASCADE
    * `CODCBT`: ON DELETE RESTRICT, ON UPDATE CASCADE
    * `REMITO`: ON DELETE RESTRICT, ON UPDATE CASCADE
    * `CODTXT`: ON DELETE SET NULL, ON UPDATE CASCADE
* **Tentative Code Fields:** `CODPRE`, `CODASO`, `TIPCBTORIG`, `NROCBTORIG`, `FECCBTORIG` (Reference to original document, logical self-reference)