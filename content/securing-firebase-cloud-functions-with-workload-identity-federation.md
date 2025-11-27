---
title: Securing Firebase Cloud Functions with Workload Identity Federation
date: 2025-11-27
authors:
  - name: robcost
    link: https://github.com/robcost
tags:
  - GCP
  - IAM
  - Firebase
  - Cloud Functions
excludeSearch: false
---
If you're building a mobile app with Firebase and want to protect Cloud Functions that call paid APIs—AI services, databases, or external integrations—you need to understand a critical security gap: **Firebase Auth validation happens inside your function, after it's already running and billing you.**

This means every request to your function is billable, even if the token is invalid. Malicious actors can spam your endpoints with garbage tokens and drain your budget. Your Firebase Auth validation will reject them, sure—but not before GCP bills you for each function invocation.

The standard advice? "Just use Firebase Auth and App Check." But here's the uncomfortable truth:

| Auth Method | When Validation Happens | Billing Implication |
|-------------|------------------------|---------------------|
| Firebase Auth | Inside function (after invocation) | **You pay for invalid requests** |
| App Check | Inside function (after invocation) | **You pay for invalid requests** |
| GCP IAM | Before function invocation | **No billing for rejected requests** |

Enter **Workload Identity Federation (WIF)**—Google Cloud's mechanism for letting external identities assume GCP service accounts. Combined with what I call the **dual-token pattern**, you get IAM protection *in front of* your functions, while preserving Firebase's seamless mobile auth experience.

This isn't a theoretical exercise. I spent weeks implementing this for a production application with 26+ Cloud Functions across three global regions. Here's everything I learned, including the billing attack vector that motivated it all.

---

## The Problem: Billing Attacks on Firebase Functions

Firebase Authentication is phenomenal for mobile apps. Users sign in with Google, Apple, or email—and you get a JWT token you can validate server-side. The problem? **Token validation happens after your function starts running.**

### The Billing Attack Vector

Consider a typical Firebase callable function:

```typescript
// Mobile app - calling a Cloud Function
const generateToken = httpsCallable(functions, 'generateAIToken');
const result = await generateToken({ prompt: 'Hello' });
```

```typescript
// Cloud Function - using Firebase Auth
export const generateAIToken = onCall(async (request) => {
  // This check happens INSIDE the function
  // The function has ALREADY been invoked (billable!)
  if (!request.auth) {
    throw new HttpsError('unauthenticated', 'Must be authenticated');
  }

  // Even if we reject here, we've already paid for invocation
  const aiResult = await callExpensiveAPI();
  return aiResult;
});
```

