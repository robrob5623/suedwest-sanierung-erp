```markdown
# A2_db-schema-rls.md

---

## Inhaltsverzeichnis

1. [ENUM Definitionen](#a-enum-definitionen)
2. [Tabellen-DDL](#b-tabellen-ddl)
3. [Indizes](#c-indizes)
4. [RLS Helper Functions](#d-rls-helper-functions)
5. [RLS Policies](#e-rls-policies)
6. [Views](#f-views)
7. [Konsistenz-Mechanik (Trigger)](#g-konsistenz-mechanik)
8. [Dokument-Inbox](#h-dokument-inbox)
9. [Storage Design](#i-storage-design)
10. [Assumptions & Decisions](#assumptions--decisions)

---

## (A) ENUM Definitionen

```sql
-- =============================================
-- ENUM TYPES
-- =============================================

CREATE TYPE tenant_type AS ENUM (
  'HAUSVERWALTUNG', 'VERSICHERUNG', 'MAKLER', 'REGULIERER',
  'SACHVERSTAENDIGER', 'KOMMUNE', 'BAUBIOLOGE', 'SONSTIGE'
);

CREATE TYPE user_role AS ENUM (
  'SYSTEM_ADMIN', 'PROJEKTLEITER', 'INNENDIENST',
  'MULTIPLIKATOR_ADMIN', 'MULTIPLIKATOR_USER', 'SACHVERSTAENDIGER'
);

CREATE TYPE damage_type AS ENUM (
  'WASSERSCHADEN', 'BRANDSCHADEN', 'STURMSCHADEN', 'SCHIMMEL', 'SONSTIGE'
);

CREATE TYPE phase_type AS ENUM (
  'LECKORTUNG', 'LECKAGE_BEHEBUNG', 'RUECKBAU', 'TROCKNUNG', 'WIEDERHERSTELLUNG'
);

CREATE TYPE phase_status AS ENUM (
  'NICHT_BEGONNEN', 'TERMIN_OFFEN', 'TERMIN_GEPLANT', 'VOR_ORT_ERFOLGT',
  'DOKUMENTATION', 'FREIGABE_AUSSTEHEND', 'IN_DURCHFUEHRUNG',
  'ABNAHME', 'ABRECHNUNG', 'ABGESCHLOSSEN'
);

CREATE TYPE priority AS ENUM (
  'NIEDRIG', 'NORMAL', 'HOCH', 'KRITISCH'
);

CREATE TYPE property_type AS ENUM (
  'MEHRFAMILIENHAUS', 'EINFAMILIENHAUS', 'GEWERBE', 'OEFFENTLICH', 'SONSTIGE'
);

CREATE TYPE contact_type AS ENUM (
  'MIETER', 'EIGENTUEMER', 'HAUSMEISTER', 'VERSICHERUNG',
  'SACHVERSTAENDIGER', 'SONSTIGE'
);

CREATE TYPE salutation AS ENUM (
  'HERR', 'FRAU', 'DIVERS', 'FIRMA'
);

CREATE TYPE event_type AS ENUM (
  'CLAIM_CREATED', 'CLAIM_UPDATED', 'CLAIM_CLOSED',
  'PHASE_CHANGED', 'STATUS_CHANGED',
  'DOCUMENT_UPLOADED', 'DOCUMENT_ASSIGNED',
  'APPOINTMENT_CREATED', 'APPOINTMENT_UPDATED', 'APPOINTMENT_COMPLETED',
  'COMMENT_ADDED',
  'OFFER_CREATED', 'OFFER_STATUS_CHANGED',
  'INVOICE_CREATED', 'INVOICE_STATUS_CHANGED',
  'USER_ASSIGNED', 'CONTACT_ADDED', 'ERP_SYNC'
);

CREATE TYPE event_category AS ENUM (
  'STATUS', 'DOKUMENT', 'KOMMUNIKATION', 'TERMIN', 'FINANZEN', 'SYSTEM'
);

CREATE TYPE appointment_type AS ENUM (
  'LECKORTUNG', 'BEGEHUNG', 'ABNAHME', 'TROCKNUNG_MESSUNG',
  'RUECKBAU', 'WIEDERHERSTELLUNG', 'SONSTIGE'
);

CREATE TYPE appointment_status AS ENUM (
  'GEPLANT', 'BESTAETIGT', 'DURCHGEFUEHRT', 'VERSCHOBEN', 'STORNIERT'
);

CREATE TYPE folder_type AS ENUM (
  'PROTOKOLLE', 'ANGEBOTE', 'RECHNUNGEN', 'FOTOS',
  'GUTACHTEN', 'KORRESPONDENZ', 'SONSTIGE', 'CUSTOM', 'INBOX'
);

CREATE TYPE document_type AS ENUM (
  'FOTO', 'PROTOKOLL', 'ANGEBOT', 'RECHNUNG', 'GUTACHTEN',
  'KORRESPONDENZ', 'SONSTIGE'
);

CREATE TYPE ocr_status AS ENUM (
  'PENDING', 'PROCESSING', 'COMPLETED', 'FAILED', 'NOT_APPLICABLE'
);

CREATE TYPE ocr_assignment_status AS ENUM (
  'UNASSIGNED', 'AUTO_ASSIGNED', 'MANUAL_REQUIRED', 'MANUAL_ASSIGNED', 'NOT_APPLICABLE'
);

CREATE TYPE offer_status AS ENUM (
  'ENTWURF', 'ERSTELLT', 'VERSENDET', 'ANGENOMMEN', 'ABGELEHNT', 'STORNIERT'
);

CREATE TYPE invoice_status AS ENUM (
  'ENTWURF', 'ERSTELLT', 'VERSENDET', 'BEZAHLT', 'MAHNUNG', 'STORNIERT'
);

CREATE TYPE notification_type AS ENUM (
  'STATUS_CHANGE', 'NEW_COMMENT', 'MENTION', 'DOCUMENT_ADDED',
  'APPOINTMENT', 'DEADLINE', 'SYSTEM'
);

CREATE TYPE integration_type AS ENUM (
  'ERP', 'OCR', 'STORAGE'
);

CREATE TYPE sync_direction AS ENUM (
  'INBOUND', 'OUTBOUND', 'BIDIRECTIONAL'
);

CREATE TYPE sync_status AS ENUM (
  'SUCCESS', 'FAILED', 'PENDING'
);

CREATE TYPE claim_assignment_role AS ENUM (
  'PROJEKTLEITER', 'BEARBEITER', 'BEOBACHTER'
);

CREATE TYPE claim_contact_role AS ENUM (
  'GESCHAEDIGTER', 'ANSPRECHPARTNER', 'HAUSMEISTER',
  'VERSICHERUNGSNEHMER', 'SONSTIGE'
);

CREATE TYPE audit_action AS ENUM (
  'CREATE', 'UPDATE', 'DELETE', 'ACCESS'
);
```

---

## (B) Tabellen-DDL

### 1. tenants

```sql
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  external_id VARCHAR(100),
  name VARCHAR(255) NOT NULL,
  type tenant_type NOT NULL,
  address_street VARCHAR(255),
  address_zip VARCHAR(10),
  address_city VARCHAR(100),
  contact_email VARCHAR(255) NOT NULL,
  contact_phone VARCHAR(50),
  logo_url VARCHAR(500),
  is_active BOOLEAN NOT NULL DEFAULT true,
  settings JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_tenants_external_id ON tenants(external_id) WHERE external_id IS NOT NULL;
```

### 2. users

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY, -- = auth.users.id
  tenant_id UUID REFERENCES tenants(id) ON DELETE SET NULL,
  external_id VARCHAR(100),
  email VARCHAR(255) NOT NULL UNIQUE,
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL,
  role user_role NOT NULL,
  phone VARCHAR(50),
  avatar_url VARCHAR(500),
  is_active BOOLEAN NOT NULL DEFAULT true,
  notification_preferences JSONB DEFAULT '{"email": true, "push": true}',
  last_login_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_tenant_id ON users(tenant_id);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_email ON users(email);
```

### 3. properties

```sql
CREATE TABLE properties (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  external_id VARCHAR(100),
  name VARCHAR(255),
  property_type property_type NOT NULL,
  address_street VARCHAR(255) NOT NULL,
  address_zip VARCHAR(10) NOT NULL,
  address_city VARCHAR(100) NOT NULL,
  geo_lat DECIMAL(10,7),
  geo_lng DECIMAL(10,7),
  unit_count INTEGER,
  construction_year INTEGER,
  notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_properties_tenant_id ON properties(tenant_id);
```

### 4. contacts

```sql
CREATE TABLE contacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  external_id VARCHAR(100),
  contact_type contact_type NOT NULL,
  salutation salutation,
  first_name VARCHAR(100),
  last_name VARCHAR(100) NOT NULL,
  company VARCHAR(255),
  email VARCHAR(255),
  phone VARCHAR(50),
  phone_mobile VARCHAR(50),
  address_street VARCHAR(255),
  address_zip VARCHAR(10),
  address_city VARCHAR(100),
  notes TEXT,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contacts_tenant_id ON contacts(tenant_id);
CREATE INDEX idx_contacts_type ON contacts(contact_type);
```

### 5. claims

