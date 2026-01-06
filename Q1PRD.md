# PRD: Company Profile Scraping, Verification & Chat Integration

**Version:** 1.0  
**Date:** 2025-01-06  
**Status:** Draft

---

## Overview

Build a two-phase company profile scraping system that:
1. Performs lightweight identity scraping during account creation (preview)
2. Triggers deep profile/product scraping after first user email verification
3. Populates wizard with pre-filled data that users verify/edit
4. Powers chat conversations with verified company context

**Core Philosophy:** "Wow factor" over pristine accuracy. Pre-populate aggressively, let users correct. The easiest tool to use is the first tool used.

---

## Goals

### Primary Goals
- **Reduce onboarding friction**: Users see their company info and products pre-populated
- **Increase data quality**: Verification tracking ensures chat accuracy
- **Enable smart conversations**: Chat has full context about company, products, and user preferences

### Success Metrics
- Time-to-first-chat reduced by 50%
- 70%+ of scraped data verified by users (not deleted)
- Chat conversations reference specific products in 60%+ of recommendations

---

## User Journey

### Phase 1: Account Creation (Light Scrape)
```
User submits company URL
  ↓
Preview screen shows scraped data (name, description, social links)
  ↓
User creates account
  ↓
Save light scrape data to companies.profile (marked as "unverified")
  ↓
Verification email sent
```

### Phase 2: Email Verification (Deep Scrape Trigger)
```
User clicks verification link
  ↓
activateUser() checks: "Is this the first verified user at this company?"
  ↓
IF YES:
  - Mark profile._metadata.deepScrapePending = true
  - Fire async deep scrape job (non-blocking)
  - Send email: "We're gathering your product catalog..."
  ↓
User redirected to wizard (sees light scrape data immediately)
```

### Phase 3: Background Deep Scrape
```
Deep scrape job runs (5-30 seconds)
  ↓
Scrape products, extended profile data, brand assets
  ↓
Populate products table (status: 'unverified')
  ↓
Merge into companies.profile (respecting any user edits)
  ↓
Send email: "Your profile is ready! We found X products."
```

### Phase 4: Wizard Verification
```
User opens wizard
  ↓
See profile fields with badges: "Scraped from website" | "Verified" | empty
  ↓
Banner: "Verify your info to improve chat accuracy"
  ↓
User reviews/edits fields → mark as "verified"
  ↓
User reviews products table → edit/delete/verify
  ↓
Profile completion score updates
```

### Phase 5: Chat with Context
```
User starts chat conversation
  ↓
Select products relevant to this conversation (optional)
  ↓
Chat includes:
  - Full conversation history
  - Company profile from JSONB
  - Selected products (or all if none selected)
  ↓
IF unverified fields exist:
  - Show banner: "You have unverified data. See Brand tab to increase chat fidelity."
  ↓
Chat provides recommendations with full context
```

---

## Data Architecture

### 1. Profile JSONB Structure

```typescript
companies.profile = {
  // Basic identity (from light scrape)
  name: string,
  website: string,
  description: string,
  socialLinks: string[],
  
  // Branding (from light scrape)
  branding: {
    favicon: string,
    themeColor: string,
    logo: string | null  // Added by deep scrape
  },
  
  // Extended profile (from deep scrape + user edits)
  tagline: string | null,
  foundingYear: number | null,
  headquartersAddress: string | null,
  priceTier: 'Budget' | 'Mid' | 'Premium' | 'Luxury' | null,
  salesChannels: string[],
  targetMarkets: string[],
  categories: string[],
  
  // Operations
  wholesale: {
    moq: string | null,
    leadTime: string | null,
    casePack: string | null
  },
  fulfillment: string | null,
  returnPolicy: string | null,
  
  // Brand identity
  style: {
    primaryColor: string | null,
    secondaryColor: string | null,
    tone: string | null,
    packaging: string | null
  },
  
  // Leadership (from deep scrape)
  leaderData: {
    founder: string | null,
    ceo: string | null,
    cto: string | null
  } | null,
  
  // Metadata (tracking system)
  _metadata: {
    version: number,
    scraped: boolean,
    scrapedAt: timestamp,
    deepScrapePending: boolean,
    deepScrapedAt: timestamp | null,
    verified: boolean,
    lastUserModified: timestamp | null,
    completionScore: number,  // 0.0 - 1.0
    source: 'initial_preview' | 'deep_scrape' | 'user_entry',
    
    // Per-field tracking
    fieldSources: {
      [fieldName]: {
        source: 'scraped' | 'deep_scraped' | 'user_modified' | 'ai_enriched',
        verificationStatus: 'null' | 'unverified' | 'verified',
        confidence: number,  // 0.0 - 1.0
        scrapedFrom: string,  // URL or 'jsonld' or 'meta_og'
        scrapedAt: timestamp,
        modifiedAt: timestamp | null
      }
    },
    
    // Scraping history
    scrapingAttempts: [
      {
        timestamp: timestamp,
        phase: 'light' | 'deep',
        url: string,
        method: string,  // 'enrichURL' | 'enrichLinkedin' | 'productScraper'
        status: 'success' | 'partial' | 'failed',
        fieldsExtracted: string[],
        productsFound: number | null
      }
    ]
  }
}
```

