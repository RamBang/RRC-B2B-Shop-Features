# B2B Minimum Order Feature - Extracted Code

This file contains all the B2B minimum order enforcement code that was temporarily added to the theme. This code has been extracted to restore the theme to its original production state.

## Files Modified and Code Added:

### 1. snippets/mini-cart.liquid

**B2B Minimum Order Logic Added:**
```liquid
{%- comment -%}B2B minimum order validation{%- endcomment -%}
{%- assign b2b_minimum_order_met = true -%}
{%- assign b2b_minimum_alert = '' -%}

{%- if customer.b2b? and customer.current_company.metafields.b2b.custentity_minimum_credit_limit -%}
  {%- assign minimum_order_amount = customer.current_company.metafields.b2b.custentity_minimum_credit_limit | times: 100 -%}
  {%- if cart.total_price < minimum_order_amount -%}
    {%- assign b2b_minimum_order_met = false -%}
    {%- assign remaining_amount = minimum_order_amount | minus: cart.total_price -%}
    {%- capture b2b_minimum_alert -%}
      <div class="alert alert--error cart-recap__minimum-alert">
        <p><strong>Minimum order of {{ customer.current_company.metafields.b2b.custentity_minimum_credit_limit | money }} required.</strong> Add {{ remaining_amount | money }} more to checkout.</p>
      </div>
    {%- endcapture -%}
  {%- endif -%}
{%- endif -%}

{%- comment -%}B2B minimum order data attributes{%- endcomment -%}
{%- if customer.b2b? and customer.current_company.metafields.b2b.custentity_minimum_credit_limit -%}
  <input type="hidden" data-b2b-minimum-amount="{{ customer.current_company.metafields.b2b.custentity_minimum_credit_limit }}" />
{%- endif -%}
<input type="hidden" data-cart-total="{{ cart.total_price | divided_by: 100.0 }}" />

{%- comment -%}B2B minimum alert display{%- endcomment -%}
{%- if b2b_minimum_alert != blank -%}
  {{- b2b_minimum_alert -}}
{%- endif -%}

{%- comment -%}B2B checkout button disabling{%- endcomment -%}
<button type="submit" name="checkout" class="button button--primary{% unless b2b_minimum_order_met %} button--disabled{% endunless %}" {% unless b2b_minimum_order_met %}disabled data-b2b-minimum-disabled="true"{% endunless %}>{{ 'cart.general.checkout' | t }}</button>
```

### 2. sections/main-cart.liquid

**B2B Minimum Order Logic Added:**
```liquid
{%- comment -%}B2B minimum order validation{%- endcomment -%}
{%- assign b2b_minimum_order_met = true -%}
{%- assign b2b_minimum_alert = '' -%}

{%- if customer.b2b? and customer.current_company.metafields.b2b.custentity_minimum_credit_limit -%}
  {%- assign minimum_order_amount = customer.current_company.metafields.b2b.custentity_minimum_credit_limit | times: 100 -%}
  {%- if cart.total_price < minimum_order_amount -%}
    {%- assign b2b_minimum_order_met = false -%}
    {%- assign remaining_amount = minimum_order_amount | minus: cart.total_price -%}
    {%- capture b2b_minimum_alert -%}
      <div class="alert alert--error cart-recap__minimum-alert">
        <p><strong>Minimum order of {{ customer.current_company.metafields.b2b.custentity_minimum_credit_limit | money }} required.</strong> Add {{ remaining_amount | money }} more to checkout.</p>
      </div>
    {%- endcapture -%}
  {%- endif -%}
{%- endif -%}

{%- comment -%}B2B minimum order data attributes{%- endcomment -%}
{%- if customer.b2b? and customer.current_company.metafields.b2b.custentity_minimum_credit_limit -%}
  <input type="hidden" data-b2b-minimum-amount="{{ customer.current_company.metafields.b2b.custentity_minimum_credit_limit }}" />
{%- endif -%}

{%- comment -%}B2B minimum alert display{%- endcomment -%}
{%- if b2b_minimum_alert != blank -%}
  {{- b2b_minimum_alert -}}
{%- endif -%}

{%- comment -%}B2B checkout button modifications{%- endcomment -%}
<button type="submit" name="checkout" class="button button--primary{% unless b2b_minimum_order_met %} button--disabled{% endunless %}" {% unless b2b_minimum_order_met %}disabled data-b2b-minimum-disabled="true"{% endunless %}>{{ 'cart.general.checkout' | t }}</button>
```