```sql
CREATE TABLE claims (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  property_id UUID REFERENCES properties(id) ON DELETE SET NULL,
  external_id VARCHAR(100),
  claim_number VARCHAR(50) NOT NULL UNIQUE,
  insurance_claim_number VARCHAR(100),
  title VARCHAR(255) NOT NULL,
  description TEXT,
  damage_type damage_type NOT NULL,
  damage_cause VARCHAR(255),
  damage_date DATE,
  reported_date DATE NOT NULL,
  address_street VARCHAR(255) NOT NULL,
  address_zip VARCHAR(10) NOT NULL,
  address_city VARCHAR(100) NOT NULL,
  address_floor VARCHAR(50),
  address_unit VARCHAR(50),
  geo_lat DECIMAL(10,7),
  geo_lng DECIMAL(10,7),
  current_phase phase_type NOT NULL DEFAULT 'LECKORTUNG',
  current_status phase_status NOT NULL DEFAULT 'NICHT_BEGONNEN',
  priority priority NOT NULL DEFAULT 'NORMAL',
  is_archived BOOLEAN NOT NULL DEFAULT false,
  closed_at TIMESTAMPTZ,
  erp_last_sync TIMESTAMPTZ,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_damage_date CHECK (damage_date IS NULL OR damage_date <= reported_date),
  CONSTRAINT chk_reported_date CHECK (reported_date <= CURRENT_DATE)
);

CREATE INDEX idx_claims_tenant_id ON claims(tenant_id);
CREATE INDEX idx_claims_property_id ON claims(property_id);
CREATE INDEX idx_claims_current_phase ON claims(current_phase);
CREATE INDEX idx_claims_current_status ON claims(current_status);
CREATE INDEX idx_claims_is_archived ON claims(is_archived);
CREATE INDEX idx_claims_created_at ON claims(created_at DESC);
```

### 6. claim_phases

```sql
CREATE TABLE claim_phases (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  claim_id UUID NOT NULL REFERENCES claims(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE, -- redundant für RLS
  phase_type phase_type NOT NULL,
  status phase_status NOT NULL DEFAULT 'NICHT_BEGONNEN',
  erp_substatus VARCHAR(100),
  sequence INTEGER NOT NULL CHECK (sequence BETWEEN 1 AND 5),
  is_applicable BOOLEAN NOT NULL DEFAULT true,
  is_skipped BOOLEAN NOT NULL DEFAULT false,
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT uq_claim_phase UNIQUE (claim_id, phase_type)
);

CREATE INDEX idx_claim_phases_claim_id ON claim_phases(claim_id);
CREATE INDEX idx_claim_phases_tenant_id ON claim_phases(tenant_id);
CREATE INDEX idx_claim_phases_status ON claim_phases(status);
```

### 7. events

```sql
CREATE TABLE events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  claim_id UUID NOT NULL REFERENCES claims(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE, -- redundant für RLS
  user_id UUID REFERENCES users(id) ON DELETE SET NULL,
  event_type event_type NOT NULL,
  event_category event_category NOT NULL,
  title VARCHAR(255) NOT NULL,
  description TEXT,
  reference_type VARCHAR(50),
  reference_id UUID,
  metadata JSONB DEFAULT '{}',
  is_visible_to_tenant BOOLEAN NOT NULL DEFAULT true,
  event_timestamp TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_events_claim_id ON events(claim_id);
CREATE INDEX idx_events_tenant_id ON events(tenant_id);
CREATE INDEX idx_events_event_timestamp ON events(event_timestamp DESC);
CREATE INDEX idx_events_event_type ON events(event_type);
CREATE INDEX idx_events_visible ON events(is_visible_to_tenant) WHERE is_visible_to_tenant = true;
```

### 8. calendar_items

```sql
CREATE TABLE calendar_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  claim_id UUID NOT NULL REFERENCES claims(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE, -- redundant für RLS
  external_id VARCHAR(100),
  appointment_type appointment_type NOT NULL,
  title VARCHAR(255) NOT NULL,
  description TEXT,
  start_datetime TIMESTAMPTZ NOT NULL,
  end_datetime TIMESTAMPTZ,
  is_all_day BOOLEAN NOT NULL DEFAULT false,
  location VARCHAR(255),
  status appointment_status NOT NULL DEFAULT 'GEPLANT',
  assigned_team VARCHAR(255),
  notes TEXT,
  is_visible_to_tenant BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_calendar_items_claim_id ON calendar_items(claim_id);
CREATE INDEX idx_calendar_items_tenant_id ON calendar_items(tenant_id);
CREATE INDEX idx_calendar_items_start ON calendar_items(start_datetime);
CREATE INDEX idx_calendar_items_status ON calendar_items(status);
```

### 9. document_folders

```sql
CREATE TABLE document_folders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  claim_id UUID NOT NULL REFERENCES claims(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE, -- redundant für RLS
  parent_folder_id UUID REFERENCES document_folders(id) ON DELETE CASCADE,
  name VARCHAR(100) NOT NULL,
  folder_type folder_type NOT NULL,
  sequence INTEGER NOT NULL DEFAULT 0,
  is_system_folder BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_document_folders_claim_id ON document_folders(claim_id);
CREATE INDEX idx_document_folders_tenant_id ON document_folders(tenant_id);
CREATE INDEX idx_document_folders_parent ON document_folders(parent_folder_id);
```

### 10. documents

```sql
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  claim_id UUID REFERENCES claims(id) ON DELETE CASCADE, -- NULL für Inbox-Dokumente
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  folder_id UUID REFERENCES document_folders(id) ON DELETE SET NULL,
  external_id VARCHAR(100),
  file_name VARCHAR(255) NOT NULL,
  display_name VARCHAR(255) NOT NULL,
  file_type VARCHAR(100) NOT NULL,
  file_size BIGINT NOT NULL,
  storage_path VARCHAR(500) NOT NULL,
  document_type document_type NOT NULL DEFAULT 'SONSTIGE',
  document_date DATE,
  version INTEGER NOT NULL DEFAULT 1,
  previous_version_id UUID REFERENCES documents(id) ON DELETE SET NULL,
  ocr_status ocr_status NOT NULL DEFAULT 'PENDING',
  ocr_confidence DECIMAL(5,2),
  ocr_extracted_data JSONB,
  ocr_assignment_status ocr_assignment_status NOT NULL DEFAULT 'UNASSIGNED',
  uploaded_by_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
  is_visible_to_tenant BOOLEAN NOT NULL DEFAULT true,
  is_inbox BOOLEAN NOT NULL DEFAULT false, -- True wenn noch nicht zugeordnet
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_inbox_or_claim CHECK (
    (is_inbox = true AND claim_id IS NULL) OR 
    (is_inbox = false AND claim_id IS NOT NULL)
  )
);

CREATE INDEX idx_documents_claim_id ON documents(claim_id);
CREATE INDEX idx_documents_tenant_id ON documents(tenant_id);
CREATE INDEX idx_documents_folder_id ON documents(folder_id);
CREATE INDEX idx_documents_ocr_status ON documents(ocr_status);
CREATE INDEX idx_documents_is_inbox ON documents(is_inbox) WHERE is_inbox = true;
CREATE INDEX idx_documents_assignment_status ON documents(ocr_assignment_status);
```

### 11. offers

```sql
CREATE TABLE offers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  claim_id UUID NOT NULL REFERENCES claims(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE, -- redundant für RLS
  claim_phase_id UUID REFERENCES claim_phases(id) ON DELETE SET NULL,
  external_id VARCHAR(100) NOT NULL,
  offer_number VARCHAR(50) NOT NULL,
  title VARCHAR(255) NOT NULL,
  description TEXT,
  net_amount DECIMAL(12,2) NOT NULL,
  tax_amount DECIMAL(12,2) NOT NULL,
  gross_amount DECIMAL(12,2) NOT NULL,
  currency VARCHAR(3) NOT NULL DEFAULT 'EUR',
  status offer_status NOT NULL DEFAULT 'ENTWURF',
  offer_date DATE NOT NULL,
  valid_until DATE,
  accepted_at TIMESTAMPTZ,
  document_id UUID REFERENCES documents(id) ON DELETE SET NULL,
  erp_last_sync TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_offer_amounts CHECK (gross_amount = net_amount + tax_amount)
);

CREATE INDEX idx_offers_claim_id ON offers(claim_id);
CREATE INDEX idx_offers_tenant_id ON offers(tenant_id);
CREATE INDEX idx_offers_status ON offers(status);
CREATE UNIQUE INDEX idx_offers_external_id ON offers(external_id);
```

### 12. invoices

```sql
CREATE TABLE invoices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  claim_id UUID NOT NULL REFERENCES claims(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE, -- redundant für RLS
  claim_phase_id UUID REFERENCES claim_phases(id) ON DELETE SET NULL,
  offer_id UUID REFERENCES offers(id) ON DELETE SET NULL,
  external_id VARCHAR(100) NOT NULL,
  invoice_number VARCHAR(50) NOT NULL,
  title VARCHAR(255) NOT NULL,
  description TEXT,
  net_amount DECIMAL(12,2) NOT NULL,
  tax_amount DECIMAL(12,2) NOT NULL,
  gross_amount DECIMAL(12,2) NOT NULL,
  currency VARCHAR(3) NOT NULL DEFAULT 'EUR',
  status invoice_status NOT NULL DEFAULT 'ENTWURF',
  invoice_date DATE NOT NULL,
  due_date DATE,
  paid_at TIMESTAMPTZ,
  paid_amount DECIMAL(12,2),
  document_id UUID REFERENCES documents(id) ON DELETE SET NULL,
  erp_last_sync TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_invoice_amounts CHECK (gross_amount = net_amount + tax_amount),
  CONSTRAINT chk_paid_amount CHECK (paid_amount IS NULL OR paid_amount <= gross_amount)
);

CREATE INDEX idx_invoices_claim_id ON invoices(claim_id);
CREATE INDEX idx_invoices_tenant_id ON invoices(tenant_id);
CREATE INDEX idx_invoices_status ON invoices(status);
CREATE UNIQUE INDEX idx_invoices_external_id ON invoices(external_id);
```