### 2. Products Table Schema

**Migration Addition:**
```sql
ALTER TABLE products ADD COLUMN verification_status TEXT DEFAULT 'unverified';
ALTER TABLE products ADD COLUMN scraped BOOLEAN DEFAULT false;
ALTER TABLE products ADD COLUMN confidence_score DECIMAL(3,2);
ALTER TABLE products ADD COLUMN scraped_from TEXT;
ALTER TABLE products ADD COLUMN scraped_at TIMESTAMPTZ;

-- Verification status: 'unverified' | 'verified' | 'flagged_for_review'
CREATE INDEX idx_products_verification ON products(company_id, verification_status);
```

**Product Row Example:**
```typescript
{
  id: uuid,
  company_id: uuid,
  name: "The Neon Lights 5.5\" Swim Trunks",
  description: "Electric summer vibes...",
  category: "Swimwear",
  subcategory: "Swim Trunks",
  wholesale_price: 29.50,
  retail_price: 59.50,
  moq: 12,
  case_pack: 6,
  attributes: {
    materials: ["Polyester", "Spandex"],
    colors: ["Neon Pink", "Electric Blue"],
    sizes: ["S", "M", "L", "XL"]
  },
  images: ["https://..."],
  status: 'active',
  
  // Verification fields
  verification_status: 'unverified',  // or 'verified'
  scraped: true,
  confidence_score: 0.89,
  scraped_from: "https://chubbiesshorts.com/products/neon-lights",
  scraped_at: "2025-01-06T15:30:00Z",
  
  created_at: timestamp,
  updated_at: timestamp
}
```

### 3. Conversation Context Model

```typescript
conversations.context = {
  // Selected products for this chat
  selectedProductIds: uuid[],
  
  // User preferences (extracted during chat)
  preferences: {
    regions: string[],
    storeSize: string,
    priceTier: string
  },
  
  // Topics discussed (lightweight tracking)
  topicsDiscussed: string[],
  retailersMentioned: string[],
  
  // Verification warning shown
  unverifiedDataWarningShown: boolean,
  
  // Last known intent
  lastKnownIntent: string
}
```

---

## Technical Implementation

### Phase 1: Save Light Scrape Data (Existing + Enhancement)

**File:** `src/views/dashboard/brand/BrandAccountCreate.vue`

**Enhancement Needed:**
```typescript
async function createCompany(previewData) {
  const profile = {
    name: previewData.name,
    website: previewData.website,
    description: previewData.shortDescription,
    socialLinks: previewData.socialLinks || [],
    branding: {
      favicon: previewData.favicon,
      themeColor: previewData.meta?.themeColor
    },
    
    _metadata: {
      version: 1,
      scraped: true,
      scrapedAt: new Date().toISOString(),
      deepScrapePending: true,
      verified: false,
      source: 'initial_preview',
      completionScore: 0.2,  // Basic fields only
      
      fieldSources: {
        name: { 
          source: 'scraped', 
          verificationStatus: 'unverified',
          confidence: 1.0, 
          scrapedFrom: 'meta_og',
          scrapedAt: new Date().toISOString()
        },
        website: { 
          source: 'scraped', 
          verificationStatus: 'unverified',
          confidence: 1.0, 
          scrapedFrom: 'url',
          scrapedAt: new Date().toISOString()
        },
        description: { 
          source: 'scraped', 
          verificationStatus: 'unverified',
          confidence: 0.9, 
          scrapedFrom: 'meta_description',
          scrapedAt: new Date().toISOString()
        }
      },
      
      scrapingAttempts: [{
        timestamp: new Date().toISOString(),
        phase: 'light',
        url: previewData.website,
        method: 'enrichURL',
        status: 'success',
        fieldsExtracted: ['name', 'website', 'description', 'socialLinks'],
        productsFound: null
      }]
    }
  };
  
  await api.post('/companies', {
    name: previewData.name,
    website: previewData.website,
    profile: profile
  });
}
```

