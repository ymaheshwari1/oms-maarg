# Archive: `split#OrderASNItems` — Original Implementation

**Source:** `runtime/component/oms/service/co/hotwax/oms/order/OrderServices.xml` (Lines 1341–1503)
**Archived:** 2026-02-23
**Reason:** Replaced with a simplified version that uses `OrderItem.shipGroupSeqId` directly instead of the deprecated `OldOrderItemShipGroupAssoc` entity.

---

## 🧠 Logic Explanation

**Purpose:** Move pre-order and backorder items from their current ship group (typically a `_NA_` facility parking group) into a dedicated `PRE_ORDER` or `BACKORDER` facility ship group.

### Phase 1: Determine WHICH items to split
*Find all preorder/backorder tagged items and group them by target facility type.*

1. Validate the order exists.
2. Check the `ProductStore.allowSplit` flag.
3. Find items with the `PreOrderItemProperty` attribute → `preOrderItemSeqIds`.
4. Find items with the `BackOrderItemProperty` attribute → `backOrderItemSeqIds`.
5. Build a map: `{ facilityTypeId → [itemSeqIds] }`
   - **If `allowSplit = Y` (default):** Only the tagged items move. Pre-order → `PRE_ORDER`, backorder → `BACKORDER`.
   - **If `allowSplit = N`:** If ANY item is pre-order → move ALL items to `PRE_ORDER`. Else if ANY is backorder → move ALL to `BACKORDER`. (The entire order stays together).

### Phase 2: Find target facility
*Resolve the facility ID for the destination.*

For each entry (`PRE_ORDER` or `BACKORDER`):
- Find a `Facility` of that `facilityTypeId`.
- If no such facility exists, log a warning and skip.

### Phase 3: Load all items & groups
*Pre-fetch data to avoid N+1 queries during the filter loop.*

- `orderItemList` → ALL OrderItem records for this order.
- `orderItemBySeq` → Map: `{ orderItemSeqId → OrderItem }` (used for status lookup in Phase 4).
- `shipGroupList` → ALL OrderItemShipGroup records for this order.
- `shipGroupBySeq` → Map: `{ shipGroupSeqId → OrderItemShipGroup }` (used for facility lookup in Phase 4).

### Phase 4: Find eligible assocs and filter
*Use `OldOrderItemShipGroupAssoc` to find items currently parked at `_NA_`.*

Query `OldOrderItemShipGroupAssoc` for the target item seq IDs, then filter OUT where:
- The OrderItem doesn't exist, or its ship group doesn't exist.
- The ship group's facility is NOT `_NA_` (i.e., it's already assigned to a real facility).
- The item is `ITEM_CANCELLED` or `ITEM_COMPLETED`.

**Result:** `filteredAssocs` — only active items currently parked at `_NA_`.

### Phase 5: Group by ship group and determine target
*Decide whether to create a new ship group or reuse an existing one.*

For each unique `shipGroupSeqId` in `filteredAssocs`:

1. Collect the `orderItemSeqId`s in that group → `groupItemSeqIds`.
2. **Check if target facility already has a ship group** for this order → reuse it.
3. **If not, check if current ship group can be reused (in-place update):** Count items remaining in the group (NOT in `groupItemSeqIds`). If `remainingCount == 0` → all items are moving → update the ship group's facility directly.
4. **If neither works → create a new ship group:** Clone current group's properties (shipping method, address, etc.), assign a new `shipGroupSeqId` and the target facility.

### Phase 6: Move items (The Dual-Write Pattern)
*Move the items using the old assoc entity.*

Only if `newShipGroupSeqId != shipGroupSeqId` (items actually need to move):

For each item — **dual-write pattern:**
1. Find old assoc record `(orderId, orderItemSeqId, oldShipGroupSeqId)`.
2. If not found → skip.
3. Check if assoc already exists at the new ship group → don't create duplicate.
4. **[CRUD 1] Create** new assoc at `newShipGroupSeqId` (copying quantity/cancelQuantity).
5. **[CRUD 2] Delete** old assoc at `oldShipGroupSeqId`.
6. **[CRUD 3] Update** `OrderItem.shipGroupSeqId` to `newShipGroupSeqId`.

> **Why this was refactored:** Steps 1–5 maintain `OldOrderItemShipGroupAssoc` as a redundant shadow copy of reality. `OrderItem.shipGroupSeqId` (step 6) is the canonical source of truth in the modern data model.

---

## 📜 Original XML Implementation

