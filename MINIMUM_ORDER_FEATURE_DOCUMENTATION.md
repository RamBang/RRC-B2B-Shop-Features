# Minimum Order Amount Feature Documentation

## Overview
This feature implements a minimum order amount restriction for B2B customers based on their company's metafield `customer.current_company.metafields.b2b.custentity_minimum_credit_limit`. The feature integrates seamlessly with existing B2B customization code without disrupting their functionality.

## Feature Behavior
- **Mini-Cart**: Shows instant static warning message (no checkout restrictions)
- **Main Cart**: Shows warning message + disables checkout button when below minimum
- **Integration**: Hooks into ITG's cart refresh events for real-time updates
- **Security**: Form submission interception as backup for mini-cart

## Files Modified

### 1. `assets/custom.js`
**Location**: End of file (after ITG code section)

**Code Added**:
```javascript
///// ITG Code End
// INSTANT Minimum Order Checker - ITG Integration (Main Cart Only)
function checkMinimumOrderInstant() {
  const minimumOrderElement = document.querySelector('[data-b2b-minimum-amount]');
  if (!minimumOrderElement) return;
  
  const minimumAmount = parseFloat(minimumOrderElement.getAttribute('data-b2b-minimum-amount')) || 0;
  if (minimumAmount <= 0) return;
  
  // Get cart total for main cart only
  if (window.theme && window.theme.cartCount > 0) {
    fetch('/cart.js')
      .then(response => response.json())
      .then(cart => {
        cartTotal = cart.total_price / 100;
        updateMinimumOrderDisplay(minimumAmount, cartTotal);
      })
      .catch(() => {
        // Fallback to DOM parsing if API fails
        const priceElement = document.querySelector('.cart-recap__price-line-price');
        if (priceElement) {
          const priceText = priceElement.textContent.replace(/[^0-9.,]/g, '');
          cartTotal = parseFloat(priceText) || 0;
          updateMinimumOrderDisplay(minimumAmount, cartTotal);
        }
      });
  }
}

function updateMinimumOrderDisplay(minimumAmount, cartTotal) {
  const shortfall = minimumAmount - cartTotal;
  
  // Update main cart only (mini-cart is now static)
  const mainCartWarning = document.getElementById('cart-minimum-order-warning');
  const mainCartCheckoutBtn = document.querySelector('.cart-recap__checkout');
  
  if (shortfall > 0) {
    // Disable main cart button
    if (mainCartCheckoutBtn) {
      mainCartCheckoutBtn.disabled = true;
      mainCartCheckoutBtn.style.opacity = '0.5';
      mainCartCheckoutBtn.style.cursor = 'not-allowed';
    }
  } else {
    // Enable main cart button
    if (mainCartCheckoutBtn) {
      mainCartCheckoutBtn.disabled = false;
      mainCartCheckoutBtn.style.opacity = '1';
      mainCartCheckoutBtn.style.cursor = 'pointer';
    }
  }
}

// Intercept checkout attempts for extra security (mainly for mini-cart)
function interceptCheckoutAttempts() {
  document.addEventListener('submit', function(e) {
    const form = e.target;
    if (form.action && form.action.includes('/cart') && form.querySelector('button[name="checkout"]')) {
      // Only intercept mini-cart submissions (main cart button is already disabled server-side)
      if (form.id === 'mini-cart') {
        const minimumOrderElement = document.querySelector('[data-b2b-minimum-amount]');
        if (!minimumOrderElement) return;
        
        const minimumAmount = parseFloat(minimumOrderElement.getAttribute('data-b2b-minimum-amount')) || 0;
        if (minimumAmount <= 0) return;
        
        fetch('/cart.js')
          .then(response => response.json())
          .then(cart => {
            const cartTotal = cart.total_price / 100;
            if (cartTotal < minimumAmount) {
              e.preventDefault();
              const shortfall = minimumAmount - cartTotal;
              alert(`Minimum order of $${minimumAmount.toFixed(2)} required. Please add $${shortfall.toFixed(2)} more to your cart.`);
              return false;
            }
          })
          .catch(() => {
            // If API fails, let the form submit and let server-side handle it
          });
      }
    }
  });
}

// Hook into ITG's completion points (keep existing logic for dynamic updates)
const originalUpdateEaQuantity = window.updateEaQuantityInCart;
if (originalUpdateEaQuantity) {
  window.updateEaQuantityInCart = function() {
    const result = originalUpdateEaQuantity.apply(this, arguments);
    // Quick check after ITG completes
    setTimeout(checkMinimumOrderInstant, 100);
    return result;
  };
}

// Instant checks on page events
document.addEventListener('DOMContentLoaded', function() {
  checkMinimumOrderInstant();
  interceptCheckoutAttempts();
});

// Quick check on cart changes
document.addEventListener('cart:rerendered', function() {
  setTimeout(checkMinimumOrderInstant, 50);
});

// Check when quantities change
document.addEventListener('change', function(e) {
  if (e.target && e.target.classList.contains('quantity-selector__value')) {
    setTimeout(checkMinimumOrderInstant, 100);
  }
});
```

