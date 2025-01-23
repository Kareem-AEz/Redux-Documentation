# **Redux & Redux Toolkit Reference Guide**

This guide provides an overview of setting up Redux and Redux Toolkit, including the modern approach with Redux Toolkit and the traditional way of implementing actions, reducers, and thunks. It also includes best practices, and common questions.

## Version Information

This guide is based on:

- Redux: v5.x
- Redux Toolkit: v2.x

Please check the official documentation for any updates or changes in newer versions.

---

## Table of Contents

1. [Setting Up Redux Toolkit](#1-setting-up-redux-toolkit)
   - [Installation](#installation)
   - [Store Configuration with Redux Toolkit](#store-configuration-with-redux-toolkit)
   - [Explanation](#explanation)
2. [Redux Flow and Application Structure](#2-redux-flow-and-application-structure)
3. [Creating Slices with Redux Toolkit](#3-creating-slices-with-redux-toolkit)
   - [Key Features](#key-features)
4. [Using Thunks in Redux Toolkit](#4-using-thunks-in-redux-toolkit)
   - [Modern Way: `createAsyncThunk`](#modern-way-createasyncthunk)
   - [Manual Thunks](#manual-thunks)
5. [The Old Way (Plain Redux)](#5-the-old-way-plain-redux)
   - [Setting Up the Reducer](#setting-up-the-reducer)
   - [Creating Actions](#creating-actions)
6. [Selectors](#6-Selectors-and-useSelector)
7. [DOs and DON'Ts](#8-dos-and-donts)
8. [Performance Considerations](#9-performance-considerations)
9. [Common Questions](#10-common-questions)

---

## **1. Setting Up Redux Toolkit**

### **Installation**

Install the required packages:

```bash
npm install @reduxjs/toolkit react-redux
```

### **Store Configuration with Redux Toolkit**

```javascript
import { configureStore } from "@reduxjs/toolkit";
import accountReducer from "./features/accounts/AccountSlice";
import customerReducer from "./features/customers/CustomerSlice";

const store = configureStore({
	reducer: {
		account: accountReducer,
		customer: customerReducer,
	},
	middleware: (getDefaultMiddleware) =>
		getDefaultMiddleware({
			thunk: true, // Thunk middleware is enabled by default
		}),
	devTools: process.env.NODE_ENV === "development", // Enable DevTools in development
});

export default store;
```

### **Explanation:**

1. **Reducers**: Combine multiple reducers into a single root reducer.
2. **Middleware**: `getDefaultMiddleware` includes `thunk` by default. Customize it if needed.
3. **DevTools**: Enabled only in development mode for debugging.

---

## **2. Redux Flow and Application Structure**

Understanding the Redux flow and how to structure your application is crucial for effective state management.

### Redux Flow

![Redux Flow Diagram](https://redux.js.org/assets/images/ReduxDataFlowDiagram-49fa8c3968371d9ef6f2a1486bd40a26.gif)

1. User interacts with the View (React components)
2. View dispatches Actions
3. Reducers process Actions and update the Store
4. Store notifies the View of state changes
5. View re-renders based on new state

### Typical Redux Application Structure

```
src/
├── features/
│   ├── feature1/
│   │   ├── Feature1Component.js
│   │   ├── feature1Slice.js
│   │   └── feature1Thunks.js
│   └── feature2/
│       ├── Feature2Component.js
│       ├── feature2Slice.js
│       └── feature2Thunks.js
├── app/
│   └── store.js
├── components/
│   └── SharedComponents.js
└── App.js
```

This structure organizes code by feature, promoting modularity and scalability.

---

## **3. Creating Slices with Redux Toolkit**

Redux Toolkit introduces `createSlice` to simplify reducer and action creation. Here's an example based on the `AccountSlice`:

```javascript
import { createSlice } from "@reduxjs/toolkit";

const initialState = {
	balance: 0,
	loan: 0,
	loanPurpose: "",
	isLoading: false,
};

const accountSlice = createSlice({
	name: "account",
	initialState,
	reducers: {
		deposit(state, action) {
			state.balance += action.payload;
			state.isLoading = false;
		},
		withdraw(state, action) {
			state.balance -= action.payload;
		},
		requestLoan: {
			prepare(loanAmount, loanPurpose) {
				return {
					payload: {
						loanAmount,
						loanPurpose,
					},
				};
			},
			reducer(state, action) {
				if (state.loan > 0) return;
				state.loan = action.payload.loanAmount;
				state.loanPurpose = action.payload.loanPurpose;
				state.balance += action.payload.loanAmount;
			},
		},
		payLoan(state) {
			state.balance -= state.loan;
			state.loanPurpose = "";
			state.loan = 0;
		},
		convertingCurrency(state) {
			state.isLoading = true;
		},
	},
});

export const { withdraw, requestLoan, payLoan } = accountSlice.actions;
export default accountSlice.reducer;
```

### **Key Features:**

1. **Reducers and Actions**: Reducers and actions are defined in one place.
2. **Prepare Callback**: Used in `requestLoan` to preprocess the payload.

---

## **4. Using Thunks in Redux Toolkit**

### **Modern Way: `createAsyncThunk`**

`createAsyncThunk` simplifies handling async logic like API calls.

```javascript
import { createAsyncThunk, createSlice } from "@reduxjs/toolkit";

export const fetchUser = createAsyncThunk("user/fetchUser", async () => {
	const response = await fetch("/api/user");
	const data = await response.json();
	return data;
});

const userSlice = createSlice({
	name: "user",
	initialState: { data: null, status: "idle", error: null },
	reducers: {},
	extraReducers: (builder) => {
		builder
			.addCase(fetchUser.pending, (state) => {
				state.status = "loading";
			})
			.addCase(fetchUser.fulfilled, (state, action) => {
				state.status = "succeeded";
				state.data = action.payload;
			})
			.addCase(fetchUser.rejected, (state, action) => {
				state.status = "failed";
				state.error = action.error.message;
			});
	},
});

export default userSlice.reducer;
```

### **Manual Thunks**

You can still write thunks manually if needed.

```javascript
export const deposit = (amount, currency) => {
	if (currency === "USD") return { type: "account/deposit", payload: amount };

	return async function (dispatch) {
		dispatch({ type: "account/convertingCurrency" });

		const res = await fetch(
			`https://api.frankfurter.dev/v1/latest?base=${currency}&symbols=USD`
		);
		const data = await res.json();
		const convertedAmount = amount * data.rates.USD;

		dispatch({ type: "account/deposit", payload: convertedAmount });
	};
};
```

---

## **5. The Old Way (Plain Redux)**

### **Setting Up the Reducer**

```javascript
const initialStateAccount = {
	balance: 0,
	loan: 0,
	loanPurpose: "",
	isLoading: false,
};

export default function accountReducer(state = initialStateAccount, action) {
	switch (action.type) {
		case "account/convertingCurrency":
			return { ...state, isLoading: true };

		case "account/deposit":
			return {
				...state,
				balance: state.balance + action.payload,
				isLoading: false,
			};

		case "account/withdraw":
			return {
				...state,
				balance: state.balance - action.payload,
			};

		case "account/requestLoan":
			if (state.loan > 0) return state;
			return {
				...state,
				loan: action.payload.amount,
				loanPurpose: action.payload.loanPurpose,
				balance: state.balance + action.payload.amount,
			};

		case "account/payLoan":
			return {
				...state,
				loan: 0,
				loanPurpose: "",
				balance: state.balance - state.loan,
			};

		default:
			return state;
	}
}
```

### **Creating Actions**

```javascript
export function deposit(amount, currency) {
	if (currency === "USD") return { type: "account/deposit", payload: amount };

	return async function (dispatch) {
		dispatch({ type: "account/convertingCurrency" });

		const res = await fetch(
			`https://api.frankfurter.dev/v1/latest?base=${currency}&symbols=USD`
		);
		const data = await res.json();
		const convertedAmount = amount * data.rates.USD;

		dispatch({ type: "account/deposit", payload: convertedAmount });
	};
}

export function withdraw(amount) {
	return { type: "account/withdraw", payload: amount };
}

export function requestLoan(amount, loanPurpose) {
	return {
		type: "account/requestLoan",
		payload: { amount: amount, loanPurpose: loanPurpose },
	};
}

export function payLoan() {
	return { type: "account/payLoan" };
}
```

---

## **6. Selectors and useSelector**

Selectors are used to access state efficiently. The `useSelector` hook from React-Redux allows components to extract data from the Redux store.

```javascript
import { useSelector } from "react-redux";

// Selector functions
export const getAccountBalance = (state) => state.account.balance;
export const getLoanDetails = (state) => ({
	loan: state.account.loan,
	loanPurpose: state.account.loanPurpose,
});

// Using selectors in a component
function AccountInfo() {
	const balance = useSelector(getAccountBalance);
	const { loan, loanPurpose } = useSelector(getLoanDetails);

	return (
		<div>
			<p>Balance: ${balance}</p>
			<p>Loan: ${loan}</p>
			<p>Loan Purpose: {loanPurpose}</p>
		</div>
	);
}
```

---

## **7. DOs and DON'Ts**

### **DOs:**

1. Use Redux Toolkit for better readability and maintainability.
2. Use `createSlice` and `createAsyncThunk` for modern Redux.
3. Normalize state for complex data.
4. Use selectors to encapsulate state access logic.

### **DON'Ts:**

1. Avoid deeply nested state for better performance.
2. Don't directly mutate state outside reducers.
3. Avoid overusing Redux for local component state.

---

## **8. Performance Considerations**

Optimizing Redux performance is crucial for maintaining a responsive application. Here are some key considerations:

1. **Use Reselect for Memoized Selectors**: Reselect can help prevent unnecessary re-renders by memoizing derived state.

---

<div>

### Created with ❤️ by [Kareem](https://github.com/Kareem-AEz)

</div>
