# Transcript Platform - Storage Strategy & Implementation

## Executive Summary

This document outlines the storage strategy for the Transcript Platform, analyzing persistence options, database types, Redis alternatives, static files, and compiled data approaches. It includes implementation examples, vendor offerings, cost analysis, and utilization calculations for each storage method.

## 1. Storage Requirements Analysis

### 1.1 Data Volume Estimates
- **Vendor Profiles**: 25+ vendors × 5KB = 125KB
- **Feature Matrices**: 25 vendors × 50 features × 100B = 125KB
- **Pricing Data**: 25 vendors × 5 tiers × 200B = 25KB
- **User Reviews**: 100 reviews × 1KB = 100KB
- **Total Static Data**: ~375KB (Phase I), ~2MB (Phase V)

### 1.2 Access Patterns
- **Read-Heavy**: 95% read operations (vendor lookups, comparisons)
- **Write-Light**: 5% write operations (reviews, user preferences)
- **Real-Time**: Minimal (vendor data updates weekly/monthly)
- **Caching**: High benefit due to infrequent data changes

## 2. Storage Options Analysis

### 2.1 Compiled Files (Recommended for Phase I)

#### **Implementation: Build-Time Data Compilation**
```typescript
// /data/vendors.json
[
  {
    "id": "otter",
    "name": "Otter.ai",
    "features": ["meeting", "real-time", "collaboration"],
    "pricing": {
      "free": "300 min/month",
      "pro": "$16.99/month",
      "business": "$30/user/month"
    },
    "accuracy": 95,
    "languages": ["en", "es", "fr", "de"],
    "integrations": ["zoom", "teams", "slack"]
  }
]

// /lib/vendors.ts (Auto-generated)
import vendorData from '../data/vendors.json'

export const vendors = vendorData as Vendor[]
export const getVendor = (id: string) => vendors.find(v => v.id === id)
export const getAllVendors = () => vendors
export const searchVendors = (query: string) => 
  vendors.filter(v => 
    v.name.toLowerCase().includes(query.toLowerCase()) ||
    v.features.some(f => f.toLowerCase().includes(query.toLowerCase()))
  )

// Type definitions
export interface Vendor {
  id: string
  name: string
  features: string[]
  pricing: Record<string, string>
  accuracy: number
  languages: string[]
  integrations: string[]
}
```

#### **Build Script: CSV to TypeScript**
```typescript
// scripts/build-vendor-data.ts
import csv from 'csv-parser'
import fs from 'fs'
import path from 'path'

interface VendorRow {
  id: string
  name: string
  features: string
  pricing_free: string
  pricing_pro: string
  pricing_business: string
  accuracy: string
  languages: string
  integrations: string
}

const buildVendorData = async () => {
  const vendors: VendorRow[] = []
  
  // Read CSV source
  fs.createReadStream('data/vendors.csv')
    .pipe(csv())
    .on('data', (row) => vendors.push(row))
    .on('end', () => {
      // Transform to final format
      const transformedVendors = vendors.map(vendor => ({
        id: vendor.id,
        name: vendor.name,
        features: vendor.features.split(',').map(f => f.trim()),
        pricing: {
          free: vendor.pricing_free,
          pro: vendor.pricing_pro,
          business: vendor.pricing_business
        },
        accuracy: parseInt(vendor.accuracy),
        languages: vendor.languages.split(',').map(l => l.trim()),
        integrations: vendor.integrations.split(',').map(i => i.trim())
      }))

      // Generate TypeScript file
      const tsContent = `// Auto-generated from CSV - ${new Date().toISOString()}
export const vendors = ${JSON.stringify(transformedVendors, null, 2)} as const

export type Vendor = typeof vendors[number]
export type VendorId = Vendor['id']

export const getVendor = (id: string): Vendor | undefined => 
  vendors.find(v => v.id === id)

export const getAllVendors = (): Vendor[] => vendors

export const searchVendors = (query: string): Vendor[] => 
  vendors.filter(v => 
    v.name.toLowerCase().includes(query.toLowerCase()) ||
    v.features.some(f => f.toLowerCase().includes(query.toLowerCase()))
  )

