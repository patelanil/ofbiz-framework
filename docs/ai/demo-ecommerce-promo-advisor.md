# Implementation Plan — Ecommerce Promo Advisor Demo

> **Lives in:** `plugins/ai/` (framework plugin hosts the demo)
> **Branch (plugins):** feature/ai-plugin
> **Status:** TODO

---

## What we are building

A demo AI agent named **PromoAdvisor** that helps a merchandising manager decide
which products to promote and at what price, then sets the promotional price after
manager approval. Uses only existing OFBiz demo data (GZ-7000, WG-9943, etc.).

**Conversation flow (two turns):**

Turn 1 — Analysis:
> User: "Which gizmos or widgets should we promote this week?"
> Agent calls 3 read-only tools, reasons over real price/inventory/sales data,
> returns a recommendation with specific price suggestions.

Turn 2 — Action (approval-gated):
> User: "Set up the $539.99 promo on GZ-7000 for 7 days."
> Agent calls setProductPromoPrice (requires-approval=true), suspends,
> manager approves in /ai/control/FindAiAgentProposal, price goes live.

---

## Files to create/modify

| File | Action |
|---|---|
| `plugins/ai/servicedef/services.xml` | Add 4 service definitions |
| `plugins/ai/groovyScripts/GetProductPriceSummary.groovy` | New — price lookup tool |
| `plugins/ai/groovyScripts/GetProductInventorySummary.groovy` | New — inventory lookup tool |
| `plugins/ai/groovyScripts/GetRecentOrderActivity.groovy` | New — sales history tool |
| `plugins/ai/groovyScripts/SetProductPromoPrice.groovy` | New — approval-gated price write |
| `plugins/ai/ai/ecommerce-promo.tools.xml` | New — tool declarations |
| `plugins/ai/ai/ecommerce-promo-advisor.agent.xml` | New — agent declaration |

---

## Key entity facts (from demo data)

**ProductPrice PK:** productId + productPriceTypeId + productPricePurposeId +
currencyUomId + productStoreGroupId + fromDate

All demo selling prices use:
- productPricePurposeId = "PURCHASE"
- currencyUomId = "USD"
- productStoreGroupId = "_NA_"

Active price = fromDate <= now AND (thruDate IS NULL OR thruDate > now).
Creating a new SPECIAL_PROMO_PRICE with fromDate=now and thruDate=now+7days
overrides any prior promo for that window.

**ProductFacility.lastInventoryCount** = ATP (available to promise), updated hourly.

**OrderItem** joins **OrderHeader** on orderId. Filter:
- OrderHeader.orderTypeId = "SALES_ORDER"
- OrderHeader.orderDate >= cutoff
- OrderHeader.statusId != "ORDER_CANCELLED"

---

## Step 1 — Service definitions in services.xml

Add four services inside `<services>`:

```xml
<service name="getProductPriceSummary" engine="groovy"
         location="component://ai/groovyScripts/GetProductPriceSummary.groovy"
         invoke="getProductPriceSummary" auth="true">
    <description>Return all active USD selling prices for a product (default, list, cost, promo, competitive)</description>
    <attribute name="productId"       mode="IN"  type="String"     optional="false"/>
    <attribute name="productName"     mode="OUT" type="String"     optional="false"/>
    <attribute name="defaultPrice"    mode="OUT" type="BigDecimal" optional="true"/>
    <attribute name="listPrice"       mode="OUT" type="BigDecimal" optional="true"/>
    <attribute name="averageCost"     mode="OUT" type="BigDecimal" optional="true"/>
    <attribute name="competitivePrice" mode="OUT" type="BigDecimal" optional="true"/>
    <attribute name="activePromoPrice" mode="OUT" type="BigDecimal" optional="true"/>
    <attribute name="currencyUomId"   mode="OUT" type="String"     optional="false"/>
</service>

<service name="getProductInventorySummary" engine="groovy"
         location="component://ai/groovyScripts/GetProductInventorySummary.groovy"
         invoke="getProductInventorySummary" auth="true">
    <description>Return total available-to-promise quantity for a product across all facilities</description>
    <attribute name="productId"          mode="IN"  type="String"     optional="false"/>
    <attribute name="totalAtpQuantity"   mode="OUT" type="BigDecimal" optional="false"/>
    <attribute name="facilityBreakdown"  mode="OUT" type="String"     optional="false"/>
</service>

<service name="getRecentOrderActivity" engine="groovy"
         location="component://ai/groovyScripts/GetRecentOrderActivity.groovy"
         invoke="getRecentOrderActivity" auth="true">
    <description>Return order count, quantity sold, and revenue for a product over the last N days</description>
    <attribute name="productId"    mode="IN"  type="String"     optional="false"/>
    <attribute name="days"         mode="IN"  type="Integer"    optional="true"/>
    <attribute name="orderCount"   mode="OUT" type="Integer"    optional="false"/>
    <attribute name="quantitySold" mode="OUT" type="BigDecimal" optional="false"/>
    <attribute name="totalRevenue" mode="OUT" type="BigDecimal" optional="false"/>
    <attribute name="periodDays"   mode="OUT" type="Integer"    optional="false"/>
</service>

<service name="setProductPromoPrice" engine="groovy"
         location="component://ai/groovyScripts/SetProductPromoPrice.groovy"
         invoke="setProductPromoPrice" auth="true">
    <description>Create a SPECIAL_PROMO_PRICE entry for a product. Requires manager approval before execution.</description>
    <attribute name="productId"   mode="IN"  type="String"     optional="false"/>
    <attribute name="promoPrice"  mode="IN"  type="BigDecimal" optional="false"/>
    <attribute name="durationDays" mode="IN" type="Integer"    optional="true"/>
    <attribute name="confirmedProductId" mode="OUT" type="String"     optional="false"/>
    <attribute name="confirmedPrice"     mode="OUT" type="BigDecimal" optional="false"/>
    <attribute name="confirmedFromDate"  mode="OUT" type="String"     optional="false"/>
    <attribute name="confirmedThruDate"  mode="OUT" type="String"     optional="false"/>
</service>
```