Here's the attack scenario:
1. Attacker discovers your function endpoint (easy—it's in your app bundle)
2. Attacker writes a script to spam requests with invalid/no tokens
3. Each request **invokes your function** (billable event)
4. Your validation rejects them... but you still pay

**At $0.40 per million invocations plus compute time, an attacker running 1,000 requests/second could cost you $34/day in invocations alone—without making a single legitimate request.**

### Why App Check Doesn't Solve This

You might think Firebase App Check fixes this. It doesn't. App Check attestation is also validated **inside** your function:

```typescript
export const protectedFunction = onCall({
  enforceAppCheck: true, // Validated INSIDE the function
}, async (request) => {
  // Function has already invoked before App Check validation
});
```

While App Check makes it harder to forge valid requests, determined attackers can:
- Extract attestation data from your app
- Replay valid attestations (they have a lifetime)
- Use device farms with valid attestations

The fundamental issue remains: **validation happens after invocation.**

### The IAM Solution

GCP IAM authentication works differently. When a Cloud Run service (which backs Cloud Functions v2) has IAM restrictions, authentication happens at the **load balancer level**:

```
Invalid/missing token → Load balancer rejects → No function invocation → No billing
Valid token → Load balancer accepts → Function invokes → Billing starts
```

But here's the catch: Firebase tokens aren't IAM tokens. Your mobile app has Firebase credentials, not GCP service account keys.

**This is the gap Workload Identity Federation bridges.**

---

## Understanding the Token Exchange Flow

The core idea is simple: exchange your Firebase Auth token for a GCP identity token. But the implementation? That's where it gets interesting.

Here's the flow:

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Mobile App    │     │  Token Exchange  │     │  IAM-Protected  │
│  (Firebase Auth)│────▶│ Cloud Function   │────▶│ Cloud Function  │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │   GCP STS API    │
                    │ (Token Exchange) │
                    └──────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ IAM Credentials  │
                    │ (ID Token Gen)   │
                    └──────────────────┘
```

1. **Mobile app** calls `exchangeToken` with Firebase Auth
2. **Token Exchange function** validates Firebase token, exchanges it via STS
3. **STS returns** a federated access token
4. **Function generates** an OIDC ID token for the target service
5. **Mobile app receives** both tokens: GCP (for IAM) + Firebase (for user identity)

That last point is crucial—it's the **dual-token pattern**. You need both tokens because IAM authenticates the request, but the target function still needs to know *who* is making the request.

---

## Step 1: Setting Up Workload Identity Federation

WIF requires three components: a **Workload Identity Pool**, an **OIDC Provider** configured for Firebase, and a **Service Account** that authenticated users can impersonate.

Here's the Pulumi configuration (you can also use Terraform or gcloud CLI):

```typescript
// Create Workload Identity Pool
const pool = new gcp.iam.WorkloadIdentityPool('firebase-mobile-pool', {
  workloadIdentityPoolId: 'firebase-mobile-pool',
  displayName: 'Firebase Mobile Pool',
  description: 'Workload Identity Pool for Firebase Auth federation',
  disabled: false,
});

// Configure Firebase as OIDC Provider
const provider = new gcp.iam.WorkloadIdentityPoolProvider('firebase-provider', {
  workloadIdentityPoolId: pool.workloadIdentityPoolId,
  workloadIdentityPoolProviderId: 'firebase-provider',
  displayName: 'Firebase OIDC Provider',

  oidc: {
    // Firebase's OIDC discovery endpoint
    issuerUri: `https://securetoken.google.com/${projectId}`,
    allowedAudiences: [projectId],
  },

  // Map Firebase claims to GCP attributes
  attributeMapping: {
    'google.subject': 'assertion.sub',        // Firebase UID
    'attribute.email': 'assertion.email',     // User email
    'attribute.email_verified': 'string(assertion.email_verified)',
  },
});
```

The `issuerUri` is key—Firebase exposes OIDC-compatible discovery at `https://securetoken.google.com/{PROJECT_ID}`. This tells GCP how to validate Firebase tokens.

### Creating the Service Account

```typescript
// Service account the mobile app will impersonate
const mobileAppInvoker = new gcp.serviceaccount.Account('mobile-app-invoker', {
  accountId: 'mobile-app-invoker',
  displayName: 'Mobile App Cloud Functions Invoker',
});

// Allow authenticated Firebase users to impersonate this SA
new gcp.serviceaccount.IAMBinding('mobile-app-wif-binding', {
  serviceAccountId: mobileAppInvoker.name,
  role: 'roles/iam.workloadIdentityUser',
  members: [
    // Any authenticated user from our WIF pool
    `principalSet://iam.googleapis.com/projects/${projectNumber}/locations/global/workloadIdentityPools/${pool.workloadIdentityPoolId}/*`,
  ],
});
```

**Gotcha #1**: The `principalSet://` URN format is different from standard IAM members. Miss this and your token exchange will fail with cryptic permission errors.

---

## Step 2: The Token Exchange Function

This is the bootstrap function—it's the only public function in the system. It validates Firebase Auth (via `onCall`), then performs the WIF exchange:

```typescript
export const exchangeToken = onCall<TokenExchangeRequest, Promise<TokenExchangeResponse>>(
  {
    region: 'us-central1',
    memory: '256MiB',
    maxInstances: 10, // Prevent abuse
    // Note: This function is NOT IAM-protected (it's the bootstrap!)
  },
  async (request) => {
    // Step 1: Validate Firebase Auth
    if (!request.auth) {
      throw new HttpsError('unauthenticated', 'User must be authenticated');
    }

    const uid = request.auth.uid;
    const { deviceId, targetServiceUrl } = request.data;

    // Step 2: Rate limiting (VERY important!)
    await checkRateLimit(deviceId, targetServiceUrl);

    // Step 3: Extract Firebase ID token from request headers
    const authHeader = request.rawRequest.headers.authorization;
    const firebaseToken = authHeader?.split('Bearer ')[1];

    // Step 4: Exchange Firebase token for federated credential
    const federatedToken = await exchangeViaSTS(firebaseToken);

    // Step 5: Generate service account ID token for target service
    const gcpToken = await generateIdToken(federatedToken, targetServiceUrl);

    // Return BOTH tokens (the dual-token pattern)
    return {
      gcpToken,
      firebaseToken,
      expiresIn: 3600,
      expiresAt: Date.now() + 3600000,
    };
  }
);
```

### The STS Exchange

Google's Security Token Service (STS) is where the magic happens:

```typescript
async function exchangeViaSTS(firebaseToken: string): Promise<string> {
  const response = await fetch('https://sts.googleapis.com/v1/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      grantType: 'urn:ietf:params:oauth:grant-type:token-exchange',
      audience: WIF_AUDIENCE, // Your pool/provider path
      scope: 'https://www.googleapis.com/auth/cloud-platform',
      requestedTokenType: 'urn:ietf:params:oauth:token-type:access_token',
      subjectToken: firebaseToken,
      subjectTokenType: 'urn:ietf:params:oauth:token-type:jwt',
    }),
  });

  const data = await response.json();
  return data.access_token;
}
```

The `audience` is critical—it's the full path to your WIF provider:
```
//iam.googleapis.com/projects/{PROJECT_NUMBER}/locations/global/workloadIdentityPools/{POOL_ID}/providers/{PROVIDER_ID}
```

### Generating the ID Token

Cloud Run (which backs Cloud Functions v2) requires an OIDC ID token, not an access token. Use the IAM Credentials API:

```typescript
async function generateIdToken(
  federatedToken: string,
  targetServiceUrl: string
): Promise<string> {
  const response = await fetch(
    `https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/${SERVICE_ACCOUNT}:generateIdToken`,
    {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${federatedToken}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        audience: targetServiceUrl, // Must match the Cloud Run service URL
        includeEmail: true,
      }),
    }
  );

  const data = await response.json();
  return data.token;
}
```

**Gotcha #2**: The `audience` for the ID token must **exactly match** the Cloud Run service URL. Not your custom domain. Not a function name. The actual `https://functionname-xyz-uc.a.run.app` URL.

---

## Step 3: IAM-Protected Cloud Functions

Now your target functions can use IAM authentication:

```typescript
export const generateAIToken = onRequest(
  {
    region: 'us-central1',
    invoker: ['mobile-app-invoker@my-project.iam.gserviceaccount.com'],
    // The invoker list restricts who can call this function
  },
  async (request, response) => {
    // ========================================
    // Dual-Token Pattern: Get user identity
    // ========================================
    const firebaseToken = request.headers['x-firebase-auth'] as string;
    if (!firebaseToken) {
      response.status(401).json({ error: 'Firebase token required' });
      return;
    }

    // Verify and extract user identity
    const decodedToken = await getAuth().verifyIdToken(firebaseToken);
    const uid = decodedToken.uid;

    // Now we know:
    // 1. Request is IAM-authenticated (from the service account)
    // 2. User identity is uid (from the Firebase token)

    // ... your function logic
  }
);
```

The dual-token pattern sends:
- `Authorization: Bearer {gcpToken}` — IAM authentication
- `X-Firebase-Auth: {firebaseToken}` — User identity

**Gotcha #3**: Firebase Functions v2's `invoker` parameter is **just metadata**. It doesn't actually create IAM policies. You need to manually grant `roles/run.invoker` to your service account via Pulumi/Terraform/gcloud.

```typescript
new gcp.cloudrunv2.ServiceIamMember('function-invoker', {
  project: projectId,
  location: 'us-central1',
  name: 'generateaitoken', // Lowercase function name
  role: 'roles/run.invoker',
  member: `serviceAccount:mobile-app-invoker@${projectId}.iam.gserviceaccount.com`,
});
```

---

## The Callable vs HTTP Trade-Off