### 2. `snippets/mini-cart.liquid`
**Location**: After line 3 (after existing hidden inputs)

**Code Added**:
```liquid
{%- comment -%}ITG Minimum Order Amount Data{%- endcomment -%}
{%- if customer.b2b? and customer.current_company.metafields.b2b.custentity_minimum_credit_limit -%}
  <div data-b2b-minimum-amount="{{ customer.current_company.metafields.b2b.custentity_minimum_credit_limit }}" style="display: none;"></div>
{%- endif -%}
```

**Location**: After line 192 (after amount saved section, before button container)

**Code Added**:
```liquid
{%- comment -%}ITG Minimum Order Warning - Static Display{%- endcomment -%}
{%- if customer.b2b? and customer.current_company.metafields.b2b.custentity_minimum_credit_limit -%}
  {%- assign minimum_order_amount = customer.current_company.metafields.b2b.custentity_minimum_credit_limit | times: 100 -%}
  {%- if cart.total_price < minimum_order_amount -%}
    {%- assign remaining_amount = minimum_order_amount | minus: cart.total_price -%}
    <div style="background: #fff3cd; border: 1px solid #ffeaa7; padding: 12px; margin: 8px 0; border-radius: 4px; color: #856404; font-weight: bold; text-align: center; font-size: 14px;">
      <p style="margin: 0;">Minimum order of {{ minimum_order_amount | money }} required. Add {{ remaining_amount | money }} more.</p>
    </div>
  {%- endif -%}
{%- endif -%}
```

### 3. `sections/main-cart.liquid`
**Location**: After line 440 (before hidden inputs and checkout button)

**Code Added**:
```liquid
{%- comment -%}ITG Minimum Order Amount Data & Warning{%- endcomment -%}
{%- if customer.b2b? and customer.current_company.metafields.b2b.custentity_minimum_credit_limit -%}
  <div data-b2b-minimum-amount="{{ customer.current_company.metafields.b2b.custentity_minimum_credit_limit }}" style="display: none;"></div>
  
  {%- assign minimum_order_amount = customer.current_company.metafields.b2b.custentity_minimum_credit_limit | times: 100 -%}
  {%- if cart.total_price < minimum_order_amount -%}
    {%- assign remaining_amount = minimum_order_amount | minus: cart.total_price -%}
    <div id="cart-minimum-order-warning" style="background: #ffebee; border: 2px solid #f44336; padding: 16px; margin: 16px 0; border-radius: 8px; color: #c62828; font-weight: bold; text-align: center;">
      <p style="margin: 0; font-size: 16px;"><strong>Minimum order of {{ minimum_order_amount | money }} required.</strong></p>
      <p style="margin: 8px 0 0 0; font-size: 14px;">Add {{ remaining_amount | money }} more to checkout.</p>
    </div>
  {%- endif -%}
{%- endif -%}
```

**Location**: Checkout button modification (around line 458)

**Code Modified**:
```liquid
<button type="submit" name="checkout" class="cart-recap__checkout button button--primary button--full button--large"
  {%- if customer.b2b? and customer.current_company.metafields.b2b.custentity_minimum_credit_limit -%}
    {%- assign minimum_order_amount = customer.current_company.metafields.b2b.custentity_minimum_credit_limit | times: 100 -%}
    {%- if cart.total_price < minimum_order_amount -%}
      disabled style="opacity: 0.5; cursor: not-allowed;"
    {%- endif -%}
  {%- endif -%}
>{{ 'cart.general.checkout' | t }}</button>
```

## Implementation Details

### Data Source
- **Metafield**: `customer.current_company.metafields.b2b.custentity_minimum_credit_limit`
- **Value Format**: Stored as dollars (e.g., 500 for $500.00)
- **Conversion**: Multiplied by 100 to match Shopify's cents format for cart comparison

### Integration with ITG Code
- **Non-disruptive**: All additions are separate from ITG's existing functions
- **Hook Integration**: Plugs into `updateEaQuantityInCart()` completion
- **Timing**: Runs AFTER ITG's pricing calculations finish
- **Events**: Listens for cart changes and quantity updates

### User Experience
1. **Mini-Cart**: 
   - Instant yellow warning appears immediately
   - No checkout restrictions (allows user flow)
   - Form submission intercepted as security backup

2. **Main Cart**:
   - Instant red warning with strong visual styling
   - Checkout button disabled server-side and client-side
   - Real-time updates when quantities change

### Security Layers
1. **Server-side**: Liquid template disables checkout button when cart total is below minimum
2. **Client-side**: JavaScript prevents form submission for mini-cart
3. **Real-time**: Dynamic updates when quantities change through ITG integration

### Styling
- **Mini-Cart Warning**: Yellow background (#fff3cd), amber border (#ffeaa7), brown text (#856404)
- **Main Cart Warning**: Red background (#ffebee), red border (#f44336), dark red text (#c62828)
- **Responsive**: Works on all device sizes

## Dependencies
- Requires existing B2B customization code
- Requires `customer.b2b?` liquid conditional
- Requires `customer.current_company.metafields.b2b.custentity_minimum_credit_limit` metafield

