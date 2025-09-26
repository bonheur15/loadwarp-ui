# LoadWarp - Dynamic DNS Health & Service Discovery

## Overview / Description

LoadWarp is a Cloudflare Worker-based application that provides dynamic DNS updates and server health monitoring. It's designed to help manage service discovery and simple load balancing by maintaining Cloudflare TXT records that list healthy server instances for defined groups. When servers report their health status, LoadWarp automatically updates the corresponding DNS records, allowing clients to discover healthy endpoints.

## Features

*   **User Management:** Create users and issue API keys for secure API access.
*   **Server Grouping:** Organize servers into logical groups (e.g., per application, service, or environment).
*   **Health Reporting:** Servers can report their status (`healthy` or `unhealthy`) via an API endpoint.
*   **Dynamic Cloudflare DNS:**
    *   Automatically creates and updates TXT DNS records in your Cloudflare zone.
    *   TXT records are named `_wrap.{groupSlug}.yourdomain.com`.
    *   Record content is a comma-separated list of healthy server addresses.
    *   Indicates `"no-healthy-servers"` if no instances are healthy in a group.
*   **Configurable Strategies:** Each server group can have a discovery strategy (e.g., `round-robin`, `random`, `failover` - default `round-robin`), although the worker currently only stores this and provides it via an API; client-side implementation of the strategy is implied.
*   **Public DNS Helper:** An unauthenticated endpoint to query the current healthy servers and strategy for a group.
*   **Database Backend:** Utilizes Cloudflare D1 for storing user, group, server, and health check information.
*   **Serverless Architecture:** Built as a Cloudflare Worker for scalability and ease of deployment.

## Architecture

LoadWarp consists of:

1.  **Cloudflare Worker (`src/index.ts`):** The core application logic handling API requests, authentication, database operations, and Cloudflare API interactions.
2.  **Cloudflare D1 Database:** Stores all persistent data including user accounts, server group configurations, server statuses, and health check logs. Drizzle ORM is used for database interactions.
3.  **Cloudflare API:** Used by the worker to manage DNS TXT records within a specified Cloudflare Zone.

**Flow:**
1. A user creates an account and obtains an API key.
2. The user creates a "server group" for their application/service.
3. Servers within that group are configured to periodically report their health status to the LoadWarp API using the API key.
4. When a health status is reported:
    a. The server's status is updated in the D1 database.
    b. LoadWarp fetches all healthy servers for that group.
    c. LoadWarp updates a TXT DNS record in Cloudflare (e.g., `_wrap.my-service-group.yourdomain.com`) with the comma-separated list of healthy server addresses.
5. Client applications can then perform a DNS TXT lookup for `_wrap.my-service-group.yourdomain.com` to get the list of currently healthy servers.

## Setup & Installation

### Prerequisites

*   A Cloudflare account.
*   `wrangler` CLI installed and configured.
*   Node.js and npm/bun.
*   A Cloudflare Zone ID and an API Token with DNS edit permissions for that zone.
*   A Cloudflare D1 database.

### Local Development

1.  **Clone the repository (if applicable).**
2.  **Install dependencies:**
    ```bash
    bun install
    # or
    npm install
    ```
3.  **Configure Environment Variables:**
    *   Rename or copy `wrangler.jsonc.example` to `wrangler.jsonc` (if an example is provided - otherwise, ensure `wrangler.jsonc` is correctly set up).
    *   Update `wrangler.jsonc` with your Cloudflare Account ID.
    *   In the `[vars]` section of `wrangler.jsonc`, set:
        *   `CF_API_TOKEN`: Your Cloudflare API token (Permissions: Zone.DNS:Edit).
        *   `CF_ZONE_ID`: The ID of the Cloudflare zone where DNS records will be managed.
        *   `CF_DOMAIN`: The domain name (e.g., `yourdomain.com`) associated with the `CF_ZONE_ID`.
    *   Configure your D1 database binding in `wrangler.jsonc` under `d1_databases`. Ensure the `database_id` matches your D1 database.
        ```json
        "d1_databases": [
            {
                "binding": "DB", // Must match the binding used in src/index.ts
                "database_name": "your-d1-database-name",
                "database_id": "your-d1-database-id"
            }
        ]
        ```
4.  **Run the development server:**
    ```bash
    bun run dev
    # or
    npm run dev
    ```
    This will start a local server, typically on `http://localhost:8787`.

5.  **Initialize Database Schema:**
    Once the dev server is running, send a request to the `/setup` endpoint to create the necessary tables in your D1 database:
    ```bash
    curl http://localhost:8787/setup
    ```
    You should see a "Tables created successfully!" message.

