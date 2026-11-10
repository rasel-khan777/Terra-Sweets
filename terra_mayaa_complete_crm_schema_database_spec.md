# Terra Mayaa (তেʼৰা মায়া) — Hospitality CRM & Operations Database Specification
**Version:** 4.0.0-PROD  
**Database Engine:** PostgreSQL 16+ (Supabase / AWS RDS Aurora compatible)  
**Security Model:** Role-Based Access Control (RBAC) + PostgreSQL Row-Level Security (RLS)  
**Timezone:** `Asia/Kolkata` (IST)  
**Currency Standard:** INR (₹) represented in numeric(12,2) with integer paisa conversion support

---

## 1. Architectural Overview & Entity-Relationship Architecture

The Terra Mayaa database ecosystem is structured into seven tightly decoupled, transaction-safe domains:

```
[ GUESTS & PROFILES ] <──────> [ RESERVATIONS & FLOOR ] <──────> [ POS & TABLE ORDERS ]
         │                               │                                │
         ├──────> [ CRM & MARKETING ]    └──────> [ INVENTORY & BEV ]     │
         │                                                                │
         └──────> [ LOYALTY & REWARDS ] <─────────────────────────────────┘
```

1. **Identity & Core Customer CRM (`guests`, `guest_preferences`, `guest_dietary_safety`)**
2. **Floor Operations & Table Inventory (`zones`, `restaurant_tables`, `table_combinations`, `reservations`)**
3. **Menu, Culinary & Brewery Ledger (`menu_categories`, `menu_items`, `menu_variants`, `beverage_kegs`)**
4. **Orders, Folios & Payments (`dining_folios`, `folio_line_items`, `folio_payments`, `kitchen_tickets`)**
5. **Loyalty, Circles & Rewards (`loyalty_accounts`, `loyalty_transactions`, `privilege_vouchers`)**
6. **Host & Sommelier Service Dossier (`guest_host_notes`, `service_briefings`, `audit_logs`)**
7. **Marketing, Messaging & WhatsApp Automation (`campaigns`, `message_dispatches`, `automated_triggers`)**

---

## 2. Production PostgreSQL DDL Schema

