# zzy-react-research-note
react 探索笔记

# Redux Ducks
> 在 Redux 数据流中创建一个 Container/Component 需要新建多少个文件？添加或修改一对 Reducer/Actions 要打开多少个窗口？是不是很多， Ducks 就是解决这烦人问题的一套方案。  
参考链接：[https://github.com/erikras/ducks-modular-redux](https://github.com/erikras/ducks-modular-redux)

实际使用 Redux 的过程中，大多数情况下只有一对 Reducer/Actions 会用到对应的 actions，那这些代码放在一个文件中似乎会根据方便，我们来看一下修改后的代码：  
```javascript
// widgets.js

// Actions
const LOAD   = 'my-app/widgets/LOAD';
const CREATE = 'my-app/widgets/CREATE';
const UPDATE = 'my-app/widgets/UPDATE';
const REMOVE = 'my-app/widgets/REMOVE';

// Reducer
export default function reducer(state = {}, action = {}) {
  switch (action.type) {
    // do reducer stuff
    default: return state;
  }
}

// Action Creators
export function loadWidgets() {
  return { type: LOAD };
}

export function createWidget(widget) {
  return { type: CREATE, widget };
}

export function updateWidget(widget) {
  return { type: UPDATE, widget };
}

export function removeWidget(widget) {
  return { type: REMOVE, widget };
}
```
### 规则
一个模块...
1. 必须 `export default` 函数名为 `reducer()` 的 reducer
2. 必须 作为函数 `export` 它的 action creators
3. 必须 把 action types 定义成形为 `npm-module-or-app/reducer/ACTION_TYPE` 的字符串
4. 如果有外部的reducer需要监听这个action type，或者作为可重用的库发布时， 可以用 `UPPER_SNAKE_CASE` 形式 export 它的 action types。

上述规则也推荐用在可重用的redux 库中用来组织 `{actionType, action, reducer}`

### 命名
大家可能好奇 Ducks 名字为什么是 Ducks 这么奇怪，其实这是 Redux 的尾音联想出来的，
对于新建的文件名 Ducks 作者也推荐使用 Ducks 结尾，如 TodosDucks、UserDucks。

### 建议
为了方便获取 Action Creators ，可以将 Action Creators 包装成一个 Object ，如：
```javascript
// Action Creators
export actions = {
  loadWidgets: () => {
    return { type: LOAD };
  },
  createWidget: (widget) => {
    return { type: CREATE, widget };
  },
  updateWidget: (widget) => {
    return { type: UPDATE, widget };
  },
  removeWidget: (widget) => {
    return { type: REMOVE, widget };
  }
}
```

### 例子
可以参考 [React Redux Universal Hot Example](https://github.com/erikras/react-redux-universal-hot-example)

---

# Redux Reselect
> 为了解决 Redux 衍生计算和重复渲染的导致的性能问题， [reselect](https://github.com/reactjs/reselect) 应运而生。 [reselect](https://github.com/reactjs/reselect) 库可以创建可记忆、可组合的 selector 函数，从而高效的计算 Redux Store 里的数据。  
参考链接：  
[https://github.com/reactjs/reselect](https://github.com/reactjs/reselect)  
[http://redux.js.org/docs/recipes/ComputingDerivedData.html](http://redux.js.org/docs/recipes/ComputingDerivedData.html)

## Todo List
首先我们先看一个例子，这是 Todo List：

#### Reducers
`reducers.js`
```javascript
import { combineReducers } from 'redux'
import { ADD_TODO, COMPLETE_TODO, SET_VISIBILITY_FILTER, VisibilityFilters } from './actions'
const { SHOW_ALL } = VisibilityFilters

function visibilityFilter(state = SHOW_ALL, action) {
  switch (action.type) {
    case SET_VISIBILITY_FILTER:
      return action.filter
    default:
      return state
  }
}

function todos(state = [], action) {
  switch (action.type) {
    case ADD_TODO:
      return [
        ...state,
        {
          text: action.text,
          completed: false
        }
      ]
    case COMPLETE_TODO:
      return [
        ...state.slice(0, action.index),
        Object.assign({}, state[action.index], {
          completed: true
        }),
        ...state.slice(action.index + 1)
      ]
    default:
      return state
  }
}

const todoApp = combineReducers({
  visibilityFilter,
  todos
})

export default todoApp
```

#### Container
`containers/App.js`
```javascript
import React, { Component, PropTypes } from 'react'
import { connect } from 'react-redux'
import { addTodo, completeTodo, setVisibilityFilter, VisibilityFilters } from '../actions'
import AddTodo from '../components/AddTodo'
import TodoList from '../components/TodoList'
import Footer from '../components/Footer'

class App extends Component {
  render() {
    // Injected by connect() call:
    const { dispatch, visibleTodos, visibilityFilter } = this.props
    return (
      <div>
        <AddTodo
          onAddClick={text =>
            dispatch(addTodo(text))
          } />
        <TodoList
          todos={visibleTodos}
          onTodoClick={index =>
            dispatch(completeTodo(index))
          } />
        <Footer
          filter={visibilityFilter}
          onFilterChange={nextFilter =>
            dispatch(setVisibilityFilter(nextFilter))
          } />
      </div>
    )
  }
}

App.propTypes = {
  visibleTodos: PropTypes.arrayOf(PropTypes.shape({
    text: PropTypes.string.isRequired,
    completed: PropTypes.bool.isRequired
  }).isRequired).isRequired,
  visibilityFilter: PropTypes.oneOf([
    'SHOW_ALL',
    'SHOW_COMPLETED',
    'SHOW_ACTIVE'
  ]).isRequired
}

function selectTodos(todos, filter) {
  switch (filter) {
    case VisibilityFilters.SHOW_ALL:
      return todos
    case VisibilityFilters.SHOW_COMPLETED:
      return todos.filter(todo => todo.completed)
    case VisibilityFilters.SHOW_ACTIVE:
      return todos.filter(todo => !todo.completed)
  }
}

// Which props do we want to inject, given the global state?
// Note: use https://github.com/faassen/reselect for better performance.
function select(state) {
  return {
    visibleTodos: selectTodos(state.todos, state.visibilityFilter),
    visibilityFilter: state.visibilityFilter
  }
}

// 包装 component ，注入 dispatch 和 state 到其默认的 connect(select)(App) 中；
export default connect(select)(App)
```

#### 改造一下
> action 和 reducer 就不赘述了，state 的结构发生了变化：  
`{ todos: [ text: '', completed: false ], visibilityFilter: 'SHOW_ALL' }`

`Container/VisibleTodoList.js`  
```javascript
import { connect } from 'react-redux'
import { toggleTodo } from '../actions'
import TodoList from '../components/TodoList'

const getVisibleTodos = (todos, filter) => {
  switch (filter) {
    case 'SHOW_ALL':
      return todos
    case 'SHOW_COMPLETED':
      return todos.filter(t => t.completed)
    case 'SHOW_ACTIVE':
      return todos.filter(t => !t.completed)
  }
}

const mapStateToProps = (state) => {
  return {
    todos: getVisibleTodos(state.todos, state.visibilityFilter)
  }
}

const mapDispatchToProps = (dispatch) => {
  return {
    onTodoClick: (id) => {
      dispatch(toggleTodo(id))
    }
  }
}

const VisibleTodoList = connect(
  mapStateToProps,
  mapDispatchToProps
)(TodoList)

export default VisibleTodoList
```
上面的 Todo List 通过 getVisibleTodos 来计算 todos ，不过有个问题：每次更新组件包括更新 state 产生的衍生计算都会重新计算 todos 。在计算量和 state tree 很大的时候就会产生性能问题。

## Reselect 创建可记忆的 selector
Reselect 通过 `createSelector` 来创建一个可记忆的 selector 来计算 state ，在这个例子中，只有 `state.todos` 和 `state.visibilityFilter` 发生变化时才会重新计算 todos ，否则返回上一次的计算的结果。看下代码：

`selectors.js`
```javascript
import { createSelector } from 'reselect'

const getVisibilityFilter = (state) => state.visibilityFilter
const getTodos = (state) => state.todos

export const getVisibleTodos = createSelector(
  [ getVisibilityFilter, getTodos ],
  (visibilityFilter, todos) => {
    switch (visibilityFilter) {
      case 'SHOW_ALL':
        return todos
      case 'SHOW_COMPLETED':
        return todos.filter(t => t.completed)
      case 'SHOW_ACTIVE':
        return todos.filter(t => !t.completed)
    }
  }
)
```

#### 组合 Selector
selector 可作为另一个 selector 的 input-selector ，如下面 `getVisibleTodos` 就作为 getKeyword 的 input-selector：
```javascript
const getKeyword = (state) => state.keyword

const getVisibleTodosFilteredByKeyword = createSelector(
  [ getVisibleTodos, getKeyword ],
  (visibleTodos, keyword) => visibleTodos.filter(
    todo => todo.text.indexOf(keyword) > -1
  )
)
```

#### 连接 Selector 和 Redux Store
`containers/VisibleTodoList.js`
```javascript
import { connect } from 'react-redux'
import { toggleTodo } from '../actions'
import TodoList from '../components/TodoList'
import { getVisibleTodos } from '../selectors'

const mapStateToProps = (state) => {
  return {
    todos: getVisibleTodos(state)
  }
}

const mapDispatchToProps = (dispatch) => {
  return {
    onTodoClick: (id) => {
      dispatch(toggleTodo(id))
    }
  }
}

const VisibleTodoList = connect(
  mapStateToProps,
  mapDispatchToProps
)(TodoList)

export default VisibleTodoList
```

## Tip
上面是 reselect 的一些基础用法，进一步请往：  
[https://github.com/reactjs/reselect](https://github.com/reactjs/reselect)  
[http://redux.js.org/docs/recipes/ComputingDerivedData.html](http://redux.js.org/docs/recipes/ComputingDerivedData.html)