### 13. comments

```sql
CREATE TABLE comments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  claim_id UUID NOT NULL REFERENCES claims(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE, -- redundant für RLS
  thread_id UUID REFERENCES comments(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  content TEXT NOT NULL,
  is_internal BOOLEAN NOT NULL DEFAULT false,
  is_edited BOOLEAN NOT NULL DEFAULT false,
  edited_at TIMESTAMPTZ,
  is_deleted BOOLEAN NOT NULL DEFAULT false,
  deleted_at TIMESTAMPTZ,
  mentions UUID[] DEFAULT '{}',
  attachments JSONB DEFAULT '[]',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comments_claim_id ON comments(claim_id);
CREATE INDEX idx_comments_tenant_id ON comments(tenant_id);
CREATE INDEX idx_comments_thread_id ON comments(thread_id);
CREATE INDEX idx_comments_user_id ON comments(user_id);
CREATE INDEX idx_comments_created_at ON comments(created_at DESC);
CREATE INDEX idx_comments_is_internal ON comments(is_internal);
```

### 14. notifications

```sql
CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE, -- redundant für RLS (NULL für interne)
  claim_id UUID REFERENCES claims(id) ON DELETE CASCADE,
  notification_type notification_type NOT NULL,
  title VARCHAR(255) NOT NULL,
  message TEXT NOT NULL,
  action_url VARCHAR(500),
  reference_type VARCHAR(50),
  reference_id UUID,
  is_read BOOLEAN NOT NULL DEFAULT false,
  read_at TIMESTAMPTZ,
  is_email_sent BOOLEAN NOT NULL DEFAULT false,
  email_sent_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_user_id ON notifications(user_id);
CREATE INDEX idx_notifications_tenant_id ON notifications(tenant_id);
CREATE INDEX idx_notifications_is_read ON notifications(is_read) WHERE is_read = false;
CREATE INDEX idx_notifications_created_at ON notifications(created_at DESC);
```

### 15. integration_mappings

```sql
CREATE TABLE integration_mappings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  integration_type integration_type NOT NULL,
  entity_type VARCHAR(50) NOT NULL,
  entity_id UUID NOT NULL,
  external_system VARCHAR(100) NOT NULL,
  external_entity_type VARCHAR(100) NOT NULL,
  external_id VARCHAR(255) NOT NULL,
  external_url VARCHAR(500),
  sync_direction sync_direction NOT NULL,
  last_sync_at TIMESTAMPTZ,
  last_sync_status sync_status NOT NULL DEFAULT 'PENDING',
  last_sync_error TEXT,
  sync_metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT uq_integration_mapping UNIQUE (integration_type, entity_type, entity_id, external_system)
);

CREATE INDEX idx_integration_mappings_entity ON integration_mappings(entity_type, entity_id);
CREATE INDEX idx_integration_mappings_external ON integration_mappings(external_system, external_id);
```

### 16. claim_assignments (Junction Table)

```sql
CREATE TABLE claim_assignments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  claim_id UUID NOT NULL REFERENCES claims(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE, -- redundant für RLS
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role claim_assignment_role NOT NULL,
  assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  assigned_by_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT uq_claim_assignment UNIQUE (claim_id, user_id, role)
);

CREATE INDEX idx_claim_assignments_claim_id ON claim_assignments(claim_id);
CREATE INDEX idx_claim_assignments_user_id ON claim_assignments(user_id);
CREATE INDEX idx_claim_assignments_tenant_id ON claim_assignments(tenant_id);
```

### 17. claim_contacts (Junction Table)

```sql
CREATE TABLE claim_contacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  claim_id UUID NOT NULL REFERENCES claims(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE, -- redundant für RLS
  contact_id UUID NOT NULL REFERENCES contacts(id) ON DELETE CASCADE,
  role_in_claim claim_contact_role NOT NULL,
  is_primary BOOLEAN NOT NULL DEFAULT false,
  notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT uq_claim_contact UNIQUE (claim_id, contact_id, role_in_claim)
);

CREATE INDEX idx_claim_contacts_claim_id ON claim_contacts(claim_id);
CREATE INDEX idx_claim_contacts_contact_id ON claim_contacts(contact_id);
CREATE INDEX idx_claim_contacts_tenant_id ON claim_contacts(tenant_id);
```

### 18. audit_logs

```sql
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  entity_type VARCHAR(50) NOT NULL,
  entity_id UUID NOT NULL,
  action audit_action NOT NULL,
  user_id UUID REFERENCES users(id) ON DELETE SET NULL,
  tenant_id UUID REFERENCES tenants(id) ON DELETE SET NULL,
  old_values JSONB,
  new_values JSONB,
  changed_fields TEXT[],
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partitionierung nach Monat für Performance
CREATE INDEX idx_audit_logs_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_tenant_id ON audit_logs(tenant_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at DESC);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);
```

---

## (C) Indizes

### Geo-Index für Karten-Queries (Bounding Box)

```sql
-- Extension aktivieren
CREATE EXTENSION IF NOT EXISTS postgis;

-- GiST Index für Geo-Queries auf claims
CREATE INDEX idx_claims_geo ON claims 
  USING gist (
    ST_MakePoint(geo_lng, geo_lat)
  ) 
  WHERE geo_lat IS NOT NULL AND geo_lng IS NOT NULL;

-- GiST Index für Geo-Queries auf properties
CREATE INDEX idx_properties_geo ON properties 
  USING gist (
    ST_MakePoint(geo_lng, geo_lat)
  ) 
  WHERE geo_lat IS NOT NULL AND geo_lng IS NOT NULL;
```

### Composite Indizes für häufige Query-Pfade

```sql
-- Tenant Claims List (aktive, sortiert nach Datum)
CREATE INDEX idx_claims_tenant_list ON claims(tenant_id, is_archived, created_at DESC);

-- Tenant Claims by Status
CREATE INDEX idx_claims_tenant_status ON claims(tenant_id, current_status, is_archived);

-- Claim Timeline (Events sortiert)
CREATE INDEX idx_events_claim_timeline ON events(claim_id, event_timestamp DESC);

-- Documents by Claim
CREATE INDEX idx_documents_claim_list ON documents(claim_id, created_at DESC) WHERE claim_id IS NOT NULL;

-- Inbox Documents by Tenant
CREATE INDEX idx_documents_inbox_tenant ON documents(tenant_id, created_at DESC) WHERE is_inbox = true;

-- Calendar Items by Date Range
CREATE INDEX idx_calendar_date_range ON calendar_items(tenant_id, start_datetime, end_datetime);

-- Full-text search auf Claims
CREATE INDEX idx_claims_search ON claims 
  USING gin (to_tsvector('german', coalesce(title, '') || ' ' || coalesce(description, '') || ' ' || claim_number));
```

---

## (D) RLS Helper Functions

```sql
-- =============================================
-- HELPER FUNCTIONS FÜR RLS
-- =============================================

-- Aktuelle User-ID aus JWT
CREATE OR REPLACE FUNCTION auth.uid() 
RETURNS UUID 
LANGUAGE sql STABLE
AS $$
  SELECT NULLIF(current_setting('request.jwt.claim.sub', true), '')::UUID;
$$;

-- User Role aus app_metadata
CREATE OR REPLACE FUNCTION auth.role() 
RETURNS user_role 
LANGUAGE sql STABLE
AS $$
  SELECT NULLIF(current_setting('request.jwt.claim.app_metadata', true)::jsonb->>'role', '')::user_role;
$$;

-- Tenant ID aus app_metadata (NULL für interne User)
CREATE OR REPLACE FUNCTION auth.tenant_id() 
RETURNS UUID 
LANGUAGE sql STABLE
AS $$
  SELECT NULLIF(current_setting('request.jwt.claim.app_metadata', true)::jsonb->>'tenant_id', '')::UUID;
$$;

-- Prüft ob User intern ist (kein tenant_id)
CREATE OR REPLACE FUNCTION auth.is_internal_user() 
RETURNS BOOLEAN 
LANGUAGE sql STABLE
AS $$
  SELECT auth.tenant_id() IS NULL AND auth.role() IN ('SYSTEM_ADMIN', 'PROJEKTLEITER', 'INNENDIENST');
$$;

-- Prüft ob User Tenant-Admin ist
CREATE OR REPLACE FUNCTION auth.is_tenant_admin() 
RETURNS BOOLEAN 
LANGUAGE sql STABLE
AS $$
  SELECT auth.role() = 'MULTIPLIKATOR_ADMIN';
$$;

-- Prüft ob User Multiplikator (Admin oder User) ist
CREATE OR REPLACE FUNCTION auth.is_multiplikator() 
RETURNS BOOLEAN 
LANGUAGE sql STABLE
AS $$
  SELECT auth.role() IN ('MULTIPLIKATOR_ADMIN', 'MULTIPLIKATOR_USER');
$$;

-- Prüft Tenant-Zugehörigkeit
CREATE OR REPLACE FUNCTION auth.has_tenant_access(check_tenant_id UUID) 
RETURNS BOOLEAN 
LANGUAGE sql STABLE
AS $$
  SELECT 
    auth.is_internal_user() 
    OR (auth.tenant_id() IS NOT NULL AND auth.tenant_id() = check_tenant_id);
$$;
```

