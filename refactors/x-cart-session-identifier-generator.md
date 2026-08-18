# Context
`X-Cart-Session`, the identifier for a user's cart, was initially created by using `Math.random()` + timestamp which is 
not truly random and therefore also not unguessable which the cart identifier should be. Realistically, the practical 
impact is low (carts hold no PII and pricing is re-validated server-side at checkout), but shipping a guessable security 
identifier is a weakness worth closing regardless

Opting to use `crypto.randomUUID()` instead gives the identifier a 122 bit, cryptographically-random identifier Also 
included fallback code as `randomUUID()` is only available in a secure context (HTTPS, or localhost during development). 
Should not be an issue but as production choices still undecided, thought better to include the fallback option vs not 
and need to add later whilst going live. The fallback includes `crypto.getRandomValues` which is also secure in the
same way as `crypto.randomUUID`.

// before
`const generateGuestId = () => 'guest_' + Math.random().toString(36).substring(2,15) + Date.now().toString(36);`
// after
`const generateGuestId = () => {
if (crypto?.randomUUID) {
return 'guest_' + crypto.randomUUID();
}
const bytes = crypto.getRandomValues(new Uint8Array(16));
return 'guest_' + Array.from(bytes, b => b.toString(16).padStart(2, '0')).join('');
}`


### Learnings: 
`Math.random()` is not genuinely random. When looking to implement something which can have a real negative effect on a 
user's experience - e.g. their cart not being 100% owned by them and all it needing is somebody to guess their identifier.
Much better to use a UUID.
