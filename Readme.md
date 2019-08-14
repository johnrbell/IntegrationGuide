# Integration Cheatsheet

This cheatsheet is to help you quickly get up and running with a fully localized storefront. This assumes you have installed Flow Connect and Flow Checkout Apps as documented here: https://docs.flow.io/shopify/guide/shopify-app-install

Much of this code is dependant on theme. Class names of DOM elements may change, therefore simply copy and pasting code may not work. This should be used as a guide explaing the *ideas* behind each step of integrating. 

### 1. Rendering Localized Pricing: 

#### Script Tag

Add the flow.js JavaScript by adding the following script tag to your theme.liquid file. We recommend that the tag be added as high as possible in the head tag (typically before vendor.js and theme.js.)

- Replace {flow_organization} with the organization id from Flow. The organization id can be seen in the URL when selecting an organization (sandbox for development) in Console or retrieved via the Organizations API.
- Replace {myshopify_domain} with the internal name of your shop (e.g. yourshop.myshopify.com).

```js
<script>
(function (f, l, o, w, i, n, g) {
  f[i] = f[i] || {};
  f[i].set = f[i].set || function () {
    (f[i].q = f[i].q || []).push(arguments);
  };
  f[i].initialized = 0;
  f[i].fraud = 'enabled';
n = l.createElement(o);
  n.src = w;
  g = l.getElementsByTagName(o)[0];
  g.parentNode.insertBefore(n, g);
})(window, document, 'script', 'https://shopify-cdn.flow.io/{flow_organization}/js/v0/flow.js?shop={myshopify_domain}', 'Flow');
</script>
```

#### Snippets

Add the two Flow snippets to the snippets directory: 

- http://cdn.flow.io/docs/shopify/flow-utility-snippets/flow_metafields_prices.liquid

- http://cdn.flow.io/docs/shopify/flow-utility-snippets/flow_localized_price.liquid
	
#### Metafield Tags

Product-template.liquid should use product-price.liquid to render the price. Replace the shopify price with code similar to this:

```html
<div class="price__regular">
  <dt>
    <span class="visually-hidden visually-hidden--inline">{{ 'products.product.regular_price' | t }}</span>
  </dt>
  <dd>
      {% if available %}<!-- if available -->
        {% if compare_at_price > price %}<!-- if on sale -->
          <span class="price-item price-item--regular" data-regular-price>
            {% if cart.attributes.flow_metafield_key %}<!-- if in an experience -->
              {% include 'flow_localized_price', flow_tag_display: 'compare_at' %}
            {% else %}<!-- if domestic -->
              {{ compare_at_price | money }}
            {% endif %}<!-- endif domestic check -->
          </span>
        {% else %}<!-- else normal price -->
          <span class="price-item price-item--regular" data-regular-price>
            {% if cart.attributes.flow_metafield_key %}<!-- if in an experience -->
              {% include 'flow_localized_price', flow_tag_display: 'item_price' %}<br>
            {% else %}<!-- if domestic -->
              {{ money_price }}
            {% endif %}<!-- endif domestic check -->
          </span>
        {% endif %}<!-- end if on sale -->
      {% else %} <!-- else for if available -->

        <span class="price-item price-item--regular" data-regular-price>
          {{ 'products.product.sold_out' | t }}
        </span>

      {% endif %}<!-- endif for if available  -->
  </dd>
</div>
<div class="price__sale">
  <dt>
    <span class="visually-hidden visually-hidden--inline">{{ 'products.product.sale_price' | t }}</span>
  </dt>
  <dd>
    <span class="price-item price-item--sale" data-sale-price>
      {% if cart.attributes.flow_metafield_key %}<!-- if in an experience -->
        {% include 'flow_localized_price', flow_tag_display: 'item_price' %}<br>
      {% else %}<!-- if domestic -->
        {{ money_price }}
      {% endif %}<!-- endif domestic check -->      
    </span>
    <span class="price-item__label" aria-hidden="true">{{ 'products.product.on_sale' | t }}</span>
  </dd>
</div>
```

Before rendering any shopify price (whether on sale or not) this code will check if you are currently in a Flow experience. This is done with `{% if cart.attributes.flow_metafield_key %}` 

If you are in an experience, it will render the `flow_localized_price`

### 2. Adding the Country Picker:

Adding the country picker next allows for easier testing moving forward. 

Download provided css and add it to theme.scss.liquid

- https://cdn.flow.io/country-picker/example.css

Add the following code to a javascript file. We recommend creating and including a file called flow.js site-wide. 