---

## (E) RLS Policies

### Enable RLS auf allen Tabellen

```sql
ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE properties ENABLE ROW LEVEL SECURITY;
ALTER TABLE contacts ENABLE ROW LEVEL SECURITY;
ALTER TABLE claims ENABLE ROW LEVEL SECURITY;
ALTER TABLE claim_phases ENABLE ROW LEVEL SECURITY;
ALTER TABLE events ENABLE ROW LEVEL SECURITY;
ALTER TABLE calendar_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE document_folders ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE offers ENABLE ROW LEVEL SECURITY;
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;
ALTER TABLE comments ENABLE ROW LEVEL SECURITY;
ALTER TABLE notifications ENABLE ROW LEVEL SECURITY;
ALTER TABLE integration_mappings ENABLE ROW LEVEL SECURITY;
ALTER TABLE claim_assignments ENABLE ROW LEVEL SECURITY;
ALTER TABLE claim_contacts ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;
```

### tenants

```sql
-- SELECT: Interne sehen alle, Externe nur eigenen Tenant
CREATE POLICY "tenants_select" ON tenants FOR SELECT USING (
  auth.is_internal_user() OR id = auth.tenant_id()
);

-- INSERT: Nur SYSTEM_ADMIN
CREATE POLICY "tenants_insert" ON tenants FOR INSERT WITH CHECK (
  auth.role() = 'SYSTEM_ADMIN'
);

-- UPDATE: SYSTEM_ADMIN alle, MULTIPLIKATOR_ADMIN eigenen Tenant (nur settings)
CREATE POLICY "tenants_update" ON tenants FOR UPDATE USING (
  auth.role() = 'SYSTEM_ADMIN' 
  OR (auth.is_tenant_admin() AND id = auth.tenant_id())
);

-- DELETE: Nur SYSTEM_ADMIN
CREATE POLICY "tenants_delete" ON tenants FOR DELETE USING (
  auth.role() = 'SYSTEM_ADMIN'
);
```

### users

```sql
-- SELECT: Interne sehen alle, Externe nur eigenen Tenant + sich selbst
CREATE POLICY "users_select" ON users FOR SELECT USING (
  auth.is_internal_user() 
  OR id = auth.uid()
  OR tenant_id = auth.tenant_id()
);

-- INSERT: SYSTEM_ADMIN alle, MULTIPLIKATOR_ADMIN für eigenen Tenant
CREATE POLICY "users_insert" ON users FOR INSERT WITH CHECK (
  auth.role() = 'SYSTEM_ADMIN'
  OR (auth.is_tenant_admin() AND tenant_id = auth.tenant_id())
);

-- UPDATE: SYSTEM_ADMIN alle, User sich selbst, MULTIPLIKATOR_ADMIN eigenen Tenant
CREATE POLICY "users_update" ON users FOR UPDATE USING (
  auth.role() = 'SYSTEM_ADMIN'
  OR id = auth.uid()
  OR (auth.is_tenant_admin() AND tenant_id = auth.tenant_id())
);

-- DELETE: SYSTEM_ADMIN alle, MULTIPLIKATOR_ADMIN eigenen Tenant (nicht sich selbst)
CREATE POLICY "users_delete" ON users FOR DELETE USING (
  auth.role() = 'SYSTEM_ADMIN'
  OR (auth.is_tenant_admin() AND tenant_id = auth.tenant_id() AND id != auth.uid())
);
```

### properties

```sql
-- SELECT: Tenant-basiert
CREATE POLICY "properties_select" ON properties FOR SELECT USING (
  auth.has_tenant_access(tenant_id)
);

-- INSERT: Interne oder MULTIPLIKATOR_ADMIN für eigenen Tenant
CREATE POLICY "properties_insert" ON properties FOR INSERT WITH CHECK (
  auth.is_internal_user()
  OR (auth.is_tenant_admin() AND tenant_id = auth.tenant_id())
);

-- UPDATE: Interne oder MULTIPLIKATOR_ADMIN für eigenen Tenant
CREATE POLICY "properties_update" ON properties FOR UPDATE USING (
  auth.is_internal_user()
  OR (auth.is_tenant_admin() AND tenant_id = auth.tenant_id())
);

-- DELETE: Nur Interne
CREATE POLICY "properties_delete" ON properties FOR DELETE USING (
  auth.is_internal_user()
);
```

### contacts

```sql
CREATE POLICY "contacts_select" ON contacts FOR SELECT USING (
  auth.has_tenant_access(tenant_id)
);

CREATE POLICY "contacts_insert" ON contacts FOR INSERT WITH CHECK (
  auth.is_internal_user()
  OR (auth.is_multiplikator() AND tenant_id = auth.tenant_id())
);

CREATE POLICY "contacts_update" ON contacts FOR UPDATE USING (
  auth.is_internal_user()
  OR (auth.is_multiplikator() AND tenant_id = auth.tenant_id())
);

CREATE POLICY "contacts_delete" ON contacts FOR DELETE USING (
  auth.is_internal_user()
);
```

### claims

```sql
-- SELECT: Tenant-basiert, archivierte nur lesbar
CREATE POLICY "claims_select" ON claims FOR SELECT USING (
  auth.has_tenant_access(tenant_id)
);

-- INSERT: Interne oder Multiplikatoren für eigenen Tenant
CREATE POLICY "claims_insert" ON claims FOR INSERT WITH CHECK (
  auth.is_internal_user()
  OR (auth.is_multiplikator() AND tenant_id = auth.tenant_id())
);

-- UPDATE: Interne oder Multiplikatoren (nicht archivierte) für eigenen Tenant
CREATE POLICY "claims_update" ON claims FOR UPDATE USING (
  auth.is_internal_user()
  OR (auth.is_multiplikator() AND tenant_id = auth.tenant_id() AND is_archived = false)
);

-- DELETE: Nur SYSTEM_ADMIN
CREATE POLICY "claims_delete" ON claims FOR DELETE USING (
  auth.role() = 'SYSTEM_ADMIN'
);
```

### claim_phases

```sql
CREATE POLICY "claim_phases_select" ON claim_phases FOR SELECT USING (
  auth.has_tenant_access(tenant_id)
);

-- INSERT/UPDATE/DELETE: Nur Interne (ERP-gesteuert)
CREATE POLICY "claim_phases_insert" ON claim_phases FOR INSERT WITH CHECK (
  auth.is_internal_user()
);

CREATE POLICY "claim_phases_update" ON claim_phases FOR UPDATE USING (
  auth.is_internal_user()
);

CREATE POLICY "claim_phases_delete" ON claim_phases FOR DELETE USING (
  auth.role() = 'SYSTEM_ADMIN'
);
```

### events

```sql
-- SELECT: Tenant-basiert, aber nur visible_to_tenant für Multiplikatoren
CREATE POLICY "events_select" ON events FOR SELECT USING (
  auth.is_internal_user()
  OR (auth.has_tenant_access(tenant_id) AND is_visible_to_tenant = true)
);

-- INSERT: Interne oder Multiplikatoren (nur sichtbare Events)
CREATE POLICY "events_insert" ON events FOR INSERT WITH CHECK (
  auth.is_internal_user()
  OR (auth.is_multiplikator() AND tenant_id = auth.tenant_id())
);

-- UPDATE/DELETE: Nur Interne
CREATE POLICY "events_update" ON events FOR UPDATE USING (
  auth.is_internal_user()
);

CREATE POLICY "events_delete" ON events FOR DELETE USING (
  auth.role() = 'SYSTEM_ADMIN'
);
```

### calendar_items

```sql
CREATE POLICY "calendar_items_select" ON calendar_items FOR SELECT USING (
  auth.is_internal_user()
  OR (auth.has_tenant_access(tenant_id) AND is_visible_to_tenant = true)
);

CREATE POLICY "calendar_items_insert" ON calendar_items FOR INSERT WITH CHECK (
  auth.is_internal_user()
);

CREATE POLICY "calendar_items_update" ON calendar_items FOR UPDATE USING (
  auth.is_internal_user()
);

CREATE POLICY "calendar_items_delete" ON calendar_items FOR DELETE USING (
  auth.is_internal_user()
);
```

### document_folders

```sql
CREATE POLICY "document_folders_select" ON document_folders FOR SELECT USING (
  auth.has_tenant_access(tenant_id)
);

CREATE POLICY "document_folders_insert" ON document_folders FOR INSERT WITH CHECK (
  auth.is_internal_user()
  OR (auth.is_multiplikator() AND tenant_id = auth.tenant_id())
);

CREATE POLICY "document_folders_update" ON document_folders FOR UPDATE USING (
  auth.is_internal_user()
  OR (auth.is_multiplikator() AND tenant_id = auth.tenant_id() AND is_system_folder = false)
);

CREATE POLICY "document_folders_delete" ON document_folders FOR DELETE USING (
  (auth.is_internal_user() OR (auth.is_multiplikator() AND tenant_id = auth.tenant_id()))
  AND is_system_folder = false
);
```

### documents