Moving from `onCall` (Firebase callable functions) to `onRequest` (HTTP functions) for IAM protection comes with real trade-offs. Here's what you gain, what you lose, and how to handle it.

### What You Lose

**1. Automatic Firebase Auth context**

With `onCall`, Firebase SDK automatically attaches the user's auth token and deserializes it for you:

```typescript
// onCall - Firebase handles auth automatically
export const myFunction = onCall(async (request) => {
  const uid = request.auth?.uid;  // Just works
  const email = request.auth?.token.email;
});
```

With `onRequest`, you're on your own:

```typescript
// onRequest - Manual auth handling
export const myFunction = onRequest(async (request, response) => {
  const firebaseToken = request.headers['x-firebase-auth'] as string;
  if (!firebaseToken) {
    response.status(401).json({ error: 'Missing token' });
    return;
  }
  const decoded = await getAuth().verifyIdToken(firebaseToken);
  const uid = decoded.uid;
});
```

**2. Automatic request/response serialization**

Callable functions automatically parse JSON bodies and serialize responses:

```typescript
// onCall - Automatic serialization
export const myFunction = onCall(async (request) => {
  const { name, age } = request.data;  // Already parsed
  return { success: true, user: { name, age } };  // Auto-serialized
});
```

HTTP functions require manual handling:

```typescript
// onRequest - Manual serialization
export const myFunction = onRequest(async (request, response) => {
  const body = request.body;  // May need JSON.parse() depending on middleware
  // ... do work
  response.status(200).json({ success: true });  // Manual response
});
```

**3. Client-side SDK convenience**

```typescript
// Callable - Clean SDK usage
const result = await httpsCallable(functions, 'myFunction')({ name: 'Alice' });
console.log(result.data);

// HTTP - Manual fetch
const response = await fetch(url, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${gcpToken}`,
    'X-Firebase-Auth': firebaseToken,
  },
  body: JSON.stringify({ name: 'Alice' }),
});
const data = await response.json();
```

### What You Gain

**1. IAM-level rejection before invocation (the whole point)**

Invalid requests never instantiate your function. No billing for attacks.

**2. Fine-grained access control**

IAM policies let you restrict which service accounts, users, or groups can invoke each function—independently managed from your application code.

**3. GCP audit logging**

Every function invocation appears in Cloud Audit Logs with the authenticated identity, enabling compliance and forensic capabilities.

**4. Consistent security model**

Your functions integrate with GCP's broader security ecosystem—VPC Service Controls, Organization Policies, etc.

### The Hybrid Approach

You don't have to choose all-or-nothing. I use a hybrid:

| Function Type | Auth Method | Why |
|---------------|-------------|-----|
| Token exchange (`exchangeToken`) | `onCall` + Firebase Auth | Bootstrap function—must be callable without IAM tokens |
| AI/expensive functions | `onRequest` + IAM | Billing protection critical |
| Admin functions | `onRequest` + IAM | Restricted to admin service account |
| Low-cost utilities | `onCall` + Firebase Auth | Convenience outweighs billing risk |

The key insight: **use IAM protection for functions where abuse would hurt**, not everywhere. A function that returns static config? Maybe not worth the complexity. A function that calls a $0.01/request AI API? Absolutely.

### Helper Functions to Smooth the Transition

To reduce boilerplate in IAM-protected functions, I created helpers:

```typescript
/**
 * Extract and verify Firebase user from dual-token request
 */
async function getAuthenticatedUser(request: Request): Promise<DecodedIdToken | null> {
  const firebaseToken = request.headers['x-firebase-auth'] as string;
  if (!firebaseToken) return null;

  try {
    return await getAuth().verifyIdToken(firebaseToken);
  } catch {
    return null;
  }
}

/**
 * Standard error response helper
 */
function sendError(response: Response, status: number, message: string) {
  response.status(status).json({ success: false, error: message });
}

/**
 * Wrapper for IAM-protected function handler
 */
