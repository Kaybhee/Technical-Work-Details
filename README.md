# EzzyExpress Smart Shopping Assistant — Revised Plan v3

## Approach: AI-Powered Intent-Driven Shopping, Not Just Meals

The system handles ANY shopping intent, not just "cook jollof rice":

- **Meal intent**: "I want to cook jollof rice" → ingredient-based cart
- **Event intent**: "I'm throwing a party for 20 people" → party supplies, drinks, snacks cart
- **Stocking intent**: "Stock my kitchen for the week" → staples + user's usual items from order history
- **Occasion intent**: "Ramadan provisions" or "Christmas groceries" → culturally relevant products
- **Budget intent**: "₦10,000 weekly groceries" → optimized cart within budget
- **Diet intent**: "I want to lose weight" → filtered healthy products
- **Hybrid**: Any combination of the above

Plus **personalization** powered by existing data:

- Past orders → "buy again" suggestions, personal staples detection
- Wishlist → incorporate wished items into recommendations
- Ratings → prefer products user has rated highly
- Popular products → trending/best-rated as fallback
- Spending patterns → budget-aware suggestions

---

## Phase 1: Gemini Integration & Product Auto-Tagging

### 1.1 Gemini Service

**New:** `src/modules/smart-shopping/services/gemini.service.ts`

- `@google/generative-ai` SDK wrapper with retry, rate limiting, JSON response enforcement
- Config: `GEMINI_API_KEY` in env
- Model: Gemini Flash for cheap batch work, Pro for complex intent

### 1.2 Automated Product Tagging

**New:** `src/modules/smart-shopping/services/product-tagger.service.ts`

- Sends product `name + description + category + brandName + additionalInformation` to Gemini
- Returns: `{ tags: ["rice", "grain", "staple"], healthTags: ["high_carb"], contextTags: ["party", "everyday", "nigerian"] }`
- **contextTags** = new — tells us WHEN/WHERE a product is relevant (party, breakfast, holiday, office snack, etc.)
- Batch processing: 20 products per Gemini call (batch in prompt, not multiple calls)
- Admin trigger: `POST /admin/smart-shopping/auto-tag-products`
- Auto-tag on new product creation

### 1.3 Enhance Product Schema

**Modify:** `src/modules/admin/product-management/products/product.schema.ts`

- Add `tags: [String]` — functional tags (rice, chicken, oil)
- Add `healthTags: [String]` — dietary tags (low_sugar, high_protein)
- Add `contextTags: [String]` — situational tags (party, breakfast, ramadan, christmas, everyday, staple)
- Add `isAiTagged: Boolean` (default: false)
- Add `aiTaggedAt: Date`

---

## Phase 2: AI Intent Engine (The Brain)

### 2.1 Intent Extraction Service

**New:** `src/modules/smart-shopping/services/intent-engine.service.ts`

Unlike the old plan that only extracted meal/diet/condition, this extracts **any shopping intent**:

**Input:** Free text from user
**Output (structured):**

```
{
  intentType: "meal" | "event" | "stocking" | "occasion" | "budget" | "diet" | "general",
  mealName?: string,           // "jollof rice"
  eventType?: string,          // "party", "picnic", "bbq"
  occasion?: string,           // "ramadan", "christmas", "birthday"
  dietaryGoal?: string,        // "weight_loss", "fitness"
  healthCondition?: string,    // "diabetes"
  servings?: number,           // 4
  budget?: number,             // 10000 (naira)
  duration?: string,           // "week", "month"
  preferences?: string[],      // any explicit preferences mentioned
}
```

Gemini prompt includes the full taxonomy of intent types with examples. Responds with structured JSON.

### 2.2 AI Cart Generator Service

**New:** `src/modules/smart-shopping/services/ai-cart-generator.service.ts`

This replaces the old "recipe generator" with a broader concept — it generates a **shopping list** for ANY intent, not just meals.

**Flow:**

1. Receive structured intent
2. Check MongoDB cache (keyed by normalized intent hash)
3. **Cache HIT** → return cached shopping list
4. **Cache MISS** → Call Gemini:
   - Meal: "What ingredients to cook jollof rice for 4?" → ingredient list
   - Event: "What products for a Nigerian party for 20?" → snacks, drinks, supplies list
   - Stocking: "Weekly kitchen staples for a Nigerian household?" → essentials list
   - Occasion: "Typical Ramadan provisions list?" → dates, grains, drinks, etc.
   - Budget: "₦10,000 weekly groceries for 1 person in Nigeria?" → optimized list
5. Parse, validate, cache, return

**Cache Schema (`CachedShoppingList`):**