export const getVendorsByFeature = (feature: string): Vendor[] =>
  vendors.filter(v => v.features.includes(feature))

export const getVendorsByLanguage = (language: string): Vendor[] =>
  vendors.filter(v => v.languages.includes(language))
`
      
      fs.writeFileSync('lib/generated-vendors.ts', tsContent)
      console.log(`Generated vendor data: ${transformedVendors.length} vendors`)
    })
}

buildVendorData()
```

#### **Package.json Scripts**
```json
{
  "scripts": {
    "build:data": "tsx scripts/build-vendor-data.ts",
    "build:all": "npm run build:data && npm run build:types",
    "predev": "npm run build:all",
    "prebuild": "npm run build:all",
    "dev": "next dev",
    "build": "next build"
  }
}
```

### 2.2 Static Files

#### **Implementation: Public Directory Storage**
```typescript
// /public/data/vendors.json
[
  {
    "id": "otter",
    "name": "Otter.ai",
    "features": ["meeting", "real-time", "collaboration"],
    "pricing": {
      "free": "300 min/month",
      "pro": "$16.99/month"
    }
  }
]

// /app/api/vendors/route.ts
import { NextResponse } from 'next/server'

export async function GET() {
  try {
    // Fetch from public directory
    const response = await fetch(`${process.env.VERCEL_URL}/data/vendors.json`)
    const vendors = await response.json()
    
    return NextResponse.json(vendors)
  } catch (error) {
    return NextResponse.json({ error: 'Failed to load vendors' }, { status: 500 })
  }
}

// /app/api/vendors/[id]/route.ts
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  try {
    const response = await fetch(`${process.env.VERCEL_URL}/data/vendors.json`)
    const vendors = await response.json()
    
    const vendor = vendors.find((v: any) => v.id === params.id)
    if (!vendor) {
      return NextResponse.json({ error: 'Vendor not found' }, { status: 404 })
    }
    
    return NextResponse.json(vendor)
  } catch (error) {
    return NextResponse.json({ error: 'Failed to load vendor' }, { status: 500 })
  }
}
```

### 2.3 Redis (External Service)

#### **Implementation: Upstash Redis Integration**
```typescript
// /lib/redis.ts
import { Redis } from '@upstash/redis'

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
})

export default redis

// /lib/vendors-redis.ts
import redis from './redis'
import { Vendor } from './types'

const VENDORS_KEY = 'vendors'
const VENDOR_PREFIX = 'vendor:'
const CACHE_TTL = 3600 // 1 hour

export const loadVendorsToRedis = async (vendors: Vendor[]) => {
  // Store all vendors as a set
  await redis.set(VENDORS_KEY, JSON.stringify(vendors), { ex: CACHE_TTL })
  
  // Store individual vendors for fast lookup
  for (const vendor of vendors) {
    await redis.set(
      `${VENDOR_PREFIX}${vendor.id}`, 
      JSON.stringify(vendor), 
      { ex: CACHE_TTL }
    )
  }
  
  console.log(`Loaded ${vendors.length} vendors to Redis`)
}

export const getVendor = async (id: string): Promise<Vendor | null> => {
  const vendor = await redis.get(`${VENDOR_PREFIX}${id}`)
  return vendor ? JSON.parse(vendor as string) : null
}

export const getAllVendors = async (): Promise<Vendor[]> => {
  const vendors = await redis.get(VENDORS_KEY)
  return vendors ? JSON.parse(vendors as string) : []
}

export const searchVendors = async (query: string): Promise<Vendor[]> => {
  const vendors = await getAllVendors()
  return vendors.filter(v => 
    v.name.toLowerCase().includes(query.toLowerCase()) ||
    v.features.some(f => f.toLowerCase().includes(query.toLowerCase()))
  )
}

export const updateVendor = async (id: string, data: Partial<Vendor>) => {
  const existing = await getVendor(id)
  if (!existing) return false
  
  const updated = { ...existing, ...data, updatedAt: new Date().toISOString() }
  await redis.set(`${VENDOR_PREFIX}${id}`, JSON.stringify(updated), { ex: CACHE_TTL })
  
  // Update the main vendors list
  const allVendors = await getAllVendors()
  const vendorIndex = allVendors.findIndex(v => v.id === id)
  if (vendorIndex !== -1) {
    allVendors[vendorIndex] = updated
    await redis.set(VENDORS_KEY, JSON.stringify(allVendors), { ex: CACHE_TTL })
  }
  
  return true
}
```

