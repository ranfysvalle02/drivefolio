# mdb-overdrive

Portfolio powered by Google Drive.

# Putting MongoDB Atlas in Overdrive: Direct-to-Database Architecture with m-stash

---

## 1. The Frontend Developer's Dream: Direct to Cloud Database

Modern frontend developers expect velocity. When prototyping a web application, portfolio, or internal dashboard, the ideal workflow is straightforward: build a responsive single-page application and connect it directly to a scalable cloud database without wrestling with hundreds of lines of boilerplate backend routing.

MongoDB Atlas remains the premier operational document database on the market—unmatched in schema flexibility, document modeling, and managed cloud scaling.

However, when developers build Jamstack sites, client-side Vue/React apps, or mobile tools, they often hit a common dilemma: **how do you securely query MongoDB Atlas directly from the client without building, containerizing, and maintaining an entire Node.js/Express server just to route requests?**

For many teams, spinning up heavy custom backend services just to authenticate users and filter document access creates unnecessary friction. 

To bridge this exact gap, we built **[m-stash](https://github.com/ranfysvalle02/m-stash)**—a lightweight, single-binary Go security proxy that acts as an instant on-ramp to MongoDB Atlas. And to prove how seamlessly it works in production, we built **[mdb-overdrive](https://mdb-overdrive.vercel.app)**.

---

## 2. The Proof of Concept: mdb-overdrive

**[mdb-overdrive](https://mdb-overdrive.vercel.app)** (source code hosted at [github.com/ranfysvalle02/mdb-overdrive](https://github.com/ranfysvalle02/mdb-overdrive), formerly known during initial prototyping as *drivefolio*) is a high-octane media showcase designed for creators, motion designers, developers, and agencies who need to present rich visual assets—4K reels, design portfolios, client PDFs, and presentation decks—with zero hosting overhead.

### How it Works:
- **Zero File Ingestion:** Files never touch our application servers. Media streams directly through Google's official `/preview` sandbox iframe, offloading player UI rendering and heavy bandwidth onto Google's global CDN.
- **Dynamic Namespaces:** Anyone can claim a `#namespace` (e.g., `mdb-overdrive.vercel.app/#motion-reel`), organize slides, and share an interactive presentation in seconds.
- **Native MongoDB Atlas Backend:** The entire frontend is an ultra-fast, single-file Vue 3 application that writes and reads presentation metadata straight from MongoDB Atlas via `m-stash`.

---

## 3. Enter m-stash: The Friction-Free Bridge to MongoDB Atlas

`m-stash` was designed to solve a single problem with maximum elegance: gives client-side applications secure, controlled access to MongoDB Atlas with **zero custom backend code**.

Written in Go, `m-stash` sits as a tiny reverse security proxy between your client frontend and your MongoDB Atlas cluster. It inspects incoming JWT bearer tokens and automatically injects owner-level access rules before requests ever hit your database.

```
+--------------------+        JWT Bearer + Payload        +--------------------+
|                    | ---------------------------------> |                    |
|   mdb-overdrive    |                                    |      m-stash       |
|  (Vue 3 Frontend)  | <--------------------------------- |   (Go Container)   |
|                    |        Safe Query Results          |                    |
+--------------------+                                    +--------------------+
                                                                     |
                                                          Native Mongo Driver (BSON)
                                                                     v
                                                          +--------------------+
                                                          |   MongoDB Atlas    |
                                                          |     (Cluster)      |
                                                          +--------------------+
```

### Edge Security, Rate Limiting & Connection Pooling
Whenever developers hear "direct client querying," immediate concerns arise around connection exhaustion, open CORS exposure, and DDoS attacks. `m-stash` handles these natively at the Go runtime layer:
- **Production Connection Pooling:** Uses the official Go MongoDB driver (`go.mongodb.org/mongo-driver/v2`) with bounded connection pools, preventing spikes in browser traffic from saturating database connections.
- **Automatic CORS & Preflight Handling:** Responds to `OPTIONS` preflights with strict, configurable header allowlists and origins.
- **Built-in Rate Limiting:** Enforces token-bucket rate limiting per IP and per authenticated user UID before queries are evaluated, shielding your MongoDB Atlas cluster from scraping and credential stuffing.

### Declarative Access Control (DLAC)

Client-supplied filters should never be trusted blindly. `m-stash` enforces Declarative Document-Level Access Control (DLAC) directly in the Go proxy:

```json
{
  "stashes": {
    "read": {
      "$or": [
        { "ownerId": "$auth.uid" },
        { "isPublic": true }
      ]
    },
    "write": {
      "ownerId": "$auth.uid"
    }
  }
}
```

When an authenticated client requests records matching `{"query": {"namespace": "creative"}}`, the Go proxy extracts `$auth.uid` from their validated token and recursively rewrites the filter using MongoDB's native `$and` clauses:

```json
{
  "$and": [
    { "namespace": "creative" },
    {
      "$or": [
        { "ownerId": "usr_9421a8b" },
        { "isPublic": true }
      ]
    }
  ]
}
```

Developers get the security of custom application servers without writing a single line of backend glue code.

---

## 4. Flexible Auth Architecture: Native or Third-Party

A major strength of `m-stash` is architectural flexibility regarding identity:

1. **Zero-Config Native Auth:** Out of the box, `m-stash` includes built-in `/v1/auth/signup` and `/v1/auth/login` endpoints with bcrypt password hashing that issue signed HS256/RS256 JWTs.
2. **Third-Party Provider Support:** You are not locked into the native auth engine. If your stack already relies on **Clerk, Auth0, Supabase, or Firebase**, simply configure `m-stash` with your external public key or shared secret. The proxy validates the incoming Bearer token, extracts the subject identifier (`sub` or `uid`), and immediately maps it to `$auth.uid` across all DLAC security rules.

---

## 5. Why Go Delivers Maximum Cloud Efficiency

When building a high-throughput proxy layer in front of a database, runtime efficiency translates directly into lower hosting bills and instant responsiveness:

| Metric | m-stash (Go Engine) | Traditional Node/TS Gateway |
| :--- | :--- | :--- |
| **Container Size** | **~15MB** (scratch base) | **150MB – 350MB+** |
| **Idle Memory Footprint** | **~15MB – 20MB** | **120MB – 200MB+** |
| **Boot Time** | Sub-millisecond instant start | JIT initialization & module load |
| **Concurrency** | Goroutines handle high-concurrency I/O | Single-threaded event loop |
| **Dependencies** | Self-contained static binary | Nested package tree (`node_modules`) |

Because `m-stash` operates with a tiny memory footprint, developers can run it reliably on the smallest cloud container tiers (such as Fly.io or Render free tiers) right alongside their free MongoDB Atlas M0 cluster, creating a robust production-capable stack for \$0.

---

## 6. Get Started in 30 Seconds

### Step 1: Spin up m-stash with Your Atlas Cluster
Run the official container pointed to your MongoDB Atlas connection string:

```bash
docker run -d \
  -p 4000:4000 \
  -e MONGO_URI="mongodb+srv://user:pass@cluster.mongodb.net/app_db" \
  -e JWT_SECRET="your-production-secret" \
  oblivio/m-stash
```

### Step 2: Register a User (or Use Your Existing JWT)
```bash
# Using m-stash built-in auth:
curl -X POST "http://localhost:4000/v1/auth/signup" \
  --header "Content-Type: application/json" \
  --data '{"email":"dev@example.com","password":"secure-unique-password"}'
```

### Step 3: Insert and Query Documents
```bash
# Insert a record (ownerId is automatically injected by m-stash)
curl -X POST "http://localhost:4000/v1/db/stashes/insertOne" \
  --header "Authorization: Bearer <TOKEN>" \
  --header "Content-Type: application/json" \
  --data '{"payload":{"namespace":"showcase","title":"2026 Creative Reel","isPublic":true}}'

# Query protected records
curl -X POST "http://localhost:4000/v1/db/stashes/find" \
  --header "Authorization: Bearer <TOKEN>" \
  --header "Content-Type: application/json" \
  --data '{"query":{"namespace":"showcase"},"limit":10}'
```

---

## 7. Open Source & Getting Involved

Both projects are open-source and ready for exploration:

- **Live Showcase Demo:** [mdb-overdrive.vercel.app](https://mdb-overdrive.vercel.app)
- **Frontend Source:** [github.com/ranfysvalle02/mdb-overdrive](https://github.com/ranfysvalle02/mdb-overdrive) *(migrated from the prototype drivefolio repository)*
- **m-stash Go Gateway:** [github.com/ranfysvalle02/m-stash](https://github.com/ranfysvalle02/m-stash)
- **Official Docker Hub Container:** [hub.docker.com/r/oblivio/m-stash](https://hub.docker.com/r/oblivio/m-stash)

If you want to bring client-side speed to your MongoDB Atlas projects without maintaining heavy custom backends, pull the container and take it for a spin!
