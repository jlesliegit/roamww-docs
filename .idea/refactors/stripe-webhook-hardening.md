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

[//]: # (Controller)

    public function handle(Request $request, StripeService $stripeService)
    {
        $handlers = [
            'checkout.session.completed' => ProcessCheckoutSession::class,
        ];

        $payload = $request->getContent();
        $sig_header = $request->header('Stripe-Signature');
        $event = $stripeService->signatureVerification($payload, $sig_header);

        if (! $event) {
            return response()->json([
                'error' => 'Invalid signature',
            ], 400);
        }

        if (array_key_exists($event->type, $handlers)) {
            $handlerClass = $handlers[$event->type];
            app($handlerClass)
                ->handle($event);
        }

        return response()->json([
            'message' => 'Order received',
        ], 200);
    }
}

[//]: # (Helper class)
class ProcessCheckoutSession implements WebhookHandlerInterface
{
    public function __construct(
        private OrderService $orderService,
        private StripeService $stripeService
    ) {}

    public function handle(Event $event)
    {
        $session = $event->data->object;
        $finalOrder = $this->orderService->fulfilOrder($session);
        if ($finalOrder === null) {
            return null;
        }
        if ($finalOrder->status === 'cancelled') {
            $this->stripeService->refund($session->id);
        } elseif ($finalOrder->status === 'paid') {
            Mail::to($finalOrder->customer_email)->queue(new OrderConfirmed($finalOrder));
        }
    }
}

[//]: # (OrderService)

 public function fulfilOrder(Session $session): ?Order
    {
        $order = Order::where('stripe_session_id', $session->id)->first();

        if (! $order || $order->status === 'paid') {
            return null;
        } else {
            return DB::transaction(function () use ($session, $order) {
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
        }
    }
```

### Learnings
Implementing the `WebhookHandlerInterface` allows the system to accept new webhooks events without needing to directly 
re-write logic directly within the controller. The controller can now simply focus on the HTTP requests and leave the
responses and domain logic to the specific parts of the app. 

A large part of this refactor also focused on increasing the efficiency of the methods associated with the third-party 
Stripe webhook. This was done by moving all external API calls *outside* of the `DB::Transaction` closure to prevent slow
third-party responses impacting transaction times and keeping database rows locked for longer than necessary. 
Synchronous `(->send())` operation was also switched for asynch background queueing `(->queue())` as slow SMTP dispatch 
could have caused webhook timeouts. 

HTTP requests are now fully decoupled from the domain logic, allowing the controller to simply deal with these requests 
and return a response. `OrderService` now manages its own Eloquent queries, pessimistic locking and inventory checks vs 
the webhook controller.

Guard clauses implemented across `OrderService` domain logic to handle negative states and exit operations at earliest 
point of failure.

Leveraging Laravel `app()` helper allows the application to dynamically resolve and inject dependencies (like `OrderService`
or `StripeService`) directly into the handler class, again keeping the controller 'blind' to service requirements and 
focusing just on the HTTP request / responses.