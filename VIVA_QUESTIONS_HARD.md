# BookQuest - Hard Difficulty Viva Questions

## 1. Architecture & System Design

### Q1: Scalability Issues
**Question:** If BookQuest suddenly gets 1 million users, what bottlenecks will you face? Design a solution.

**Expected Answer Structure:**
- Database bottlenecks:
  - `populate()` in recommendBooks will cause N+1 queries or load all data at once (memory issue)
  - No indexes on genre.name, author, publisher → full collection scans
  - Recommendation query with `$or` and `$nin` becomes slow
- API bottlenecks:
  - Single-threaded Node.js → need clustering or load balancing
  - JWT verification on every request → consider Redis caching
  - Search endpoint returns all books from NavBar without pagination
- Frontend bottlenecks:
  - Loading all books into memory → need pagination/infinite scroll
  - Large image downloads without compression

**Solution Design:**
- Add database indexes: `db.books.createIndex({ "genre.name": 1, author: 1, publisher: 1 })`
- Implement caching layer (Redis) for recommendations
- Use connection pooling for MongoDB
- Add pagination to book listing endpoints
- Implement CDN for book cover images
- Use message queues (RabbitMQ/Redis) for async recommendation computation
- Horizontal scaling with PM2 clusters

---

### Q2: Authentication Security Vulnerabilities
**Question:** Analyze the authentication flow. What security issues exist in `authController.js` and how would you fix them?

**Expected Answer Structure:**
- Current vulnerabilities:
  - JWT stored in localStorage (XSS vulnerable) → stored in secure httpOnly cookies instead
  - No token refresh mechanism → tokens valid for full 1 hour (risk if stolen)
  - No rate limiting on `/register` and `/login` → brute force attacks possible
  - Password validation enforced on frontend only, not backend
  - No HTTPS enforcement → token can be intercepted in transit
  - User ID exposed in JWT payload → not critical but unnecessary
  - No logout mechanism → token remains valid until expiry
  - No email verification on registration

- Fixes:
  ```javascript
  // Add rate limiting
  const rateLimit = require('express-rate-limit');
  const loginLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 5, // 5 attempts
    message: 'Too many login attempts'
  });
  app.post('/api/users/login', loginLimiter, loginUser);
  
  // Use httpOnly cookies
  res.cookie('token', token, {
    httpOnly: true,
    secure: true, // HTTPS only
    sameSite: 'strict',
    maxAge: 3600000
  });
  
  // Add refresh token mechanism
  const refreshToken = jwt.sign({ id }, REFRESH_SECRET, { expiresIn: '7d' });
  res.cookie('refreshToken', refreshToken, { httpOnly: true, secure: true });
  ```

---

### Q3: Database Schema Design Flaws
**Question:** The `Book` schema has `reviews` array storing user ObjectIds and content. What problems arise with this design at scale? Propose an alternative.

**Expected Answer Structure:**
- Problems:
  - Reviews array grows unbounded → document size limit in MongoDB (16MB)
  - Querying reviews requires loading entire book document
  - No pagination for reviews → inefficient
  - No indexing on review metadata (date, helpful votes)
  - Updating reviews requires re-saving entire book
  - Difficult to query across all reviews (e.g., "most recent reviews")
  
- Alternative design (Separate collection):
  ```javascript
  // Current schema
  Book {
    reviews: [{ user, content, createdAt }]
  }
  
  // Better schema
  Book {
    reviewCount: 5,
    avgRating: 4.2
  }
  
  Review {
    _id: ObjectId,
    bookId: ref(Book),
    userId: ref(User),
    content: String,
    rating: Number,
    createdAt: Date,
    helpfulCount: Number
  }
  
  // Index on bookId for efficient queries
  db.reviews.createIndex({ bookId: 1, createdAt: -1 })
  ```

---

## 2. Recommendation Algorithm Deep Dive

### Q4: Recommendation Algorithm Limitations
**Question:** The `recommendBooks()` function uses content-based filtering. Explain 3 critical limitations and how you'd solve each.

**Expected Answer Structure:**
1. **Cold Start Problem for New Users**
   - Issue: New user has no books → empty preference sets → no recommendations
   - Solution: Recommend popular/trending books or ask user to rate books on signup
   
