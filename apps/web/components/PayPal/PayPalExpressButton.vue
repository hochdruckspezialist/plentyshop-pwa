<template>
  <div v-if="paypalUuid" ref="paypalButton" :id="'paypal-' + paypalUuid" class="z-0 relative paypal-button" />
</template>

<script setup lang="ts">
import type { FUNDING_SOURCE, OnApproveData, OnInitActions } from '@paypal/paypal-js';
import { orderGetters, productGetters, cartGetters } from '@plentymarkets/shop-sdk';
import { v4 as uuid } from 'uuid';
import type { PaypalButtonPropsType } from '~/components/PayPal/types';

const paypalButton = ref<HTMLElement | null>(null);
const { loadScript, createTransaction, approveOrder, executeOrder } = usePayPal();
const { createOrder } = useMakeOrder();
const { shippingPrivacyAgreement } = useAdditionalInformation();
const { data: cart, addToCart, clearCartItems } = useCart();
const currency = computed(() => cartGetters.getCurrency(cart.value) || (useAppConfig().fallbackCurrency as string));
const localePath = useLocalePath();
const emits = defineEmits(['on-click']);

const props = withDefaults(defineProps<PaypalButtonPropsType>(), {
  disabled: false,
});

const { type, disabled, value } = toRefs(props);

const TypeCartPreview = 'CartPreview';
const TypeSingleItem = 'SingleItem';
const TypeCheckout = 'Checkout';

const isCommit = type.value === TypeCheckout;
const paypal = await loadScript(currency.value, isCommit);
const paypalUuid = ref('');

const onInit = (actions: OnInitActions) => {
  if (type.value === TypeCheckout) {
    true === disabled.value ? actions.disable() : actions.enable();
    watch(disabled, () => (true === disabled.value ? actions.disable() : actions.enable()));
  } else {
    actions.enable();
  }
};

const onClick = async () => {
  if (
    type.value === TypeSingleItem &&
    !disabled.value &&
    value.value &&
    productGetters.isSalable(value.value.product)
  ) {
    await addToCart({
      productId: Number(productGetters.getId(value.value.product)),
      quantity: value.value.quantity,
      basketItemOrderParams: value.value.basketItemOrderParams,
    });
  }
};

const onApprove = async (data: OnApproveData) => {
  const result = await approveOrder(data.orderID, data.payerID ?? '');

  if ((type.value === TypeCartPreview || type.value === TypeSingleItem) && result?.url)
    navigateTo(localePath(paths.readonlyCheckout + `/?payerId=${data.payerID}&orderId=${data.orderID}`));

  if (type.value === TypeCheckout) {
    const order = await createOrder({
      paymentId: cart.value.methodOfPaymentId,
      shippingPrivacyHintAccepted: shippingPrivacyAgreement.value,
    });

    await executeOrder({
      mode: 'paypal',
      plentyOrderId: Number.parseInt(orderGetters.getId(order)),
      paypalTransactionId: data.orderID,
    });

    clearCartItems();

    if (order?.order?.id)
      navigateTo(localePath(paths.thankYou + '/?orderId=' + order.order.id + '&accessKey=' + order.order.accessKey));
  }
};

const renderButton = (fundingSource: FUNDING_SOURCE) => {
  if (paypal?.Buttons && fundingSource) {
    const button = paypal?.Buttons({
      style: {
        layout: 'vertical',
        label: type.value === TypeCartPreview || type.value === TypeSingleItem ? 'checkout' : 'buynow',
        color: 'blue',
      },
      fundingSource: fundingSource,
      async onClick() {
        await onClick();
        emits('on-click');
      },
      onInit(data, actions) {
        onInit(actions);
      },
      onError() {
        // eslint-disable-next-line unicorn/expiring-todo-comments
        // TODO: handle error
      },
      async createOrder() {
        const order = await createTransaction(fundingSource);
        return order?.id ?? '';
      },
      async onApprove(data) {
        await onApprove(data);
      },
    });

    if (button.isEligible() && paypalButton.value) button.render('#' + paypalButton.value?.id);
  }
};

const createPaypalUuid = async () => (paypalUuid.value = uuid());

onMounted(async () => {
  await createPaypalUuid().then(() => {
    if (paypal) {
      const FUNDING_SOURCES = [paypal.FUNDING?.PAYPAL, paypal.FUNDING?.PAYLATER];
      FUNDING_SOURCES.forEach((fundingSource) => renderButton(fundingSource as FUNDING_SOURCE));
    }
    return true;
  });
});
</script>