```sql
-- ============================================================================
-- TERRA MAYAA RESTAURANT & CRM MANAGEMENT SYSTEM
-- MASTER PRODUCTION DATABASE SCHEMA (POSTGRESQL 16+)
-- ============================================================================

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "citext";

-- ----------------------------------------------------------------------------
-- ENUMS & TYPES
-- ----------------------------------------------------------------------------

CREATE TYPE guest_tier AS ENUM ('REGULAR', 'SILVER', 'GOLD_VIP', 'PLATINUM_VIP', 'ROYAL_AMBASSADOR');
CREATE TYPE reservation_status AS ENUM ('PENDING', 'CONFIRMED', 'ARRIVED', 'SEATED', 'COMPLETED', 'CANCELLED', 'NO_SHOW');
CREATE TYPE table_status AS ENUM ('AVAILABLE', 'RESERVED', 'OCCUPIED', 'TURNOVER_CLEANING', 'BLOCKED_MAINTENANCE');
CREATE TYPE zone_type AS ENUM ('SUNSET_ROOFTOP_DECK', 'DECK_CABANA', 'THE_VAULT_VIP', 'BREWERY_TAP_BAR', 'INDOOR_LUXURY_DINING');
CREATE TYPE dietary_lifestyle AS ENUM ('VEGETARIAN', 'NON_VEGETARIAN', 'VEGAN', 'JAIN', 'PESCATARIAN', 'HALAL');
CREATE TYPE spice_tolerance AS ENUM ('MILD', 'MEDIUM', 'ELEVATED', 'AUTHENTIC_ASSAM_JOLOKIA');
CREATE TYPE payment_mode AS ENUM ('UPI', 'CREDIT_CARD', 'DEBIT_CARD', 'CASH', 'AMEX', 'HOUSE_FOLIO_CREDIT');
CREATE TYPE payment_status AS ENUM ('PENDING', 'SETTLED', 'PARTIAL', 'REFUNDED', 'VOIDED');
CREATE TYPE order_destination AS ENUM ('KITCHEN_HOT', 'KITCHEN_COLD_SUSHI', 'WOODFIRE_PIZZA_OVEN', 'ROOFTOP_BREWERY_BAR', 'DESSERT_STATION');
CREATE TYPE message_channel AS ENUM ('WHATSAPP', 'SMS', 'EMAIL', 'PUSH');
CREATE TYPE dispatch_status AS ENUM ('QUEUED', 'SENT', 'DELIVERED', 'READ', 'FAILED', 'OPTED_OUT');

-- ----------------------------------------------------------------------------
-- 1. GUESTS & CUSTOMER CRM
-- ----------------------------------------------------------------------------

CREATE TABLE guests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guest_code VARCHAR(32) UNIQUE NOT NULL, -- e.g. TM-8821
    salutation VARCHAR(16),                 -- 'Dr.', 'Mr.', 'Ms.', 'Justice'
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    full_name VARCHAR(210) GENERATED ALWAYS AS (
        TRIM(COALESCE(salutation || ' ', '') || first_name || ' ' || last_name)
    ) STORED,
    phone_country_code VARCHAR(8) DEFAULT '+91',
    mobile_number VARCHAR(20) NOT NULL,
    alternate_phone VARCHAR(20),
    email CITEXT,
    designation VARCHAR(120),               -- e.g. 'Chief Cardiologist, Excel Care Hospital'
    company_affiliation VARCHAR(160),
    date_of_birth DATE,
    wedding_anniversary DATE,
    home_city VARCHAR(100) DEFAULT 'Guwahati',
    pin_code VARCHAR(10) DEFAULT '781005',
    tier guest_tier DEFAULT 'REGULAR' NOT NULL,
    is_vip BOOLEAN DEFAULT FALSE NOT NULL,
    total_lifetime_spend NUMERIC(12, 2) DEFAULT 0.00 NOT NULL,
    total_visits_count INTEGER DEFAULT 0 NOT NULL,
    completed_covers_count INTEGER DEFAULT 0 NOT NULL,
    no_show_count INTEGER DEFAULT 0 NOT NULL,
    cancellation_count INTEGER DEFAULT 0 NOT NULL,
    first_visited_at TIMESTAMPTZ,
    last_visited_at TIMESTAMPTZ,
    preferred_communication_channel message_channel DEFAULT 'WHATSAPP' NOT NULL,
    marketing_consent BOOLEAN DEFAULT TRUE NOT NULL,
    privacy_consent_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    CONSTRAINT chk_mobile_format CHECK (length(mobile_number) >= 8)
);

CREATE UNIQUE INDEX idx_guests_mobile ON guests(phone_country_code, mobile_number);
CREATE INDEX idx_guests_email ON guests(email) WHERE email IS NOT NULL;
CREATE INDEX idx_guests_tier_spend ON guests(tier, total_lifetime_spend DESC);
CREATE INDEX idx_guests_last_visit ON guests(last_visited_at DESC NULLS LAST);
CREATE INDEX idx_guests_birthday ON guests(EXTRACT(MONTH FROM date_of_birth), EXTRACT(DAY FROM date_of_birth));

-- ----------------------------------------------------------------------------
-- 2. GUEST CULINARY, SOMMELIER & BEHAVIORAL PREFERENCES
-- ----------------------------------------------------------------------------

CREATE TABLE guest_preferences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guest_id UUID NOT NULL REFERENCES guests(id) ON DELETE CASCADE,
    lifestyle dietary_lifestyle DEFAULT 'NON_VEGETARIAN' NOT NULL,
    spice_level spice_tolerance DEFAULT 'MEDIUM' NOT NULL,
    favorite_cuisine VARCHAR(80),               -- 'Neo-Indian', 'Pan-Asian', 'Wood-fired'
    favorite_signature_dish VARCHAR(120),       -- 'TM Dhuni Butter Chicken'
    favorite_cocktail_or_brew VARCHAR(120),     -- 'Clarified Cold-Brew Espresso Martini'
    arrival_beverage_ritual TEXT,               -- 'Serve sparkling water in chilled glassware upon arrival'
    preferred_zone zone_type DEFAULT 'SUNSET_ROOFTOP_DECK',
    preferred_table_id UUID,                    -- FK reference added post table creation
    favorite_table_label VARCHAR(64) DEFAULT 'Deck Cabana T-14',
    special_dining_requests TEXT,               -- 'Requires unobstructed sunset view; dislikes loud DJ speakers'
    sugar_free_beverages BOOLEAN DEFAULT FALSE,
    low_sodium_preferred BOOLEAN DEFAULT FALSE,
    verified_by_chef BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    CONSTRAINT uq_guest_preferences UNIQUE(guest_id)
);

CREATE TABLE guest_allergens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guest_id UUID NOT NULL REFERENCES guests(id) ON DELETE CASCADE,
    allergen_type VARCHAR(64) NOT NULL,         -- 'CRUSTACEANS', 'PEANUTS', 'DAIRY', 'GLUTEN', 'SOY', 'SULPHITES'
    severity_level VARCHAR(32) DEFAULT 'SEVERE', -- 'MILD_INTOLERANCE', 'SEVERE_ALLERGY', 'ANAPHYLACTIC'
    verified_by_staff_id UUID,
    service_alert_note TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);

CREATE INDEX idx_allergens_guest ON guest_allergens(guest_id);

-- ----------------------------------------------------------------------------
-- 3. FLOOR PLAN, ZONES & PHYSICAL TABLE ASSETS
-- ----------------------------------------------------------------------------

CREATE TABLE zones (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    zone_code VARCHAR(32) UNIQUE NOT NULL,      -- 'ROOFTOP_DECK', 'CABANAS', 'VAULT', 'BREWERY_BAR'
    name VARCHAR(100) NOT NULL,
    zone_category zone_type NOT NULL,
    floor_level VARCHAR(32) DEFAULT '6th Floor Rooftop' NOT NULL,
    total_capacity_seats INTEGER NOT NULL,
    is_smoking_permitted BOOLEAN DEFAULT FALSE,
    has_acoustic_live_view BOOLEAN DEFAULT TRUE,
    has_unobstructed_sunset_view BOOLEAN DEFAULT FALSE,
    is_air_conditioned BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE NOT NULL
);

CREATE TABLE restaurant_tables (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    zone_id UUID NOT NULL REFERENCES zones(id) ON DELETE RESTRICT,
    table_number VARCHAR(32) UNIQUE NOT NULL,   -- 'T-01', 'T-14', 'CAB-01', 'V-01', 'B-04'
    min_covers INTEGER DEFAULT 2 NOT NULL,
    max_covers INTEGER DEFAULT 4 NOT NULL,
    optimal_covers INTEGER DEFAULT 2 NOT NULL,
    status table_status DEFAULT 'AVAILABLE' NOT NULL,
    is_vip_reserved_only BOOLEAN DEFAULT FALSE,
    shape VARCHAR(32) DEFAULT 'RECTANGULAR',    -- 'ROUND', 'RECTANGULAR', 'HIGH_TOP_BOOTH', 'CABANA'
    pos_grid_x INTEGER DEFAULT 0,
    pos_grid_y INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    CONSTRAINT chk_covers CHECK (max_covers >= min_covers)
);

CREATE INDEX idx_tables_zone_status ON restaurant_tables(zone_id, status);

ALTER TABLE guest_preferences 
    ADD CONSTRAINT fk_preferred_table 
    FOREIGN KEY (preferred_table_id) REFERENCES restaurant_tables(id) ON DELETE SET NULL;

-- ----------------------------------------------------------------------------
-- 4. RESERVATIONS & GUEST BOOKINGS
-- ----------------------------------------------------------------------------

CREATE TABLE reservations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_reference VARCHAR(32) UNIQUE NOT NULL, -- e.g. TM-RES-20251028-4091
    guest_id UUID NOT NULL REFERENCES guests(id) ON DELETE RESTRICT,
    table_id UUID REFERENCES restaurant_tables(id) ON DELETE SET NULL,
    zone_id UUID REFERENCES zones(id) ON DELETE RESTRICT,
    reservation_date DATE NOT NULL,
    shift_name VARCHAR(32) DEFAULT 'DINNER' NOT NULL, -- 'LUNCH_DINE', 'SUNSET_HIGH_TEA', 'ROOFTOP_DINNER'
    slot_time TIME NOT NULL,
    party_size INTEGER NOT NULL,
    status reservation_status DEFAULT 'CONFIRMED' NOT NULL,
    special_occasion VARCHAR(64),               -- 'BIRTHDAY', 'ANNIVERSARY', 'CORPORATE_GALA', 'CASUAL_DINING'
    host_notes TEXT,                            -- 'Pre-chill flute for Dom Perignon; Guest prefers Bikash as server'
    dietary_summary_tag VARCHAR(120),           -- 'Mild spice & low sugar (No Peanuts)'
    grace_period_minutes INTEGER DEFAULT 15,
    checked_in_at TIMESTAMPTZ,
    seated_at TIMESTAMPTZ,
    departed_at TIMESTAMPTZ,
    created_by_staff_id UUID,
    cancellation_reason TEXT,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);

CREATE INDEX idx_reservations_date_slot ON reservations(reservation_date, slot_time, status);
CREATE INDEX idx_reservations_guest ON reservations(guest_id, reservation_date DESC);
CREATE INDEX idx_reservations_table ON reservations(table_id, reservation_date);

-- ----------------------------------------------------------------------------
-- 5. MENU, RECIPES & CRAFT BREWERY INVENTORY
-- ----------------------------------------------------------------------------

CREATE TABLE menu_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug VARCHAR(64) UNIQUE NOT NULL,
    display_name VARCHAR(100) NOT NULL,
    assamese_title VARCHAR(120),
    sort_order INTEGER DEFAULT 1,
    kitchen_destination order_destination DEFAULT 'KITCHEN_HOT',
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE menu_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category_id UUID NOT NULL REFERENCES menu_categories(id) ON DELETE RESTRICT,
    sku_code VARCHAR(32) UNIQUE NOT NULL,       -- 'TM-DISH-001'
    name VARCHAR(160) NOT NULL,
    assamese_name VARCHAR(160),
    short_description TEXT,
    lifestyle dietary_lifestyle DEFAULT 'NON_VEGETARIAN' NOT NULL,
    is_chef_signature BOOLEAN DEFAULT FALSE,
    is_gluten_free BOOLEAN DEFAULT FALSE,
    base_price NUMERIC(10, 2) NOT NULL,
    gst_rate_percent NUMERIC(4, 2) DEFAULT 5.00 NOT NULL,
    prep_time_minutes INTEGER DEFAULT 15,
    is_available BOOLEAN DEFAULT TRUE NOT NULL,
    beverage_pairing_suggestion VARCHAR(160),
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);

CREATE TABLE beverage_kegs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tap_number INTEGER UNIQUE NOT NULL,         -- Tap 1 to 6
    brew_name VARCHAR(120) NOT NULL,            -- 'Kamakshya Golden Ale', 'Brahmaputra Belgian Wit'
    style VARCHAR(80) NOT NULL,
    abv_percent NUMERIC(3, 1) NOT NULL,
    ibu_rating INTEGER,
    keg_total_litres NUMERIC(6, 2) DEFAULT 50.00 NOT NULL,
    keg_remaining_litres NUMERIC(6, 2) DEFAULT 50.00 NOT NULL,
    tapped_at TIMESTAMPTZ DEFAULT clock_timestamp(),
    status VARCHAR(32) DEFAULT 'ACTIVE_TAPPING'
);

-- ----------------------------------------------------------------------------
-- 6. DINING FOLIOS, POS TICKETS & BILLING
-- ----------------------------------------------------------------------------

CREATE TABLE dining_folios (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_number VARCHAR(40) UNIQUE NOT NULL, -- e.g. TM-INV-99214
    reservation_id UUID REFERENCES reservations(id) ON DELETE SET NULL,
    guest_id UUID NOT NULL REFERENCES guests(id) ON DELETE RESTRICT,
    table_id UUID REFERENCES restaurant_tables(id) ON DELETE SET NULL,
    covers_seated INTEGER NOT NULL,
    server_staff_id UUID,
    server_name VARCHAR(100) DEFAULT 'Bikash H.',
    subtotal_amount NUMERIC(12, 2) DEFAULT 0.00 NOT NULL,
    discount_amount NUMERIC(12, 2) DEFAULT 0.00 NOT NULL,
    cgst_amount NUMERIC(10, 2) DEFAULT 0.00 NOT NULL,
    sgst_amount NUMERIC(10, 2) DEFAULT 0.00 NOT NULL,
    service_charge_amount NUMERIC(10, 2) DEFAULT 0.00 NOT NULL,
    grand_total NUMERIC(12, 2) DEFAULT 0.00 NOT NULL,
    payment_status payment_status DEFAULT 'PENDING' NOT NULL,
    opened_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    settled_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);

CREATE TABLE folio_line_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    folio_id UUID NOT NULL REFERENCES dining_folios(id) ON DELETE CASCADE,
    menu_item_id UUID NOT NULL REFERENCES menu_items(id) ON DELETE RESTRICT,
    item_name_snapshot VARCHAR(160) NOT NULL,
    unit_price NUMERIC(10, 2) NOT NULL,
    quantity INTEGER DEFAULT 1 NOT NULL,
    line_total NUMERIC(10, 2) GENERATED ALWAYS AS (unit_price * quantity) STORED,
    special_kitchen_instruction TEXT,
    order_destination order_destination DEFAULT 'KITCHEN_HOT',
    fired_at TIMESTAMPTZ DEFAULT clock_timestamp(),
    served_at TIMESTAMPTZ
);

CREATE TABLE folio_payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    folio_id UUID NOT NULL REFERENCES dining_folios(id) ON DELETE CASCADE,
    amount NUMERIC(12, 2) NOT NULL,
    mode payment_mode NOT NULL,
    transaction_reference VARCHAR(120),         -- Razorpay txn ID, UPI UTR #, or Amex auth code
    settled_by_staff_id UUID,
    settled_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);

CREATE INDEX idx_folios_guest ON dining_folios(guest_id, opened_at DESC);
CREATE INDEX idx_folios_date ON dining_folios(opened_at);

-- ----------------------------------------------------------------------------
-- 7. LOYALTY LEDGER & PRIVILEGE REWARDS
-- ----------------------------------------------------------------------------

CREATE TABLE loyalty_accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guest_id UUID UNIQUE NOT NULL REFERENCES guests(id) ON DELETE CASCADE,
    current_points_balance INTEGER DEFAULT 0 NOT NULL,
    lifetime_points_earned INTEGER DEFAULT 0 NOT NULL,
    lifetime_points_redeemed INTEGER DEFAULT 0 NOT NULL,
    current_tier guest_tier DEFAULT 'REGULAR' NOT NULL,
    points_to_next_tier INTEGER DEFAULT 1550,
    next_tier_target guest_tier DEFAULT 'ROYAL_AMBASSADOR',
    updated_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);

CREATE TABLE loyalty_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    loyalty_account_id UUID NOT NULL REFERENCES loyalty_accounts(id) ON DELETE CASCADE,
    folio_id UUID REFERENCES dining_folios(id) ON DELETE SET NULL,
    points_delta INTEGER NOT NULL,              -- Positive for credit, negative for redemption
    balance_after INTEGER NOT NULL,
    transaction_type VARCHAR(40) NOT NULL,      -- 'DINING_SPEND_CREDIT', 'MANUAL_COMPLIMENTARY', 'REDEMPTION'
    description TEXT,                           -- 'Dining spend of ₹14,280 on Cabana T-14'
    authorized_by_staff_id UUID,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);

-- ----------------------------------------------------------------------------
-- 8. HOST, SOMMELIER SERVICE LOGS & CONCIERGE DOSSIER
-- ----------------------------------------------------------------------------

CREATE TABLE guest_host_notes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guest_id UUID NOT NULL REFERENCES guests(id) ON DELETE CASCADE,
    author_staff_id UUID NOT NULL,
    author_staff_name VARCHAR(100) DEFAULT 'Pranjal Baruah (General Manager)' NOT NULL,
    author_role VARCHAR(64) DEFAULT 'General Manager & Sommelier' NOT NULL,
    note_category VARCHAR(40) DEFAULT 'SERVICE_BRIEFING', -- 'WINE_PREFERENCE', 'SEATING', 'INCIDENT', 'BIRTHDAY_SURPRISE'
    note_text TEXT NOT NULL,
    is_pinned_to_shift_briefing BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);

CREATE INDEX idx_host_notes_guest ON guest_host_notes(guest_id, is_pinned_to_shift_briefing DESC);

-- ----------------------------------------------------------------------------
-- 9. WHATSAPP & AUTOMATED CAMPAIGN DISPATCHES
-- ----------------------------------------------------------------------------

CREATE TABLE automated_campaign_triggers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trigger_slug VARCHAR(64) UNIQUE NOT NULL,   -- 'BIRTHDAY_DOM_PERIGNON_QUEUE', 'POST_DINNER_FEEDBACK'
    name VARCHAR(120) NOT NULL,
    days_offset INTEGER DEFAULT 0,              -- e.g. -3 for 3 days before birthday
    message_template_text TEXT NOT NULL,
    is_active BOOLEAN DEFAULT TRUE NOT NULL
);

CREATE TABLE communication_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guest_id UUID NOT NULL REFERENCES guests(id) ON DELETE CASCADE,
    channel message_channel DEFAULT 'WHATSAPP' NOT NULL,
    destination_target VARCHAR(64) NOT NULL,    -- WhatsApp Number
    template_slug VARCHAR(64),
    rendered_content TEXT NOT NULL,
    status dispatch_status DEFAULT 'QUEUED' NOT NULL,
    external_provider_message_id VARCHAR(120),
    sent_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);

CREATE INDEX idx_comm_logs_guest ON communication_logs(guest_id, created_at DESC);
```