#### **API Routes with Redis**
```typescript
// /app/api/vendors/route.ts
import { NextResponse } from 'next/server'
import { getAllVendors, searchVendors } from '@/lib/vendors-redis'

export async function GET(request: Request) {
  try {
    const { searchParams } = new URL(request.url)
    const query = searchParams.get('q')
    
    if (query) {
      const vendors = await searchVendors(query)
      return NextResponse.json(vendors)
    }
    
    const vendors = await getAllVendors()
    return NextResponse.json(vendors)
  } catch (error) {
    return NextResponse.json({ error: 'Failed to load vendors' }, { status: 500 })
  }
}

// /app/api/vendors/[id]/route.ts
export async function PUT(
  request: Request,
  { params }: { params: { id: string } }
) {
  try {
    const body = await request.json()
    const success = await updateVendor(params.id, body)
    
    if (!success) {
      return NextResponse.json({ error: 'Vendor not found' }, { status: 404 })
    }
    
    return NextResponse.json({ success: true })
  } catch (error) {
    return NextResponse.json({ error: 'Failed to update vendor' }, { status: 500 })
  }
}
```

### 2.4 PostgreSQL + Prisma (Full Database)

#### **Implementation: Relational Database**
```typescript
// /prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Vendor {
  id          String   @id @default(cuid())
  name        String   @unique
  description String?
  website     String?
  founded     Int?
  employees   Int?
  funding     String?
  
  // Features as JSON for flexibility
  features    Json     // Array of feature strings
  languages   Json     // Array of language codes
  integrations Json    // Array of integration names
  
  // Pricing models
  pricingModels PricingModel[]
  
  // User reviews
  reviews     Review[]
  
  // Metadata
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  @@map("vendors")
}

model PricingModel {
  id          String   @id @default(cuid())
  name        String   // "Free", "Pro", "Business"
  price       String   // "$16.99/month", "300 min/month"
  description String?
  features    Json     // Array of included features
  
  vendorId    String
  vendor      Vendor   @relation(fields: [vendorId], references: [id], onDelete: Cascade)
  
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  @@map("pricing_models")
}

model Review {
  id          String   @id @default(cuid())
  rating      Int      // 1-5 stars
  comment     String?
  pros        Json     // Array of positive points
  cons        Json     // Array of negative points
  
  vendorId    String
  vendor      Vendor   @relation(fields: [vendorId], references: [id], onDelete: Cascade)
  
  userId      String?
  userName    String?
  
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  @@map("reviews")
}
```