### 3. snippets/wcp_cart.liquid

**B2B Minimum Order Logic Added:**
```liquid
// B2B Minimum Order Validation
{% if customer.b2b? and customer.current_company.metafields.b2b.custentity_minimum_credit_limit %}
  var minimumOrderAmount = {{ customer.current_company.metafields.b2b.custentity_minimum_credit_limit | json }};
  var cartTotal = {{ cart.total_price | divided_by: 100.0 | json }};
  
  if (cartTotal < minimumOrderAmount) {
    var remainingAmount = minimumOrderAmount - cartTotal;
    alert('Minimum order of $' + minimumOrderAmount.toFixed(2) + ' required. Add $' + remainingAmount.toFixed(2) + ' more to checkout.');
    return false;
  }
{% endif %}
```

### 4. assets/custom.js

**B2B Minimum Order Logic Added:**
```javascript
// B2B Minimum Order Amount Validation - PASSIVE ONLY
const checkB2BMinimumOrder = () => {
  try {
    // Wait for ITG system to determine customer type first
    const wholesaleType = document.body.getAttribute('data-wholesale');
    
    // If customer type not determined yet, allow checkout (don't block)
    if (!wholesaleType) return true;
    
    // Apply minimum order logic to both B2B and Wholesale customers (not Retail)
    if (wholesaleType === 'Retail') return true;
    
    // Check if customer has company and minimum order amount
    const minimumOrderElement = document.querySelector('[data-b2b-minimum-amount]');
    const cartTotalElement = document.querySelector('[data-cart-total]');
    
    if (!minimumOrderElement || !cartTotalElement) return true;
    
    const minimumAmount = parseFloat(minimumOrderElement.getAttribute('data-b2b-minimum-amount')) || 0;
    const cartTotal = parseFloat(cartTotalElement.getAttribute('data-cart-total')) || 0;
    
    // If no minimum set, allow checkout
    if (minimumAmount <= 0) return true;
    
    return cartTotal >= minimumAmount;
  } catch (error) {
    // If any error occurs, don't block checkout
    console.warn('B2B minimum order check failed:', error);
    return true;
  }
};

const updateCheckoutButtons = () => {
  // VERY specific targeting - only cart checkout buttons, never interfere with ITG
  const checkoutButtons = document.querySelectorAll(
    '.mini-cart__recap button[name="checkout"], .cart-recap button[name="checkout"]'
  );
  
  const minimumMet = checkB2BMinimumOrder();
  
  checkoutButtons.forEach(button => {
    // Double check - absolutely no product form buttons
    if (button.closest('.product-form, .product-info, .itg-product-main')) {
      return;
    }
    
    if (minimumMet) {
      button.disabled = false;
      button.classList.remove('button--disabled');
      button.removeAttribute('data-b2b-minimum-disabled');
    } else {
      button.disabled = true;
      button.classList.add('button--disabled');
      button.setAttribute('data-b2b-minimum-disabled', 'true');
    }
  });
};

// Prevent checkout if minimum order not met - GENTLE approach
const preventB2BCheckoutIfNeeded = (event) => {
  try {
    const button = event.target;
    
    // Only prevent if this specific button was disabled by our B2B logic
    if (button.hasAttribute('data-b2b-minimum-disabled') && 
        button.getAttribute('data-b2b-minimum-disabled') === 'true') {
      
      event.preventDefault();
      event.stopPropagation();
      
      // Gentle alert highlighting
      const alertMessage = document.querySelector('.alert--error');
      if (alertMessage) {
        alertMessage.scrollIntoView({ behavior: 'smooth', block: 'center' });
        alertMessage.classList.add('alert--pulse');
        setTimeout(() => alertMessage.classList.remove('alert--pulse'), 500);
      }
      
      return false;
    }
    
    return true;
  } catch (error) {
    // If any error occurs, allow checkout to proceed
    console.warn('B2B checkout prevention failed:', error);
    return true;
  }
};

// Initialize B2B minimum order checking - PASSIVE validation only
document.addEventListener('DOMContentLoaded', () => {
  // Only run on cart pages, and delay to avoid conflicts with ITG system
  setTimeout(() => {
    if (document.querySelector('.template-cart, .mini-cart')) {
      // Very specific checkout button click handler - only for actual checkout buttons
      document.addEventListener('click', (event) => {
        const target = event.target;
        
        // ONLY target the exact checkout buttons in cart areas
        if (target.matches('button[name="checkout"]') && 
            (target.closest('.mini-cart__recap') || target.closest('.cart-recap'))) {
          
          if (!preventB2BCheckoutIfNeeded({ target: target, preventDefault: () => event.preventDefault(), stopPropagation: () => event.stopPropagation() })) {
            event.preventDefault();
            event.stopPropagation();
            return false;
          }
        }
      });
      
      // Passive validation only - don't interfere with ITG cart operations
      document.addEventListener('cart:rerendered', () => {
        setTimeout(updateCheckoutButtons, 100);
      });
    }
  }, 5000); // Wait well after ITG system initializes
});

// Add gentle pulse animation for error alerts
const style = document.createElement('style');
style.textContent = `
  .alert--pulse {
    animation: b2b-gentle-pulse 0.5s ease-in-out;
  }
  
  @keyframes b2b-gentle-pulse {
    0% { opacity: 1; }
    50% { opacity: 0.7; }
    100% { opacity: 1; }
  }
  
  .cart-recap__minimum-alert {
    margin-bottom: 1rem;
  }
