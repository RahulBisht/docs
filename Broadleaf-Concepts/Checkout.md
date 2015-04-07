# Checkout

> Note: Please read the [[Checkout Page Design]] doc if you haven't yet familiarized yourself with the Checkout Flow provided out-of-the-box by Broadleaf.

The checkout process in Broadleaf is controlled by a workflow, which is extensible like all other Broadleaf workflows. This workflow is responsible for some basic verification, validating and confirming payments, and completing the order. Let's dive in.

### Important Considerations

Note that the checkout workflow can be called asynchronously depending on what Payment Gateway you integrate with. The implication of this means that you need to be careful and not rely on any session state when calling `checkoutService.performCheckout()`. For example, some payment gateways like Authorize.net and PayPal PayFlow will issue a silent post to your server to complete the checkout process. In this case, the request will come directly from the gateway and not from the Customer's browser so you can't use `CartState.getCart()` or any other request/session based parameters reliably. Take this into consideration when customizing your checkout workflow.

## <a name="wiki-config"></a>Configuration

Let's take a look at the default configuration in Broadleaf Commerce for Checkout:

```xml
<!-- Checkout Workflow Configuration -->
<bean p:order="1000" id="blVerifyCustomerMaxOfferUsesActivity" class="org.broadleafcommerce.core.offer.service.workflow.VerifyCustomerMaxOfferUsesActivity"/>
<bean p:order="2000" id="blValidateProductOptionsActivity" class="org.broadleafcommerce.core.checkout.service.workflow.ValidateProductOptionsActivity"/>
<bean p:order="3000" id="blValidateAndConfirmPaymentActivity" class="org.broadleafcommerce.core.checkout.service.workflow.ValidateAndConfirmPaymentActivity">
    <property name="rollbackHandler" ref="blConfirmPaymentsRollbackHandler" />
</bean>
<bean p:order="4000" id="blRecordOfferUsageActivity" class="org.broadleafcommerce.core.offer.service.workflow.RecordOfferUsageActivity">
    <property name="rollbackHandler" ref="blRecordOfferUsageRollbackHandler" />
</bean>
<bean p:order="5000" id="blCommitTaxActivity" class="org.broadleafcommerce.core.checkout.service.workflow.CommitTaxActivity">
    <property name="rollbackHandler" ref="blCommitTaxRollbackHandler" />
</bean>
<bean p:order="6000" id="blDecrementInventoryActivity" class="org.broadleafcommerce.core.checkout.service.workflow.DecrementInventoryActivity">
    <property name="rollbackHandler" ref="blDecrementInventoryRollbackHandler" />
</bean>
<bean p:order="7000" id="blCompleteOrderActivity" class="org.broadleafcommerce.core.checkout.service.workflow.CompleteOrderActivity">
    <property name="rollbackHandler" ref="blCompleteOrderRollbackHandler" />
</bean>

<bean id="blCheckoutWorkflow" class="org.broadleafcommerce.core.workflow.SequenceProcessor">
    <property name="processContextFactory">
        <bean class="org.broadleafcommerce.core.checkout.service.workflow.CheckoutProcessContextFactory"/>
    </property>
    <property name="activities">
        <list>
            <ref bean="blVerifyCustomerMaxOfferUsesActivity" />
            <ref bean="blValidateProductOptionsActivity" />
            <ref bean="blValidateAndConfirmPaymentActivity" />
            <ref bean="blRecordOfferUsageActivity" />
            <ref bean="blCommitTaxActivity" />
            <ref bean="blDecrementInventoryActivity" />
            <ref bean="blCompleteOrderActivity" />
        </list>
    </property>
    <property name="defaultErrorHandler">
        <bean class="org.broadleafcommerce.core.workflow.DefaultErrorHandler">
            <property name="unloggedExceptionClasses">
                <list>
                    <value>org.broadleafcommerce.core.inventory.service.InventoryUnavailableException</value>
                </list>
            </property>
        </bean>
    </property>
</bean>
```

Checkout workflows are similar to workflows in general in that they require a ProcessContextFactory, a list of activities and an error handler to be configured. Checkout workflows will use an instance of CheckoutProcessContextFactory. By default, Broadleaf Commerce supplies activities for executing pricing, payment and order reset. In addition, Broadleaf utilizes the DefaultErrorHandler implementation, which simply logs any exceptions to the console and bubbles the exceptions.


At the very least, most users will want to keep the default activities included with Broadleaf for checkout, which represent the core tasks for most order checkouts. However, some users will want to add additional checkout activities that do not fit neatly in the given workflows Next, we'll look at customizing the workflow. 

## <a name="wiki-customization"></a>Customization

Sometimes it is desirable to add in a custom checkout activity. For example, you may have a scenario where you need to notify a fulfillment provider of a completed order to schedule shipment to your customer. Let's take some time to look at this scenario more in-depth and develop an example.

First, we need to create a custom checkout activity implementation.

```java
public class MyFulfillmentActivity extends BaseActivity {

    @Resource(name="myFulfillmentService")
    private FulfillmentService myFulfillmentService;

    public ProcessContext execute(ProcessContext context) throws Exception {
        CheckoutSeed seed = ((CheckoutContext) context).getSeedData();
        myFulfillmentService.fulfillOrder(seed.getOrder());
        return context;
    }
}
```

In the `MyFulfillmentActivity` class, we've injected an instance of a new service interface, `FulfillmentService`. This is arbitrary and could actually be any service capable of fulfilling your order with your fulfillment provider. Like all activities, we implement the execute method. In our example, we retrieve the order from the ProcessContext and simply pass it to our conceptual fulfillment service.

With the logic in place, all that's left is to re-configure the checkout workflow to include our new activity. Using the activities list defined above, we can add in our new activity to the default Broadleaf Checkout Workflow. That can easily be done by the Broadleaf Merge process. In your application context, you would override the key `blCheckoutWorkflow` to provide our new custom workflow definition and define the `p:order` of where in the activity list it should go. In our case, we want it in between the `blCommitTaxActivity` and the `blCompleteOrderActivity`. So in this case, we will define it as `p:order="5500"`  Let's check out the end result:


```xml
<bean id="blCheckoutWorkflow" class="org.broadleafcommerce.core.workflow.SequenceProcessor">
    <property name="activities">
        <list>
            <bean p:order="5500" class="com.mycompany.checkout.service.workflow.MyFulfillmentActivity"/>
        </list>
    </property>
</bean>
```