---

### Phase 2: Deep Scrape Trigger (New)

**File:** `src/services/UserService.ts`

**Implementation:**
```typescript
public static async activateUser(userId: string) {
  const db = PostgresService.getInstance();
  const { rows } = await db.query(
    `UPDATE users
        SET status = 'active'
      WHERE id = $1::uuid
        AND status = 'inactive'
      RETURNING id, company_id, email, name, status, deleted_at, created_at, updated_at`,
    [userId]
  );

  if (!rows[0]) {
    throw new Error('Activation failed: user not found or already active');
  }

  const user = mapRowToUser(rows[0]);
  
  // Check if first verified user at company
  const isFirstUser = await this.isFirstVerifiedUser(user.companyId);
  
  if (isFirstUser) {
    // Fire async deep scrape (non-blocking)
    this.triggerDeepScrape(user.companyId, userId).catch(err => {
      logger.error('Deep scrape trigger failed', { 
        companyId: user.companyId, 
        userId, 
        error: err 
      });
    });
  }

  return user;
}

private static async isFirstVerifiedUser(companyId: string): Promise<boolean> {
  const db = PostgresService.getInstance();
  const { rows } = await db.query(
    `SELECT COUNT(*) as count
     FROM users
     WHERE company_id = $1::uuid
       AND status = 'active'
       AND deleted_at IS NULL`,
    [companyId]
  );
  
  return parseInt(rows[0].count) === 1;
}

private static async triggerDeepScrape(companyId: string, userId: string): Promise<void> {
  // Simple fire-and-forget for now
  // Later: Replace with Bull/BullMQ job queue
  
  setImmediate(async () => {
    try {
      await CompanyScraperService.performDeepScrape(companyId, userId);
    } catch (error) {
      logger.error('Deep scrape failed', { companyId, userId, error });
      // Don't throw - user experience isn't blocked by scrape failure
    }
  });
}
```

---

### Phase 3: Deep Scrape Service (New)

**File:** `src/services/CompanyScraperService.ts` (NEW)