**Verify:** `./gradlew classes` — BUILD SUCCESSFUL
**Commit:** `feat(ai): add ecommerce promo advisor service definitions`

---

## Step 2 — GetProductPriceSummary.groovy

```groovy
import org.apache.ofbiz.base.util.UtilDateTime

def getProductPriceSummary() {
    String productId = parameters.productId
    if (!productId) return error("productId is required")

    GenericValue product = from("Product").where("productId", productId).queryOne()
    if (!product) return error("Product not found: ${productId}")

    Timestamp now = UtilDateTime.nowTimestamp()
    List<GenericValue> prices = from("ProductPrice")
            .where("productId", productId,
                   "productPricePurposeId", "PURCHASE",
                   "currencyUomId", "USD",
                   "productStoreGroupId", "_NA_")
            .filterByDate()
            .queryList()

    Map<String, BigDecimal> byType = [:]
    for (GenericValue p : prices) {
        byType[p.productPriceTypeId] = p.getBigDecimal("price")
    }

    Map result = success()
    result.productName = product.getString("productName") ?: product.getString("internalName") ?: productId
    result.defaultPrice    = byType["DEFAULT_PRICE"]
    result.listPrice       = byType["LIST_PRICE"]
    result.averageCost     = byType["AVERAGE_COST"]
    result.competitivePrice = byType["COMPETITIVE_PRICE"]
    result.activePromoPrice = byType["SPECIAL_PROMO_PRICE"]
    result.currencyUomId   = "USD"
    return result
}
```

**Verify:** `./gradlew classes && ./gradlew checkstyleMain`
**Commit:** `feat(ai): add GetProductPriceSummary Groovy service`

---

## Step 3 — GetProductInventorySummary.groovy

```groovy
def getProductInventorySummary() {
    String productId = parameters.productId
    if (!productId) return error("productId is required")

    List<GenericValue> facilities = from("ProductFacility")
            .where("productId", productId)
            .queryList()

    if (!facilities) {
        return success([totalAtpQuantity: BigDecimal.ZERO,
                        facilityBreakdown: "No facility records found for ${productId}"])
    }

    BigDecimal total = BigDecimal.ZERO
    List<String> lines = []
    for (GenericValue pf : facilities) {
        BigDecimal atp = pf.getBigDecimal("lastInventoryCount") ?: BigDecimal.ZERO
        total = total.add(atp)
        lines << "${pf.facilityId}: ${atp}"
    }

    return success([totalAtpQuantity:  total,
                    facilityBreakdown: lines.join(", ")])
}
```

**Verify:** `./gradlew classes && ./gradlew checkstyleMain`
**Commit:** `feat(ai): add GetProductInventorySummary Groovy service`

---

## Step 4 — GetRecentOrderActivity.groovy