---

## 3. Automation Triggers, Rollups & Business Logic

```sql
-- ----------------------------------------------------------------------------
-- AUTOMATION TRIGGER 1: Synchronize Guest Lifetime Spend & Visit Counts
-- ----------------------------------------------------------------------------

CREATE OR REPLACE FUNCTION fn_update_guest_folio_aggregates()
RETURNS TRIGGER AS $$
BEGIN
    IF (NEW.payment_status = 'SETTLED' AND (OLD.payment_status IS DISTINCT FROM 'SETTLED')) THEN
        UPDATE guests
        SET 
            total_lifetime_spend = total_lifetime_spend + NEW.grand_total,
            total_visits_count = total_visits_count + 1,
            last_visited_at = NEW.settled_at,
            tier = CASE 
                WHEN (total_lifetime_spend + NEW.grand_total) >= 200000 THEN 'PLATINUM_VIP'::guest_tier
                WHEN (total_lifetime_spend + NEW.grand_total) >= 50000 THEN 'GOLD_VIP'::guest_tier
                WHEN (total_lifetime_spend + NEW.grand_total) >= 15000 THEN 'SILVER'::guest_tier
                ELSE 'REGULAR'::guest_tier
            END,
            is_vip = CASE 
                WHEN (total_lifetime_spend + NEW.grand_total) >= 50000 THEN TRUE
                ELSE is_vip
            END,
            updated_at = clock_timestamp()
        WHERE id = NEW.guest_id;

        -- Automatically award loyalty points: 1 Point per ₹100 spent
        INSERT INTO loyalty_transactions (
            loyalty_account_id,
            folio_id,
            points_delta,
            balance_after,
            transaction_type,
            description
        )
        SELECT 
            la.id,
            NEW.id,
            FLOOR(NEW.grand_total / 100)::INTEGER,
            la.current_points_balance + FLOOR(NEW.grand_total / 100)::INTEGER,
            'DINING_SPEND_CREDIT',
            'Earned on invoice ' || NEW.invoice_number
        FROM loyalty_accounts la
        WHERE la.guest_id = NEW.guest_id;

        UPDATE loyalty_accounts
        SET 
            current_points_balance = current_points_balance + FLOOR(NEW.grand_total / 100)::INTEGER,
            lifetime_points_earned = lifetime_points_earned + FLOOR(NEW.grand_total / 100)::INTEGER,
            updated_at = clock_timestamp()
        WHERE guest_id = NEW.guest_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_folio_settled_aggregates
AFTER UPDATE OF payment_status ON dining_folios
FOR EACH ROW
EXECUTE FUNCTION fn_update_guest_folio_aggregates();
```