**Structure:**
```typescript
export class CompanyScraperService {
  
  static async performDeepScrape(companyId: string, triggeredByUserId: string) {
    const startTime = Date.now();
    logger.info('Starting deep scrape', { companyId });
    
    try {
      // 1. Get company profile
      const company = await CompanyService.getById(companyId);
      const website = company.profile.website;
      
      // 2. Scrape products (details TBD)
      const scrapedProducts = await this.scrapeProducts(website);
      
      // 3. Scrape extended profile data (about page, team, etc.)
      const extendedProfile = await this.scrapeExtendedProfile(website);
      
      // 4. Save products to database
      if (scrapedProducts.length > 0) {
        await this.saveScrapedProducts(companyId, scrapedProducts);
      }
      
      // 5. Merge extended profile (respecting user edits)
      await this.mergeExtendedProfile(companyId, extendedProfile);
      
      // 6. Update metadata
      await this.markDeepScrapeComplete(companyId, {
        success: true,
        productsFound: scrapedProducts.length,
        duration: Date.now() - startTime
      });
      
      // 7. Send notification email
      await EmailService.sendDeepScrapeComplete(triggeredByUserId, {
        productsFound: scrapedProducts.length,
        profileFieldsUpdated: Object.keys(extendedProfile).length
      });
      
      logger.info('Deep scrape completed', { 
        companyId, 
        productsFound: scrapedProducts.length,
        duration: Date.now() - startTime
      });
      
    } catch (error) {
      logger.error('Deep scrape failed', { companyId, error });
      
      // Mark as failed but don't throw
      await this.markDeepScrapeComplete(companyId, {
        success: false,
        error: error.message
      });
    }
  }
  
  private static async scrapeProducts(website: string): Promise<any[]> {
    // TODO: Implement product scraping
    
    // For now, return empty array (fail gracefully)
    return [];
  }
  
  private static async scrapeExtendedProfile(website: string): Promise<any> {
    // TODO: Scrape about page, team info, etc.
    // Similar to existing enrichURL but more comprehensive
    
    return {};
  }
  
  private static async saveScrapedProducts(
    companyId: string, 
    products: any[]
  ): Promise<void> {
    const db = PostgresService.getInstance();
    
    for (const product of products) {
      await db.query(
        `INSERT INTO products (
          company_id, name, description, category, subcategory,
          wholesale_price, retail_price, moq, case_pack,
          attributes, images, status,
          verification_status, scraped, confidence_score, 
          scraped_from, scraped_at
        ) VALUES (
          $1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, 
          $13, $14, $15, $16, $17
        )`,
        [
          companyId,
          product.name,
          product.description,
          product.category,
          product.subcategory,
          product.wholesale_price,
          product.retail_price,
          product.moq,
          product.case_pack,
          JSON.stringify(product.attributes || {}),
          JSON.stringify(product.images || []),
          'active',
          'unverified',
          true,
          product.confidence_score || 0.8,
          product.scraped_from,
          new Date()
        ]
      );
    }
  }
  
  private static async mergeExtendedProfile(
    companyId: string, 
    extendedData: any
  ): Promise<void> {
    const db = PostgresService.getInstance();
    
    // Get current profile
    const { rows } = await db.query(
      `SELECT profile FROM companies WHERE id = $1::uuid`,
      [companyId]
    );
    
    const currentProfile = rows[0].profile;
    
    // Merge logic: Only update fields that are:
    // 1. Currently null/empty
    // 2. NOT user_modified
    Object.entries(extendedData).forEach(([field, value]) => {
      const fieldMeta = currentProfile._metadata?.fieldSources?.[field];
      
      // Skip if user has modified this field
      if (fieldMeta?.source === 'user_modified') {
        return;
      }
      
      // Update if empty or from light scrape
      if (!currentProfile[field] || fieldMeta?.source === 'scraped') {
        currentProfile[field] = value;
        currentProfile._metadata.fieldSources[field] = {
          source: 'deep_scraped',
          verificationStatus: 'unverified',
          confidence: extendedData._confidence?.[field] || 0.8,
          scrapedFrom: extendedData._sources?.[field] || 'deep_scrape',
          scrapedAt: new Date().toISOString()
        };
      }
    });
    
    // Save merged profile
    await db.query(
      `UPDATE companies 
       SET profile = $1::jsonb 
       WHERE id = $2::uuid`,
      [currentProfile, companyId]
    );
  }
  
  private static async markDeepScrapeComplete(
    companyId: string, 
    result: { success: boolean; productsFound?: number; error?: string; duration?: number }
  ): Promise<void> {
    const db = PostgresService.getInstance();
    
    await db.query(
      `UPDATE companies
       SET profile = jsonb_set(
         jsonb_set(
           jsonb_set(
             profile,
             '{_metadata,deepScrapePending}',
             'false'
           ),
           '{_metadata,deepScrapedAt}',
           to_jsonb(now())
         ),
         '{_metadata,scrapingAttempts}',
         profile->'_metadata'->'scrapingAttempts' || $1::jsonb
       )
       WHERE id = $2::uuid`,
      [
        JSON.stringify([{
          timestamp: new Date().toISOString(),
          phase: 'deep',
          status: result.success ? 'success' : 'failed',
          productsFound: result.productsFound || 0,
          error: result.error || null,
          duration: result.duration || null
        }]),
        companyId
      ]
    );
  }
}
```

---

### Phase 4: Wizard Verification UI (Enhancement)

**File:** `src/views/dashboard/brand/BrandWizard.vue`

**New Features:**

1. **Field Status Badges:**
```vue
<template>
  <v-text-field
    v-model="draft.name"
    label="Company Name"
  >
    <template v-slot:append-inner>
      <VerificationBadge :field="'name'" :metadata="profile._metadata" />
    </template>
  </v-text-field>
</template>

<!-- VerificationBadge.vue component -->
<script setup>
const props = defineProps(['field', 'metadata']);

const status = computed(() => {
  const fieldSource = props.metadata?.fieldSources?.[props.field];
  if (!fieldSource) return 'empty';
  return fieldSource.verificationStatus || 'unverified';
});

const badge = computed(() => {
  switch (status.value) {
    case 'verified': return { text: 'Verified', color: 'success', icon: 'mdi-check-circle' };
    case 'unverified': return { text: 'Scraped from website', color: 'warning', icon: 'mdi-alert-circle' };
    default: return { text: 'Add info', color: 'grey', icon: 'mdi-pencil' };
  }
});
</script>

<template>
  <v-chip :color="badge.color" size="small" variant="flat">
    <v-icon start :icon="badge.icon" size="x-small" />
    {{ badge.text }}
  </v-chip>
</template>
```