```groovy
import org.apache.ofbiz.base.util.UtilDateTime
import org.apache.ofbiz.entity.condition.EntityCondition
import org.apache.ofbiz.entity.condition.EntityOperator
import java.sql.Timestamp

def getRecentOrderActivity() {
    String productId = parameters.productId
    if (!productId) return error("productId is required")

    int days = parameters.days ? parameters.days as int : 30
    Timestamp fromDate = new Timestamp(System.currentTimeMillis() - (days * 24L * 60 * 60 * 1000))

    // Step 1: get eligible sales order IDs in the window
    List<GenericValue> headers = from("OrderHeader")
            .where(EntityCondition.makeCondition([
                EntityCondition.makeCondition("orderTypeId", EntityOperator.EQUALS, "SALES_ORDER"),
                EntityCondition.makeCondition("orderDate",   EntityOperator.GREATER_THAN_EQUAL_TO, fromDate),
                EntityCondition.makeCondition("statusId",    EntityOperator.NOT_EQUAL, "ORDER_CANCELLED")
            ], EntityOperator.AND))
            .select("orderId")
            .queryList()

    if (!headers) {
        return success([orderCount: 0, quantitySold: BigDecimal.ZERO,
                        totalRevenue: BigDecimal.ZERO, periodDays: days])
    }

    List<String> orderIds = headers.collect { it.getString("orderId") }

    // Step 2: find items for this product in those orders
    List<GenericValue> items = from("OrderItem")
            .where(EntityCondition.makeCondition([
                EntityCondition.makeCondition("productId", EntityOperator.EQUALS, productId),
                EntityCondition.makeCondition("orderId",   EntityOperator.IN, orderIds)
            ], EntityOperator.AND))
            .select("orderId", "quantity", "unitPrice")
            .queryList()

    BigDecimal qtyTotal = BigDecimal.ZERO
    BigDecimal revTotal = BigDecimal.ZERO
    Set<String> seenOrders = []
    for (GenericValue item : items) {
        BigDecimal qty   = item.getBigDecimal("quantity")  ?: BigDecimal.ZERO
        BigDecimal price = item.getBigDecimal("unitPrice") ?: BigDecimal.ZERO
        qtyTotal = qtyTotal.add(qty)
        revTotal = revTotal.add(qty.multiply(price))
        seenOrders.add(item.getString("orderId"))
    }

    return success([orderCount:   seenOrders.size(),
                    quantitySold: qtyTotal,
                    totalRevenue: revTotal,
                    periodDays:   days])
}
```

**Verify:** `./gradlew classes && ./gradlew checkstyleMain`
**Commit:** `feat(ai): add GetRecentOrderActivity Groovy service`

---

## Step 5 — SetProductPromoPrice.groovy

```groovy
import org.apache.ofbiz.base.util.UtilDateTime
import java.sql.Timestamp

def setProductPromoPrice() {
    String productId    = parameters.productId
    BigDecimal promoPrice = parameters.promoPrice
    int durationDays    = parameters.durationDays ? parameters.durationDays as int : 7

    if (!productId)   return error("productId is required")
    if (promoPrice == null || promoPrice <= BigDecimal.ZERO) {
        return error("promoPrice must be a positive value")
    }

    GenericValue product = from("Product").where("productId", productId).queryOne()
    if (!product) return error("Product not found: ${productId}")

    Timestamp fromDate = UtilDateTime.nowTimestamp()
    Timestamp thruDate = new Timestamp(fromDate.time + (durationDays * 24L * 60 * 60 * 1000))

    GenericValue priceRecord = delegator.makeValue("ProductPrice")
    priceRecord.set("productId",              productId)
    priceRecord.set("productPriceTypeId",     "SPECIAL_PROMO_PRICE")
    priceRecord.set("productPricePurposeId",  "PURCHASE")
    priceRecord.set("currencyUomId",          "USD")
    priceRecord.set("productStoreGroupId",    "_NA_")
    priceRecord.set("fromDate",               fromDate)
    priceRecord.set("thruDate",               thruDate)
    priceRecord.set("price",                  promoPrice)
    priceRecord.set("createdDate",            fromDate)
    priceRecord.set("createdByUserLogin",     userLogin?.getString("userLoginId") ?: "system")
    priceRecord.set("lastModifiedDate",       fromDate)
    priceRecord.set("lastModifiedByUserLogin", userLogin?.getString("userLoginId") ?: "system")
    delegator.create(priceRecord)

    return success([confirmedProductId:  productId,
                    confirmedPrice:      promoPrice,
                    confirmedFromDate:   fromDate.toString(),
                    confirmedThruDate:   thruDate.toString()])
}
```

**Verify:** `./gradlew classes && ./gradlew checkstyleMain`
**Commit:** `feat(ai): add SetProductPromoPrice Groovy service`

---

## Step 6 — ecommerce-promo.tools.xml