```sql
-- SELECT: Tenant-basiert, nur sichtbare für Multiplikatoren
CREATE POLICY "documents_select" ON documents FOR SELECT USING (
  auth.is_internal_user()
  OR (auth.has_tenant_access(tenant_id) AND is_visible_to_tenant = true)
);

-- INSERT: Jeder für eigenen Tenant (Upload)
CREATE POLICY "documents_insert" ON documents FOR INSERT WITH CHECK (
  auth.is_internal_user()
  OR (auth.is_multiplikator() AND tenant_id = auth.tenant_id())
);

-- UPDATE: Interne oder Uploader für eigene Dokumente
CREATE POLICY "documents_update" ON documents FOR UPDATE USING (
  auth.is_internal_user()
  OR (auth.is_multiplikator() AND tenant_id = auth.tenant_id() AND uploaded_by_user_id = auth.uid())
);

-- DELETE: Nur Interne
CREATE POLICY "documents_delete" ON documents FOR DELETE USING (
  auth.is_internal_user()
);
```

### offers (Read-Only für Tenants)

```sql
CREATE POLICY "offers_select" ON offers FOR SELECT USING (
  auth.has_tenant_access(tenant_id)
);

-- INSERT/UPDATE/DELETE: Nur Interne (ERP-Sync)
CREATE POLICY "offers_insert" ON offers FOR INSERT WITH CHECK (
  auth.is_internal_user()
);

CREATE POLICY "offers_update" ON offers FOR UPDATE USING (
  auth.is_internal_user()
);

CREATE POLICY "offers_delete" ON offers FOR DELETE USING (
  auth.is_internal_user()
);
```

### invoices (Read-Only für Tenants)

```sql
CREATE POLICY "invoices_select" ON invoices FOR SELECT USING (
  auth.has_tenant_access(tenant_id)
);

CREATE POLICY "invoices_insert" ON invoices FOR INSERT WITH CHECK (
  auth.is_internal_user()
);

CREATE POLICY "invoices_update" ON invoices FOR UPDATE USING (
  auth.is_internal_user()
);

CREATE POLICY "invoices_delete" ON invoices FOR DELETE USING (
  auth.is_internal_user()
);
```

### comments

```sql
-- SELECT: Tenant-basiert, interne Kommentare nur für Interne
CREATE POLICY "comments_select" ON comments FOR SELECT USING (
  auth.is_internal_user()
  OR (auth.has_tenant_access(tenant_id) AND is_internal = false)
);

-- INSERT: Jeder für eigenen Tenant
CREATE POLICY "comments_insert" ON comments FOR INSERT WITH CHECK (
  auth.is_internal_user()
  OR (auth.is_multiplikator() AND tenant_id = auth.tenant_id())
);

-- UPDATE: Autor oder Interne
CREATE POLICY "comments_update" ON comments FOR UPDATE USING (
  auth.is_internal_user()
  OR (user_id = auth.uid() AND tenant_id = auth.tenant_id())
);

-- DELETE (Soft): Autor oder Interne
CREATE POLICY "comments_delete" ON comments FOR DELETE USING (
  auth.is_internal_user()
  OR (user_id = auth.uid() AND tenant_id = auth.tenant_id())
);
```

### notifications

```sql
-- SELECT: Nur eigene Notifications
CREATE POLICY "notifications_select" ON notifications FOR SELECT USING (
  user_id = auth.uid()
);

-- INSERT: System oder Interne
CREATE POLICY "notifications_insert" ON notifications FOR INSERT WITH CHECK (
  auth.is_internal_user()
);

-- UPDATE: Nur eigene (für read-Status)
CREATE POLICY "notifications_update" ON notifications FOR UPDATE USING (
  user_id = auth.uid()
);

-- DELETE: Nur eigene
CREATE POLICY "notifications_delete" ON notifications FOR DELETE USING (
  user_id = auth.uid()
);
```

### integration_mappings

```sql
-- Nur für Interne (System-Tabelle)
CREATE POLICY "integration_mappings_select" ON integration_mappings FOR SELECT USING (
  auth.is_internal_user()
);

CREATE POLICY "integration_mappings_insert" ON integration_mappings FOR INSERT WITH CHECK (
  auth.is_internal_user()
);

CREATE POLICY "integration_mappings_update" ON integration_mappings FOR UPDATE USING (
  auth.is_internal_user()
);

CREATE POLICY "integration_mappings_delete" ON integration_mappings FOR DELETE USING (
  auth.is_internal_user()
);
```

### claim_assignments

```sql
CREATE POLICY "claim_assignments_select" ON claim_assignments FOR SELECT USING (
  auth.has_tenant_access(tenant_id)
);

CREATE POLICY "claim_assignments_insert" ON claim_assignments FOR INSERT WITH CHECK (
  auth.is_internal_user()
);

CREATE POLICY "claim_assignments_update" ON claim_assignments FOR UPDATE USING (
  auth.is_internal_user()
);

CREATE POLICY "claim_assignments_delete" ON claim_assignments FOR DELETE USING (
  auth.is_internal_user()
);
```

### claim_contacts

```sql
CREATE POLICY "claim_contacts_select" ON claim_contacts FOR SELECT USING (
  auth.has_tenant_access(tenant_id)
);

CREATE POLICY "claim_contacts_insert" ON claim_contacts FOR INSERT WITH CHECK (
  auth.is_internal_user()
  OR (auth.is_multiplikator() AND tenant_id = auth.tenant_id())
);

CREATE POLICY "claim_contacts_update" ON claim_contacts FOR UPDATE USING (
  auth.is_internal_user()
  OR (auth.is_multiplikator() AND tenant_id = auth.tenant_id())
);

CREATE POLICY "claim_contacts_delete" ON claim_contacts FOR DELETE USING (
  auth.is_internal_user()
);
```

### audit_logs

```sql
-- SELECT: Interne sehen alle, Tenant-Admins eigene Tenant-Logs
CREATE POLICY "audit_logs_select" ON audit_logs FOR SELECT USING (
  auth.is_internal_user()
  OR (auth.is_tenant_admin() AND tenant_id = auth.tenant_id())
);

-- INSERT: Nur via Trigger (kein direkter User-Insert)
CREATE POLICY "audit_logs_insert" ON audit_logs FOR INSERT WITH CHECK (
  true -- Trigger-basiert
);

-- UPDATE/DELETE: Niemals (immutable)
CREATE POLICY "audit_logs_update" ON audit_logs FOR UPDATE USING (
  false
);

CREATE POLICY "audit_logs_delete" ON audit_logs FOR DELETE USING (
  false
);
```

---

## (F) Views

### v_tenant_claims (Portal-Liste)

```sql
CREATE OR REPLACE VIEW v_tenant_claims AS
SELECT 
  c.id,
  c.tenant_id,
  c.claim_number,
  c.insurance_claim_number,
  c.title,
  c.damage_type,
  c.damage_date,
  c.reported_date,
  c.address_street,
  c.address_zip,
  c.address_city,
  c.address_floor,
  c.address_unit,
  c.geo_lat,
  c.geo_lng,
  c.current_phase,
  c.current_status,
  c.priority,
  c.is_archived,
  c.created_at,
  c.updated_at,
  -- Aggregierte Counts
  (SELECT COUNT(*) FROM documents d WHERE d.claim_id = c.id AND d.is_visible_to_tenant = true) AS document_count,
  (SELECT COUNT(*) FROM comments cm WHERE cm.claim_id = c.id AND cm.is_internal = false AND cm.is_deleted = false) AS comment_count,
  (SELECT MAX(e.event_timestamp) FROM events e WHERE e.claim_id = c.id AND e.is_visible_to_tenant = true) AS last_activity_at
FROM claims c
WHERE auth.has_tenant_access(c.tenant_id);

-- Security Definer für Performance
GRANT SELECT ON v_tenant_claims TO authenticated;
```

### v_tenant_claim_detail (Portal-Detail)