2. **Verification Alert Banner:**
```vue
<template>
  <v-alert v-if="hasUnverifiedFields" type="warning" variant="tonal" class="mb-4">
    <v-alert-title>Verify Your Information</v-alert-title>
    <div>
      You have {{ unverifiedFieldCount }} unverified fields. 
      Review and verify to improve chat response accuracy.
    </div>
    <template v-slot:append>
      <v-btn size="small" @click="markAllAsReviewed">
        Mark All as Reviewed
      </v-btn>
    </template>
  </v-alert>
</template>

<script setup>
const hasUnverifiedFields = computed(() => {
  const sources = profile._metadata?.fieldSources || {};
  return Object.values(sources).some(s => s.verificationStatus === 'unverified');
});

const unverifiedFieldCount = computed(() => {
  const sources = profile._metadata?.fieldSources || {};
  return Object.values(sources).filter(s => s.verificationStatus === 'unverified').length;
});
</script>
```

3. **Auto-verify on Edit:**
```typescript
// When user edits a field, mark as verified
function onFieldEdit(fieldName: string, newValue: any) {
  draft[fieldName] = newValue;
  
  // Update metadata
  if (!draft._metadata) draft._metadata = { fieldSources: {} };
  if (!draft._metadata.fieldSources) draft._metadata.fieldSources = {};
  
  draft._metadata.fieldSources[fieldName] = {
    ...draft._metadata.fieldSources[fieldName],
    source: 'user_modified',
    verificationStatus: 'verified',
    modifiedAt: new Date().toISOString()
  };
  
  // Update completion score
  draft._metadata.completionScore = calculateCompletionScore(draft);
}
```

4. **Products Verification Table:**
```vue
<!-- In BrandPanel.vue or ProductSidebar.vue -->
<template>
  <v-data-table
    :items="products"
    :headers="headers"
  >
    <template v-slot:item.verification_status="{ item }">
      <v-chip 
        :color="item.verification_status === 'verified' ? 'success' : 'warning'"
        size="small"
      >
        {{ item.verification_status === 'verified' ? 'Verified' : 'Needs Review' }}
      </v-chip>
    </template>
    
    <template v-slot:item.actions="{ item }">
      <v-btn 
        v-if="item.verification_status === 'unverified'"
        size="small" 
        @click="verifyProduct(item.id)"
      >
        Verify
      </v-btn>
      <v-btn size="small" icon="mdi-pencil" @click="editProduct(item)" />
      <v-btn size="small" icon="mdi-delete" @click="deleteProduct(item.id)" />
    </template>
  </v-data-table>
</template>

<script setup>
async function verifyProduct(productId: string) {
  await api.patch(`/products/${productId}`, {
    verification_status: 'verified'
  });
  // Refresh products list
}
</script>
```

---

### Phase 5: Chat Integration (Enhancement)

**File:** `src/services/ChatService.ts`

**Implementation:**
```typescript
export class ChatService {
  
  static async sendMessage(
    companyId: string,
    conversationId: string,
    userMessage: string,
    selectedProductIds: string[] = []
  ) {
    // 1. Get company profile
    const company = await CompanyService.getById(companyId);
    const profile = company.profile;
    
    // 2. Get selected products (or all active if none selected)
    const products = selectedProductIds.length > 0
      ? await ProductService.getByIds(selectedProductIds)
      : await ProductService.getByCompany(companyId, { status: 'active', limit: 50 });
    
    // 3. Get conversation history
    const conversation = await ConversationService.getById(conversationId);
    
    // 4. Check for unverified data
    const hasUnverifiedProfile = this.hasUnverifiedFields(profile);
    const hasUnverifiedProducts = products.some(p => p.verification_status === 'unverified');
    
    // 5. Build system prompt
    const systemPrompt = this.buildSystemPrompt(profile, products, {
      hasUnverifiedProfile,
      hasUnverifiedProducts
    });
    
    // 6. Call Claude API
    const response = await anthropic.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 4096,
      system: systemPrompt,
      messages: [
        ...conversation.messages,
        { role: 'user', content: userMessage }
      ]
    });
    
    // 7. Save messages
    await ConversationService.addMessage(conversationId, 'user', userMessage);
    await ConversationService.addMessage(conversationId, 'assistant', response.content[0].text);
    
    // 8. Update context
    await ConversationService.updateContext(conversationId, {
      selectedProductIds,
      unverifiedDataWarningShown: hasUnverifiedProfile || hasUnverifiedProducts
    });
    
    // 9. Check conversation length warning
    const messageCount = conversation.messages.length + 2;
    const shouldWarnLongConversation = messageCount > 30;
    
    return {
      response: response.content[0].text,
      warnings: {
        unverifiedData: hasUnverifiedProfile || hasUnverifiedProducts,
        longConversation: shouldWarnLongConversation,
        messageCount
      }
    };
  }
  
  private static buildSystemPrompt(
    profile: any, 
    products: any[], 
    flags: { hasUnverifiedProfile: boolean; hasUnverifiedProducts: boolean }
  ): string {
    let prompt = `You are a wholesale matchmaking assistant for ${profile.name}.