- `intentHash: string` (unique, indexed) — hash of normalized intent
- `intentType: string`
- `displayTitle: string` — "Jollof Rice for 4" / "Party Supplies for 20"
- `items[]`:
  - `name: string` — "rice", "soft drinks"
  - `searchTerms: string[]` — ["rice", "parboiled rice", "basmati rice"]
  - `category: string` — "grains", "beverages"
  - `suggestedQuantity: number`
  - `unit: string`
  - `isEssential: boolean`
  - `group: string` — "ingredients", "drinks", "snacks", "supplies" (for UI grouping)
- `createdAt, updatedAt`

---

## Phase 3: Personalization Layer

### 3.1 User Preferences Service

**New:** `src/modules/smart-shopping/services/personalization.service.ts`

Leverages EXISTING data — no new tracking infrastructure needed:

**Methods:**

- `getUserStaples(userId)` — Analyze past completed orders, find frequently purchased products (bought 3+ times). These are the user's "staples."
- `getUserCategoryPreferences(userId)` — Rank categories by purchase frequency
- `getWishlistProducts(userId)` — Fetch current wishlist items
- `getUserBudgetRange(userId)` — Calculate average order total from history
- `getUserTopRatedProducts(userId)` — Products this user rated 4-5 stars

### 3.2 How Personalization Feeds Into Cart Generation

When generating a cart, the matching engine considers personalization:

- **Stocking intent** → start with user's staples, supplement with popular products they haven't tried
- **Any intent** → if user has a wishlist item matching the shopping list, prefer that specific product
- **Any intent** → when multiple products match an ingredient, prefer ones user has bought/rated before
- **Budget intent** → use user's avg budget if not specified
- **Trending** → surface high-averageRating products as alternatives

---

## Phase 4: Matching Engine

### 4.1 Matching Engine Service

**New:** `src/modules/smart-shopping/services/matching-engine.service.ts`

For each item in the AI-generated shopping list:

1. **Tag match** — product `tags` intersect item `searchTerms`
2. **Text search** — product name contains search terms (fallback)
3. **Rank by:**
   - User bought before (personalization boost)
   - User wishlisted (boost)
   - In stock (must-have)
   - Price ascending
   - Average rating descending
4. **Diet filtering** (only when diet/health intent active):
   - Remove products with `healthTags` matching restricted tags
   - Boost products with allowed health tags
5. **Context filtering** — if intent is "party", boost products with contextTag "party"
6. Return top match + 2–3 alternatives per item

### 4.2 Budget Optimizer

When budget is specified:

- After matching, total the cart
- If over budget: swap items for cheaper alternatives, reduce non-essential quantities
- If under budget: suggest add-ons from user's staples or popular items
- Return: items + total + budget utilization %

### 4.3 Quantity Scaling

- Scale quantities from shopping list based on servings/duration
- Round up to nearest sellable unit from product's unitPricings

---

## Phase 5: Diet Profiles (Hardcoded Constants)

### 5.1 Diet Profile Constants

**New:** `src/modules/smart-shopping/constants/diet-profiles.ts`

- 5 profiles: Weight Loss, Fitness, Diabetes-Friendly, Low Carb, Low Sodium
- Each: `allowedTags[]`, `restrictedTags[]`, `moderationTags[]`
- Applied as optional filter layer, not the core

### 5.2 Health Tag Vocabulary

**New:** `src/modules/smart-shopping/constants/health-tags.ts`

- Controlled vocabulary for AI tagging consistency

---

## Phase 6: Smart Shopping API

### 6.1 Controller

**New:** `src/modules/smart-shopping/smart-shopping.controller.ts`

**Endpoints:**