2. **Filter Bubble / Lack of Diversity**
   - Issue: If user likes only Fantasy by Tolkien, they'll only get Fantasy/Tolkien recommendations forever
   - Solution: Implement diversity algorithm → mix 70% relevant + 30% adjacent genres; use serendipity index
   
3. **No Quality Ranking**
   - Issue: A 5-star book and 2-star book both match equally → returned in arbitrary order
   - Solution: Score recommendations: `score = (genreMatch * 3 + authorMatch * 1) * rating * 0.5`; sort by score

4. **Data Quality Issues**
   - Issue: Missing genres, inconsistent author names ("J. Smith" vs "J Smith"), sparse publisher data
   - Solution: Data cleaning pipeline; normalize on save; make genre required; validate on insert

5. **No Temporal Dynamics**
   - Issue: Recent books not prioritized; user preferences change over time
   - Solution: Weight recent books higher; use exponential decay on old interactions; track preference drift

---

### Q5: Content-Based vs Collaborative Filtering
**Question:** Should BookQuest switch from content-based to collaborative filtering? Analyze pros/cons in the context of this project.

**Expected Answer Structure:**

| Aspect | Content-Based (Current) | Collaborative |
|--------|-------------------------|----------------|
| **Cold Start** | ✅ Works day 1 | ❌ Needs user history |
| **Diversity** | ❌ Limited | ✅ Serendipitous |
| **Sparsity** | ✅ Works with few ratings | ❌ Needs dense ratings matrix |
| **Scalability** | ✅ O(n) | ❌ O(n²) user comparisons |
| **Implementation** | ✅ Simple | ❌ Complex (ALS, SVD) |
| **Data Privacy** | ⚠️ Okay | ❌ User ratings exposed |

**Recommendation:** Implement **Hybrid** approach
- First 3 months: Use content-based (no cold start)
- After user has 5+ ratings: Switch to collaborative filtering
- Always include 10% random/trending for diversity

---

### Q6: Recommendation Quality Metrics
**Question:** How would you evaluate if the recommendation algorithm is actually good? Define metrics and a testing methodology.

**Expected Answer Structure:**

**Offline Metrics:**
- **Precision@k**: Of top 10 recommendations, how many did user actually interact with?
- **Recall@k**: Of all books user could have liked, how many did we recommend?
- **NDCG (Normalized Discounted Cumulative Gain)**: Ranking quality (higher-ranked recommendations matter more)
- **Coverage**: Percentage of catalog recommended to all users (avoid recommending same 100 books)

**Online Metrics (A/B Testing):**
- Click-through rate (CTR) on recommendations
- Conversion rate (user adds recommended book to want-to-read)
- User engagement (time spent, returns)

**Testing Methodology:**
```javascript
// Historical evaluation: Split user data
- Training set (80%): Build user profiles
- Test set (20%): Treat as new interactions, check if recommended

- For each test user:
  - Get recommendations using training data
  - Check if test interactions appear in top-k
  - Calculate precision/recall/NDCG

// A/B Testing in production:
- Control: Current algorithm
- Variant A: New ranking-based algorithm
- Metric: user engagement, CTR
- Run 2 weeks, measure significance (p-value < 0.05)
```

---

## 3. Frontend-Backend Integration

### Q7: NavBar Search Performance Issue
**Question:** In `NavBar.js`, every keystroke fetches ALL books from `/api/books`. At 100k books, this will break. How do you fix this?

**Expected Answer Structure:**
- Current problem:
  ```javascript
  const response = await axios.get("http://localhost:5000/api/books"); // Fetch ALL books
  const filteredSuggestions = allBooks.filter(...); // Filter in frontend
  ```
  - O(n) memory for all books
  - Network bandwidth wasted
  - Latency on first keystroke
  