## Company Profile

**Name:** ${profile.name}
**Website:** ${profile.website}
**Description:** ${profile.description || 'N/A'}
**Industry:** ${profile.categories?.join(', ') || 'N/A'}
**Price Tier:** ${profile.priceTier || 'N/A'}
**Target Markets:** ${profile.targetMarkets?.join(', ') || 'N/A'}

**Wholesale Terms:**
- MOQ: ${profile.wholesale?.moq || 'N/A'}
- Lead Time: ${profile.wholesale?.leadTime || 'N/A'}
- Case Pack: ${profile.wholesale?.casePack || 'N/A'}

**Brand Style:** ${profile.style?.tone || 'N/A'}

## Product Catalog (${products.length} products)

`;

    products.forEach(product => {
      prompt += `
### ${product.name}
- Category: ${product.category}
- Description: ${product.description || 'N/A'}
- Wholesale Price: $${product.wholesale_price || 'N/A'}
- Retail Price: $${product.retail_price || 'N/A'}
- MOQ: ${product.moq || 'N/A'}
`;
    });

    // Add data quality notice
    if (flags.hasUnverifiedProfile || flags.hasUnverifiedProducts) {
      prompt += `

## IMPORTANT: Data Quality Notice
Some of the company information or products have not been verified by the user. 
If making specific recommendations, note that details should be confirmed by the user.
`;
    }

    prompt += `

## Your Task
Help this brand find ideal retail partners. Use the profile and product information 
to make intelligent, personalized recommendations. Reference specific products when 
relevant. Consider brand alignment, not just category matching.
`;

    return prompt;
  }
  
  private static hasUnverifiedFields(profile: any): boolean {
    const fieldSources = profile._metadata?.fieldSources || {};
    return Object.values(fieldSources).some(
      (source: any) => source.verificationStatus === 'unverified'
    );
  }
}
```

**Frontend Chat Component:**
```vue
<!-- ChatView.vue -->
<template>
  <div class="chat-container">
    <!-- Verification Warning Banner -->
    <v-alert
      v-if="showUnverifiedWarning"
      type="warning"
      variant="tonal"
      closable
      class="mb-4"
    >
      You have unverified profile data or products. 
      <router-link to="/brand">Review your Brand tab</router-link> 
      to increase chat response accuracy.
    </v-alert>
    
    <!-- Long Conversation Warning -->
    <v-alert
      v-if="showLongConversationWarning"
      type="info"
      variant="tonal"
      closable
      class="mb-4"
    >
      This conversation is getting long ({{ messageCount }} messages). 
      Consider starting a new conversation for better performance.
    </v-alert>
    
    <!-- Chat messages -->
    <div class="messages">
      <!-- ... messages display ... -->
    </div>
    
    <!-- Product Selector (optional) -->
    <v-expansion-panels v-if="products.length > 0" class="mb-4">
      <v-expansion-panel title="Select Products for This Chat (Optional)">
        <v-expansion-panel-text>
          <v-chip-group v-model="selectedProductIds" multiple column>
            <v-chip
              v-for="product in products"
              :key="product.id"
              :value="product.id"
              filter
              variant="outlined"
            >
              {{ product.name }}
              <v-icon 
                v-if="product.verification_status === 'unverified'" 
                end 
                color="warning"
                size="small"
              >
                mdi-alert-circle
              </v-icon>
            </v-chip>
          </v-chip-group>
          <div class="text-caption mt-2">
            Leave empty to include all products. Products with 
            <v-icon color="warning" size="x-small">mdi-alert-circle</v-icon> 
            need verification.
          </div>
        </v-expansion-panel-text>
      </v-expansion-panel>
    </v-expansion-panels>
    
    <!-- Message input -->
    <v-text-field
      v-model="messageInput"
      label="Type your message..."
      @keydown.enter="sendMessage"
    />
  </div>
</template>

<script setup>
const showUnverifiedWarning = ref(false);
const showLongConversationWarning = ref(false);
const messageCount = ref(0);
const selectedProductIds = ref([]);

async function sendMessage() {
  const response = await api.post(`/conversations/${conversationId}/messages`, {
    content: messageInput.value,
    selectedProductIds: selectedProductIds.value
  });
  
  // Show warnings if needed
  if (response.warnings?.unverifiedData) {
    showUnverifiedWarning.value = true;
  }
  
  if (response.warnings?.longConversation) {
    showLongConversationWarning.value = true;
    messageCount.value = response.warnings.messageCount;
  }
  
  // Clear input and refresh messages
  messageInput.value = '';
  await loadMessages();
}
</script>
```