- `POST /smart-shopping/interpret` — Free text → structured intent
- `POST /smart-shopping/generate-cart` — Structured intent → smart cart with matched products
- `GET /smart-shopping/suggestions` — Browseable suggestions (popular cached lists, user's staples, trending)
- `GET /smart-shopping/reorder/:orderId` — Re-generate cart from a past order
- `POST /smart-shopping/apply-to-cart` — Push smart cart into user's real cart

### 6.2 Service

**New:** `src/modules/smart-shopping/smart-shopping.service.ts`

**Core flow:**

1. Receive structured intent + hubId + userId
2. Generate shopping list (from cache or AI)
3. Load personalization data (staples, wishlist, ratings)
4. Run matching engine for each item in shopping list
5. Apply diet filtering if applicable
6. Apply budget optimization if applicable
7. Apply context filtering (party, occasion, etc.)
8. Return: `{ title, items: [{ product, alternatives[], quantity, unit, group, isPersonalized }], totalEstimate, budgetUtilization?, warnings[], disclaimer }`

### 6.3 Module

**New:** `src/modules/smart-shopping/smart-shopping.module.ts`

- Providers: SmartShoppingService, GeminiService, ProductTaggerService, AiCartGeneratorService, MatchingEngineService, PersonalizationService
- Controllers: SmartShoppingController
- Imports: RepositoryModule, MongooseModelsModule

---

## Phase 7: Admin Endpoints (Automation Only)

### 7.1 Controller

**New:** `src/modules/admin/smart-shopping-management/smart-shopping-admin.controller.ts`

- `POST /admin/smart-shopping/auto-tag-products` — Trigger batch AI tagging
- `GET /admin/smart-shopping/tagging-status` — Progress dashboard
- `POST /admin/smart-shopping/retag-product/:id` — Re-tag single product
- `GET /admin/smart-shopping/cached-lists` — View cached shopping lists
- `DELETE /admin/smart-shopping/cached-lists/:id` — Invalidate a cached list

---

## Phase 8: Integration & Safety

### 8.1 Product Creation Hook

- Auto-tag on new product creation in existing product service

### 8.2 Cart Integration

- Extend cart-management service with `addSmartCartItems(employeeId, hubId, items[])`
- Respect single-hub constraint
- User can edit/remove/swap after applying

### 8.3 Safety

- Medical disclaimer on health-related responses
- Diet labels framed as "dietary preferences"
- AI prompts include guardrails
- Rate limiting on AI endpoints

---

## File Summary

### New Files (14)

| File                                                                | Purpose                          |
| ------------------------------------------------------------------- | -------------------------------- |
| `src/modules/smart-shopping/smart-shopping.module.ts`               | Module                           |
| `src/modules/smart-shopping/smart-shopping.controller.ts`           | API endpoints                    |
| `src/modules/smart-shopping/smart-shopping.service.ts`              | Core orchestration               |
| `src/modules/smart-shopping/services/gemini.service.ts`             | Gemini wrapper                   |
| `src/modules/smart-shopping/services/product-tagger.service.ts`     | AI auto-tagging                  |
| `src/modules/smart-shopping/services/intent-engine.service.ts`      | Intent extraction                |
| `src/modules/smart-shopping/services/ai-cart-generator.service.ts`  | Shopping list generation + cache |
| `src/modules/smart-shopping/services/matching-engine.service.ts`    | Product matching                 |
| `src/modules/smart-shopping/services/personalization.service.ts`    | User preference analysis         |
| `src/modules/smart-shopping/schemas/cached-shopping-list.schema.ts` | Cache model                      |
| `src/modules/smart-shopping/constants/diet-profiles.ts`             | Diet profiles                    |
| `src/modules/smart-shopping/constants/health-tags.ts`               | Tag vocabulary                   |
| `src/modules/smart-shopping/dtos/`                                  | DTOs                             |
| `src/modules/admin/smart-shopping-management/`                      | Admin endpoints                  |

### Existing Files to Modify (8)

| File                     | Change                                                    |
| ------------------------ | --------------------------------------------------------- |
| Product schema           | Add tags, healthTags, contextTags, isAiTagged, aiTaggedAt |
| Product DTOs             | New fields                                                |
| Product creation service | Hook auto-tag                                             |
| MongooseModelsModule     | Register CachedShoppingList                               |
| RepositoryModule         | Add CachedShoppingList repo                               |
| Cart management service  | Bulk-add from smart cart                                  |
| app.module.ts            | Import SmartShoppingModule                                |
| Config files             | Add GEMINI_API_KEY                                        |

---

## End-to-End Examples

### Example 1: "I want to cook jollof rice for 4 people"

→ Intent: { type: "meal", mealName: "jollof_rice", servings: 4 }
→ AI generates: rice, tomato paste, tomatoes, pepper, onions, oil, seasoning, chicken
→ Matching engine finds products per ingredient in user's hub
→ Personalization: user bought "Mama Gold Rice" before → prefer it
→ Quantities scaled for 4 servings
→ Cart: 8 products with alternatives

### Example 2: "I'm throwing a birthday party for 20"

→ Intent: { type: "event", eventType: "birthday_party", servings: 20 }
→ AI generates: soft drinks, juice, chips, cake mix, disposable plates, small chops ingredients, rice, chicken
→ Products matched + boosted by contextTag "party"
→ Budget-aware if user specifies budget

### Example 3: "Stock my kitchen for the week"

→ Intent: { type: "stocking", duration: "week" }
→ Personalization: fetch user's staples from order history (rice, bread, eggs, milk → bought 5+ times)
→ AI supplements: common weekly staples user hasn't tried
→ Cart: personalized weekly grocery list

### Example 4: "₦15,000 budget, healthy food for the week"

→ Intent: { type: "budget", budget: 15000, dietaryGoal: "healthy", duration: "week" }
→ AI generates: healthy weekly shopping list
→ Matching engine + budget optimizer keeps total under ₦15K
→ Diet filtering removes junk food options

### Example 5: "Ramadan provisions for my family"

→ Intent: { type: "occasion", occasion: "ramadan", servings: 5 }
→ AI generates: dates, rice, pasta, beverages, cooking oil, spices, flour
→ Matched and scaled for family of 5
