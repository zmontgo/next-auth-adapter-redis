# NextAuth.js Redis Adapter
This is an adapter for NextAuth that allows it to interface with Redis. This is primarily a package for users looking to use NextAuth.js' reliability and security at scale. Because of the inherent complexity of managing data at scale, operations in this package have a higher chance of failing. We make **zero** assumptions about how you want to handle failed operations.

This package is designed to be as unopinionated as possible, and to allow you to build your own solution on top of it. Our provided backend (while highly configurable) is more opinionated, but we can assure you that it has the correct opinions. Please note that this adapter is incapable of being used without a backend, as the backend is responsible for setting the data in the cache, which is how this adapter retrieves data.

### Use Cases
- You are planning a project that uses Redis as a database or cache.
- You have built an application using NextAuth.js, and scale has become an issue. You need an interum solution while you migrate to a more scalable solution.
- In many cases, this adapter will be enough as as permanent solution. Alongside a holistic scaling strategy, this adapter can be used to scale NextAuth.js to millions of users.

### Features
Much of the actual functionality is handled by the backend package. We provide a performant and highly configurable backend built in Rust, but this package is backend-agnostic and you may use any backend you wish, so long as it conforms to the API. You may even create your own.
- Persistent database management offloaded to the backend behind a [Redis Queue](https://redis.com/glossary/event-queue/).
- Fine-grained queueing system allows for monitoring and provisioning to be done on a per-queue basis.
- Redis-based cache allows for its O(1) lookup time to be used to its full potential.
- Read-through and write-through cache writing ensures that data is always up-to-date.
- Ensures that all operations are [idempotent](https://en.wikipedia.org/wiki/Idempotence#Computer_science_meaning) and, in conjuction with a well-built backend, [ACID-compliant](https://en.wikipedia.org/wiki/ACID). By default, the adapter itself can only guarantee idempotence, and supports atomicity, consistency, and isolation. Durability is not enabled by default. (See [Persistence](#persistence) for more information.)
- Out-of-the-box support for Redis Sentinel and Redis Cluster ([See Below](#scaling)).
- Type definitions for native TypeScript support.

# Persistence
This package uses a Redis Queue system to communicate with a backend, which handles communication with a persistent database. By default, no data is stored persistently (breaking the 'D' in ACID), and the Redis Adapter is used as a cache. This is useful for development, or for applications that do not require persistent storage. Individual tables may be made persistent by setting the corresponding `persistent` flag to `true` when initializing this adapter. How-to's, sensible defaults, and examples are established in the [Configuration](#configuration) section.

Here is a list of all tasks that are enqueued by this adapter, and their corresponding queue names. We will imagine that the Redis enqueue command is `ENQUEUE queue_name data`. All data is stringified JSON. For any get operations, the frontend expects the backend to respond to these commands by setting the data in the key-value store. Idempotency keys are generated as described in the [Idempotency](#idempotency) section. The expected data formats for users, sessions, verification requests, and accounts come from NextAuth, and are documented in (types.ts)[./types.ts].
- `ENQUEUE CREATE_USER { ...user, idempotency_key }`
- `ENQUEUE UPDATE_USER { ...user, idempotency_key }`
- `ENQUEUE DELETE_USER { user_id, idempotency_key }`
- `ENQUEUE CREATE_SESSION { ...session, idempotency_key }`
- `ENQUEUE UPDATE_SESSION { ...session, idempotency_key }`
- `ENQUEUE DELETE_SESSION { session_id, idempotency_key }`
- `ENQUEUE CREATE_VERIFICATION_REQUEST { ...verification_request, idempotency_key }`
- `ENQUEUE USE_VERIFICATION_REQUEST { ...verification_request, idempotency_key }`
- `ENQUEUE LINK_ACCOUNT { ...account, idempotency_key }`
- `ENQUEUE UNLINK_ACCOUNT { ...account, idempotency_key }`
- `ENQUEUE GET_USER { user_id }`
- `ENQUEUE GET_USER_BY_EMAIL { email }`
- `ENQUEUE GET_USER_BY_ACCOUNT { providerAccountId, provider }`
- `ENQUEUE GET_SESSION_AND_USER { session_id }`

It is worth noting that persistence may be provided by Redis itself. Keep all queues disabled, and enable some form of persistence in your Redis configuration to ensure that data is not lost in the event of a crash. This is not a substitute for a proper backup strategy.

### Cacheing
This package keeps track of recently-used data in Redis. This allows for faster access to data that is used often. For instance, when a user logs in, we cache their session token. This allows us to quickly verify that the session token is valid without needing to hit the backend. This cache is invalidated when the data is updated, and is updated when the data is retrieved. This allows for a consistent view of the data, and ensures that the cache is always up-to-date. All caches are enabled by default, but may be disabled by setting the corresponding `cache` flag to `false` when initializing this adapter. Additionally, each cache has its own TTL, set in milliseconds. All are set to `0`, which means the cache will never expire. This is useful for development, but should be changed in production. How-to's, sensible defaults, and examples are established in the [Configuration](#configuration) section.

**Danger: As noted above, by default, no caches have a TTL. This means that session tokens and verification tokens will never expire. This is useful for development, but should be changed in production unless you have a good reason not to.**

Caches are key-value stores in Redis. The following key prefixes are defined:
- `USER_DATA`: This is the key prefix for user data. The key is the `user_id`.
- `EMAIL_INDEX`: This is the key prefix for email index data. The key is the `email`.
- `ACCOUNT_INDEX`: This is the key prefix for account index data. The key is `provider:providerAccountId`
- `SESSION_DATA`: This is the key prefix for session data. The key is the `session_id`.
- `VERIFICATION_REQUEST_DATA`: This is the key prefix for verification request data. The key is the `verification_request_id`.
- `ACCOUNT_DATA`: This is the key prefix for account data. The key is the `account_id`.

Here are an example of the key-value pair that is set in Redis when a user is created:
- `USER_DATA:1234 { ...user }`
- `EMAIL_INDEX:johndoe@example.com 1234`
- `ACCOUNT_INDEX:google:1234567890 1234`

Similarly, here is an example of the key-value pair that is set in Redis when a session is created:
- `SESSION_DATA:1234 { ...session }`

Verification requests...
- `VERIFICATION_REQUEST_DATA:1234 { ...verification_request }`

Accounts...
- `ACCOUNT_DATA:1234 { ...account }`


### Idempotency
This package ensures that every exposed operation is idempotent, except read operations. Idempotency keys are generated by creating a string like the following: `user_id:operation_name:stringified_data`. This string is then hashed using [MurmurHash3](https://www.npmjs.com/package/murmurhash3) to allow for predictable length and character set. Keeping track of idempotency keys, deduplication, and atomicity are the responsibility of the backend.

### Configuration
Configuration is done by passing an object to the `createAdapter` function. Here is an example of the configuration's type definition:
```typescript
type AdapterConfig = {
  // Redis client. The RedisClient type is exported from this package.
  redis: RedisClient;
  // Granular control over which persistence methods are enabled. Defaults to false for all.
  persistence: {
    user: boolean;
    account: boolean;
    session: boolean;
    verificationRequest: boolean;
  };
  // Granular control over which caches are enabled. Defaults to true for all. This is the recommended configuration.
  caches: {
    user: boolean;
    account: boolean;
    session: boolean;
    verificationRequest: boolean;
  };
  // Granular control over cache TTLs, in seconds. Defaults to 0 for all.
  cacheTTLs: {
    user: number;
    account: number;
    session: number;
    verificationRequest: number;
  };
  // Queue name. Defaults to 'authenticator'.
  queueName: string;
  // Cache prefix. Defaults to 'authenticator'.
  cachePrefix: string;
  // Cache key separator. Defaults to ':'.
  cacheKeySeparator: string;
  // Timeouts, in milliseconds. Defaults to 10000 for all.
  timeouts: {
    // Timeout for enqueueing a job.
    enqueue: number;
    // Timeout for dequeueing a job.
    dequeue: number;
    // Timeout for processing a job.
    process: number;
  };
  // Maximum number of retries. Defaults to 3.
  maxRetries: number;
  // Maximum number of concurrent jobs. Defaults to 1.
  maxConcurrency: number;
  // Maximum number of jobs that can be processed in a single tick. Defaults to 1.
  maxJobsPerTick: number;
  // Whether or not to log debug messages. Defaults to false.
  debug: boolean;
};
```

**Suggested Persistence**
- `user`: true
- `account`: true
- `session`: false
- `verificationRequest`: false

**Additional Suggestions**
It is suggested that you keep all caches enabled. TTLs should be set on a per-project basis, but sensible defaults are as follows. These are in seconds. The verification request number is the only one that is derived from the NextAuth.js documentation.
- `user`: 3600
- `account`: 3600
- `session`: 86400
- `verificationRequest`: 86400


### Errors and Logging
Any errors that are thrown by this package are of the type `AuthenticatorError`. This is a subclass of `Error`, and has the following type definition:
```typescript
type AuthenticatorError = {
  // Error message.
  message: string;
  // Error code.
  code: string;
  // Error stack.
  stack?: string;
  // Error data.
  data?: any;
};
```

This package ships with a `debug` flag. When set to `true`, debug messages will be logged to the console. These messages are useful for debugging, but are not necessary for normal operation. They are disabled by default.

This package does not make consideration for error monitoring or reporting, nor does it make any assumptions about how you would like to handle errors. It is up to you to decide how to handle errors, and whether or not to report them.

### Clients
This package is designed to be somewhat client-agnostic. We expose a `RedisClient` type that is exported from this package. This type is used to define the function signature of the object you pass to the `redis` parameter. This allows you to use any Redis client you would like, as long as it conforms to the `RedisClient` type.

Both `ioredis` and `redis` are supported out of the box. Because we directly call the methods of the Redis client, any client that does not conform to the `RedisClient` type that you would like to use will need to be wrapped in a class that conforms to the `RedisClient` type. This is not difficult, and is left as an exercise to the reader.

My personal recommendation is to use `ioredis`, as it is a more modern implementation of a Redis client, and has a more robust feature set. For instance, it allows for Redis clustering and sentinel support, which is outside the scope of this package.

### Contribution
I am not the best developer in the world. If you find a bug, or would like to contribute, please feel free to open an issue or a pull request. I will do my best to respond in a timely manner.

### Attribution
I am the best developer in the world. While you don't need to attribute me, it would be nice. If you are using this as a permanent solution for a large project, please consider letting me know! I would love to see what you are building.

Feel free to fuel my ego at [ego@zachmontgomery.com](mailto:ego@zachmontgomery.com).

Alternatively, you can keep me eating (thereby keeping me alive and able to maintain this package) by [buying me a coffee](https://www.buymeacoffee.com/zmontgo).