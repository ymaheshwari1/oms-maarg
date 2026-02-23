# Archive: `set#PreOrderPromisedDate` — Original Implementation

**Source:** `runtime/component/oms/service/co/hotwax/oms/order/OrderServices.xml` (Lines 1482–1557)
**Archived:** 2026-02-23
**Reason:** Replaced with a simplified version that uses `OrderItem.shipGroupSeqId` directly instead of the deprecated `OldOrderItemShipGroupAssoc` entity, effectively eliminating an N+1 query pattern.

---

## 🧠 Logic Explanation

**Purpose:** Set the `promisedDatetime` and link a `correspondingPoId` for pre-order and backorder sales items by locating an incoming Purchase Order (PO) that has available items (`availableToPromise > 0`).

### Phase 1: Validations
1. Verify the order exists.
2. Ensure the order status is NOT `ORDER_CANCELLED`, `ORDER_COMPLETED`, or `ORDER_REJECTED`.

### Phase 2: Find Pre-Order / Backorder Ship Groups
1. Find all `Facility` records with type `PRE_ORDER` or `BACKORDER`.
2. Find `OrderItemShipGroup` records for the order that are assigned to any of those facilities.

### Phase 3: The Data Loading (The N+1 Anti-Pattern)
*This is the section that involves refactoring.*

1. Iterate over the found ship groups.
2. Query `OldOrderItemShipGroupAssoc` to find items in that ship group.
3. Iterate over the assocs (`assocs`):
   - **(N+1 Query Issue)** For *each* assoc, run `entity-find-one` to load the actual `OrderItem` record.
   - If the item already has a `correspondingPoId`, skip it.
   - Query `OrderItemAttribute` to verify the item is truly tagged with `PreOrderItemProperty` or `BackOrderItemProperty`.
   - Resolve the target quantity using the assoc `quantity - cancelQuantity` (falling back to item quantity).

### Phase 4: Find Incoming Purchase Order Items
For each eligible item:
1. Find Purchase Order items (`OrderItem` where `orderTypeId = PURCHASE_ORDER`) for the same `productId`.
2. Filter for PO items that have `availableToPromise > 0` and an `estimatedDeliveryDate`.
3. Order them chronologically by `estimatedDeliveryDate`.

### Phase 5: Reserve Inventory & Link
1. Iterate over the candidate PO items.
2. Check if the candidate's PO header is valid (not cancelled/completed/rejected).
3. If the PO item has enough `availableToPromise` to cover the sales item quantity, it is selected as the winning candidate.
4. **Link:** Update the sales `OrderItem` with the PO's `estimatedDeliveryDate` (as `promisedDatetime`), the `correspondingPoId`, etc.
5. **Deduct:** Update the PO's `OrderItem` by reducing its `availableToPromise` by the sales item's quantity.

---

## 📜 Original XML Implementation

```xml
    <service verb="set" noun="PreOrderPromisedDate">
        <description>Set promised date and corresponding PO for preorder/backorder items</description>
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
            <if condition="['ORDER_CANCELLED','ORDER_COMPLETED','ORDER_REJECTED'].contains(orderHeader.statusId)"><return/></if>

            <entity-find entity-name="org.apache.ofbiz.product.facility.Facility" list="facilityList">
                <econdition field-name="facilityTypeId" operator="in" from="['PRE_ORDER','BACKORDER']"/>
            </entity-find>
            <set field="facilityIds" from="facilityList*.facilityId.findAll{it}"/>
            <if condition="!facilityIds"><return/></if>

            <entity-find entity-name="org.apache.ofbiz.order.order.OrderItemShipGroup" list="shipGroupList">
                <econdition field-name="orderId" from="orderId"/>
                <econdition field-name="facilityId" operator="in" from="facilityIds"/>
            </entity-find>
            <iterate list="shipGroupList" entry="shipGroup">
                <entity-find entity-name="org.apache.ofbiz.order.order.OldOrderItemShipGroupAssoc" list="assocs">
                    <econdition field-name="orderId" from="orderId"/>
                    <econdition field-name="shipGroupSeqId" from="shipGroup.shipGroupSeqId"/>
                </entity-find>
                <iterate list="assocs" entry="assoc">
                    <entity-find-one entity-name="org.apache.ofbiz.order.order.OrderItem" value-field="orderItem">
                        <field-map field-name="orderId" from="orderId"/>
                        <field-map field-name="orderItemSeqId" from="assoc.orderItemSeqId"/>
                    </entity-find-one>
                    <if condition="!orderItem || orderItem.correspondingPoId"><continue/></if>
                    <entity-find entity-name="org.apache.ofbiz.order.order.OrderItemAttribute" list="orderItemAttrList" limit="1">
                        <econdition field-name="orderId" from="orderId"/>
                        <econdition field-name="orderItemSeqId" from="assoc.orderItemSeqId"/>
                        <econdition field-name="attrName" operator="in" from="['PreOrderItemProperty','BackOrderItemProperty']"/>
                    </entity-find>
                    <if condition="!orderItemAttrList"><continue/></if>
                    <set field="orderItemQuantity" from="(assoc.quantity ?: orderItem.quantity) - (assoc.cancelQuantity ?: 0)"/>
                    <if condition="!orderItemQuantity || orderItemQuantity &lt;= 0"><continue/></if>

                    <entity-find entity-name="org.apache.ofbiz.order.order.OrderItem" list="poItemList">
                        <econdition field-name="productId" from="orderItem.productId"/>
                        <econdition field-name="availableToPromise" operator="greater" value="0"/>
                        <econdition field-name="estimatedDeliveryDate" operator="is-not-null"/>
                        <order-by field-name="estimatedDeliveryDate"/>
                    </entity-find>
                    <set field="candidate" from="null"/>
                    <iterate list="poItemList" entry="poItem">
                        <if condition="candidate"><continue/></if>
                        <entity-find-one entity-name="org.apache.ofbiz.order.order.OrderHeader" value-field="poHeader">
                            <field-map field-name="orderId" from="poItem.orderId"/>
                        </entity-find-one>
                        <if condition="!poHeader"><continue/></if>
                        <if condition="poHeader.orderTypeId != 'PURCHASE_ORDER'"><continue/></if>
                        <if condition="['ORDER_CANCELLED','ORDER_COMPLETED','ORDER_REJECTED'].contains(poHeader.statusId)"><continue/></if>
                        <if condition="poItem.availableToPromise &gt;= orderItemQuantity">
                            <set field="candidate" from="poItem"/>
                        </if>
                    </iterate>
                    <if condition="!candidate"><continue/></if>

                    <service-call name="update#org.apache.ofbiz.order.order.OrderItem"
                                  in-map="[orderId:orderId, orderItemSeqId:orderItem.orderItemSeqId, promisedDatetime:candidate.estimatedDeliveryDate,
                                    correspondingPoId:candidate.orderId, autoCancelDate:null, isNewProduct:candidate.isNewProduct]"/>
                    <set field="newAtp" from="candidate.availableToPromise - orderItemQuantity"/>
                    <if condition="newAtp &lt; 0"><set field="newAtp" from="0"/></if>
                    <service-call name="update#org.apache.ofbiz.order.order.OrderItem"
                                  in-map="[orderId:candidate.orderId, orderItemSeqId:candidate.orderItemSeqId, availableToPromise:newAtp]"/>
                </iterate>
            </iterate>
        </actions>
    </service>
```