- Solutions (ranked by effectiveness):
  1. **Server-side search with pagination:**
     ```javascript
     GET /api/books/search?q=harry&limit=10&offset=0
     
     // Backend
     router.get('/search', async (req, res) => {
       const regex = new RegExp(req.query.q, 'i');
       const results = await Book.find({ 
         $or: [{ title: regex }, { author: regex }] 
       })
       .limit(parseInt(req.query.limit))
       .skip(parseInt(req.query.offset));
       res.json(results);
     });
     ```
  
  2. **Add database indexes:**
     ```javascript
     db.books.createIndex({ title: 'text', author: 'text' })
     // Use text search: { $text: { $search: query } }
     ```
  
  3. **Debounce requests in frontend:**
     ```javascript
     const [searchTimeout, setSearchTimeout] = useState(null);
     
     const handleSearch = (value) => {
       clearTimeout(searchTimeout);
       setSearchTimeout(
         setTimeout(() => {
           fetchSuggestions(value); // Fetch after 300ms of no typing
         }, 300)
       );
     };
     ```
  
  4. **Cache results + lazy load:**
     ```javascript
     // Cache in localStorage
     const cached = JSON.parse(localStorage.getItem('booksCache'));
     if (cached && cacheAge < 1hour) return cached;
     ```

---

### Q8: JWT Token Handling Issues
**Question:** The app passes JWT in localStorage and header on each request. What are the risks and better approaches?

**Expected Answer Structure:**
- Current risks:
  - **XSS vulnerability**: If any script injection happens, attacker reads JWT from localStorage
  - **Token theft**: Long-lived token (1 hour) → if stolen, full access
  - **No revocation**: Can't invalidate token before expiry
  - **Token exposure**: Sent on every request, more attack surface

- Better approaches:
  1. **HttpOnly Cookies** (most secure):
     ```javascript
     // Backend: Set token in httpOnly cookie
     res.cookie('authToken', token, {
       httpOnly: true,      // Cannot be accessed by JS
       secure: true,        // Only over HTTPS
       sameSite: 'strict',  // CSRF protection
       maxAge: 15 * 60 * 1000 // 15 minutes
     });
     
     // Frontend: No manual JWT handling needed
     // Browser sends cookie automatically
     ```
  
  2. **Short-lived + Refresh Token**:
     ```javascript
     // Access token: 15 minutes (httpOnly cookie)
     // Refresh token: 7 days (httpOnly cookie)
     
     GET /api/refresh
     -> Backend validates refresh token → issues new access token
     ```
  
  3. **Token Blacklist for Logout**:
     ```javascript
     // On logout, add token to Redis blacklist
     redis.setex(`blacklist_${token}`, 3600, true);
     
     // On each request, check if token is blacklisted
     if (redis.get(`blacklist_${token}`)) throw UnauthorizedError;
     ```

---

## 4. Data Modeling & Relationships

### Q9: N+1 Query Problem
**Question:** The `recommendBooks()` uses `.populate("favourites completed wantToRead")`. Explain the N+1 problem and how to solve it.

**Expected Answer Structure:**
- N+1 Problem:
  ```javascript
  // Query 1: Find user
  const user = await User.findById(userId);
  
  // Queries 2-N: For each populate field, additional queries
  // 1 query to fetch user
  // 1 query to populate favourites
  // 1 query to populate completed  
  // 1 query to populate wantToRead
  // = 4 queries total (1 + 3)
  
  // If you had 100 users → 400 queries! 🔴
  ```

- Solutions:
  1. **Batch populate (current approach, but inefficient for many users):**
     ```javascript
     // Better for single user
     User.findById(userId).populate([
       { path: 'favourites' },
       { path: 'completed' },
       { path: 'wantToRead' }
     ]);
     ```
  
  2. **Aggregation Pipeline (better for scale):**
     ```javascript
     const users = await User.aggregate([
       { $match: { _id: ObjectId(userId) } },
       { $lookup: {
           from: 'books',
           localField: 'favourites',
           foreignField: '_id',
           as: 'favourites'
       }},
       { $lookup: {
           from: 'books',
           localField: 'completed',
           foreignField: '_id',
           as: 'completed'
       }}
     ]);
     ```
  
  3. **Selective population (only fetch needed fields):**
     ```javascript
     User.findById(userId).populate([
       { path: 'favourites', select: 'genre author publisher' },
       { path: 'completed', select: 'genre author publisher' },
       { path: 'wantToRead', select: 'genre author publisher' }
     ]);
     ```

---

### Q10: Book Schema Design Redundancy
**Question:** Notice that `User.js` has `favoriteGenres` field but also has genre info in the favorite books. Why is this redundant? What's the tradeoff?