### Deployment

1.  **Ensure `wrangler.jsonc` is configured** with your production Cloudflare account details, D1 database ID, and environment variables.
2.  **Deploy to Cloudflare Workers:**
    ```bash
    bun run deploy
    # or
    npm run deploy
    ```
3.  **Initialize Database Schema (if not already done for the deployed D1 instance):**
    Access `https://your-worker-url.your-account.workers.dev/setup` in your browser or via curl.

## API Endpoints

Base URL: Your Cloudflare Worker URL (e.g., `https://your-worker.your-account.workers.dev`) or `http://localhost:8787` for local development.

### Authentication

Most endpoints require Bearer Token authentication. Include an `Authorization` header with your API key:
`Authorization: Bearer <YOUR_API_KEY>`

### User Management

#### Create User
*   **Endpoint:** `POST /api/users/create`
*   **Body (JSON):**
    ```json
    {
        "email": "user@example.com"
    }
    ```
*   **Response (201 Created):**
    ```json
    {
        "message": "User created successfully. Store this API key securely!",
        "user": {
            "id": "uuid-string",
            "email": "user@example.com",
            "apiKey": "generated-api-key",
            "passwordHash": "not-implemented", // Password auth is not implemented
            "createdAt": "timestamp"
        }
    }
    ```
    **Note:** Store the `apiKey` securely. It will not be shown again.

### Server Group Management

#### Create Server Group
*   **Endpoint:** `POST /api/groups`
*   **Authentication:** Required.
*   **Body (JSON):**
    ```json
    {
        "name": "My Web Servers",
        "strategy": "round-robin" // Optional, defaults to 'round-robin'. Others: 'random', 'failover'
    }
    ```
*   **Response (201 Created):**
    ```json
    {
        "id": "uuid-string",
        "userId": "user-uuid",
        "name": "My Web Servers",
        "groupSlug": "my-web-servers-xxxxxx", // Auto-generated slug
        "strategy": "round-robin",
        "createdAt": "timestamp"
    }
    ```
    An initial DNS TXT record (`_wrap.{groupSlug}.{CF_DOMAIN}`) will be created with content `"no-healthy-servers"`.

#### List Server Groups
*   **Endpoint:** `GET /api/groups`
*   **Authentication:** Required.
*   **Response (200 OK):**
    ```json
    [
        {
            "id": "uuid-string",
            "userId": "user-uuid",
            "name": "My Web Servers",
            "groupSlug": "my-web-servers-xxxxxx",
            "strategy": "round-robin",
            "createdAt": "timestamp",
            "servers": [
                // Array of server objects associated with this group
                {
                    "id": "server-uuid",
                    "groupId": "group-uuid",
                    "address": "192.168.1.100",
                    "status": "healthy",
                    "lastHeartbeatAt": "timestamp",
                    "createdAt": "timestamp"
                }
            ]
        }
    ]
    ```

### Health Reporting

#### Report Server Health
*   **Endpoint:** `POST /api/report`
*   **Authentication:** Required.
*   **Body (JSON):**
    ```json
    {
        "groupSlug": "my-web-servers-xxxxxx",
        "address": "server1.example.com", // Or IP address
        "status": "healthy" // or "unhealthy"
    }
    ```
*   **Response (200 OK):**
    ```json
    {
        "message": "Status for server1.example.com reported as healthy. DNS record sync initiated."
    }
    ```
    This endpoint will:
    1.  Update or create the server entry in the database with the new status and current timestamp.
    2.  Log the health check.
    3.  Trigger a DNS sync to update the Cloudflare TXT record for the specified `groupSlug` with the latest list of healthy servers.

### Public DNS Information

#### Get Group DNS Information
*   **Endpoint:** `GET /dns/{groupSlug}`
*   **Authentication:** Not required.
*   **Example:** `GET /dns/my-web-servers-xxxxxx`
*   **Response (200 OK):**
    ```json
    {
        "strategy": "round-robin",
        "servers": [
            "server1.example.com",
            "server2.example.com"
        ]
    }
    ```
    If the group is not found, returns a 404. If no servers are healthy, `servers` will be an empty array.
    **Note:** This endpoint provides a direct way to query server lists. The primary intended method of discovery is via DNS TXT record lookup (e.g., `dig TXT _wrap.my-web-servers-xxxxxx.yourdomain.com`).

## How it Works (DNS Updates)

