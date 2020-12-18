---
layout: post
title:  "React + Redux 应用的开发流程，以购物车为例"
date: 2017-09-02
categories: ["React"]
description: "Redux 是最热门的 React 架构。介绍 Redux 的文章不少，官方文档的内容也非常丰富。  
本文根据个人对 Redux 的理解，提供一种和官方文档不一样的视角，阐述如何从零开始开发一个 React + Redux 应用..."
---


**项目的 repository：[react-redux-shopping-cart-demo](https://github.com/chasecs/react-redux-shopping-cart-demo)**

Redux 是最热门的 React 架构。介绍 Redux 的文章不少，官方文档的内容也非常丰富。  
本文根据个人对 Redux 的理解，提供一种和官方文档不一样的视角，阐述如何从零开始开发一个 React + Redux 应用。  
<!--more-->

阅读之前，假设你已经了解 Redux 的三个核心概念：Action，Reducer，Store，并且知道三者在 Redux 的工作流程所承担的角色和功能。  
文中使用的实例 “shopping-cart”，参考自官方的示例[shopping-cart](https://github.com/reactjs/redux/tree/master/examples/shopping-cart)，虽然实现同样的效果，但是代码有较大的差异。

## 零、 react-redux 目录结构

使用 `create-react-app` 创建项目，并安装相关依赖：
```bash
$ create-react-app react-redux-shopping-cart-demo
$ cd react-redux-shopping-cart-demo
$ npm i -S redux react-redux
```

在 `src/` 下创建项目的目录结构:
```
|- /src
  |- /actions
  |- /constants
    |- action_types.js
  |- /components
  |- /containers
    |- app.js
  |- /reducers
    |- index.js
  |- index.js
```


### 一、`<Provider>` 封装根组件 `<App/>`

使用 `react-redux` 提供 `<Provider>` ，它提供 store 属性，并封装应用的根组件 `App`，所有子组件都可以访问 state。
`src/index.js`
```js
import React from 'react';
import { render } from 'react-dom';
import { createStore } from 'redux';
import { Provider } from 'react-redux'
import App from './containers/App';
import reducer from './reducers';

const store = createStore(reducer);

render (
  <Provider store={store}>
    <App/>
  </Provider>,
  document.getElementById('root')
)
```

store 由 `Redux` 提供，它需要 Reducer 作为参数。因此，我们还需要创建 App 和 Reducer。  
先创建 Reducer，因为项目刚刚创建，我们可以创建一个没有内容的 Reducer。  
`reducers/index.js`
```js
import { combineReducers } from 'redux'

export default combineReducers({});
```

其次是 App，可以把它看作是应用的首页，同样可以先建一个没有实际内容的页面。  
```js
import React from 'react'

const App = () => (
  <div>
    <h2>hello world</h2>
  </div>
)

export default App
```

这样，一个 React + Redux 应用就可以通过 `npm start` 运行起来了。你也可以把它看作是一个 boilerplate。

## 二、设计 state 的结构

上一个步骤完成了项目的初始化，在开始真正的开发之前，最好先思考一下整个应用的 state 结构（state shape）。  
state 的设计可以化整为零，比如以 Container（容器组件）为单位划分，该部分 state 的初始值定义在和 Container 对应的 Reducer 中，该 Reducer 就处理这个部分的 state。  
以 shopping cart 应用为例，页面展示上需要两个独立的容器组件：
- ProductContainer，用于显示库存商品 products ；
- CartContainer，用于显示购物车的商品；

因此，state 的结构可以设计如下：  
```js
state = {
  products,
  cart
}
```

至于 products 和 cart 的数据类型应该是怎样，可以在各自的 Reducer 里在具体定义。

## 三、展示组件开发

步骤一只是搭建了 react/redux 框架，里面并没有我们的数据和 UI，如何把这些呈现出来？  
在 React/Redux，数据由 Action 提供，它可以是从 api 获取的，也可以是本地的数据。  
而 UI 部分，React-Redux 将所有组件分成两大类：展示组件（presentational component）和容器组件（container component），分别保存在 `components/` 和 `containers/` 两个目录。

展示组件负责视觉呈现，和 redux 几乎没有联系。如果是复杂的应用，还可以把展示组件分别保存在 `components/` 和 `pages/` 两个目录，`pages/` 对应页面，由 component 按需组合起来。  
容器组件负责管理数据和业务逻辑，这里的数据并不是直接从 Action 获取，而是从 state 获取，而 Action 和 state 需要通过 Reducer 才能互动。

因为展示组件和 redux 关系不大，可以先行开发，

```js
|- /components
  |- product.js
  |- product_item.js
  |- products_list.js
  |- cart.js
```
展示组件的开发这里不做介绍，可以直接看源码 `[components](https://github.com/chasecs/react-redux-shopping-cart-demo/tree/master/src/components)`

## 四、容器组件开发

展示组件完成后，要使它和数据产生联系，需要通过容器组件。  
因为容器组件相对复杂一些，所以不采用 const FuntionName 的方式来创建，使用 class XX extends Component 创建。  
创建容器组件 `products_list_container.js`：  
```js
import React, { Component } from 'react'
import PropTypes from 'prop-types'
import ProductsList from '../components/products_list';
import ProductItem from '../components/product_item';

class ProductsListContainer extends Component {

  static propTypes = {
    products: PropTypes.arrayOf(PropTypes.shape({
      id: PropTypes.number.isRequired,
      title: PropTypes.string.isRequired,
      price: PropTypes.number.isRequired,
      inventory: PropTypes.number.isRequired
    })).isRequired,
    addToCart: PropTypes.func.isRequired
  }

  render(){
    return (
      <ProductsList title="Product List">
        {products.map( product =>
          <ProductItem
            key={product.id}
            product={product}
            onAddToCartClicked={() => {}}
          />
        )}
      </ProductsList>
    )
  }
}
```

如果仅仅是上述的代码，`ProductsListContainer` 仍然只是一个展示组件，把它变为容器组件，你还需要 `react-redux` 提供的 `connect()` 方法，所以，在文件最后加上：
```js
....
ProductsListContainer = connect()(ProductsListContainer);
export default ProductsListContainer;
```

下一步是往 `ProductsListContainer` 里面传入 `products` 数据和 `onAddToCartClicked` 方法。  
这里需要的数据是：库存商品数据，因此，可以在 `constants/action_types.js` 里定义一个 `type`:
```js
export const SHOW_INVENTORY = 'SHOW_INVENTORY'
```

传入数据可以调用 `mapStateToProps` 方法，把数据传给组件。
修改 `products_list_container.js`, 最终代码如下：

```js
import React from 'react'
import PropTypes from 'prop-types'
import ProductsList from '../components/products_list';
import Product from '../components/product';
import { connect } from 'react-redux'

const ProductsContainer =({products}) => (
  <ProductsList title="hehe">
    {products.map( product =>
      <Product
        key={product.id}
        title={product.title}
        price={product.price}
        inventory={product.inventory}
      />
    )}
  </ProductsList>
)

const mapStateToProps = () => ({
  /* 不规范的代码 BEGIN*/
  products: [
    {"id": 1, "title": "iPad 4 Mini", "price": 500.01, "inventory": 2},
    {"id": 2, "title": "H&M T-Shirt White", "price": 10.99, "inventory": 10},
    {"id": 3, "title": "Charli XCX - Sucker CD", "price": 19.99, "inventory": 5}
  ]
  /* 不规范的代码是 END*/
})

export default connect(mapStateToProps)(ProductsContainer)
```

然后在 `containers/app.js` 修改 `<App>` 组件，加入 `ProductsListContainer`
```js
import ProductsListContainer from './products_list_container'

const App = () => (
  <div>
    <ProductsListContainer/>
  </div>
)
```

再运行一下，就可以在浏览器页面看到商品的列表了。

## 五、Action 和 Reducer 处理数据

正如 `ProductsContainer.js` 里标注的，`mapStateToProps` 里的代码是错误的，数据不能直接赋值。正确的做法是，把 Redux 的 store 里保存的 state 输出为组件的 props。    
不过，store 里的 state 又从哪里获取？  
按照 Redux 的工作流程，它应该从 Action 和 Reducer 产生。Action 提供原始数据，经过 Reducer 的处理，更新到 state。  
先在 `actions/products_action.js` 实现这个 Action：  
```js
import * as types from '../constants/action_types';

export const showInventory = () => {
  let products = {
    1: {"id": 1, "title": "iPad 4 Mini", "price": 500.01, "inventory": 2},
    2: {"id": 2, "title": "H&M T-Shirt White", "price": 10.99, "inventory": 10},
    3: {"id": 3, "title": "Charli XCX - Sucker CD", "price": 19.99, "inventory": 5}
  }
  return {
    type: types.SHOW_INVENTORY,
    products
  }
}
```

数据要经过 Reducer 才能和 state 产生关联，新建一个 `reducers/products_reducer.js`：  
```js
import * as types from '../constants/action_types';

const initialState = {}

const productsReducer = (state = initialState, action) => {
  switch (action.type) {
    case types.SHOW_INVENTORY:
      return Object.assign({}, action.products)
    default:
      return state
  }
}

export default productsReducer
```

把 `productsReducer` 添加到 `reducers/index.js`
```js
import products from './products_reducer';

export default combineReducers({
  products: productsReducer
});
```

现在 Action 已经有了，但是还需要调用 `dispatch(action)` 这个方法，使用 `mapDispatchToProps` 方法。如果向 `mapDispatchToProps` 返回一个对象，它的 key 对应的 value 必须是一个返回 Action 对象的函数，调用该函数时，返回的 Action 会由 Redux 自动 `dispatch`。  
```js
import { showInventory, addToCart } from '../actions/products_action';
...
const mapDispatchToProps = {
  showInventory,
}
export default connect(
  mapStateToProps,
  mapDispatchToProps
)(ProductsListContainer)
```

`showInventory` 方法就可以在 `ProductsListContainer` 通过 `this.props` 来访问  
```js
componentDidMount() {
  const { showInventory } = this.props;
  showInventory()
}
```
这一步也类似于现实应用中向api请求数据。  
`showInventory()` 调用成功后，store 中的 state 就已经更新了。容器组件就可以通过 `mapStateToProps` 获取 state 并映射到 props。
因为 `state.products` 的格式不是和展示组件要求的格式不太一样，所以我们要用 `parseProducts` 转换一下：  
```js
const parseProducts = (products) => {
  let mappedProducts = []
  for(let key in products){
    mappedProducts.push(products[key])
  }
  return mappedProducts
}
const mapStateToProps = (state) => (
  {products: parseProducts(state.products)}
)
```

同样，最终的 `products` 可以在 `ProductsListContainer` 通过 `this.props` 来访问，
```js
  render(){
    const {products} = this.props;
    return (
      <ProductsList title="Product List">
        {products.map( product =>
          <ProductItem
            key={product.id}
            product={product}
            onAddToCartClicked={() => {}}
          />
        )}
      </ProductsList>
    )
  }
```

再运行一下，就可以在浏览器页面看到商品的列表了。至此，这个应用已经是一个完整地运用 redux 的 Action，Reducer，Store。

## 六、实现交互操作

迄今为止，我们的应用只是展示了商品，作为一个购物车应用，还需要增加交互操作。下来将实现 addToCart 按钮的功能。  
在 Redux 的工作流程里，一个引发 state 变化的 function，必须是一次 dispatch(Action)。  
先不考虑购物车 `CartContainer` 的情况，对于 `ProductsContainer` 这个容器组件，每点击一下，库存数量就应该减少一个。  

实现 `ADD_TO_CART` 的 Action：
`constants/action_types.js`   
```js
export const ADD_TO_CART = 'ADD_TO_CART'
```

`actions/products_action.js`  
```js
export const addToCart = productId => {
  return ({
    type: types.ADD_TO_CART,
    productId
  })
}
```

在 Reducer 实现对应的方法，修改 `reducers/products_reducer.js`
```js
const decreaseInventory = (product) => {
  return {
    ...product,
    inventory: product.inventory - 1
  }
}

const productsReducer = (state = initialState, action) => {
  switch (action.type) {
    case types.SHOW_INVENTORY:
      return Object.assign({}, action.products)
    case types.ADD_TO_CART:
      const { productId } = action
      return {
        ...state,
        [productId]: decreaseInventory(state[productId])
      }
    default:
      return state
  }
}
```

修改 `products_list_container.js`
```js
...
render(){
  const {products, addToCart} = this.props;
  return (
    <ProductsList title="Product List">
      {products.map( product =>
        <ProductItem
          key={product.id}
          product={product}
          onAddToCartClicked={() => addToCart(product.id)}
        />
      )}
    </ProductsList>
  )
}

...

const mapDispatchToProps = {
  showInventory,
  addToCart
}
```

这样，`ProductsContainer` 的 `addToCart` 功能就已经实现了，每点击一次，对应商品的数量就会减少一个。

## 一个 Action 引发多个 Reducer

在商品列表中，点击 "Add to Cart" 按钮之后，发生的变化不仅是库存减少，还有购物车物品增多、购物金额变化。    
因为库存和购物车被划分在不同的 Reducer 里，所以就会出现一个 Action 触发两个 Reducer 的情况。  
实现 `reducers/cart_reducer.js`
```js
import * as types from '../constants/action_types';

const initialState = {
  addedIds: [],
  quantityById: {}
}

const addProduct = (state = initialState.addedIds, productId) => {
  if(state.indexOf(productId) === -1){
    return [...state, productId]
  }
  return state
}

const addQuantity = (state = initialState.quantityById, productId) => {
  return {
    ...state,
    [productId]: (state[productId] || 0) + 1
  }
}


const cartReducer = (state = initialState, action) => {
  switch (action.type) {
    case types.ADD_TO_CART:
      return Object.assign({}, state, {
        addedIds: addProduct(state.addedIds, action.productId),
        quantityById: addQuantity(state.quantityById, action.productId)
      })
    default:
      return state
  }
}

export default cart
```

将 `cartReducer` 加入 `reducers/index.js`
```js
....
+import cartReducer from './cart_reducer'

export default combineReducers({
  ...
+  cart: cartReducer
});
```

实现 `CartContainer`,
```js
import React, { Component} from 'react'
import PropTypes from 'prop-types'
import Cart from '../components/cart'
import { connect } from 'react-redux'

class CartContainer extends Component {

  render(){
    const { products, total } = this.props;

    return(
      <Cart
        products={products}
        total={total}
        onCheckoutClicked={() => alert('checkout')}
      />
    )
  }
}

const productsInCart = (state) => {
  const { products, cart } = state;
  let result = [];
  let total = 0;
  for(let id in cart.quantityById){
    result.push({
      ...products[id],
      quantity:cart.quantityById[id]
    });
    total += products[id].price * cart.quantityById[id]
  }

  return {
    products: result,
    total: total.toFixed(2)
  }
}

const mapStateToProps = (state) => productsInCart(state)

export default connect(
  mapStateToProps,
)(CartContainer)

```

其中 `productsInCart` 方法用于解析 state 中的购物车数据，输出我们需要的格式，从而在组件中渲染出来。
至此，点击 "add to cart" 按钮之后，会同时发生库存减少、购物车商品数量、金额变化，最基本的购物车 demo 就已经完成。

当然，目前这个购物车是十分简陋的，如果想继续利用它来熟悉 Redux 或者 React，还有许多可以开发的方向，比如：减少购物车商品数量，购物车商品可选、全选，加入 React-Router，页面样式...

(完)

### 参考资料  
- [Read Me · Redux](http://redux.js.org/)
- [react-redux API](https://github.com/reactjs/react-redux/blob/master/docs/api.md#api)
- [shopping-cart](https://github.com/reactjs/redux/tree/master/examples/shopping-cart)
- [tj/frontend-boilerplate](https://github.com/tj/frontend-boilerplate)
- [Redux 入门教程（三）：React-Redux 的用法](http://www.ruanyifeng.com/blog/2016/09/redux_tutorial_part_three_react-redux.html)