`;
document.head.appendChild(style);
```

### 5. assets/theme.css

**B2B Minimum Order Styles Added:**
```css
/* B2B Minimum Order Validation Styles */
.alert--error {
  background-color: #fef2f2;
  border: 1px solid #fecaca;
  color: #dc2626;
  border-radius: 0.375rem;
  padding: 0.75rem 1rem;
  margin-bottom: 1rem;
}

.button--disabled {
  opacity: 0.5;
  cursor: not-allowed;
  pointer-events: none;
}

.button--disabled:hover {
  opacity: 0.5;
  transform: none;
}

.cart-recap__minimum-alert {
  margin-bottom: 1rem;
}

.mini-cart__button-container .alert--error {
  margin-bottom: 0.5rem;
  font-size: 0.875rem;
}

@keyframes pulse {
  0% { transform: scale(1); }
  50% { transform: scale(1.02); }
  100% { transform: scale(1); }
}

.alert--pulse {
  animation: pulse 0.5s ease-in-out;
}

/* Additional checkout button states */
.additional-checkout-button:disabled,
.shopify-payment-button__button:disabled {
  opacity: 0.5 !important;
  cursor: not-allowed !important;
  pointer-events: none !important;
}
```

## Implementation Notes:

1. **Metafield Used:** `customer.current_company.metafields.b2b.custentity_minimum_credit_limit`
2. **Applies To:** B2B customers with company metafields only
3. **Validation Points:** Mini-cart, main cart, WCP wholesale checkout
4. **Features:** 
   - Real-time validation
   - Error messaging showing required amounts
   - Button state management
   - Integration with existing checkout flows

## Usage:

To re-implement this feature, the code above would need to be carefully integrated back into the respective files, ensuring it doesn't interfere with the ITG wholesale quantity management system. 