1.  **Group Creation:** When a new server group is created (e.g., `my-app`), a unique slug is generated (e.g., `my-app-a1b2c3`). A DNS TXT record `_wrap.my-app-a1b2c3.{CF_DOMAIN}` is immediately created in your Cloudflare zone. Initially, its content will be `"no-healthy-servers"`.
2.  **Health Reporting:** An agent or process on your servers makes a `POST` request to `/api/report` with the `groupSlug`, its `address` (IP or hostname), and `status` (`healthy` or `unhealthy`).
3.  **Database Update:** The worker updates the server's status in the D1 database.
4.  **DNS Synchronization:**
    *   The worker queries the database for all servers in that `groupSlug` marked as `healthy`.
    *   It constructs a comma-separated string of these healthy server addresses (e.g., `"10.0.0.1,10.0.0.2"`).
    *   If no servers are healthy, the string is `"no-healthy-servers"`.
    *   The worker then uses the Cloudflare API to update the content of the TXT record `_wrap.my-app-a1b2c3.{CF_DOMAIN}`.
    *   If the record doesn't exist (e.g., manual deletion), it attempts to create it.

Client applications can then perform a standard DNS TXT lookup on `_wrap.my-app-a1b2c3.{CF_DOMAIN}`. The result will be the list of healthy servers, which the client can then use according to the group's defined strategy (though strategy implementation is client-side).

## Configuration

Configuration is primarily managed through:

*   **`wrangler.jsonc`:**
    *   `name`: Name of the worker.
    *   `main`: Entry point for the worker (`src/index.ts`).
    *   `compatibility_date`: Cloudflare worker compatibility date.
    *   `vars`:
        *   `CF_API_TOKEN`: Cloudflare API Token (Zone.DNS:Edit permission for the target zone).
        *   `CF_ZONE_ID`: The Zone ID of your domain in Cloudflare.
        *   `CF_DOMAIN`: The domain name (e.g., `example.com`) where TXT records will be created.
    *   `d1_databases`: Configuration for your D1 database binding.
        *   `binding`: Name used to access the database in the worker code (e.g., `DB`).
        *   `database_name`: Your D1 database's name.
        *   `database_id`: Your D1 database's unique ID.

## Database Schema

The application uses the following tables in Cloudflare D1 (managed by Drizzle ORM, schema in `src/db/schema.ts`):

*   **`users`**: Stores user information and API keys.
    *   `id` (TEXT, PK): Unique user ID (UUID).
    *   `email` (TEXT, UNIQUE): User's email address.
    *   `passwordHash` (TEXT): (Currently "not-implemented").
    *   `apiKey` (TEXT, UNIQUE): API key for authentication.
    *   `createdAt` (INTEGER, TIMESTAMP): User creation timestamp.

*   **`server_groups`**: Defines groups of servers.
    *   `id` (TEXT, PK): Unique group ID (UUID).
    *   `userId` (TEXT, FK -> users.id): Owning user.
    *   `name` (TEXT): Human-readable name for the group.
    *   `groupSlug` (TEXT, UNIQUE): URL-friendly unique slug for the group.
    *   `strategy` (TEXT): Load balancing/discovery strategy (`round-robin`, `random`, `failover`).
    *   `createdAt` (INTEGER, TIMESTAMP): Group creation timestamp.

*   **`servers`**: Stores individual server details and their status.
    *   `id` (TEXT, PK): Unique server ID (UUID).
    *   `groupId` (TEXT, FK -> server_groups.id): Group the server belongs to.
    *   `address` (TEXT): IP address or hostname of the server.
    *   `status` (TEXT): Current health status (`healthy`, `unhealthy`, `pending`).
    *   `lastHeartbeatAt` (INTEGER, TIMESTAMP): Timestamp of the last health report.
    *   `createdAt` (INTEGER, TIMESTAMP): Server record creation timestamp.
    *   Unique constraint on (`groupId`, `address`).

*   **`health_checks`**: Logs historical health check data.
    *   `id` (TEXT, PK): Unique health check ID (UUID).
    *   `serverId` (TEXT, FK -> servers.id): Server the check is for.
    *   `status` (TEXT): Status reported (`healthy`, `unhealthy`).
    *   `details` (TEXT): (Optional) Additional details about the health check.
    *   `checkedAt` (INTEGER, TIMESTAMP): Timestamp of the health check.

*   **`traffic_stats`**: (Schema defined, but not actively used in the current `index.ts` logic) Intended for tracking request counts per server/group.

## Contributing

Contributions are welcome! Please feel free to submit pull requests or open issues.
(Further details can be added here if specific contribution guidelines are established).

## License

This project is licensed under the MIT License. (Or specify another if applicable. If no license file exists, this is a placeholder).
