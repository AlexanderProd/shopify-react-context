## Installation

Add the package to your project via `npm install shopify-react-context` or `yarn add shopify-react-context`.

## Usage

Import your the `StoreContextProvider` component and wrap your components with it. For example a Layout component in `GatsbyJS`:

```JS
import React from 'react'
import { StoreContextProvider }from 'shopify-react-context'

const Layout = ({ children }) => {
  return (
    <StoreContextProvider>
      {children}
    </StoreContextProvider>
  )
}

export default Layout
```

Now to access the Context use the React `useContext` hook and pass the `StoreContext` to it.

```JS
const {
  addVariantToCart,
  store: { client, adding },
} = useContext(StoreContext)
```

## Example

This is an example for a ProductForm component using the Context exposed by `shopify-react-context`.

```JS
import React, { useState, useContext, useEffect, useCallback } from 'react'
import { StoreContext } from 'shopify-react-context'
import isEqual from 'lodash/isEqual'
import find from 'lodash/find'

const ProductForm = ({ product }) => {
  const {
    options,
    variants,
    variants: [initialVariant],
    priceRange: { minVariantPrice },
  } = product
  const [variant, setVariant] = useState({ ...initialVariant })
  const [quantity, setQuantity] = useState(1)
  const {
    addVariantToCart,
    store: { client, adding },
  } = useContext(StoreContext)

  const productVariant =
    client.product.helpers.variantForOptions(product, variant) || variant
  const [available, setAvailable] = useState(productVariant.availableForSale)

  const checkAvailability = useCallback(
    productId => {
      client.product.fetch(productId).then(fetchedProduct => {
        // this checks the currently selected variant for availability
        const result = fetchedProduct.variants.filter(
          variant => variant.id === productVariant.shopifyId
        )
        setAvailable(
          result[0]?.available ?? fetchedProduct.variants[0].available
        )
      })
    },
    [client.product, productVariant.shopifyId]
  )

  useEffect(() => {
    checkAvailability(product.shopifyId)
  }, [productVariant, checkAvailability, product.shopifyId])

  const handleQuantityChange = ({ target }) => {
    setQuantity(target.value)
  }

  const handleOptionChange = (optionIndex, { target }) => {
    const { value } = target
    const currentOptions = [...variant.selectedOptions]

    currentOptions[optionIndex] = {
      ...currentOptions[optionIndex],
      value,
    }

    const selectedVariant = find(variants, ({ selectedOptions }) =>
      isEqual(currentOptions, selectedOptions)
    )

    setVariant({ ...selectedVariant })
  }

  const handleAddToCart = () => {
    addVariantToCart(productVariant.shopifyId, quantity)
  }

  const checkDisabled = (name, value) => {
    const match = find(variants, {
      selectedOptions: [
        {
          name: name,
          value: value,
        },
      ],
    })
    if (match === undefined) return true
    if (match.availableForSale === true) return false
    return true
  }

  const price = Intl.NumberFormat(undefined, {
    currency: minVariantPrice.currencyCode,
    minimumFractionDigits: 2,
    style: 'currency',
  }).format(variant.price)

  return (
    <>
      <h3>{price}</h3>
      {options.map(({ id, name, values }, index) => (
        <React.Fragment key={id}>
          <label htmlFor={name}>{name} </label>
          <select
            name={name}
            key={id}
            onBlur={event => handleOptionChange(index, event)}
          >
            {values.map(value => (
              <option
                value={value}
                key={`${name}-${value}`}
                disabled={checkDisabled(name, value)}
              >
                {value}
              </option>
            ))}
          </select>
          <br />
        </React.Fragment>
      ))}
      <label htmlFor="quantity">Quantity </label>
      <input
        type="number"
        id="quantity"
        name="quantity"
        min="1"
        step="1"
        onChange={handleQuantityChange}
        value={quantity}
      />
      <br />
      <button
        type="submit"
        disabled={!available || adding}
        onClick={handleAddToCart}
      >
        Add to Cart
      </button>
      {!available && <p>This Product is out of Stock!</p>}
    </>
  )
}
```