```xml
    <service verb="split" noun="OrderASNItems">
        <description>Split sales order items into preorder/backorder ship groups</description>
        <in-parameters>
            <parameter name="orderId" required="true"/>
        </in-parameters>
        <actions>
            <entity-find-one entity-name="org.apache.ofbiz.order.order.OrderHeader" value-field="orderHeader">
                <field-map field-name="orderId"/>
            </entity-find-one>
            <if condition="!orderHeader">
                <return type="warning" message="Order not found for orderId ${orderId}"/>
            </if>

            <entity-find-one entity-name="org.apache.ofbiz.product.store.ProductStore" value-field="productStore" cache="true">
                <field-map field-name="productStoreId" from="orderHeader.productStoreId"/>
            </entity-find-one>
            <set field="allowSplit" from="productStore?.allowSplit != 'N'"/>

            <entity-find entity-name="org.apache.ofbiz.order.order.OrderItemAttribute" list="preOrderAttrList" distinct="true">
                <econdition field-name="orderId" from="orderId"/>
                <econdition field-name="attrName" value="PreOrderItemProperty"/>
                <select-field field-name="orderItemSeqId"/>
            </entity-find>
            <set field="preOrderItemSeqIds" from="preOrderAttrList*.orderItemSeqId"/>

            <entity-find entity-name="org.apache.ofbiz.order.order.OrderItemAttribute" list="backOrderAttrList" distinct="true">
                <econdition field-name="orderId" from="orderId"/>
                <econdition field-name="attrName" value="BackOrderItemProperty"/>
                <select-field field-name="orderItemSeqId"/>
            </entity-find>
            <set field="backOrderItemSeqIds" from="backOrderAttrList*.orderItemSeqId"/>

            <set field="preOrderOrBackOrder" from="[:]"/>
            <if condition="allowSplit">
                <if condition="preOrderItemSeqIds">
                    <set field="preOrderOrBackOrder" from="preOrderOrBackOrder + ['PRE_ORDER': preOrderItemSeqIds]"/>
                </if>
                <if condition="backOrderItemSeqIds">
                    <set field="preOrderOrBackOrder" from="preOrderOrBackOrder + ['BACKORDER': backOrderItemSeqIds]"/>
                </if>
            </if>
            <if condition="!allowSplit">
                <entity-find entity-name="org.apache.ofbiz.order.order.OrderItem" list="allOrderItems" distinct="true">
                    <econdition field-name="orderId" from="orderId"/>
                    <select-field field-name="orderItemSeqId"/>
                </entity-find>
                <set field="allItemSeqIds" from="allOrderItems*.orderItemSeqId"/>
                <if condition="allItemSeqIds &amp;&amp; preOrderItemSeqIds">
                    <set field="preOrderOrBackOrder" from="['PRE_ORDER': allItemSeqIds]"/>
                </if>
                <if condition="allItemSeqIds &amp;&amp; !preOrderItemSeqIds &amp;&amp; backOrderItemSeqIds">
                    <set field="preOrderOrBackOrder" from="['BACKORDER': allItemSeqIds]"/>
                </if>
            </if>

            <if condition="!preOrderOrBackOrder"><return/></if>

            <iterate list="preOrderOrBackOrder.entrySet()" entry="entry">
                <set field="facilityTypeId" from="entry.key"/>
                <set field="orderItemSeqIds" from="entry.value"/>
                <if condition="!orderItemSeqIds"><continue/></if>

                <entity-find entity-name="org.apache.ofbiz.product.facility.Facility" list="facilityList" limit="1">
                    <econdition field-name="facilityTypeId" from="facilityTypeId"/>
                </entity-find>
                <set field="facility" from="facilityList ? facilityList[0] : null"/>
                <if condition="!facility">
                    <log level="warn" message="No facility found for facilityTypeId ${facilityTypeId} while splitting order ${orderId}"/>
                    <continue/>
                </if>

                <entity-find entity-name="org.apache.ofbiz.order.order.OrderItem" list="orderItemList">
                    <econdition field-name="orderId" from="orderId"/>
                </entity-find>
                <set field="orderItemBySeq" from="orderItemList.collectEntries{[(it.orderItemSeqId): it]}"/>
                <entity-find entity-name="org.apache.ofbiz.order.order.OrderItemShipGroup" list="shipGroupList">
                    <econdition field-name="orderId" from="orderId"/>
                </entity-find>
                <set field="shipGroupBySeq" from="shipGroupList.collectEntries{[(it.shipGroupSeqId): it]}"/>

                <entity-find entity-name="org.apache.ofbiz.order.order.OldOrderItemShipGroupAssoc" list="assocList">
                    <econdition field-name="orderId" from="orderId"/>
                    <econdition field-name="orderItemSeqId" operator="in" from="orderItemSeqIds"/>
                </entity-find>
                <set field="filteredAssocs" from="[]"/>
                <iterate list="assocList" entry="assoc">
                    <set field="assocItem" from="orderItemBySeq[assoc.orderItemSeqId]"/>
                    <set field="assocShipGroup" from="shipGroupBySeq[assoc.shipGroupSeqId]"/>
                    <if condition="!assocItem || !assocShipGroup"><continue/></if>
                    <if condition="assocShipGroup.facilityId != '_NA_'"><continue/></if>
                    <if condition="['ITEM_CANCELLED','ITEM_COMPLETED'].contains(assocItem.statusId)"><continue/></if>
                    <set field="filteredAssocs" from="filteredAssocs + [assoc]"/>
                </iterate>
                <if condition="!filteredAssocs"><continue/></if>

                <set field="shipGroupSeqIds" from="filteredAssocs*.shipGroupSeqId.findAll{it}.unique()"/>
                <iterate list="shipGroupSeqIds" entry="shipGroupSeqId">
                    <set field="groupItemSeqIds" from="filteredAssocs.findAll{it.shipGroupSeqId == shipGroupSeqId}*.orderItemSeqId.findAll{it}.unique()"/>
                    <if condition="!groupItemSeqIds"><continue/></if>

                    <entity-find entity-name="org.apache.ofbiz.order.order.OrderItemShipGroup" list="existingGroupList" limit="1">
                        <econdition field-name="orderId" from="orderId"/>
                        <econdition field-name="facilityId" from="facility.facilityId"/>
                    </entity-find>
                    <set field="existingGroup" from="existingGroupList ? existingGroupList[0] : null"/>
                    <set field="newShipGroupSeqId" from="existingGroup?.shipGroupSeqId"/>

                    <if condition="!newShipGroupSeqId">
                        <entity-find-count entity-name="org.apache.ofbiz.order.order.OldOrderItemShipGroupAssoc" count-field="remainingCount">
                            <econdition field-name="orderId" from="orderId"/>
                            <econdition field-name="shipGroupSeqId" from="shipGroupSeqId"/>
                            <econdition field-name="orderItemSeqId" operator="not-in" from="groupItemSeqIds"/>
                        </entity-find-count>
                        <if condition="remainingCount == 0">
                            <service-call name="update#org.apache.ofbiz.order.order.OrderItemShipGroup"
                                          in-map="[orderId:orderId, shipGroupSeqId:shipGroupSeqId, facilityId:facility.facilityId, orderFacilityId:facility.facilityId]"/>
                            <set field="newShipGroupSeqId" from="shipGroupSeqId"/>
                        </if>
                        <if condition="!newShipGroupSeqId">
                            <entity-find-count entity-name="org.apache.ofbiz.order.order.OrderItemShipGroup" count-field="shipGroupCount">
                                <econdition field-name="orderId" from="orderId"/>
                            </entity-find-count>
                            <set field="newShipGroupSeqId" from="String.format('%05d', (shipGroupCount ?: 0) + 1)"/>
                            <entity-find-one entity-name="org.apache.ofbiz.order.order.OrderItemShipGroup" value-field="baseShipGroup">
                                <field-map field-name="orderId" from="orderId"/>
                                <field-map field-name="shipGroupSeqId" from="shipGroupSeqId"/>
                            </entity-find-one>
                            <set field="createMap" from="baseShipGroup?.getValueMap() ?: [:]"/>
                            <set field="createMap.orderId" from="orderId"/>
                            <set field="createMap.shipGroupSeqId" from="newShipGroupSeqId"/>
                            <set field="createMap.facilityId" from="facility.facilityId"/>
                            <set field="createMap.orderFacilityId" from="facility.facilityId"/>
                            <service-call name="create#org.apache.ofbiz.order.order.OrderItemShipGroup" in-map="createMap"/>
                        </if>
                    </if>

                    <if condition="newShipGroupSeqId &amp;&amp; newShipGroupSeqId != shipGroupSeqId">
                        <iterate list="groupItemSeqIds" entry="orderItemSeqId">
                            <entity-find-one entity-name="org.apache.ofbiz.order.order.OldOrderItemShipGroupAssoc" value-field="assocRecord">
                                <field-map field-name="orderId" from="orderId"/>
                                <field-map field-name="orderItemSeqId" from="orderItemSeqId"/>
                                <field-map field-name="shipGroupSeqId" from="shipGroupSeqId"/>
                            </entity-find-one>
                            <if condition="!assocRecord"><continue/></if>
                            <entity-find-one entity-name="org.apache.ofbiz.order.order.OldOrderItemShipGroupAssoc" value-field="existingAssoc">
                                <field-map field-name="orderId" from="orderId"/>
                                <field-map field-name="orderItemSeqId" from="orderItemSeqId"/>
                                <field-map field-name="shipGroupSeqId" from="newShipGroupSeqId"/>
                            </entity-find-one>
                            <if condition="!existingAssoc">
                                <service-call name="create#org.apache.ofbiz.order.order.OldOrderItemShipGroupAssoc"
                                              in-map="[orderId:orderId, orderItemSeqId:orderItemSeqId, shipGroupSeqId:newShipGroupSeqId, quantity:assocRecord.quantity, cancelQuantity:assocRecord.cancelQuantity]"/>
                            </if>
                            <service-call name="delete#org.apache.ofbiz.order.order.OldOrderItemShipGroupAssoc"
                                          in-map="[orderId:orderId, orderItemSeqId:orderItemSeqId, shipGroupSeqId:shipGroupSeqId]"/>
                            <service-call name="update#org.apache.ofbiz.order.order.OrderItem"
                                          in-map="[orderId:orderId, orderItemSeqId:orderItemSeqId, shipGroupSeqId:newShipGroupSeqId]"/>
                        </iterate>
                    </if>
                </iterate>
            </iterate>
        </actions>
    </service>
```