```sql
CREATE OR REPLACE VIEW v_tenant_claim_detail AS
SELECT 
  c.id,
  c.tenant_id,
  c.property_id,
  c.claim_number,
  c.insurance_claim_number,
  c.title,
  c.description,
  c.damage_type,
  c.damage_cause,
  c.damage_date,
  c.reported_date,
  c.address_street,
  c.address_zip,
  c.address_city,
  c.address_floor,
  c.address_unit,
  c.geo_lat,
  c.geo_lng,
  c.current_phase,
  c.current_status,
  c.priority,
  c.is_archived,
  c.closed_at,
  c.created_at,
  c.updated_at,
  -- Phasen als JSON Array
  (
    SELECT jsonb_agg(
      jsonb_build_object(
        'id', cp.id,
        'phase_type', cp.phase_type,
        'status', cp.status,
        'sequence', cp.sequence,
        'is_applicable', cp.is_applicable,
        'is_skipped', cp.is_skipped,
        'started_at', cp.started_at,
        'completed_at', cp.completed_at
      ) ORDER BY cp.sequence
    )
    FROM claim_phases cp 
    WHERE cp.claim_id = c.id
  ) AS phases,
  -- Nächste Termine
  (
    SELECT jsonb_agg(
      jsonb_build_object(
        'id', ci.id,
        'appointment_type', ci.appointment_type,
        'title', ci.title,
        'start_datetime', ci.start_datetime,
        'end_datetime', ci.end_datetime,
        'status', ci.status,
        'location', ci.location
      ) ORDER BY ci.start_datetime
    )
    FROM calendar_items ci 
    WHERE ci.claim_id = c.id 
      AND ci.is_visible_to_tenant = true
      AND ci.start_datetime >= NOW()
      AND ci.status NOT IN ('STORNIERT', 'DURCHGEFUEHRT')
    LIMIT 5
  ) AS upcoming_appointments,
  -- Letzte Events (Timeline)
  (
    SELECT jsonb_agg(
      jsonb_build_object(
        'id', e.id,
        'event_type', e.event_type,
        'event_category', e.event_category,
        'title', e.title,
        'description', e.description,
        'event_timestamp', e.event_timestamp
      ) ORDER BY e.event_timestamp DESC
    )
    FROM events e 
    WHERE e.claim_id = c.id 
      AND e.is_visible_to_tenant = true
    LIMIT 20
  ) AS recent_events,
  -- Dokument-Zusammenfassung
  (
    SELECT jsonb_agg(
      jsonb_build_object(
        'id', d.id,
        'display_name', d.display_name,
        'document_type', d.document_type,
        'file_type', d.file_type,
        'created_at', d.created_at
      ) ORDER BY d.created_at DESC
    )
    FROM documents d 
    WHERE d.claim_id = c.id 
      AND d.is_visible_to_tenant = true
    LIMIT 10
  ) AS recent_documents,
  -- Kontakte
  (
    SELECT jsonb_agg(
      jsonb_build_object(
        'id', ct.id,
        'contact_type', ct.contact_type,
        'first_name', ct.first_name,
        'last_name', ct.last_name,
        'company', ct.company,
        'email', ct.email,
        'phone', ct.phone,
        'role_in_claim', cc.role_in_claim,
        'is_primary', cc.is_primary
      )
    )
    FROM claim_contacts cc
    JOIN contacts ct ON ct.id = cc.contact_id
    WHERE cc.claim_id = c.id
  ) AS contacts,
  -- Angebote (nur Status, keine Beträge!)
  (
    SELECT jsonb_agg(
      jsonb_build_object(
        'id', o.id,
        'offer_number', o.offer_number,
        'title', o.title,
        'status', o.status,
        'offer_date', o.offer_date,
        'valid_until', o.valid_until
      ) ORDER BY o.offer_date DESC
    )
    FROM offers o 
    WHERE o.claim_id = c.id
  ) AS offers,
  -- Rechnungen (nur Status, keine Beträge!)
  (
    SELECT jsonb_agg(
      jsonb_build_object(
        'id', i.id,
        'invoice_number', i.invoice_number,
        'title', i.title,
        'status', i.status,
        'invoice_date', i.invoice_date
      ) ORDER BY i.invoice_date DESC
    )
    FROM invoices i 
    WHERE i.claim_id = c.id
  ) AS invoices
FROM claims c
WHERE auth.has_tenant_access(c.tenant_id);

GRANT SELECT ON v_tenant_claim_detail TO authenticated;
```

### v_internal_claims (Voll für Interne)

```sql
CREATE OR REPLACE VIEW v_internal_claims AS
SELECT 
  c.*,
  -- Phasen
  (
    SELECT jsonb_agg(
      jsonb_build_object(
        'id', cp.id,
        'phase_type', cp.phase_type,
        'status', cp.status,
        'erp_substatus', cp.erp_substatus,
        'sequence', cp.sequence,
        'is_applicable', cp.is_applicable,
        'is_skipped', cp.is_skipped,
        'started_at', cp.started_at,
        'completed_at', cp.completed_at,
        'notes', cp.notes
      ) ORDER BY cp.sequence
    )
    FROM claim_phases cp 
    WHERE cp.claim_id = c.id
  ) AS phases,
  -- Alle Termine
  (
    SELECT jsonb_agg(
      jsonb_build_object(
        'id', ci.id,
        'appointment_type', ci.appointment_type,
        'title', ci.title,
        'start_datetime', ci.start_datetime,
        'end_datetime', ci.end_datetime,
        'status', ci.status,
        'location', ci.location,
        'assigned_team', ci.assigned_team,
        'notes', ci.notes,
        'is_visible_to_tenant', ci.is_visible_to_tenant
      ) ORDER BY ci.start_datetime
    )
    FROM calendar_items ci 
    WHERE ci.claim_id = c.id
  ) AS appointments,
  -- Angebote (MIT Beträgen)
  (
    SELECT jsonb_agg(
      jsonb_build_object(
        'id', o.id,
        'offer_number', o.offer_number,
        'title', o.title,
        'status', o.status,
        'net_amount', o.net_amount,
        'tax_amount', o.tax_amount,
        'gross_amount', o.gross_amount,
        'offer_date', o.offer_date,
        'valid_until', o.valid_until,
        'accepted_at', o.accepted_at
      ) ORDER BY o.offer_date DESC
    )
    FROM offers o 
    WHERE o.claim_id = c.id
  ) AS offers,
  -- Rechnungen (MIT Beträgen)
  (
    SELECT jsonb_agg(
      jsonb_build_object(
        'id', i.id,
        'invoice_number', i.invoice_number,
        'title', i.title,
        'status', i.status,
        'net_amount', i.net_amount,
        'tax_amount', i.tax_amount,
        'gross_amount', i.gross_amount,
        'invoice_date', i.invoice_date,
        'due_date', i.due_date,
        'paid_at', i.paid_at,
        'paid_amount', i.paid_amount
      ) ORDER BY i.invoice_date DESC
    )
    FROM invoices i 
    WHERE i.claim_id = c.id
  ) AS invoices,
  -- Zugewiesene Bearbeiter
  (
    SELECT jsonb_agg(
      jsonb_build_object(
        'user_id', u.id,
        'email', u.email,
        'first_name', u.first_name,
        'last_name', u.last_name,
        'role', ca.role,
        'assigned_at', ca.assigned_at
      )
    )
    FROM claim_assignments ca
    JOIN users u ON u.id = ca.user_id
    WHERE ca.claim_id = c.id AND ca.is_active = true
  ) AS assigned_users,
  -- Tenant-Info
  (
    SELECT jsonb_build_object(
      'id', t.id,
      'name', t.name,
      'type', t.type,
      'contact_email', t.contact_email
    )
    FROM tenants t
    WHERE t.id = c.tenant_id
  ) AS tenant,
  -- Summen
  (SELECT COALESCE(SUM(o.gross_amount), 0) FROM offers o WHERE o.claim_id = c.id AND o.status = 'ANGENOMMEN') AS total_offers_accepted,
  (SELECT COALESCE(SUM(i.gross_amount), 0) FROM invoices i WHERE i.claim_id = c.id) AS total_invoiced,
  (SELECT COALESCE(SUM(i.paid_amount), 0) FROM invoices i WHERE i.claim_id = c.id WHERE i.paid_amount IS NOT NULL) AS total_paid
FROM claims c
WHERE auth.is_internal_user();

GRANT SELECT ON v_internal_claims TO authenticated;
```

---

## (G) Konsistenz-Mechanik

### Trigger für current_phase / current_status Maintenance

```sql
-- =============================================
-- TRIGGER: Synchronisiert claim.current_phase/current_status mit claim_phases
-- =============================================

CREATE OR REPLACE FUNCTION fn_sync_claim_current_phase()
RETURNS TRIGGER AS $$
DECLARE
  v_current_phase phase_type;
  v_current_status phase_status;
BEGIN
  -- Finde die erste nicht-abgeschlossene, anwendbare Phase
  SELECT cp.phase_type, cp.status
  INTO v_current_phase, v_current_status
  FROM claim_phases cp
  WHERE cp.claim_id = COALESCE(NEW.claim_id, OLD.claim_id)
    AND cp.is_applicable = true
    AND cp.is_skipped = false
    AND cp.status != 'ABGESCHLOSSEN'
  ORDER BY cp.sequence ASC
  LIMIT 1;

  -- Wenn alle Phasen abgeschlossen, nimm die letzte
  IF v_current_phase IS NULL THEN
    SELECT cp.phase_type, cp.status
    INTO v_current_phase, v_current_status
    FROM claim_phases cp
    WHERE cp.claim_id = COALESCE(NEW.claim_id, OLD.claim_id)
      AND cp.is_applicable = true
      AND cp.is_skipped = false
    ORDER BY cp.sequence DESC
    LIMIT 1;
  END IF;

  -- Update claim nur wenn sich etwas geändert hat
  UPDATE claims
  SET 
    current_phase = COALESCE(v_current_phase, current_phase),
    current_status = COALESCE(v_current_status, current_status),
    updated_at = now()
  WHERE id = COALESCE(NEW.claim_id, OLD.claim_id)
    AND (current_phase IS DISTINCT FROM v_current_phase 
         OR current_status IS DISTINCT FROM v_current_status);

  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER trg_claim_phases_sync_current
AFTER INSERT OR UPDATE OF status, is_applicable, is_skipped OR DELETE
ON claim_phases
FOR EACH ROW
EXECUTE FUNCTION fn_sync_claim_current_phase();
```

### Trigger für automatische ClaimPhases-Erstellung

```sql
-- =============================================
-- TRIGGER: Erstellt 5 ClaimPhases bei neuem Claim
-- =============================================

CREATE OR REPLACE FUNCTION fn_create_claim_phases()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO claim_phases (claim_id, tenant_id, phase_type, status, sequence)
  VALUES
    (NEW.id, NEW.tenant_id, 'LECKORTUNG', 'NICHT_BEGONNEN', 1),
    (NEW.id, NEW.tenant_id, 'LECKAGE_BEHEBUNG', 'NICHT_BEGONNEN', 2),
    (NEW.id, NEW.tenant_id, 'RUECKBAU', 'NICHT_BEGONNEN', 3),
    (NEW.id, NEW.tenant_id, 'TROCKNUNG', 'NICHT_BEGONNEN', 4),
    (NEW.id, NEW.tenant_id, 'WIEDERHERSTELLUNG', 'NICHT_BEGONNEN', 5);
  
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER trg_claims_create_phases
AFTER INSERT ON claims
FOR EACH ROW
EXECUTE FUNCTION fn_create_claim_phases();
```