**Expected Answer Structure:**
- The redundancy:
  ```javascript
  // User.js
  favourites: [{ type: ObjectId, ref: "Book" }],     // References books
  favoriteGenres: [{ type: ObjectId, ref: "Genre" }], // Duplicate genre info
  
  // Could be derived from:
  Book.find({ _id: { $in: user.favourites } })
    .distinct("genre");
  ```

- Why redundancy exists:
  - Faster access: Don't need to populate books to know genres
  - Explicit tracking: User can have favorite genres independent of books
  - Query efficiency: Direct query on `favoriteGenres` faster than through books

- Tradeoffs:
  | Aspect | Redundant | Derived |
  |--------|-----------|---------|
  | Query speed | ✅ Fast | ❌ Slower |
  | Data consistency | ❌ Risk | ✅ Always accurate |
  | Storage | ❌ More | ✅ Less |
  | Update complexity | ❌ Hard | ✅ Easy |
  
- Better solution: **Computed field (MongoDB $lookup on read)**
  ```javascript
  // Instead of storing, compute on read
  User.aggregate([
    { $lookup: {
        from: 'books',
        localField: 'favourites',
        foreignField: '_id',
        as: 'favBooks'
    }},
    { $addFields: {
        favoriteGenres: { $setUnion: ['$favBooks.genre'] }
    }}
  ]);
  ```

---

## 5. Error Handling & Edge Cases

### Q11: Race Condition in Rating Update
**Question:** In `saveRating()`, there's a potential race condition. Identify it and propose a solution.

**Expected Answer Structure:**
- The race condition:
  ```javascript
  // Current code:
  const book = await Book.findById(bookId);           // Read
  const totalRatings = book.reviews.length;
  const currentTotalScore = book.rating * totalRatings;
  const newAverage = (currentTotalScore + rating) / (totalRatings + 1);
  book.rating = newAverage;
  await book.save();                                   // Write
  
  // Race condition:
  // Thread 1: Read book (rating: 4.0)
  // Thread 2: Read book (rating: 4.0) <- Same read!
  // Thread 1: Calculate new avg, save (rating: 4.1)
  // Thread 2: Calculate new avg based on old value, save (rating: 4.1) <- Wrong!
  // Result: One rating lost
  ```

- Solutions:
  1. **Atomic operation (best):**
     ```javascript
     // Use MongoDB atomic update with $inc
     const book = await Book.findByIdAndUpdate(
       bookId,
       {
         $inc: { ratingSum: rating, ratingCount: 1 },
         $set: { rating: /* calculated average */ }
       },
       { new: true }
     );
     ```
  
  2. **Database transaction (strong consistency):**
     ```javascript
     const session = await mongoose.startSession();
     session.startTransaction();
     try {
       const book = await Book.findById(bookId).session(session);
       book.rating = newAverage;
       await book.save({ session });
       await session.commitTransaction();
     } catch (e) {
       await session.abortTransaction();
     }
     ```
  
  3. **Optimistic locking (version field):**
     ```javascript
     Book {
       rating: Number,
       __v: Number (version field)
     }
     
     // Update with version check
     Book.updateOne(
       { _id: bookId, __v: oldVersion },
       { $set: { rating: newRating, __v: oldVersion + 1 } }
     );
     // If __v changed, update fails; retry
     ```

---

### Q12: BookDetails Comment Submission Failure
**Question:** In `BookDetails.js`, `handleSubmitComment()` has async code but doesn't handle the response or error properly. What can go wrong and how to fix it?

