# Context
As a part of the TypeScript migration, the global state manager for the shopping cart (built using Zustand) required a
complete overhaul. The store acts as the bridge between the UI and the backend, managing everything from localised cart
mutations to initiating Stripe checkout. Introducing strict typing was necessary to avoid silent failures and indefinite 
loading states.

## *Roadmap Alignment: [Frontend rigour]*

### Legacy code
```
import { create } from 'zustand';
import api from "../../lib/axios";

export const useCartStore = create((set, get) => ({
    isCartOpen: false,
    openCart: () => set({isCartOpen: true}),
    closeCart: () => set({isCartOpen: false}),

    cartData: null,
    isLoading: false,
    error: null,
    clearError: () => set({error: null}),

    fetchCart: async () => {
        try {
            const response = await api.get('api/cart');
            set({cartData: response.data});
        } catch {
            console.log('No active cart data found')
        }
    },

    addToCart: async (payload) => {
        set({isLoading: true, error: null});

        try {
            const response = await api.post('api/cart/add', payload);

            set({
                cartData: response.data,
                isLoading: false, 
            })

            return true
        } catch (error) {
            let errorMessage = 'An unexpected error occurred';

            if (error.response?.status === 422) {
                errorMessage = error.response.data.message || 'Invalid product selection'
            } else if (error.response?.data?.message) {
                errorMessage = error.response.data.message
            } else {
                console.error(error);
            }

            set({
                error: errorMessage,
                isLoading: false, 
            })

            return false
        }
    },

    checkout: async () => {
        const {cartData} = get();

        if (!cartData || !cartData.session_id) {
            set({error: 'No active session found. Please add items to your cart.'});
            return;
        }

        set({isLoading: true, error: null});
        try {
            const response = await api.post('/api/checkout', {
                cart_session_id: cartData.session_id
            });

            if (response.data && response.data.url) {
                window.location.href = response.data.url;
            } else {
                Error('Invalid response from Stripe.'); 
            }
        } catch (error) {
            console.error('Checkout error: ', error);
            set({
                error: error.response?.data?.message || 'Failed to start checkout',
                isLoading: false
            })
        }
    },

    updateCartItemQuantity: async (variantId, quantity) => {
        if (quantity < 0) return false;

        set({isLoading: true, error: null});

        try {
            const response = await api.patch(`api/cart/items/${variantId}`, {
                quantity,
            });

            set({
                cartData: response.data,
                isLoading: false,
            });

            return true;
        } catch (error) {
            let errorMessage = 'Failed to update cart item';

            if (error.response?.status === 422) {
                errorMessage = error.response.data.message || 'Failed to update cart item';
            } else if (error.response?.data?.message) {
                errorMessage = error.response.data.message
            } else {
                console.error(error);
            }

            set({
                error: errorMessage,
                isLoading: false,
            });

            return false;
        }
    },

    clearCartLocal: () => set({
        cartData: null,
        isCartOpen: false,
        isLoading: false,
        error: null,
    }),

    emptyCart: async () => {
        set({isLoading: true, error: null});

        try {
            await api.delete(`api/cart`)
            set({cartData: null});
        } catch (error) {
            console.log('Failed to empty cart', error);
            set({error: 'Sorry, we could not empty your cart. Please try again'});
        } finally {
            set({isLoading: false, error: null}); 
        }
    }
}))
```

### Vulnerabilities with legacy code
There were three main weaknesses to how the cartStore initially worked which I realised during a codebase audit before 
beginning refactor and TypeScript migration:
1) Lack of API response interfaces -> no enforcement between frontend + backend
2) Instantiating an error object but never doing anything with this meaning application failed silently when reaching 
this code. Leaving users in an infinite loading state
3) Scattered isLoading: false statements within try/catch blocks made codebase brittle with some methods, e.g. `emptyCart`,
actively removing error messages before UI can render these