```js
Flow.set('on', 'ready', () => {
  window.flow.countryPicker.createCountryPicker(
    {'containerId' : 'countryPicker'}
  )
});
```

Add a countryPicker div to the header:

```html
<div id='countryPicker'></div>
```

### 3. Cart Setup

Working in cart-template.liquid

- Add `data-flow-cart-container` as an attribute in the first div.

- Add `data-flow-localize="cart-subtotal"` to `<span class="cart-subtotal_price..."`

- Find `<tr class="cart_row"` and add `data-flow-cart-item-number="{{ item.variant_id }}"` to the end of the tr tag

- Find the remaining shopify price span/tags and add the attribute: `data-flow-localize="cart-item-price"`

In theme.liquid, before the closing body tag, add: 

```html
<script type="application/json" id="flow_shopify_cart">{{ cart | json }}</script>
```

Add the following to the main SCSS file: 

```scss
$loading-circle-diameter: 4px;
$loading-circle-color: white;

@keyframes flowcalizing {
  100%  { transform: rotate(360deg); }
}
@keyframes flowcalization-fade-in {
  0% { opacity: 0 }
  100% { opacity: 1 }
}

[data-flow-cart-container] {
  [data-flow-localize] {
    animation: flowcalization-fade-in 0.7s ease;
  }

  &.flowcalizing [data-flow-localize] {
    transition: none;
    position: relative;
    visibility: hidden;
    animation: none;

    &::after {
      transform-origin: bottom left;
      content: '';
      display: inline-block;
      width: $loading-circle-diameter;
      height: $loading-circle-diameter;
      border-radius: 50%;
      background-color: $loading-circle-color;
      visibility: visible;
      animation: 0.6s ease infinite flowcalizing;
    }
  }
}
```

In you JS file (flow.js) add the following code. 

```js
Flow.set('on', 'ready', () => {
  //localize cart
  if (Flow.getExperience()) {
    Flow.cart.localize({
      success: () => {console.log('prices localized')}
    })
  }
})
```

### 4. Variant Price replacement

When switching variant variations, such as t-shirt sizes from S to XL- prices may change. In order to handle this we need to modify two files. 

#### product-price.liquid

Code needs to be modified around the conditional checking if the product is a sale item. Both sides of the conditional need to be changed to include another conditional checking if the user is in an experience or domestic. This code was provided in step one above. 

### theme.js

In theme.js we need to modify code for the updatePrice function also: 

```js
if (variant.compare_at_price > variant.price) {
  if (Flow.getExperience()){//if in an experience
    $regularPrice.html(
      `<span class="flow-price flow-price__compare-at" flow-variant="${ variant.id }" 
      flow-selector="prices.compare_at.label" 
      flow-default="${ theme.Currency.formatMoney(variant.compare_at_price, theme.moneyFormat) }">
      </span>`            
    )
    $salePrice.html(
      `<span class="flow-price flow-price__item" flow-variant="${ variant.id }" 
      flow-selector="prices.item.label" 
      flow-default="${ theme.Currency.formatMoney(variant.price, theme.moneyFormat) }">
      </span>`
    );
    $priceContainer.addClass(this.classes.productOnSale);
    Flow.variants.localize({force:true})//get price from Flow
  }else{//if domestic
    $regularPrice.html(
      theme.Currency.formatMoney(
        variant.compare_at_price,
        theme.moneyFormat
      )
    );    
    $salePrice.html(
      theme.Currency.formatMoney(variant.price, theme.moneyFormat)
    );
    $priceContainer.addClass(this.classes.productOnSale);
  }//endif experience check
} else {//end on sale 
//Regular Price
  if (Flow.getExperience()){//if in an experience
    $regularPrice.html(
      `<span class="flow-price flow-price__item" flow-variant="${ variant.id }" 
      flow-selector="prices.item.label" 
      flow-default="${ theme.Currency.formatMoney(variant.price, theme.moneyFormat) }">
      </span>`
    );
    Flow.variants.localize({force:true})//get price from Flow
  }else{//if domestic
    $regularPrice.html(
      theme.Currency.formatMoney(variant.price, theme.moneyFormat)
    );
  }//endif experience check
}//end regular price 
```

### 5. Account Page

Replace the `order.total_price` tag with a check for experience, like so: 

```html
{% if order.metafields.flow_order != empty %}
  <td data-label="{{ 'customer.orders.total' | t }}">
    {{ order.metafields.flow_order.total }}
  </td>
{% else %}                  
  <td data-label="{{ 'customer.orders.total' | t }}">
    {{ order.total_price | money }}
  </td>
{% endif %}
```