#### **Prisma Client Usage**
```typescript
// /lib/prisma.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma = globalForPrisma.prisma ?? new PrismaClient()

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma

// /lib/vendors-prisma.ts
import { prisma } from './prisma'
import { Vendor, PricingModel, Review } from '@prisma/client'

export type VendorWithRelations = Vendor & {
  pricingModels: PricingModel[]
  reviews: Review[]
  _count: {
    reviews: number
  }
}

export const getVendor = async (id: string): Promise<VendorWithRelations | null> => {
  return await prisma.vendor.findUnique({
    where: { id },
    include: {
      pricingModels: true,
      reviews: {
        orderBy: { createdAt: 'desc' },
        take: 10
      },
      _count: {
        select: { reviews: true }
      }
    }
  })
}

export const getAllVendors = async (): Promise<VendorWithRelations[]> => {
  return await prisma.vendor.findMany({
    include: {
      pricingModels: true,
      _count: {
        select: { reviews: true }
      }
    },
    orderBy: { name: 'asc' }
  })
}

export const searchVendors = async (query: string): Promise<VendorWithRelations[]> => {
  return await prisma.vendor.findMany({
    where: {
      OR: [
        { name: { contains: query, mode: 'insensitive' } },
        { description: { contains: query, mode: 'insensitive' } },
        { features: { path: '$', array_contains: [query] } }
      ]
    },
    include: {
      pricingModels: true,
      _count: {
        select: { reviews: true }
      }
    }
  })
}

export const getVendorsByFeature = async (feature: string): Promise<VendorWithRelations[]> => {
  return await prisma.vendor.findMany({
    where: {
      features: { path: '$', array_contains: [feature] }
    },
    include: {
      pricingModels: true,
      _count: {
        select: { reviews: true }
      }
    }
  })
}

export const createVendor = async (data: {
  name: string
  description?: string
  website?: string
  features: string[]
  languages: string[]
  integrations: string[]
  pricingModels: Array<{
    name: string
    price: string
    description?: string
    features: string[]
  }>
}): Promise<Vendor> => {
  return await prisma.vendor.create({
    data: {
      name: data.name,
      description: data.description,
      website: data.website,
      features: data.features,
      languages: data.languages,
      integrations: data.integrations,
      pricingModels: {
        create: data.pricingModels
      }
    }
  })
}
```

## 3. Vendor Offerings & Pricing

### 3.1 Database Hosting

#### **PlanetScale (Recommended for Phase II+)**
- **Free Tier**: 1 database, 1GB storage, 1B row reads/month
- **Hobby**: $29/month - 5 databases, 5GB storage, 10B row reads/month
- **Pro**: $99/month - 10 databases, 25GB storage, 100B row reads/month
- **Features**: Branch-based development, automatic scaling, global distribution

#### **Supabase**
- **Free Tier**: 500MB database, 2GB bandwidth, 50MB file storage
- **Pro**: $25/month - 8GB database, 250GB bandwidth, 100GB file storage
- **Features**: Built-in authentication, real-time subscriptions, auto-generated APIs

### 3.2 Redis Providers

#### **Upstash (Recommended for Vercel)**
- **Free Tier**: 10,000 requests/day, 256MB data
- **Hobby**: $20/month - 100K requests/day, 1GB data
- **Pro**: $100/month - 1M requests/day, 10GB data
- **Features**: Vercel integration, global edge deployment, REST API

#### **Redis Cloud**
- **Free Tier**: 30MB data, 30 connections
- **Fixed**: $5/month - 100MB data, 100 connections
- **Flexible**: $15/month - 1GB data, 500 connections
- **Features**: Full Redis compatibility, enterprise support

## 4. Cost Analysis & Utilization

### 4.1 Phase I: Compiled Files (0-2 months)
```
Storage Costs: $0/month
- Simple file-based approach
- Data bundled with code
- No external storage needed
- Vercel hosting includes static files
- No database setup
- No Redis service

Total Phase I Cost: $0/month
```

### 4.2 Phase II: Redis + Static Files (2-4 months)
```
Storage Costs: $0/month
- Static files in Vercel public directory
- Redis free tier (10K requests/day, 256MB)
- No database hosting needed

Total Phase II Cost: $0/month
```

### 4.3 Phase III: PostgreSQL + Redis (4-6 months)
```
Storage Costs: $29/month
- PlanetScale Hobby: $29/month
- Upstash Hobby: $20/month
- Total: $49/month

Total Phase III Cost: $49/month
```

### 4.4 Phase IV: Production Scale (6-8 months)
```
Storage Costs: $99/month
- PlanetScale Pro: $99/month
- Upstash Pro: $100/month
- Total: $199/month

Total Phase IV Cost: $199/month
```

### 4.5 Phase V: Enterprise Scale (8-10 months)
```
Storage Costs: $199/month
- PlanetScale Pro: $99/month
- Upstash Pro: $100/month
- Additional services: $50-100/month
- Total: $249-349/month

Total Phase V Cost: $249-349/month
```

## 5. Implementation Recommendations

