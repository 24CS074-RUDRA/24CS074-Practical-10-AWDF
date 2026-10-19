# Practical 10 — Asynchronous Task Notifications

A full-stack Taskflow app demonstrating asynchronous background notifications with Node.js's built-in `EventEmitter`. Task creation returns its HTTP response immediately; the notification listener logs the task title, timestamp, and assigned user two seconds later.

## Learning objective

Decouple task-related side effects from Express request handling and observe the difference between responding to an API request and finishing background notification work.

## Technologies

- Node.js 18+ and the built-in `events` module (no event-library dependency)
- Express.js and Mongoose
- MongoDB
- JWT authentication and bcrypt password hashing
- `node-cache` caching carried over from Practical 9
- Postman or the Postman extension for Visual Studio Code

## Clone and install

```powershell
git clone https://github.com/24CS074-RUDRA/24CS074-Practical-10-AWDF.git
Set-Location .\24CS074-Practical-10-AWDF
npm install
npm --prefix frontend install
```

## Configure MongoDB and JWT

Create a local `.env` from the safe template:

```powershell
Copy-Item .env.example .env
```

Set your own MongoDB URI and generate a private JWT secret. Never commit `.env` or share its credentials.

```env
PORT=5000
MONGO_URI=mongodb://127.0.0.1:27017/taskmanager
JWT_SECRET=replace-this-with-a-long-random-secret
CACHE_ENABLED=true
```

Generate a JWT secret locally:

```powershell
node -p "require('crypto').randomBytes(48).toString('hex')"
```

For MongoDB Atlas, replace the local `MONGO_URI` with your own Atlas URI and make sure the database user's access and network allow-list are configured.

## Build, test, and run

```powershell
npm --prefix frontend run build
npm test
npm run dev
```

The API and built frontend are served at [http://localhost:5000](http://localhost:5000). The server connects to MongoDB before it begins listening.

## Event-driven request flow

1. `POST /api/tasks` validates the request and saves the task in MongoDB.
2. The handler invalidates the authenticated user's task-list cache.
3. Express sends the `201 Created` response and logs the API response timestamp.
4. The route emits `task-created` with the task title, task ID, and assigned user's ID.
5. The listener schedules the notification with a two-second `setTimeout` and returns immediately.
6. After the delay, the listener logs the notification and its completion timestamp. Listener failures are reported through the registered `error` event handler.

`EventEmitter.emit()` invokes its listeners synchronously. The listener itself stays non-blocking because it schedules the slow notification with a timer rather than doing the delayed work before returning. The API response is sent before the notification event is emitted, so notification completion cannot delay the response.

The successful `DELETE /api/tasks/:id` route similarly emits `task-deleted`, whose listener logs the deleted task on the next event-loop turn.

### Event modules

- `events.js` — shared native `EventEmitter` instance.
- `listeners.js` — asynchronous `task-created` and `task-deleted` notification listeners plus notification error logging.
- `routes/tasks.js` — emits events only after the associated MongoDB operation succeeds and the HTTP response is sent.
- `server.js` — registers listeners when the application starts.

## Verify with Postman

1. Start the application and confirm the terminal reports MongoDB connected and the server listening on port `5000`.
2. Register at `POST http://localhost:5000/api/auth/register` with JSON:
   ```json
   {
     "name": "Practical User",
     "email": "your-email@example.com",
     "password": "a-password-of-at-least-6-characters"
   }
   ```
3. Log in at `POST http://localhost:5000/api/auth/login` using that email and password. Copy the returned JWT into Postman's **Bearer Token** authorization for protected requests. Never share or include the token in screenshots.
4. Send `POST http://localhost:5000/api/tasks` with `Content-Type: application/json`, Bearer authentication, and this body:
   ```json
   {
     "title": "Verify asynchronous notifications",
     "description": "The API should respond before its notification completes.",
     "status": "pending",
     "priority": "high"
   }
   ```
5. Confirm the response is `201 Created` and contains the saved task.
6. Inspect the server terminal. First observe:
   ```text
   [API] Task creation response sent at <timestamp> for task "Verify asynchronous notifications"
   ```
   About two seconds later, observe:
   ```text
   [Notification] Task "Verify asynchronous notifications" created at <timestamp> for user <user-id>
   ```
7. Verify the API returned `201` before the later notification timestamp. This demonstrates that delayed notification work did not hold up the HTTP response.
8. To demonstrate the supplementary delete event, send `DELETE http://localhost:5000/api/tasks/<task-id>` with Bearer authentication and observe the separate `[Notification] ... deleted at ...` message in the server terminal.

Capture one screenshot showing the successful Postman `201` response and another showing the corresponding API and delayed notification log lines. Hide Bearer tokens, passwords, and database credentials.

## Automated verification

Run the existing cache, authentication, and task API suite together with the Practical 10 event checks:

```powershell
npm test
```

The suite checks that task creation returns before the delayed notification, that the notification logs its timestamp and assigned user, and that a successful deletion emits its own notification.

## Practical 9 cache behavior

Caching remains user-scoped and enabled by default with a 60-second TTL. Successful task writes continue to invalidate the affected entries. Task-list and single-task GET responses expose `X-Cache: HIT` or `MISS`; use `x-skip-cache: true` or `CACHE_ENABLED=false` when measuring an uncached baseline. Cache counters are available from the authenticated `GET /api/tasks/cache-stats` endpoint.

In-memory EventEmitter listeners are appropriate for this single-process lab. A process crash loses queued timers, and multiple application instances do not share events. A durable queue or shared broker is more suitable for production workloads that require retries, persistence, or horizontal scaling.

## Project layout

```text
.
├── events.js
├── listeners.js
├── middleware/
├── models/
├── routes/
├── utils/
├── frontend/
├── server.js
└── test_practical10.js
```