---

## 4. PostgreSQL Row-Level Security (RLS) & Multi-Role Policies

```sql
-- Enable RLS across all customer data tables
ALTER TABLE guests ENABLE ROW LEVEL SECURITY;
ALTER TABLE guest_preferences ENABLE ROW LEVEL SECURITY;
ALTER TABLE guest_allergens ENABLE ROW LEVEL SECURITY;
ALTER TABLE dining_folios ENABLE ROW LEVEL SECURITY;
ALTER TABLE guest_host_notes ENABLE ROW LEVEL SECURITY;

-- ----------------------------------------------------------------------------
-- APP ROLES DEFINITIONS:
-- 'authenticated_manager' -> Full View & Write Access
-- 'authenticated_host'    -> Service and Allergen viewing, no raw phone export
-- 'authenticated_guest'   -> Self-service only (via Supabase auth.uid())
-- ----------------------------------------------------------------------------

-- 1. Management Policy (General Manager, Owner, Sommelier)
CREATE POLICY p_manager_all_access ON guests
    FOR ALL
    TO authenticated
    USING (
        (current_setting('request.jwt.claims', true)::jsonb ->> 'role') IN ('GENERAL_MANAGER', 'CRM_ADMIN', 'RESTAURANT_OWNER')
    );

-- 2. Floor Host & Server Policy (Restricted PII, Operational View)
CREATE POLICY p_floor_service_read ON guests
    FOR SELECT
    TO authenticated
    USING (
        (current_setting('request.jwt.claims', true)::jsonb ->> 'role') IN ('FLOOR_CAPTAIN', 'SOMMELIER', 'RECEPTIONIST')
    );

-- 3. Critical Allergen Policy: Available instantly to all kitchen & floor staff
CREATE POLICY p_allergens_service_safety ON guest_allergens
    FOR SELECT
    TO authenticated
    USING (TRUE);

-- 4. Guest Self-Service (Table Reservations, Folio Receipts)
CREATE POLICY p_guest_self_read ON guests
    FOR SELECT
    TO authenticated
    USING (
        auth.uid() = id
    );
```