### Trigger für Event-Logging bei Statusänderung

```sql
-- =============================================
-- TRIGGER: Event bei Phasen-/Statusänderung
-- =============================================

CREATE OR REPLACE FUNCTION fn_log_phase_change()
RETURNS TRIGGER AS $$
BEGIN
  IF OLD.status IS DISTINCT FROM NEW.status THEN
    INSERT INTO events (
      claim_id, 
      tenant_id,
      event_type, 
      event_category, 
      title, 
      description,
      metadata,
      is_visible_to_tenant,
      event_timestamp
    )
    VALUES (
      NEW.claim_id,
      NEW.tenant_id,
      'STATUS_CHANGED',
      'STATUS',
      format('Status geändert: %s → %s', OLD.status, NEW.status),
      format('Phase %s: Status wurde von %s auf %s geändert', NEW.phase_type, OLD.status, NEW.status),
      jsonb_build_object(
        'phase_type', NEW.phase_type,
        'old_status', OLD.status,
        'new_status', NEW.status
      ),
      true,
      now()
    );
  END IF;
  
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER trg_claim_phases_log_change
AFTER UPDATE OF status ON claim_phases
FOR EACH ROW
EXECUTE FUNCTION fn_log_phase_change();
```

### Trigger für updated_at

```sql
-- =============================================
-- TRIGGER: Automatisches updated_at
-- =============================================

CREATE OR REPLACE FUNCTION fn_set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Anwenden auf alle relevanten Tabellen
CREATE TRIGGER trg_tenants_updated_at BEFORE UPDATE ON tenants FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
CREATE TRIGGER trg_users_updated_at BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
CREATE TRIGGER trg_properties_updated_at BEFORE UPDATE ON properties FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
CREATE TRIGGER trg_contacts_updated_at BEFORE UPDATE ON contacts FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
CREATE TRIGGER trg_claims_updated_at BEFORE UPDATE ON claims FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
CREATE TRIGGER trg_claim_phases_updated_at BEFORE UPDATE ON claim_phases FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
CREATE TRIGGER trg_calendar_items_updated_at BEFORE UPDATE ON calendar_items FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
CREATE TRIGGER trg_document_folders_updated_at BEFORE UPDATE ON document_folders FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
CREATE TRIGGER trg_documents_updated_at BEFORE UPDATE ON documents FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
CREATE TRIGGER trg_offers_updated_at BEFORE UPDATE ON offers FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
CREATE TRIGGER trg_invoices_updated_at BEFORE UPDATE ON invoices FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
CREATE TRIGGER trg_comments_updated_at BEFORE UPDATE ON comments FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
CREATE TRIGGER trg_integration_mappings_updated_at BEFORE UPDATE ON integration_mappings FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
```

---

## (H) Dokument-Inbox

### Design-Entscheidung

Inbox-Dokumente werden in der `documents`-Tabelle mit `claim_id = NULL` und `is_inbox = true` gespeichert. Ein Constraint stellt Konsistenz sicher.

### Inbox-spezifische Funktionen

```sql
-- =============================================
-- FUNCTION: Dokument aus Inbox einem Claim zuordnen
-- =============================================

CREATE OR REPLACE FUNCTION fn_assign_inbox_document(
  p_document_id UUID,
  p_claim_id UUID,
  p_folder_id UUID DEFAULT NULL,
  p_document_type document_type DEFAULT NULL
)
RETURNS documents AS $$
DECLARE
  v_document documents;
  v_claim_tenant_id UUID;
BEGIN
  -- Prüfe Claim existiert und hole tenant_id
  SELECT tenant_id INTO v_claim_tenant_id
  FROM claims WHERE id = p_claim_id;
  
  IF v_claim_tenant_id IS NULL THEN
    RAISE EXCEPTION 'Claim not found: %', p_claim_id;
  END IF;

  -- Update Dokument
  UPDATE documents
  SET 
    claim_id = p_claim_id,
    folder_id = p_folder_id,
    document_type = COALESCE(p_document_type, document_type),
    is_inbox = false,
    ocr_assignment_status = 'MANUAL_ASSIGNED',
    updated_at = now()
  WHERE id = p_document_id
    AND is_inbox = true
    AND tenant_id = v_claim_tenant_id
  RETURNING * INTO v_document;

  IF v_document IS NULL THEN
    RAISE EXCEPTION 'Document not found or not in inbox: %', p_document_id;
  END IF;

  -- Event erstellen
  INSERT INTO events (claim_id, tenant_id, event_type, event_category, title, reference_type, reference_id, is_visible_to_tenant)
  VALUES (p_claim_id, v_claim_tenant_id, 'DOCUMENT_ASSIGNED', 'DOKUMENT', 
          format('Dokument zugeordnet: %s', v_document.display_name),
          'document', p_document_id, true);

  RETURN v_document;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- =============================================
-- VIEW: Inbox-Dokumente pro Tenant
-- =============================================

CREATE OR REPLACE VIEW v_document_inbox AS
SELECT 
  d.id,
  d.tenant_id,
  d.file_name,
  d.display_name,
  d.file_type,
  d.file_size,
  d.storage_path,
  d.document_type,
  d.ocr_status,
  d.ocr_confidence,
  d.ocr_extracted_data,
  d.uploaded_by_user_id,
  d.created_at,
  -- OCR-basierte Vorschläge für Zuordnung
  d.ocr_extracted_data->>'suggested_claim_number' AS suggested_claim_number,
  d.ocr_extracted_data->>'suggested_document_type' AS suggested_document_type,
  d.ocr_extracted_data->>'extracted_date' AS extracted_date,
  -- Uploader Info
  u.email AS uploaded_by_email,
  u.first_name || ' ' || u.last_name AS uploaded_by_name
FROM documents d
LEFT JOIN users u ON u.id = d.uploaded_by_user_id
WHERE d.is_inbox = true
  AND auth.has_tenant_access(d.tenant_id);

GRANT SELECT ON v_document_inbox TO authenticated;
```

### Trigger für automatische Zuordnung bei OCR-Completion

```sql
-- =============================================
-- TRIGGER: Auto-Assign nach OCR wenn Confidence hoch
-- =============================================

CREATE OR REPLACE FUNCTION fn_auto_assign_after_ocr()
RETURNS TRIGGER AS $$
DECLARE
  v_claim_id UUID;
  v_suggested_claim_number TEXT;
BEGIN
  -- Nur bei OCR-Completion für Inbox-Dokumente
  IF NEW.ocr_status = 'COMPLETED' 
     AND NEW.is_inbox = true 
     AND OLD.ocr_status != 'COMPLETED' THEN
    
    -- Prüfe Confidence
    IF NEW.ocr_confidence >= 85 THEN
      v_suggested_claim_number := NEW.ocr_extracted_data->>'claim_number';
      
      IF v_suggested_claim_number IS NOT NULL THEN
        -- Versuche Claim zu finden
        SELECT id INTO v_claim_id
        FROM claims
        WHERE claim_number = v_suggested_claim_number
          AND tenant_id = NEW.tenant_id;
        
        IF v_claim_id IS NOT NULL THEN
          -- Auto-Assign
          UPDATE documents
          SET 
            claim_id = v_claim_id,
            is_inbox = false,
            ocr_assignment_status = 'AUTO_ASSIGNED'
          WHERE id = NEW.id;
          
          -- Event erstellen
          INSERT INTO events (claim_id, tenant_id, event_type, event_category, title, reference_type, reference_id, metadata, is_visible_to_tenant)
          VALUES (v_claim_id, NEW.tenant_id, 'DOCUMENT_ASSIGNED', 'DOKUMENT', 
                  format('Dokument automatisch zugeordnet: %s', NEW.display_name),
                  'document', NEW.id, 
                  jsonb_build_object('assignment_type', 'AUTO', 'ocr_confidence', NEW.ocr_confidence),
                  true);
          
          RETURN NEW;
        END IF;
      END IF;
      
      -- Keine Auto-Zuordnung möglich
      UPDATE documents
      SET ocr_assignment_status = 'MANUAL_REQUIRED'
      WHERE id = NEW.id;
      
    ELSE
      -- Confidence zu niedrig
      UPDATE documents
      SET ocr_assignment_status = 'MANUAL_REQUIRED'
      WHERE id = NEW.id;
    END IF;
  END IF;
  
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER trg_documents_auto_assign_ocr
AFTER UPDATE OF ocr_status ON documents
FOR EACH ROW
EXECUTE FUNCTION fn_auto_assign_after_ocr();
```

---

## (I) Storage Design

### Bucket-Struktur