### 5.1 Phase I: Compiled Files (Recommended)
```
Why:
- Zero cost for storage and hosting
- Fastest development and deployment
- No external dependencies
- Perfect for 15-25 vendor profiles

Implementation:
- CSV/JSON source files
- Build-time compilation to TypeScript
- Data bundled with application code
- Simple npm scripts for data updates
```

### 5.2 Phase II: Redis + Static Files
```
Why:
- Minimal cost ($0/month with free tiers)
- Real-time data updates possible
- Better performance than static files
- Easy migration from compiled files

Implementation:
- Static files for vendor data
- Redis for caching and sessions
- API routes for data access
- Upstash Redis with Vercel integration
```

### 5.3 Phase III+: PostgreSQL + Redis
```
Why:
- Full database capabilities
- Complex relationships and queries
- User accounts and reviews
- Enterprise features

Implementation:
- PostgreSQL with Prisma ORM
- Redis for caching and sessions
- PlanetScale for database hosting
- Upstash for Redis service
```

## 6. Migration Strategy

### 6.1 Compiled Files → Redis
```typescript
// Migration script
import { loadVendorsToRedis } from './lib/vendors-redis'
import { vendors } from './lib/generated-vendors'

const migrateToRedis = async () => {
  await loadVendorsToRedis(vendors)
  console.log('Migration to Redis complete')
}

// Update API routes to use Redis instead of compiled data
```

### 6.2 Redis → PostgreSQL
```typescript
// Migration script
import { prisma } from './lib/prisma'
import { getAllVendors } from './lib/vendors-redis'

const migrateToPostgreSQL = async () => {
  const vendors = await getAllVendors()
  
  for (const vendor of vendors) {
    await prisma.vendor.create({
      data: {
        name: vendor.name,
        eatures: vendor.features,
        // ... other fields
      }
    })
  }
  
  console.log('Migration to PostgreSQL complete')
}
```

## 7. Performance

### 7.1 Data Access Estimates
```
Compiled Files:
- Access Time: <1ms (bundled with code)
- Memory Usage: ~375KB (Phase I)
- Bundle Size: +375KB to JavaScript bundle

Static Files:
- Access Time: 10-50ms (network request)
- Memory Usage: 0KB (no memory overhead)
- Bundle Size: 0KB (separate files)

Redis:
- Access Time: 5-20ms (network + Redis)
- Memory Usage: 0KB (no memory overhead)
- Bundle Size: 0KB (separate service)

PostgreSQL:
- Access Time: 20-100ms (network + database)
- Memory Usage: 0KB (no memory overhead)
- Bundle Size: 0KB (separate service)
```

### 7.2 Scalability Analysis
```
Compiled Files:
- Scalability: Limited by bundle size
- Max Vendors: ~100-200 before bundle becomes unwieldy
- Performance: Consistent regardless of data size

Static Files:
- Scalability: Limited by CDN and file size
- Max Vendors: ~1000+ vendors
- Performance: Degrades with file size

Redis:
- Scalability: Limited by Redis service limits
- Max Vendors: 10,000+ vendors
- Performance: Consistent with proper indexing

PostgreSQL:
- Scalability: Limited by database hosting plan
- Max Vendors: 100,000+ vendors
- Performance: Degrades with complex queries
```

## 8. Conclusion

### 8.1 Phase I Recommendation: Compiled Files
The compiled files approach provides the **fastest path to market** with **zero additional costs** and **excellent performance** for the initial 15-25 vendor profiles. This approach eliminates external dependencies and simplifies deployment while maintaining type safety and fast data access.

### 8.2 Long-term Strategy: Hybrid Approach
As the platform scales, a **hybrid approach** combining PostgreSQL for complex data relationships with Redis for caching provides the best balance of performance, scalability, and cost-effectiveness.

### 8.3 Cost
- **Phase I**: $0/month - Perfect for MVP and early users
- **Phase II**: $0/month - Free tier services sufficient
- **Phase III+**: $49-349/month - Scaling costs align with revenue growth

The storage strategy provides a **clear migration path** from simple file-based storage to enterprise-grade database solutions, ensuring the platform can grow without architectural limitations.

---

**Version**: 0.2  
**Date**: August 2025  
**Copyright**: 2025 Transcript Developers