---

## Out of Scope (For Later Phases)

### Phase 2 Enhancements (Future)
- **High-fidelity product scraping**: [Tier 1: API endpoints, Tier 2: LLM responses, Tier 3: manual scrape]
- **Product images download/storage**: Never in app, always url or CDN possibly
- **Shopify/e-commerce platform integrations**: Direct API connections
- **LinkedIn company page scraping**: Requires separate integration
- **Pre-generated profile documents**: Caching for performance optimization
- **Bull/BullMQ job queue**: Production-grade async processing
- **Conversation insights extraction**: Learning from chat patterns
- **Multi-source data reconciliation**: Merging data from multiple scrapes

### Never In Scope
- Automatic profile updates without user review
- Sharing scraped data between companies
- Scraping competitor pricing/data

---

## Success Criteria

### Phase 1 Complete When:
- [x] Light scrape data saves to profile with metadata
- [x] Email verification triggers deep scrape (async)
- [x] Deep scrape populates products table
- [x] Wizard shows verification badges
- [x] Chat includes profile and products in context
- [x] Unverified data warnings display in chat

### Quality Metrics:
- Deep scrape completes in < 60 seconds for 90% of companies
- 70%+ of scraped fields verified by users within 7 days
- < 10% of scraped products deleted (low false positive rate)
- Chat conversations reference specific products in 60%+ of messages

### Technical Requirements:
- No blocking operations during user flows
- Graceful failure handling (scrape fails → user sees empty products)
- All scraped data clearly marked as unverified
- User edits always take precedence over scraped data

---

## Database Migrations Needed

### Migration 1: Add Verification Fields to Products
```sql
-- File: src/migrations/XXXXXX_add_product_verification.js

exports.up = (pgm) => {
  pgm.addColumns('products', {
    verification_status: {
      type: 'text',
      notNull: true,
      default: 'unverified',
      comment: 'unverified | verified | flagged_for_review'
    },
    scraped: {
      type: 'boolean',
      notNull: true,
      default: false
    },
    confidence_score: {
      type: 'decimal(3,2)',
      notNull: false,
      comment: 'AI confidence score 0.00-1.00'
    },
    scraped_from: {
      type: 'text',
      notNull: false,
      comment: 'URL where product was scraped from'
    },
    scraped_at: {
      type: 'timestamptz',
      notNull: false
    }
  });
  
  pgm.createIndex('products', 'verification_status');
  pgm.createIndex('products', ['company_id', 'verification_status']);
  pgm.createIndex('products', ['company_id', 'scraped'], {
    where: 'scraped = true'
  });
};

exports.down = (pgm) => {
  pgm.dropIndex('products', 'verification_status');
  pgm.dropIndex('products', ['company_id', 'verification_status']);
  pgm.dropIndex('products', ['company_id', 'scraped']);
  
  pgm.dropColumns('products', [
    'verification_status',
    'scraped',
    'confidence_score',
    'scraped_from',
    'scraped_at'
  ]);
};
```