### Updated
```
import {AddToCartPayload, CartResponse, CheckoutResponse} from "../../types/cart";
import {create} from 'zustand';
import api from "../../lib/axios";
import axios from "axios";

interface CartState {
    cartData: CartResponse | null;
    isLoading: boolean;
    error: string | null;

    isCartOpen: boolean,
    openCart: () => void;
    closeCart: () => void;
    checkout: () => Promise<void>;
    clearCartLocal: () => void;
    emptyCart: () => void;

    addToCart: (payload: AddToCartPayload) => Promise<boolean>;
    updateCartItemQuantity: (variantId: number, quantity: number) => Promise<boolean>;
    fetchCart: () => Promise<void>;
    clearError: () => void;
}

interface ApiResponse {
    message: string;
}

export const useCartStore = create<CartState>((set, get) => ({
    isCartOpen: false,
    openCart: () => set({isCartOpen: true}),
    closeCart: () => set({isCartOpen: false}),

    cartData: null,
    isLoading: false,
    error: null,
    clearError: () => set({error: null}),

    fetchCart: async () => {
        try {
            const response = await api.get<CartResponse>('api/cart');
            set({cartData: response.data});
        } catch {
            console.log('No active cart data found')
        }
    },

    addToCart: async (payload) => {
        set({isLoading: true, error: null});

        try {
            const response = await api.post<CartResponse>('api/cart/add', payload);

            set({
                cartData: response.data,
            })

            return true
        } catch (error) {
            let errorMessage = 'An unexpected error occurred';

            if (axios.isAxiosError(error)) {
                if (error.response?.status === 422) {
                    errorMessage = error.response.data.message || 'Invalid product selection'
                } else if (error.response?.data?.message) {
                    errorMessage = error.response.data.message
                }
            } else {
                console.error(error);
            }

            set({
                error: errorMessage,
            })

            return false
        } finally {
            set({isLoading: false})
        }
    },

    checkout: async () => {
        const {cartData} = get();

        if (!cartData || !cartData.session_id) {
            set({error: 'No active session found. Please add items to your cart.'});
            return;
        }

        set({isLoading: true, error: null});
        try {
            const response = await api.post<CheckoutResponse>('/api/checkout', {
                cart_session_id: cartData.session_id
            });

            if (response.data && response.data.url) {
                window.location.href = response.data.url;
            } else {
                throw new Error('Invalid response from Stripe.');
            }
        } catch (error: unknown) {
            if (axios.isAxiosError(error)) {
                set({
                    error: error.response?.data?.message || 'Failed to start checkout',
                })
            } else {
                console.error('Checkout error: ', error);
            }
        } finally {
            set({isLoading: false});
        }
    },

    updateCartItemQuantity: async (variantId, quantity) => {
        if (quantity < 0) return false;

        set({isLoading: true, error: null});

        try {
            const response = await api.patch<CartResponse>(`api/cart/items/${variantId}`, {
                quantity,
            });

            set({
                cartData: response.data,
            });

            return true;
        } catch (error) {
            let errorMessage = 'Failed to update cart item';

            if (axios.isAxiosError(error)) {
                if (error.response?.status === 422) {
                    errorMessage = error.response.data.message || 'Failed to update cart item';
                } else if (error.response?.data?.message) {
                    errorMessage = error.response.data.message
                }
            } else {
                console.error(error);
            }

            set({
                error: errorMessage,
            });

            return false;
        } finally {
            set({isLoading: false});
        }
    },

    clearCartLocal: () => set({
        cartData: null,
        isCartOpen: false,
        isLoading: false,
        error: null,
    }),

    emptyCart: async () => {
        set({isLoading: true, error: null});

        try {
            await api.delete<ApiResponse>(`api/cart`)
            set({cartData: null});
        } catch (error) {
            console.log('Failed to empty cart', error);
            set({error: 'Sorry, we could not empty your cart. Please try again'});
        } finally {
            set({isLoading: false});
        }
    }
}))
```

### Learnings 
Migrating to TypeScript allowed me to introduce strict typing which is a necessity given the domain logic that this 
store deals with. This code is one step removed from the Stripe webhook which manages orders, deals with live inventory
levels and transaction. Enforcing strict data contracts protects checkout flow from malformed requests and silent 
failures.

Focusing on the typing of errors by using `axios.isAxiosError` as a type guard allowed me to safely narrow the unknown 
error type caught during network failures and guarantees the application can map Laravel's 422 validation messages
to user.

Adopting the `try/catch/finally` block for async state handling ensured that loading UI states always resolve 
deterministically. Centralising this by utilising the `finally` block across all methods also removed some instances of 
conditional error messages being erased before UI can render these to ensure users are aware of all failures.  