function withDualTokenAuth(
  handler: (request: Request, response: Response, user: DecodedIdToken) => Promise<void>
) {
  return async (request: Request, response: Response) => {
    // Handle CORS preflight
    if (request.method === 'OPTIONS') {
      response.status(204).send('');
      return;
    }

    const user = await getAuthenticatedUser(request);
    if (!user) {
      sendError(response, 401, 'Firebase authentication required');
      return;
    }

    await handler(request, response, user);
  };
}
```

Usage becomes cleaner:

```typescript
export const generateAIToken = onRequest(
  { region: 'us-central1', invoker: [SERVICE_ACCOUNT] },
  withDualTokenAuth(async (request, response, user) => {
    // user is guaranteed to be authenticated
    const result = await doExpensiveAIOperation(user.uid);
    response.json({ success: true, result });
  })
);
```

---

## Step 4: Client-Side Implementation

On the mobile side, you need to:
1. Call `exchangeToken` to get both tokens
2. Cache tokens to avoid rate limiting
3. Make requests with both headers

```typescript
class TokenService {
  private cachedTokens: Map<string, CachedTokens> = new Map();

  async getTokensForService(targetServiceUrl: string): Promise<CachedTokens> {
    // Check cache first
    const cached = this.cachedTokens.get(targetServiceUrl);
    if (cached && cached.expiresAt - Date.now() > 300000) {
      return cached; // Use cached if >5min until expiry
    }

    // Exchange tokens
    const exchangeTokenFn = httpsCallable(functions, 'exchangeToken');
    const result = await exchangeTokenFn({
      deviceId: await getDeviceId(),
      targetServiceUrl
    });

    const tokens = result.data;
    this.cachedTokens.set(targetServiceUrl, tokens);

    // Persist to storage for app restart survival
    await AsyncStorage.setItem(
      `@tokens/${targetServiceUrl}`,
      JSON.stringify(tokens)
    );

    return tokens;
  }