### Migration 2: Add Helper Functions for Profile Queries
```sql
-- File: src/migrations/XXXXXX_add_profile_helpers.js

exports.up = (pgm) => {
  pgm.sql(`
    -- Check if profile has unverified fields
    CREATE OR REPLACE FUNCTION profile_has_unverified_fields(profile_data jsonb)
    RETURNS boolean AS $$
      SELECT EXISTS (
        SELECT 1
        FROM jsonb_each(profile_data->'_metadata'->'fieldSources')
        WHERE value->>'verificationStatus' = 'unverified'
      );
    $$ LANGUAGE sql IMMUTABLE;
    
    -- Get profile completion score
    CREATE OR REPLACE FUNCTION profile_completion_score(profile_data jsonb)
    RETURNS decimal AS $$
      SELECT COALESCE(
        (profile_data->'_metadata'->>'completionScore')::decimal,
        0.0
      );
    $$ LANGUAGE sql IMMUTABLE;
    
    -- Check if deep scrape is pending
    CREATE OR REPLACE FUNCTION profile_deep_scrape_pending(profile_data jsonb)
    RETURNS boolean AS $$
      SELECT COALESCE(
        (profile_data->'_metadata'->>'deepScrapePending')::boolean,
        false
      );
    $$ LANGUAGE sql IMMUTABLE;
  `);
  
  pgm.createIndex('companies', 'profile_has_unverified_fields(profile)', {
    name: 'idx_companies_unverified_fields',
    where: 'profile_has_unverified_fields(profile) = true'
  });
  
  pgm.createIndex('companies', 'profile_completion_score(profile)', {
    name: 'idx_companies_completion_score'
  });
};

exports.down = (pgm) => {
  pgm.dropIndex('companies', 'profile_has_unverified_fields(profile)');
  pgm.dropIndex('companies', 'profile_completion_score(profile)');
  
  pgm.sql(`
    DROP FUNCTION IF EXISTS profile_has_unverified_fields;
    DROP FUNCTION IF EXISTS profile_completion_score;
    DROP FUNCTION IF EXISTS profile_deep_scrape_pending;
  `);
};
```

---

## API Endpoints Needed

### Company Profile
- `PATCH /api/companies/:id/profile` - Update profile fields with verification tracking
- `GET /api/companies/:id/verification-status` - Get verification summary

### Products
- `GET /api/companies/:id/products?verification_status=unverified` - Filter by verification
- `PATCH /api/products/:id/verify` - Mark product as verified
- `PATCH /api/products/:id` - Edit product (auto-verifies)
- `DELETE /api/products/:id` - Delete product

### Chat
- `POST /api/conversations/:id/messages` - Send message with context
  ```json
  {
    "content": "Find retailers in California",
    "selectedProductIds": ["uuid1", "uuid2"]
  }
  ```
- `GET /api/conversations/:id/warnings` - Get data quality warnings

---

## Timeline Estimate

### Step 1: Foundation
- [ ] Add verification fields to products table (Migration 1)
- [ ] Add profile helper functions (Migration 2)
- [ ] Update UserService.activateUser() with first-user check
- [ ] Create CompanyScraperService stub (returns empty products for now)

### Step 2: Wizard UI
- [ ] Create VerificationBadge component
- [ ] Add verification alert banner to wizard
- [ ] Implement auto-verify on edit logic
- [ ] Add products verification table/UI

### Step 3: Chat Integration
- [ ] Update ChatService to include profile + products
- [ ] Add verification warnings to chat UI
- [ ] Add product selector to chat
- [ ] Add long conversation warnings

### step 4: Polish & Testing
- [ ] Email notification for deep scrape complete
- [ ] Error handling and logging
- [ ] Test all verification flows
- [ ] Performance optimization

### Further work: Deep Scrape Implementation
- Research high-fidelity product APIs
- Implement actual product scraping
- Add extended profile scraping (about pages, etc.)

---

## Open Questions / Decisions Needed

1. **Email notification content**: What should the "deep scrape complete" email say?
2. **Product limits**: Should we cap scraped products (e.g., max 500)?
3. **Verification incentives**: Reward users for verifying data?
4. **Admin dashboard**: Should admins see verification stats across all companies?
5. **Data retention**: How long to keep scraping history in _metadata?

---

## Notes

- This PRD focuses on architecture and data flow
- Simple fire-and-forget async model for now; can upgrade to Bull/BullMQ later
- Verification system is non-blocking but visible - users can use the system with unverified data
- Chat quality improves as users verify, creating natural incentive to clean up data