### 6. Individual Past Order Page

Replace rendered Shopify Prices with Flow Prices, similar to all the other replacements: 


```html
{% if order.metafields.flow_order != empty %}
  {{ line_item.properties.flow_line_unit_price }}
{% else %}
  {{ line_item.original_price | money }}
{% endif %}
```

This needs to be done for everwhere a price appears: Line Item Total, Total Price, Discount, Shipping, etc. Dependant on theme. 

For tax/vat/duty, we need more modifications. 

Replace with something similar to this:

```html
<!-- if domestic loop shopify tax -->
{% if order.metafields.flow_order == empty %}
  {%- for tax_line in order.tax_lines -%}
    <tr>
      <th class="small--hide" scope="row" colspan="4">{{ 'customer.order.tax' | t }} ({{ tax_line.title }} {{ tax_line.rate | times: 100 }}%)</th>
      <td class="text-right" data-label="{{ 'customer.order.tax' | t }} ({{ tax_line.title }} {{ tax_line.rate | times: 100 }}%)">
        {{ tax_line.price | money }}
      </td>
    </tr>
  {%- endfor -%}
{% endif %}

<!-- look for metafields from flow -->
{% if order.metafields.flow_order.duty %}
  <tr>
  <th class="small--hide" scope="row" colspan="4">Duty</th>
    <td class="text-right" data-label="Duty">
      {{ order.metafields.flow_order.duty }}
    </td>      
  </tr>      
{% endif %}

{% if order.metafields.flow_order.vat %}
  <tr>
    <th class="small--hide" scope="row" colspan="4">{{ order.metafields.flow_order.tax_name }}</th>
    <td class="text-right" data-label="{{ 'order.metafields.tax' | t }}">
      {{ order.metafields.flow_order.vat }}
    </td>    
  </tr>       
{% endif %}
```

### 7. Checkout Button Hijack 

International orders are processed through Flow's checkout, not Shopify. If our user is in an international experience we need to redirect users when they click the "Checkout" button.

In flow.js, inside of `Flow.set('on','ready', ()> {})` we created earlier, add the following:

```js
var checkout = document.getElementById("cart__form")
if (checkout && Flow.getExperience()){
  checkout.addEventListener("submit", e => {
    e.preventDefault();
    Flow.checkout.redirect()
  });
}
```

### 8. Quantity Updates

Becase we are redirecting to Flow checkout, we need to inform shopify of any quantity changes before we do so. We also need to update the prices on the cart. On change of the input boxes we need to make an ajax call and modify DOM elements.

In flow.js, inside of `Flow.set('on','ready', ()> {})` we created earlier, add the following:

```js
//Eventlistener on Cart Row Qty Selectors
qtyInputs = document.querySelectorAll(".cart__quantity-td input")
if (qtyInputs){
  qtyInputs.forEach(qtyInput => {
    qtyInput.addEventListener("change", e => {
      e.preventDefault()
      //update row data attributes 
      qtyInput.closest(".cart__row").setAttribute('data-cart-item-quantity', qtyInput.value)
      //build object then post to shopify. 
      cartObj = {}
      buildCartObj(cartObj)
      ajaxUpdate(cartObj)
    })
  })
}

//build a cart object, update row attributes 
buildCartObj = () => {
  document.querySelectorAll("tr.cart__row").forEach(cartRow => {
    key = cartRow.getAttribute("data-cart-item-key")
    qty = cartRow.getAttribute("data-cart-item-quantity")
    cartObj[key] = qty
  })
}

//POST to Shopify informing of new quantities
ajaxUpdate = cartObj => {
  jQuery.post(
    '/cart/update.js', {updates: cartObj}).always(data => {
      //get total price from json 
      price = JSON.parse(data.responseText)
      //convert to $xx.xx
      price = formatUSD.format(price.total_price/100)
      updatePage(cartObj,price)
    })  
}

//update cart count bubble and total price
updatePage = (cartObj,price) => {
  //update total price
  if (!Flow.getExperience()){price += " USD"}
  document.querySelector(".cart-subtotal__price").innerText = price
  Flow.cart.localize({
    success: () => {console.log('prices re-localized')}
  })
  //add all qty's and update cart bubble
  total = Object.values(cartObj).reduce((a, b) => parseInt(a) + parseInt(b))
  document.querySelector('#CartCount span').innerText = total
}

//convert price from JSON response to usable format
formatUSD = new Intl.NumberFormat('en-US', {
  style: 'currency',
  currency: 'USD',
  minimumFractionDigits: 2
})
```