```
supabase-storage/
├── documents/                          # Haupt-Bucket für Dokumente
│   ├── tenants/
│   │   └── {tenant_id}/
│   │       ├── claims/
│   │       │   └── {claim_id}/
│   │       │       ├── protokolle/
│   │       │       ├── angebote/
│   │       │       ├── rechnungen/
│   │       │       ├── fotos/
│   │       │       ├── gutachten/
│   │       │       ├── korrespondenz/
│   │       │       └── sonstige/
│   │       └── inbox/                  # Unzugeordnete Dokumente
│   │           └── {document_id}_{filename}
│   └── internal/                       # Nur für interne User
│       └── {claim_id}/...
├── avatars/                            # User-Profilbilder
│   └── {user_id}.{ext}
└── logos/                              # Tenant-Logos
    └── {tenant_id}.{ext}
```

### Bucket-Erstellung

```sql
-- Via Supabase Dashboard oder SQL
INSERT INTO storage.buckets (id, name, public, file_size_limit, allowed_mime_types)
VALUES 
  ('documents', 'documents', false, 52428800, -- 50MB
   ARRAY['application/pdf', 'image/jpeg', 'image/png', 'image/webp', 'image/heic', 
         'application/msword', 'application/vnd.openxmlformats-officedocument.wordprocessingml.document']),
  ('avatars', 'avatars', true, 2097152, -- 2MB
   ARRAY['image/jpeg', 'image/png', 'image/webp']),
  ('logos', 'logos', true, 5242880, -- 5MB
   ARRAY['image/jpeg', 'image/png', 'image/svg+xml', 'image/webp']);
```

### Storage Policies

```sql
-- =============================================
-- DOCUMENTS BUCKET POLICIES
-- =============================================

-- SELECT: Tenant kann eigene Claim-Dokumente lesen, Interne alles
CREATE POLICY "documents_select" ON storage.objects FOR SELECT USING (
  bucket_id = 'documents' AND (
    -- Interne User: Zugriff auf alles
    auth.is_internal_user()
    OR
    -- Tenant: Zugriff auf eigene Dokumente (Pfad enthält tenant_id)
    (
      auth.tenant_id() IS NOT NULL 
      AND (storage.foldername(name))[2] = auth.tenant_id()::text
    )
  )
);

-- INSERT: Tenant kann in eigenen Bereich hochladen
CREATE POLICY "documents_insert" ON storage.objects FOR INSERT WITH CHECK (
  bucket_id = 'documents' AND (
    auth.is_internal_user()
    OR
    (
      auth.is_multiplikator()
      AND (storage.foldername(name))[2] = auth.tenant_id()::text
    )
  )
);

-- UPDATE: Interne oder Uploader (Metadaten)
CREATE POLICY "documents_update" ON storage.objects FOR UPDATE USING (
  bucket_id = 'documents' AND (
    auth.is_internal_user()
    OR
    (
      auth.is_multiplikator()
      AND (storage.foldername(name))[2] = auth.tenant_id()::text
      AND owner = auth.uid()
    )
  )
);

-- DELETE: Nur Interne
CREATE POLICY "documents_delete" ON storage.objects FOR DELETE USING (
  bucket_id = 'documents' AND auth.is_internal_user()
);

-- =============================================
-- AVATARS BUCKET POLICIES (Public Read, Owner Write)
-- =============================================

CREATE POLICY "avatars_select" ON storage.objects FOR SELECT USING (
  bucket_id = 'avatars'
);

CREATE POLICY "avatars_insert" ON storage.objects FOR INSERT WITH CHECK (
  bucket_id = 'avatars' 
  AND (storage.filename(name))::uuid = auth.uid()
);

CREATE POLICY "avatars_update" ON storage.objects FOR UPDATE USING (
  bucket_id = 'avatars' 
  AND (storage.filename(name))::uuid = auth.uid()
);

CREATE POLICY "avatars_delete" ON storage.objects FOR DELETE USING (
  bucket_id = 'avatars' 
  AND (storage.filename(name))::uuid = auth.uid()
);

-- =============================================
-- LOGOS BUCKET POLICIES (Public Read, Tenant Admin Write)
-- =============================================

CREATE POLICY "logos_select" ON storage.objects FOR SELECT USING (
  bucket_id = 'logos'
);

CREATE POLICY "logos_insert" ON storage.objects FOR INSERT WITH CHECK (
  bucket_id = 'logos' 
  AND (
    auth.role() = 'SYSTEM_ADMIN'
    OR (auth.is_tenant_admin() AND (storage.filename(name))::uuid = auth.tenant_id())
  )
);

CREATE POLICY "logos_update" ON storage.objects FOR UPDATE USING (
  bucket_id = 'logos' 
  AND (
    auth.role() = 'SYSTEM_ADMIN'
    OR (auth.is_tenant_admin() AND (storage.filename(name))::uuid = auth.tenant_id())
  )
);

CREATE POLICY "logos_delete" ON storage.objects FOR DELETE USING (
  bucket_id = 'logos' 
  AND (
    auth.role() = 'SYSTEM_ADMIN'
    OR (auth.is_tenant_admin() AND (storage.filename(name))::uuid = auth.tenant_id())
  )
);
```

### Helper Function für Storage Path Generation

```sql
-- =============================================
-- FUNCTION: Generiert Storage-Pfad für Dokumente
-- =============================================

CREATE OR REPLACE FUNCTION fn_generate_document_storage_path(
  p_tenant_id UUID,
  p_claim_id UUID,      -- NULL für Inbox
  p_folder_type folder_type,
  p_document_id UUID,
  p_filename TEXT
)
RETURNS TEXT AS $$
DECLARE
  v_safe_filename TEXT;
  v_folder_name TEXT;
BEGIN
  -- Sanitize filename
  v_safe_filename := regexp_replace(p_filename, '[^a-zA-Z0-9._-]', '_', 'g');
  v_folder_name := lower(p_folder_type::text);
  
  IF p_claim_id IS NULL THEN
    -- Inbox-Pfad
    RETURN format('tenants/%s/inbox/%s_%s', p_tenant_id, p_document_id, v_safe_filename);
  ELSE
    -- Claim-Pfad
    RETURN format('tenants/%s/claims/%s/%s/%s_%s', 
                  p_tenant_id, p_claim_id, v_folder_name, p_document_id, v_safe_filename);
  END IF;
END;
$$ LANGUAGE plpgsql IMMUTABLE;
```

---

## Assumptions & Decisions

### Assumptions

- **Auth-Struktur:** `auth.users.raw_app_meta_data` enthält `role` (user_role ENUM) und `tenant_id` (UUID, NULL für interne User).
- **Service Role:** ERP/n8n Sync verwendet Supabase Service Role Key → bypassed RLS automatisch.
- **PostGIS:** Extension `postgis` ist verfügbar für Geo-Queries.
- **Supabase Storage:** Standard Supabase Storage mit `storage.objects` Tabelle.
- **Timezone:** Alle Timestamps sind `TIMESTAMPTZ` (UTC).

### Design Decisions

| ID | Entscheidung | Begründung |
|----|--------------|------------|
| **D-001** | `tenant_id` redundant in allen Tabellen | Vermeidet JOINs in RLS-Policies → bessere Performance. Trigger/App-Logik stellt Konsistenz sicher. |
| **D-002** | `current_phase/current_status` in Claims wird via Trigger maintained (nicht computed View) | Häufiger Read-Zugriff, seltenere Writes. Trigger-Overhead bei Phase-Updates akzeptabel. |
| **D-003** | Inbox-Dokumente in `documents` Tabelle mit `is_inbox = true` statt separate Tabelle | Einheitliches Schema, einfachere Queries, Constraint sichert Konsistenz (`claim_id` NULL ↔ `is_inbox` true). |
| **D-004** | `ocr_assignment_status` ENUM erweitert um `UNASSIGNED` | A1 hatte kein explizites "noch nicht verarbeitet" State. |
| **D-005** | Offers/Invoices: Beträge werden für Tenants in Views ausgeblendet, nicht in RLS | RLS filtert Zeilen, nicht Spalten. Views bieten bessere Kontrolle über sichtbare Felder. |
| **D-006** | `folder_type` ENUM erweitert um `INBOX` | Für interne Kategorisierung von Inbox-Dokumenten vor Zuordnung. |
| **D-007** | `claim_phases.tenant_id` zusätzlich zu `claim_id` | RLS-Performance: direkter Tenant-Check ohne JOIN auf claims. |
| **D-008** | Keine separate `inbound_documents` Tabelle | Weniger Komplexität. Document-Move wird zu Update statt Insert+Delete. Storage-Pfad ändert sich bei Zuordnung (muss in App gehandelt werden). |
| **D-009** | Storage-Pfad enthält document_id als Prefix | Verhindert Namenskollisionen, ermöglicht direktes Lookup. |
| **D-010** | `SACHVERSTAENDIGER` Rolle für externe Gutachter | Aus A1 übernommen, RLS behandelt wie Multiplikator mit tenant_id. |
| **D-011** | AuditLog immutable (kein UPDATE/DELETE Policy) | Compliance-Anforderung, Audit Trail muss unveränderlich sein. |
| **D-012** | Notifications haben optionalen `tenant_id` | NULL für Notifications an interne User ohne Tenant-Bezug. |

### Korrekturen zu A1

| Korrektur | Beschreibung |
|-----------|--------------|
| `Document.claim_id` jetzt nullable | A1 hatte `claim_id` als Pflicht. Für Inbox-Funktion muss es nullable sein mit Constraint. |
| `ocr_assignment_status` neuer Wert `UNASSIGNED` | Initial-Status vor OCR-Verarbeitung fehlte in A1. |
| `users.id` nicht mit DEFAULT | Muss explizit = `auth.users.id` gesetzt werden bei Erstellung. |
```