**Expected Answer Structure:**
- Current issues:
  ```javascript
  // Current code in BookDetails.js
  const response = await fetch(`http://localhost:5000/api/books/review`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(requestBody),
  });
  
  // ❌ Problems:
  // 1. Response is never checked (no error handling if 500)
  // 2. Comment not cleared after successful submit
  // 3. User gets no feedback if request fails
  // 4. Commented out: if (response.ok) { ... }
  ```

- Issues:
  - User types comment → clicks submit → nothing happens (silent failure)
  - No error message if server returns 400/500
  - Comment stays in textarea → confusing UX
  - No loading state → user clicks multiple times (duplicate comments)

- Fix:
  ```javascript
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [submitError, setSubmitError] = useState(null);
  
  const handleSubmitComment = async () => {
    if (!comment.trim()) {
      setSubmitError("Comment cannot be empty!");
      return;
    }

    setIsSubmitting(true);
    setSubmitError(null);

    try {
      const response = await fetch(`http://localhost:5000/api/books/review`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(requestBody),
      });

      if (!response.ok) {
        const data = await response.json();
        throw new Error(data.message || "Failed to submit comment");
      }

      // Success
      setComment(""); // Clear textarea
      alert("Comment added successfully!");
      
      // Refresh comments (fetch updated list)
      fetchUpdatedComments();
      
    } catch (error) {
      setSubmitError(error.message);
      console.error("Error submitting comment:", error);
    } finally {
      setIsSubmitting(false);
    }
  };
  
  // UI
  {submitError && <div className="alert alert-danger">{submitError}</div>}
  <button disabled={isSubmitting}>
    {isSubmitting ? "Submitting..." : "Submit Comment"}
  </button>
  ```

---

## 6. Advanced Concepts

### Q13: Search Algorithm Complexity
**Question:** The NavBar search filters all books in memory with `.filter()`. Analyze time/space complexity and propose a production-ready solution.

**Expected Answer Structure:**
- Current complexity:
  ```javascript
  const suggestions = allBooks.filter(book =>
    book.title.toLowerCase().includes(value.toLowerCase()) ||
    book.author.toLowerCase().includes(value.toLowerCase())
  );
  
  Time: O(n * m) where n = num books, m = avg book metadata length
  Space: O(k) where k = num matches
  
  At 100k books: 100k operations per keystroke 🔴
  ```

- Production solution (Elasticsearch):
  ```bash
  # Install Elasticsearch
  npm install elasticsearch
  ```
  
  ```javascript
  // Backend: Index books
  const { Client } = require('@elastic/elasticsearch');
  const client = new Client({ node: 'http://localhost:9200' });
  
  app.post('/api/books/index', async (req, res) => {
    const books = await Book.find();
    for (const book of books) {
      await client.index({
        index: 'books',
        id: book._id.toString(),
        body: {
          title: book.title,
          author: book.author,
          genre: book.genre.name,
          description: book.description
        }
      });
    }
  });
  
  // Search endpoint (fast)
  app.get('/api/books/search', async (req, res) => {
    const results = await client.search({
      index: 'books',
      body: {
        query: {
          multi_match: {
            query: req.query.q,
            fields: ['title^2', 'author', 'genre']
          }
        },
        size: 10
      }
    });
    res.json(results.body.hits.hits.map(h => h._source));
  });
  ```
  
  Benefits:
  - ✅ Sub-millisecond search (indexes, not full scan)
  - ✅ Relevance scoring (title matches higher than description)
  - ✅ Fuzzy matching (typos handled)
  - ✅ Scaling to millions of books

---

### Q14: Authentication Flow Vulnerability
**Question:** A user logs in, gets a JWT token. Then they change their password. Why can they still access protected routes using the old token? How would you fix this?

**Expected Answer Structure:**
- The vulnerability:
  ```javascript
  // Login → Token issued
  jwt.sign({ id: user._id }, SECRET, { expiresIn: '1h' });
  
  // User changes password
  // But old token still valid for 59 minutes ← VULNERABILITY
  // Attacker with old token can still access account
  ```

- Solutions:
  1. **Token blacklist on password change:**
     ```javascript
     // On password change
     router.post('/change-password', protect, async (req, res) => {
       const { oldPassword, newPassword } = req.body;
       const user = await User.findById(req.user.id);
       
       // Validate old password
       const isMatch = await bcrypt.compare(oldPassword, user.password);
       if (!isMatch) return res.status(400).json({ error: 'Wrong password' });
       
       // Update password
       user.password = await bcrypt.hash(newPassword, 10);
       await user.save();
       
       // Blacklist all existing tokens
       const token = req.headers.authorization.split(' ')[1];
       redis.setex(`blacklist_${user._id}`, 3600, true);
       
       res.json({ message: 'Password changed, please login again' });
     });
     
     // Middleware: Check blacklist
     const protect = (req, res, next) => {
       const token = req.headers.authorization.split(' ')[1];
       const decoded = jwt.verify(token, SECRET);
       
       if (redis.get(`blacklist_${decoded.id}`)) {
         return res.status(401).json({ error: 'Token invalidated' });
       }
       next();
     };
     ```
  
  2. **Add token version/generation**:
     ```javascript
     User {
       password: String,
       tokenGeneration: Number (incremented on password change)
     }
     
     // Token stores generation
     jwt.sign({ id: user._id, gen: user.tokenGeneration }, SECRET);
     
     // Verify: check if token generation matches current
     const decoded = jwt.verify(token, SECRET);
     const user = await User.findById(decoded.id);
     if (decoded.gen !== user.tokenGeneration) {
       throw new UnauthorizedError('Token invalidated');
     }
     ```
  
  3. **Short-lived tokens + revocation**:
     ```javascript
     // Access token: 15 minutes
     // Refresh token: 7 days (stored in DB)
     
     // On password change: delete all refresh tokens
     User.updateOne({ _id: id }, { refreshTokens: [] });
     ```

---

### Q15: Pagination and Cursor Design
**Question:** How would you implement pagination for the book listing endpoints to handle 1 million books efficiently?

**Expected Answer Structure:**
- Current problem:
  ```javascript
  // Current: GET /api/books returns ALL books
  // At 1M books: 50MB JSON response, crashes
  ```

- Solutions:
  1. **Offset-based pagination (simple but inefficient at scale):**
     ```javascript
     GET /api/books?page=5&limit=20
     
     // Problem: skip(offset) still scans all skipped docs
     // Page 1000: skips 20000 documents 🔴
     
     router.get('/', async (req, res) => {
       const page = req.query.page || 1;
       const limit = 20;
       const skip = (page - 1) * limit;
       
       const books = await Book.find()
         .skip(skip)
         .limit(limit)
         .sort({ createdAt: -1 });
       
       res.json({
         data: books,
         page,
         totalPages: Math.ceil(await Book.countDocuments() / limit)
       });
     });
     ```
  
  2. **Cursor-based pagination (better for scale):**
     ```javascript
     GET /api/books?cursor=5f7d8b9c&limit=20
     
     // Cursor = last book's ID from previous page
     // Only fetches 20 docs, no scanning ✅
     
     router.get('/', async (req, res) => {
       const limit = 20;
       const cursor = req.query.cursor;
       
       let query = {};
       if (cursor) {
         query._id = { $gt: ObjectId(cursor) };
       }
       
       const books = await Book.find(query)
         .limit(limit + 1)
         .sort({ _id: 1 });
       
       const hasNextPage = books.length > limit;
       const data = books.slice(0, limit);
       const nextCursor = data.length > 0 ? data[data.length - 1]._id : null;
       
       res.json({
         data,
         nextCursor,
         hasNextPage
       });
     });
     ```
  
  3. **Keyset pagination (production-ready):**
     ```javascript
     GET /api/books?after=2024-01-01T00:00:00Z,5f7d8b9c&limit=20
     
     // Cursor includes timestamp + ID for deterministic ordering
     router.get('/', async (req, res) => {
       const [createdAt, id] = (req.query.after || '').split(',');
       
       let query = {};
       if (createdAt && id) {
         query.$or = [
           { createdAt: { $lt: new Date(createdAt) } },
           { 
             createdAt: new Date(createdAt),
             _id: { $lt: ObjectId(id) }
           }
         ];
       }
       
       const books = await Book.find(query)
         .sort({ createdAt: -1, _id: -1 })
         .limit(21);
       
       const hasMore = books.length > 20;
       const data = books.slice(0, 20);
       
       const cursor = data.length > 0 
         ? `${data[data.length - 1].createdAt},${data[data.length - 1]._id}`
         : null;
       
       res.json({ data, nextCursor: cursor, hasMore });
     });
     ```

---

## Summary: Key Concepts to Master

1. **Scalability**: Indexes, caching, pagination, connection pooling
2. **Security**: JWT handling, rate limiting, password hashing, XSS/CSRF protection
3. **Algorithms**: Content-based filtering alternatives, complexity analysis
4. **Database**: N+1 queries, normalization, atomic operations, transactions
5. **Frontend-Backend**: API design, error handling, loading states
6. **Concurrency**: Race conditions, optimistic locking, transactions

---

**Good luck with your viva! 🚀**
