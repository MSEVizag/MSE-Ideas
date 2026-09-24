# Product Table Schema

This document outlines the PostgreSQL database schema for the `products` table, including custom ENUMs, generated columns for Cloudflare R2 and full-text search, and automated triggers.

```sql
-- Enable UUID generation extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Create Custom ENUM for Stock Status
DO $$ BEGIN
    CREATE TYPE stock_status_enum AS ENUM ('in_stock', 'sold_out', 'pre_order');
EXCEPTION
    WHEN duplicate_object THEN null;
END $$;

-- Create Products Table
CREATE TABLE IF NOT EXISTS products (
    -- Primary Key UUID
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Public Information
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    category VARCHAR(100) NOT NULL,
    description TEXT,
    price NUMERIC(10, 2) NOT NULL DEFAULT 0.00 CHECK (price >= 0),
    
    -- Cloudflare R2 Storage Reference
    -- Auto-generated base folder URL pointing to /products/[id]/
    r2_folder_url TEXT GENERATED ALWAYS AS (
        'https://your-r2-bucket-domain.com/products/' || id::text || '/'
    ) STORED,

    -- Dynamic Specs & Tags
    specs JSONB DEFAULT '{}'::jsonb NOT NULL,
    tags TEXT[] DEFAULT '{}',

    -- Visibility & Stock Controls
    visibility_status visibility_status_enum DEFAULT 'published' NOT NULL,
    stock_status stock_status_enum DEFAULT 'in_stock' NOT NULL,
    stock_quantity INTEGER DEFAULT 0 CHECK (stock_quantity >= 0) NOT NULL,
    moq INTEGER DEFAULT 1 CHECK (moq >= 1) NOT NULL,

    -- Internal / Admin Metadata (NEVER expose to storefront APIs)
    sku VARCHAR(100) UNIQUE,
    internal_notes TEXT,

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,

    -- Generated Full-Text Search Vector (Weighted Title > Description)
    fts tsvector GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(description, '')), 'B')
    ) STORED
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_products_slug ON products (slug);
CREATE INDEX IF NOT EXISTS idx_products_specs ON products USING gin (specs);
CREATE INDEX IF NOT EXISTS idx_products_tags ON products USING gin (tags);
CREATE INDEX IF NOT EXISTS idx_products_fts ON products USING gin (fts);
CREATE INDEX IF NOT EXISTS idx_products_visibility ON products (is_visible);
CREATE INDEX IF NOT EXISTS idx_products_stock_status ON products (stock_status);
CREATE INDEX IF NOT EXISTS idx_products_created_at ON products (created_at DESC);

-- Updated_at Trigger Logic
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

DROP TRIGGER IF EXISTS update_products_updated_at ON products;
CREATE TRIGGER update_products_updated_at
    BEFORE UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```