File: `plugins/ai/ai/ecommerce-promo.tools.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements. ...
-->
<tools>

    <tool name="getProductPriceSummary"
          service="getProductPriceSummary">
        <description>Look up all active USD prices for a product: default (selling) price, list price,
average cost, promotional price, and competitive (competitor) price. Use this first to understand
a product's current pricing and margin before recommending a promotion.</description>
    </tool>

    <tool name="getProductInventorySummary"
          service="getProductInventorySummary">
        <description>Return the total available-to-promise (ATP) inventory quantity for a product across
all warehouses, plus a per-facility breakdown. Use this to check if we have enough stock
to support a promotion before recommending one.</description>
    </tool>

    <tool name="getRecentOrderActivity"
          service="getRecentOrderActivity">
        <description>Return the number of orders, total quantity sold, and total revenue for a product
over the last N days (default 30). Use this to assess whether a product is selling well
or needs a promotional push.</description>
    </tool>

    <tool name="setProductPromoPrice"
          service="setProductPromoPrice"
          requires-approval="true"
          required-permission="CATALOG_PRICE_MAINT">
        <description>Create a SPECIAL_PROMO_PRICE for a product in USD, active for the specified number
of days (default 7). This is a WRITE operation that changes the live store price and requires
manager approval before it executes. Only call this when the user has confirmed they want to
proceed with the promotion.</description>
    </tool>

</tools>
```

**Verify:** `./gradlew classes`
**Commit:** `feat(ai): add ecommerce-promo tools XML`

---

## Step 7 — ecommerce-promo-advisor.agent.xml

File: `plugins/ai/ai/ecommerce-promo-advisor.agent.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements. ...
-->
<agents>
    <agent name="PromoAdvisor"
           provider="openai-default"
           max-iterations="6">

        <system-prompt><![CDATA[
You are a merchandising advisor for Open For Business, an ecommerce store selling Gizmos and Widgets.

Your job is to help the merchandising team decide which products to put on promotion and at what price.

When asked about promotions:
1. Call getProductPriceSummary to understand current pricing and margins.
2. Call getProductInventorySummary to check stock availability.
3. Call getRecentOrderActivity to see how the product is selling.
4. Reason over the data: a good promotion candidate has healthy stock, slow recent sales,
   and room between cost and list price to offer a meaningful discount.
5. Recommend a specific promotional price that is above average cost but below the default price,
   and explain your reasoning with the actual numbers.

When the user asks you to apply a promotion:
- Confirm the product ID, price, and duration before calling setProductPromoPrice.
- Only call setProductPromoPrice once you have explicit user confirmation.
- Inform the user that the price change will require manager approval before it goes live.

Products available in this store: GZ-1000, GZ-1001, GZ-2002, GZ-2644, GZ-5005,
GZ-1004, GZ-1005, GZ-7000, GZ-8544, GZ-9290, WG-1111, WG-5569, WG-9943, WG-9944.
        ]]></system-prompt>

        <tool name="getProductPriceSummary"/>
        <tool name="getProductInventorySummary"/>
        <tool name="getRecentOrderActivity"/>
        <tool name="setProductPromoPrice"/>
    </agent>
</agents>
```

**Verify:** `./gradlew classes`
**Commit:** `feat(ai): add PromoAdvisor agent definition`

---

## Step 8 — Full verification

```bash
./gradlew classes
./gradlew checkstyleMain
./gradlew codenarcMain
```

Expected: zero errors, zero violations.

**Manual test via /ai webapp:**
1. Go to `/ai/control/FindAiAgents` — PromoAdvisor should appear in the list
2. Click "Run Agent" → type: "Which gizmos or widgets should we promote this week?"
3. Observe 3 tool calls in the run detail, coherent recommendation
4. Second turn: "Set up the $539.99 promo on GZ-7000 for 7 days."
5. Confirm approval gate fires — proposalId appears, status = approval_required
6. Go to `/ai/control/FindAiAgentProposal` → Approve → confirm ProductPrice record created

---

## Demo conversation (expected)

**Turn 1:**
> User: "Which gizmos or widgets should we promote this week?"
>
> [Agent calls getProductPriceSummary(GZ-7000), getProductInventorySummary(GZ-7000), getRecentOrderActivity(GZ-7000)]
>
> Agent: "GZ-7000 (Massive Gizmo) is the strongest promotion candidate this week.
> It sells for $599.99 against a list price of $999.99, with an average cost of $499.99
> from DemoSupplier — giving us a $100 margin floor. Inventory shows 100 units on hand
> at WebStoreWarehouse. It sold 0 orders in the last 30 days, meaning it needs a push.
> I recommend a 7-day flash sale at $539.99 — 10% off the default price, $40 above cost,
> and well below the $999.99 list price to create urgency."

**Turn 2:**
> User: "Go ahead and set up $539.99 on GZ-7000 for 7 days."
>
> [Agent calls setProductPromoPrice(GZ-7000, 539.99, 7) — SUSPENDED]
>
> Agent: "I've prepared a promotional price of $539.99 for GZ-7000 starting now and
> running for 7 days. This requires manager approval before it goes live. A manager can
> review and approve it at /ai/control/FindAiAgentProposal."