---

## 5. Seed Data Snapshot: Real Production Roster

```sql
-- Seed Core Zones
INSERT INTO zones (id, zone_code, name, zone_category, total_capacity_seats, has_unobstructed_sunset_view) VALUES
('11111111-1111-1111-1111-111111111111', 'SUNSET_DECK', 'Open-Air Sunset Deck', 'SUNSET_ROOFTOP_DECK', 40, TRUE),
('22222222-2222-2222-2222-222222222222', 'DECK_CABANAS', 'The Luxury Deck Cabanas', 'DECK_CABANA', 16, TRUE),
('33333333-3333-3333-3333-333333333333', 'THE_VAULT', 'The Vault Private Sommelier Lounge', 'THE_VAULT_VIP', 14, FALSE),
('44444444-4444-4444-4444-444444444444', 'BREWERY_BAR', 'Craft Brewery High-Taps', 'BREWERY_TAP_BAR', 20, FALSE);

-- Seed Key Table (T-14 Cabana)
INSERT INTO restaurant_tables (id, zone_id, table_number, min_covers, max_covers, optimal_covers, is_vip_reserved_only, shape) VALUES
('aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa', '22222222-2222-2222-2222-222222222222', 'Deck Cabana T-14', 2, 6, 4, TRUE, 'CABANA');

-- Seed VIP Guest: Dr. Arindam Sharma (from Live CRM Screen)
INSERT INTO guests (
    id, guest_code, salutation, first_name, last_name,
    mobile_number, designation, tier, is_vip,
    total_lifetime_spend, total_visits_count,
    last_visited_at, date_of_birth
) VALUES (
    '99999999-9999-9999-9999-999999999999', 'TM-8821', 'Dr.', 'Arindam', 'Sharma',
    '9864022194', 'Chief Cardiologist, Excel Care Hospital', 'PLATINUM_VIP', TRUE,
    148500.00, 28,
    clock_timestamp() - INTERVAL '3 days', '1976-11-18'
);

-- Seed Dr. Sharma Preferences
INSERT INTO guest_preferences (
    guest_id, lifestyle, spice_level, favorite_cuisine,
    favorite_signature_dish, favorite_cocktail_or_brew,
    arrival_beverage_ritual, preferred_zone, preferred_table_id,
    favorite_table_label, sugar_free_beverages, low_sodium_preferred
) VALUES (
    '99999999-9999-9999-9999-999999999999', 'NON_VEGETARIAN', 'MEDIUM', 'Neo-Indian & Sushi',
    'TM Dhuni Butter Chicken, Truffle Salmon Nigiri', 'Clarified Cold-Brew Espresso Martini',
    'Clarified Cold-Brew Espresso Martini (served chilled in vintage crystal glassware)',
    'DECK_CABANA', 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa', 'Deck Cabana T-14',
    TRUE, FALSE
);

-- Seed Host Briefing by General Manager Pranjal Baruah
INSERT INTO guest_host_notes (
    guest_id, author_staff_id, author_staff_name, author_role,
    note_category, note_text, is_pinned_to_shift_briefing
) VALUES (
    '99999999-9999-9999-9999-999999999999', gen_random_uuid(), 'Pranjal Baruah', 'General Manager & Sommelier',
    'SERVICE_BRIEFING',
    'Dr. Sharma loves corner cabana table T-14 with the open evening breeze. Entertains medical associates often. Always prefers chilled glassware for sparkling water, and likes to taste whatever new seasonal stout we have on tap.',
    TRUE
);

-- Seed Loyalty State
INSERT INTO loyalty_accounts (
    guest_id, current_points_balance, lifetime_points_earned,
    current_tier, points_to_next_tier, next_tier_target
) VALUES (
    '99999999-9999-9999-9999-999999999999', 3450, 4820,
    'PLATINUM_VIP', 1550, 'ROYAL_AMBASSADOR'
);
```