  async callProtectedFunction(url: string, body: any): Promise<any> {
    const tokens = await this.getTokensForService(url);

    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${tokens.gcpToken}`,
        'X-Firebase-Auth': tokens.firebaseToken,
      },
      body: JSON.stringify(body),
    });

    return response.json();
  }
}
```

**Gotcha #4**: GCP tokens expire after 1 hour. With a 5-minute buffer for clock skew and network latency, you need to refresh tokens before the 55-minute mark. Cache tokens per service URL since each has a different audience.

---

## The Bugs That Almost Broke Me

### Bug #1: Firebase Functions v2 CORS + Large Bodies

Here's a fun one. Firebase Functions v2 has a convenient `cors: true` option:

```typescript
export const myFunction = onRequest({
  cors: true,  // Seems helpful, right?
  // ...
});
```

Except it completely breaks when your request body exceeds ~16KB. The function receives an empty body with error `Request body is missing data`.

**Root cause**: The CORS middleware reads the body to check content, but doesn't properly restore it for the handler.

**Solution**: Handle CORS manually:

```typescript
export const myFunction = onRequest({
  // NO cors: true
}, async (request, response) => {
  // Manual CORS handling
  response.set('Access-Control-Allow-Origin', '*');
  response.set('Access-Control-Allow-Methods', 'GET, POST, OPTIONS');
  response.set('Access-Control-Allow-Headers', 'Content-Type, Authorization, X-Firebase-Auth');

  if (request.method === 'OPTIONS') {
    response.status(204).send('');
    return;
  }

  // Now request.body works correctly
});
```

### Bug #2: Function Name Case Sensitivity

When granting IAM permissions via Pulumi/Terraform, you need the Cloud Run service name—which is the **lowercase** version of your function name:

```typescript
// Function export name
export const generateEphemeralTokenNA = onRequest(...);

// Cloud Run service name (for IAM)
name: 'generateephemeraltokenna'  // All lowercase!
```

I lost two hours to this. The error message? `Resource 'generateEphemeralTokenNA' not found`. Thanks, GCP.

### Bug #3: Changing Function Configuration Requires Delete/Recreate

If you remove `cors: true` from an existing function, Firebase will error with `Changing from a callable function to an HTTPS function is not allowed`.

The solution is to delete the function first:
```bash
firebase functions:delete generateEphemeralTokenNA --region us-central1 --force
firebase deploy --only functions:generateEphemeralTokenNA
```

Then re-add IAM permissions since they're tied to the Cloud Run service instance.

---

## Rate Limiting: Your Defense Against Token Farming

Without rate limiting, an attacker could call your token exchange function millions of times, generating millions of IAM tokens. Even if each token can only call your functions, that's still a vector for abuse.

I implemented per-device, per-service rate limiting:

```typescript
// Rate limit key includes both device and target service
const rateLimitKey = `${deviceId}_${serviceName}`;

// Dev: 5 min, Prod: 50 min (tokens last 1 hour)
const MIN_EXCHANGE_INTERVAL = isDev ? 5 * 60 * 1000 : 50 * 60 * 1000;

const lastExchange = await getLastExchange(rateLimitKey);
if (Date.now() - lastExchange < MIN_EXCHANGE_INTERVAL) {
  throw new HttpsError(
    'resource-exhausted',
    `Rate limited. Try again in ${waitTimeMinutes} minutes.`
  );
}
```

The per-service key is important: users might need tokens for multiple functions (AI generation, image analysis, etc.), and each has a different target audience.

---

## Production Considerations

### Multi-Region Deployment

With WIF, you can deploy IAM-protected functions to multiple regions while sharing a single identity pool. The token exchange function stays in one region, but the target functions can be anywhere:

```typescript
// Single token exchange in us-central1
export const exchangeToken = onCall({ region: 'us-central1' }, ...);

// Regional protected functions
export const generateTokenNA = onRequest({ region: 'us-central1' }, ...);
export const generateTokenEU = onRequest({ region: 'europe-west1' }, ...);
export const generateTokenAPAC = onRequest({ region: 'asia-southeast1' }, ...);
```

Each regional function needs its own IAM binding, but they all use the same service account.

### Token Persistence

Tokens should survive app restarts to avoid hammering the exchange function:

```typescript
// On app start
async loadPersistedTokens() {
  const keys = await AsyncStorage.getAllKeys();
  const tokenKeys = keys.filter(k => k.startsWith('@tokens/'));

  for (const key of tokenKeys) {
    const data = await AsyncStorage.getItem(key);
    const tokens = JSON.parse(data);

    // Only load if still valid
    if (tokens.expiresAt - Date.now() > 300000) {
      this.cachedTokens.set(tokens.targetServiceUrl, tokens);
    } else {
      await AsyncStorage.removeItem(key);
    }
  }
}
```

### Monitoring and Observability

Log everything:
- Token exchange attempts (successful and failed)
- Rate limit hits
- IAM validation failures
- Token expiration patterns

This data is invaluable for detecting abuse and debugging auth issues.

---

## Summary

Workload Identity Federation + the dual-token pattern gives you:

1. **Billing attack protection** — Invalid requests rejected *before* function invocation means no billing for attacks
2. **True IAM protection** for Cloud Functions without exposing service account keys
3. **User identity preservation** through the Firebase token header
4. **Defense in depth** with rate limiting, IAM policies, and Firebase Auth validation
5. **Flexibility** to support mobile apps, web apps, and admin portals with the same infrastructure

The implementation is non-trivial—you're essentially bridging two authentication systems—but the security posture is dramatically better than Firebase Auth alone.

For any application where:
- Functions call paid APIs (AI services, external integrations)
- You're concerned about abuse or billing attacks
- You need audit trails that tie into GCP's IAM system
- You want to sleep at night knowing attackers can't drain your budget

This is the right approach. The one-time complexity of setting up WIF pays for itself the first time someone tries to spam your endpoints.

---

## Resources

- [GCP Workload Identity Federation Documentation](https://cloud.google.com/iam/docs/workload-identity-federation)
- [Google STS API Reference](https://cloud.google.com/iam/docs/reference/sts/rest)
- [IAM Credentials API - generateIdToken](https://cloud.google.com/iam/docs/reference/credentials/rest/v1/projects.serviceAccounts/generateIdToken)
- [Firebase Auth REST API](https://firebase.google.com/docs/reference/rest/auth)

**Note**: Firebase exposes OIDC discovery at `https://securetoken.google.com/{YOUR_PROJECT_ID}/.well-known/openid-configuration` — replace `{YOUR_PROJECT_ID}` with your Firebase project ID to verify the endpoint.
