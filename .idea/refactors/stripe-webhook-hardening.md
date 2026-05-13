# Context
Initially, the `StripeWebhookController` suffered from being incredibly bloated and trying to deal with too many 
responsibilities. Where this should have just been responsible for creating a checkout session and returning the webhook 
response, it was also dealing with order updates, database locking and sending the customer an email upon order 
completion.

## *Roadmap Alignment: [Service Layer Extraction], [Resilience]*

### Legacy code
```
class StripeWebhookApiController extends Controller
{
    public function handle(Request $request, StripeService $stripeService)
    {
        $payload = $request->getContent();
        $sig_header = $request->header('Stripe-Signature');
        $endpoint_secret = config('services.stripe.webhook_secret');

        try {
            if (app()->environment('testing')) {
                $event = json_decode($payload);
            } else {
                $event = Webhook::constructEvent(
                    $payload, $sig_header, $endpoint_secret
                );
            }
        } catch (SignatureVerificationException $e) {
            return response()->json([
                'error' => 'Invalid, signature',
            ], 400);
        }

        if ($event->type === 'checkout.session.completed') {
            $session = $event->data->object;

            $order = Order::where('stripe_session_id', $session->id)->first();

            if (! $order || $order->status === 'paid') {
                return response()->json(['status' => 'ignored'], 200);
            }

            $finalOrder = DB::transaction(function () use ($stripeService, $session, $order) {
                $lockedOrder = Order::with('items.variant')->lockForUpdate()->find($order->id);

                if (! $lockedOrder || $lockedOrder->status === 'paid') {
                    return null;
                }

                $hasStockIssue = false;
                foreach ($lockedOrder->items as $item) {
                    if (! $item->variant || $item->variant->stock < $item->quantity) {
                        $hasStockIssue = true;
                        break;
                    }
                }

                if ($hasStockIssue) {
                    $stripeService->refund($session->id);

                    $lockedOrder->update([
                        'status' => 'cancelled',
                        'cancellation_reason' => 'Out of stock',
                    ]);

                    return $lockedOrder;
                }

                foreach ($lockedOrder->items as $item) {
                    $item->variant->decrement('stock', $item->quantity);
                }

                $shipping = $session->collected_information->shipping_details
                    ?? $session->shipping_details
                    ?? null;
                $address = $shipping?->address ?? null;

                $lockedOrder->update([
                    'customer_name' => $shipping?->name ?? 'Name Missing',
                    'shipping_line1' => $address?->line1 ?? 'No Address',
                    'shipping_line2' => $address?->line2 ?? 'No Address',
                    'shipping_city' => $address?->city ?? 'No City',
                    'shipping_postal_code' => $address?->postal_code ?? 'No Postcode',
                    'customer_phone_number' => $session->customer_details?->phone ?? 'No Phone',
                    'customer_email' => $session->customer_details?->email ?? $lockedOrder->customer_email,
                    'status' => 'paid',
                ]);

                if ($lockedOrder->cart_id) {
                    CartItem::where('cart_id', $lockedOrder->cart_id)->delete();
                }

                return $lockedOrder;
            });

            if ($finalOrder) {
                if ($finalOrder->status === 'paid') {
                    Mail::to($finalOrder->customer_email)->send(new OrderConfirmed($finalOrder));
                }
            }
        }

        return response()->json([
            'message' => 'Order received',
        ], 200);
    }
}
```

### Vulnerabilities with legacy code
1) Controller bloat - the controller was initially handling HTTP requests, Stripe infrastructure and Order domain logic 
all within the same method
2) Firing a synchronous email within an asynchronous operation, delaying the response to Stripe and leading to 
occassional timeouts 
3) Calling `$stripeService->refund($session->id)` inside of the open Database Transaction causing database timeouts if 
Stripe API is slow to return

### Impact
Minimising risk of database timeouts to ensure users are never charged for out of stock items saving client manual 
refund admin and customer support

### Updated
```

```

### Learnings