---

## 6. Real-Time CRM Analytic Views

```sql
-- 1. Daily Executive Capacity & Seating Ledger View
CREATE OR REPLACE VIEW vw_live_rooftop_seating_utilization AS
SELECT 
    z.name AS zone_name,
    z.zone_category,
    COUNT(t.id) AS total_tables,
    COUNT(CASE WHEN t.status = 'OCCUPIED' THEN 1 END) AS tables_occupied,
    COUNT(CASE WHEN t.status = 'RESERVED' THEN 1 END) AS tables_reserved,
    COUNT(CASE WHEN t.status = 'AVAILABLE' THEN 1 END) AS tables_available,
    ROUND(
        (COUNT(CASE WHEN t.status IN ('OCCUPIED', 'RESERVED') THEN 1 END)::NUMERIC / NULLIF(COUNT(t.id), 0)) * 100, 1
    ) AS occupancy_percentage
FROM zones z
LEFT JOIN restaurant_tables t ON z.id = t.zone_id
GROUP BY z.id, z.name, z.zone_category;

-- 2. VIP Concierge Guest Briefing View (Daily Service Roster)
CREATE OR REPLACE VIEW vw_tonight_vip_concierge_roster AS
SELECT 
    r.booking_reference,
    r.reservation_date,
    r.slot_time,
    r.party_size,
    t.table_number,
    g.guest_code,
    g.full_name AS guest_name,
    g.mobile_number,
    g.tier,
    g.total_lifetime_spend,
    g.total_visits_count,
    gp.favorite_table_label,
    gp.arrival_beverage_ritual,
    hn.note_text AS latest_sommelier_briefing
FROM reservations r
JOIN guests g ON r.guest_id = g.id
LEFT JOIN restaurant_tables t ON r.table_id = t.id
LEFT JOIN guest_preferences gp ON g.id = gp.guest_id
LEFT JOIN LATERAL (
    SELECT note_text 
    FROM guest_host_notes 
    WHERE guest_id = g.id 
    ORDER BY is_pinned_to_shift_briefing DESC, created_at DESC 
    LIMIT 1
) hn ON TRUE
WHERE r.reservation_date = CURRENT_DATE
ORDER BY r.slot_time ASC;
```
